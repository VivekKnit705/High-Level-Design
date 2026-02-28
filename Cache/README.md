# Cache

### What is Cache?
Cache is the process of storing frequently accessed data to a temporary data storage layer, called a cache, so that it can be quickly retrieved when needed.

### Why is Cache important?
1. Save Network call
2. Avoid Recalculation
3. Reduce Latency
4. Reduce Load on the Database

### Why we can not cache everything?
1. Hardware cost will be high
2. If we add ton of data in cache then the search time will increase, and it will defeat the purpose of caching.

### Cache Invalidation Techniques
1. Time to Live
2. Event Drive
3. Write through
4. write around
5. write back/Behind


### Caching Vs CDN
1. **Caching:** Caching is the process of storing frequently accessed data to a temporary data storage layer, called a cache, so that it can be quickly retrieved when needed.
2. **CDN:** A Content Delivery Network (CDN) is a network of servers that are distributed across different geographic locations. The primary purpose of a CDN is to deliver content, such as web pages, images, videos, and other static assets, to users in a faster and more efficient manner.


### Disadvantage of Caching:
1. Cache can show the stale data if the underlying data changes and the cache is not updated.

### Disadvantage of CDN:
1. It has Consistency issues as the content is replicated across multiple servers and there can be a delay in updating the content across all servers.

