## Hybrid Cache(Experimental)

HybridCache feature enables Theine to extend the DRAM cache to NVM. With HybridCache, Theine can seamlessly move Items stored in cache across DRAM and NVM as they are accessed. Using HybridCache, you can shrink your DRAM footprint of the cache and replace it with NVM like Flash. This can also enable you to achieve large cache capacities for the same or relatively lower power and dollar cost.

#### Design
Hybrid Cache is inspired by CacheLib's HybridCache. See [introduction](https://cachelib.org/docs/Cache_Library_User_Guides/HybridCache) and [architecture](https://cachelib.org/docs/Cache_Library_Architecture_Guide/hybrid_cache) from CacheLib's guide.

When you use HybridCache, items allocated in the cache can live on NVM or DRAM based on how they are accessed. Irrespective of where they are, **when you access them, you always get them to be in DRAM**.

Items start their lifetime on DRAM. As an item becomes cold it gets evicted from DRAM when the cache is full. Theine spills it to a cache on the NVM device. Upon subsequent access through `Get()`, if the item is not in DRAM, theine looks it up in the HybridCache and if found, moves it to DRAM. When the HybridCache gets filled up, subsequent insertions into the HybridCache from DRAM will throw away colder items from HybridCache.

Same as CacheLib, Theine hybrid cache also has **BigHash** and **Block Cache**, it's highly recommended to read the CacheLib architecture design before using hybrid cache, here is a simple introduction of these 2 engines(just copy from CacheLib):

-   **BigHash**  is effectively a giant fixed-bucket hash map on the device. To read or write, the entire bucket is read (in case of write, updated and written back). Bloom filter used to reduce number of IO. When bucket is full, items evicted in FIFO manner. You don't pay any RAM price here (except Bloom filter, which is 2GB for 1TB BigHash, tunable).
-   **Block Cache**, on the other hand, divides device into equally sized regions (16MB, tunable) and fills a region with items of same size class, or, in case of log-mode fills regions sequentially with items of different size. Sometimes we call log-mode “stack alloc”. BC stores compact index in memory: key hash to offset. We do not store full key in memory and if collision happens (super rare), old item will look like evicted. In your calculations, use 12 bytes overhead per item to estimate RAM usage. For example, if your average item size is 4KB and cache size is 500GB you'll need around 1.4GB of memory.

#### Using Hybrid Cache

To use HybridCache, you need to create a nvm cache with NvmBuilder. NewNvmBuilder require 2 params, first is cache file name, second is cache size in bytes. Theine will use direct I/O to read/write file.

```go
nvm, err := theine.NewNvmBuilder[int, int]("cache", 150<<20).[settings...].Build()
```

Then enable hybrid mode in your Theine builder.
```go
client, err := theine.NewBuilder[int, int](100).Hybrid(nvm).Build()
```

#### NVM Builder Settings

All settings are optional, unless marked as "Required".

* **[Common]** `BlockSize` default 4096

    Device block size in bytes (minimum IO granularity).
* **[Common]** `KeySerializer` default JsonSerializer

    KeySerializer is used to marshal/unmarshal between your key type and bytes.
    ```go
    type Serializer[T any] interface {
	    Marshal(v T) ([]byte, error)
	    Unmarshal(raw []byte, v *T) error
    }
    ```
* **[Common]** `ValueSerializer` default JsonSerializer

    ValueSerializer is used to marshal/unmarshal between your value type and bytes. Same interface as KeySerializer.
* **[Common]** `ErrorHandler` default do nothing

    Theine evicts entries to Nvm asynchronously, so errors will be handled by this error handler.
* **[BlockCache]** `RegionSize` default 16 << 20 (16 MB)

    Region size in bytes.
* **[BlockCache]** `CleanRegionSize` default 3

    How many regions do we reserve for future writes. Set this to be equivalent to your per-second write rate. It should ensure your writes will not have to retry to wait for a region reclamation to finish.
* **[BigHash]** `BucketSize` defalut 4 << 10 (4 KB)

    Bucket size in bytes.
* **[BigHash]** `BigHashPct` default 10

    Percentage of space to reserve for BigHash. Set the percentage > 0 to enable BigHash. The remaining part is for BlockCache. The value has to be in the range of [0, 100]. Set to 100 will disable block cache.
* **[BigHash]** `BigHashMaxItemSize` default (bucketSize - 80)

    Maximum size of a small item to be stored in BigHash. Must be less than (bucket size - 80).
* **[BigHash]** `BucketBfSize` default 8 bytes

    Bloom filter size, bytes per bucket.

#### Hybrid Mode Settings

After you call `Hybrid(...)` in a cache builder. Theine will convert current builder to hybrid builder. Hybrid builder has several settings.

* `Workers` defalut 2

    Theine evicts entries in a separate policy goroutinue, but insert to NVM can be done parallel. To make this work, Theine send evicted entries to workers, and worker will sync data to NVM cache. This setting controls how many workers are used to sync data.

* `AdmProbability` defalut 1

    This is an admission policy for endurance and performance reason. When entries are evicted from DRAM cache, this policy will be used to control the insertion percentage. A value of 1 means that all entries evicted from DRAM will be inserted into NVM. Values should be in the range of [0, 1].
