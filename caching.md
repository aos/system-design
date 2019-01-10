# Caching

- Takes advantage of locality of reference principle: recently requested data is
  likely to be requested again!

- Short-term memory: limited amount of space, but typically faster than
  original data source and contains the most recently accessed items.

- Often found at the level nearest to the frontend to return data quickly
  without taxing downstream levels

## Application server cache

Place the cache directly on the request layer node -- this will be very fast.
However if you have a load balancer that randomly distributes requests across
the nodes, the same request will go to different nodes, thus increasing cache
misses.

## Distributed cache

Each node of the cache will hold part of the cached data and will send a
request to another node for the data before going to the origin. The cache is
divided using a consistent hashing function, such that if a request node is
looking for a certain piece of data, it can quickly know where to look within
the distributed cache to determine if that data is available.

One of the advantages -- ease by which we can increase the cache space. We just
add more nodes!

Disadvantage -- resolving a missing node. Possible: store multiple copies of
the data on different nodes but logic can get messy real quickly.

## Global cache

All nodes use the same single cache space. This involves adding a server or
file store of some sort, faster than original store, and accessible by all the
request layer nodes. It is possible to overwhelm this global cache if there are
too many request nodes.

Two types, based on what happens during a cache miss:
1. Make the cache itself retrieve the missing data (MOST done here)
2. Make the request node retrieve the data

## Content distribution networks (CDNs)

Usually reserved for sites serving large amounts of static media. In typical
setup, a request will first ask the CDN for a piece of static media, the CDN
will then server that content if it is locally available. If not, the CDN will
query the backend servers for the file and cache it locally.

If system isn't large enough for own CDN, serve static media off a separate
subdomain using a lightweight HTTP server like Nginx.

## Cache invalidation

Three main schemes:

1. **Write-through cache**: data is written into the cache and corresponding DB
   at the same time. Allows for fast retrieval and complete data consistency
   between cache and storage. Disadvantage: higher latency for write operations
   since all writes occur twice.
2. **Write-around cache**: similar to write-through, but data is written
   directly to permanent storage, bypassing cache. This can reduce cache
   being flooded with write operations. The downside is that a read request for
   recently written data will cause a cache miss.
3. **Write-back cache**: data is written to cache alone, and completion is
   immediately confirmed to client. Writing to permanent storage is done after
   specified intervals or under some conditions. Advantage: low latency and
   high throughput for write-intensive apps. Disadvantage: risk of data loss in
   case of a crash or adverse event because only copy of data is in cache.

## Cache eviction policies

1. FIFO: evicts first block accessed first
2. LIFO: evicts block accessed most recently first
3. LRU: discards least recently used item first
4. MRU (most recently used): opposite of LRU
5. LFU (least frequently used): counts how often an item is needed, discards
   least often first
6. RR (random replacement): randomly selects a candidate item
