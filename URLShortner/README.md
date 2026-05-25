# Design URL Shortener

## Problem Statement
Design a URL shortening service like TinyURL. The service should provide the following functionalities:

### Requirement Gathering
1. FunctionRequirements:
   - Shorten a long URL to a short URL.
   - Redirect a short URL to the original long URL.
   - Track the number of clicks for each short URL.
2. Non-Functional Requirements:
   - The service should be highly available and scalable.
   - The service should have low latency for URL redirection.
   - 1B requests per day
   - 100 Million Daily Active user
   - 1- 5B total Lifetime urls

### API Design
1. Shorten URL API
   - Endpoint: POST /shorten
   - Request Body: { "longUrl": "https://www.example.com" }
   - Response Body: { "shortUrl": "http://short.url/abc123" }
2. Redirect URL API
   - Endpoint: GET /{shortUrlId}
   - Response: Redirects to the original long URL
   

### Basic HLD Design
![img_5.png](img_5.png)

### Better HLD Design
![img_3.png](img_3.png)


### URLs Shortening Algorithm
1. **Base62 Encoding:** Use a combination of uppercase letters, lowercase letters, and digits to create a unique short URL ID.
   1. Create a counter and increment it for each new URL.: But drawback is that it can be easily guessed and can lead to security issues. and there is single point of failure.
   2. Use a Random String Generator: Generate a random string of a fixed length (e.g., 6 characters) for each new URL. This approach is more secure and less predictable than using a counter.
   3. Use a Hashing Algorithm: Hash the long URL to create a unique short URL ID. This approach can help avoid collisions and can be more efficient for large datasets.

### Final HLD Design
![img_4.png](img_4.png)



Markdown# System Design: Production-Grade URL Shortener (TinyURL / Bitly)

This repository contains the comprehensive, production-grade architecture and system design for a scalable, highly available, and low-latency URL Shortener system. It is engineered to handle massive global traffic and scale out horizontally while maintaining ultra-low latency redirection.

---

## 1. System Requirements & Scope

### Functional Requirements
* **Shortening:** Take a long destination URL and return a unique, shortened URL slug (e.g., `https://tiny.cc/abc123X`).
* **Redirection:** When a user accesses a short URL, the system must instantly redirect them to the original long URL with minimum latency using standard HTTP response codes (`302 Found`).
* **Custom Aliases:** Users can request an optional, descriptive custom text slug for their links (e.g., `https://tiny.cc/my-wedding`).
* **Expiration:** Links have a default expiration lifespan (e.g., 5 years), but users can specify a custom expiration timestamp during generation.

### Non-Functional Requirements
* **High Availability ($99.99\%$):** Redirection endpoints are highly critical infrastructure across the web. If redirection is down, shared links "break."
* **Ultra-Low Latency:** Redirection must respond within sub-100ms globally. Caching is essential.
* **Scalability & Read-Heavy Profile:** The system exhibits a highly asymmetric load. Links are clicked (read) orders of magnitude more frequently than they are generated (written). We design for a **100:1 read-to-write ratio**.

---

## 2. Capacity Estimation & Data Math

To properly engineer the underlying infrastructure, we base our resource provisioning on a global scale of **100 million new URLs created per month**.

### Traffic & Throughput (QPS)
* **Write Throughput (URL Generation):**
  $$\text{Writes per second} = \frac{100,000,000 \text{ URLs}}{30 \text{ days} \times 86,400 \text{ seconds}} \approx \mathbf{40 \text{ writes/sec}}$$
* **Read Throughput (URL Redirections):**
  Given our 100:1 read-heavy ratio:
  $$\text{Reads per second} = 40 \times 100 = \mathbf{4,000 \text{ redirections/sec (QPS)}}$$

### Storage Volume Projections (5-Year Plan)
* **Total URLs in 5 Years:** $100\text{M/month} \times 12\text{ months} \times 5\text{ years} = \mathbf{6 \text{ billion URLs}}$.
* **Data Field Size Breakdown per Row:**
  * `short_slug`: 7 bytes (Base62 encoded index)
  * `long_url`: 500 bytes (Assuming conservative average URL length with parameters)
  * `user_id`: 8 bytes (Identifier for rate limiting and link management)
  * `created_at`: 4 bytes (Epoch timestamp)
  * `expire_at`: 4 bytes (Nullable epoch timestamp)
  * *Database Overhead & Indexes:* ~27 bytes
  * **Total Record Size:** $\approx \mathbf{550 \text{ bytes}}$ per row.
* **Total Storage Allocation:** $$6,000,000,000 \text{ records} \times 550 \text{ bytes} \approx \mathbf{3.3 \text{ Terabytes (TB)}}$$

### In-Memory Cache (RAM Allocation)
Applying the **80/20 Pareto Principle**, $20\%$ of individual hot links drive $80\%$ of the redirection traffic. We cache a full day's top $20\%$ of read traffic.
* **Total Daily Reads:** $4,000 \text{ QPS} \times 86,400 \text{ seconds} \approx 345.6 \text{ million reads/day}$.
* **Unique Links to Cache (20%):** $345.6\text{M} \times 0.20 \approx 69.12 \text{ million links}$.
* **Required Cache Memory:** $$69.12 \text{ million links} \times 550 \text{ bytes} \approx \mathbf{38 \text{ GB of RAM}}$$
  *Note: 38 GB can comfortably reside within a single high-memory Redis cluster node, though sharded replication is applied for high-availability.*

---

## 3. Core Logic & Token Generation Service

We must guarantee that every automatically shortened URL maps to a completely unique 7-character identifier. Using an alphabet of **Base62** (`[a-z, A-Z, 0-9]`), the total capacity is:
$$62^7 \approx \mathbf{3.5 \text{ Trillion unique combinations}}$$
This vastly exceeds our 5-year requirement of 6 billion rows.

### Why Hashing Fails at Scale
Using an MD5/SHA256 hash of the long URL and taking the first 7 characters introduces **hash collisions**. Resolving a collision forces a Write Server to read the database to check if the slug is taken, mutating the input text, and re-hashing. This creates an expensive, unscalable read-before-write dependency.

### The Gold Standard: Distributed Counter + Base62
Instead of hashing, we use an ever-incrementing **Global Counter (Numeric ID)**. Number `10,000,000` translates deterministically into its Base62 equivalent string. Because every numeric ID is distinct, collisions are mathematically impossible. 

To eliminate a single database counter becoming a scalability bottleneck, we use a **Key Generation Service (KGS)** orchestrated via **Apache ZooKeeper**:

1. The KGS coordinates allocating dedicated **ranges (token blocks)** of IDs to individual application Write Servers in memory.
2. **Write Server 1** requests a block and receives IDs `1,000,001 to 2,000,000`.
3. **Write Server 2** simultaneously receives IDs `2,000,001 to 3,000,000`.
4. Servers consume their assigned token ranges directly in-memory without cross-network synchronization. If a server crashes, its remaining allocated token sub-range is discarded, causing a negligible gap in the auto-incremental ID sequence, but strictly guaranteeing zero duplicate slugs.

---

## 4. Architectural Design & Runtime Flows (CQRS Pattern)

To scale read paths independently from write paths without resource contention, the system implements **Command Query Responsibility Segregation (CQRS)**.

### The Write Path Flow (Creation)
1. A client submits a long URL via `POST /api/v1/shorten` to the **API Gateway**.
2. The Gateway authenticates, rate-limits, and forwards the payload to the **Write Server Load Balancer**, which routes it to an active **Write Server**.
3. The Write Server pulls an available unique numeric token directly from its internal memory buffer (pre-allocated by the ZooKeeper-orchestrated **Key Generation Service**).
4. The server encodes this numeric token into a **Base62 string** to act as the unique short slug.
5. The unified mapping row (`short_slug`, `long_url`, metadata) is persisted into the distributed **NoSQL Database**.
6. The compiled shortened URL is returned back to the client.

### The Read Path Flow (Redirection)
1. A user triggers a link click via `GET /{short_slug}`.
2. The **API Gateway** intercepts the request and routes it straight through the dedicated **Read Server Load Balancer** to the **Read Server pool**.
3. The targeted Read Server directly queries the highly-available **Redis Cache** cluster.
4. **Cache Hit:** The corresponding long URL is fetched immediately from memory, and a standard `302 Found` response redirects the browser within single-digit milliseconds.
5. **Cache Miss:** The Read Server drops back to execute a key lookup on the distributed **NoSQL Database**. It then synchronously writes this record back into the **Redis Cache** to optimize all future readers before returning the `302 Found` redirect.

---

## 5. Production-Grade Deep Dives & Edge Cases

### Data Layer: Why NoSQL Wins at Scale
While a sharded SQL instance can technically contain 3.3 TB of data, a **NoSQL Key-Value or Document Store (e.g., AWS DynamoDB or Apache Cassandra)** is explicitly selected for this architecture:
* **No Relationships:** A URL shortener represents an isolated key-value mapping (`short_slug -> long_url`). Relational constraints, joins, and ACID-transaction engines add unnecessary computing overhead.
* **Consistent $O(1)$ Operations:** NoSQL databases partition rows by a hash function executed on the Partition Key (`short_slug`). As the cluster scales horizontally from 10 nodes to 100 nodes, lookup times stay perfectly flat.

#### Handling Custom Aliases with Conditional Writes
Custom links do not follow the sequence derived from the Global Counter. This creates a race condition vulnerability if two separate users attempt to register the exact same alias (e.g., `elonmusk`) at the identical microsecond. 

To resolve this without distributed locks on the write path, we leverage **Atomic Conditional Writes** native to NoSQL storage engines:
```text
PutItem(
  TableName: "URL_Mapping",
  Item: {
    "short_slug": "elonmusk",
    "long_url": "[https://twitter.com/elonmusk](https://twitter.com/elonmusk)",
    "created_at": 1774391200
  },
  ConditionExpression: "attribute_not_exists(short_slug)"
)
```
The storage engine handles the execution atomicity. The first query to hit the data node wins. The second concurrent query is rejected with a ConditionalCheckFailedException, which the Write Server handles by returning a 409 Conflict (Alias Already Taken) response to the user.Resilience Strategy: Mitigating the Cache StampedeIf a highly public, viral link (generating 50,000 requests/sec) drops from the Redis cluster due to cache expiration or node eviction, all 50,000 simultaneous connections instantly cascade downward onto the persistent NoSQL database layer. This Cache Stampede (Thundering Herd) will over-utilize database IOPS and cause system-wide brownouts.We mitigate this inside the Read Server application logic by implementing Distributed Mutex Locking (via Redis SetNX):Pythondef get_long_url(short_slug):
    # Attempt high-speed cache read
    long_url = redis.get(short_slug)
    if long_url is not None:
        return long_url # Cache Hit

    # Cache Miss: Protect database from stampede using an atomic lock
    lock_key = f"lock:{short_slug}"
    
    # Try to set lock with 2-second expiration (NX = Only set if not exists)
    if redis.set(lock_key, "locked", nx=True, ex=2):
        try:
            # Double-check cache in case another parallel thread just built it
            long_url = redis.get(short_slug)
            if long_url is not None:
                return long_url
                
            # Single query allowed to execute against persistent storage
            long_url = db.query_long_url(short_slug)
            
            # Write back to Redis to unblock all queued/future readers
            redis.set(short_slug, long_url, ex=86400) # 24-hour TTL
            return long_url
        finally:
            redis.delete(lock_key) # Clean up lock
    else:
        # Lock acquisition failed - another process is active on the DB read
        # Back off and poll cache again after brief sleep
        time.sleep(0.05) # 50 milliseconds
        return get_long_url(short_slug)
URL Expiration & Cleanup LifecycleScanning an active, 6-billion-row production database to locate and prune expired links introduces massive disk I/O contention. We use a hybrid Lazy-Plus-Asynchronous cleanup strategy:Lazy Deletion: When a short link is read, the system checks the expire_at attribute. If the current system epoch exceeds expire_at, a 404 Not Found code is sent to the user, and an asynchronous delete signal is queued.Segmented Batch Sweeper: A background cron service scans a secondary partition index grouped by month/day of expiration. It runs continuously at low priority, deleting expired records in small, isolated batches (e.g., 1,000 rows per transaction). This distributes database cleaning overhead evenly across 24 hours, keeping database performance stable.6. High-Level Summary MatrixMetric / LayerComponent SelectionCore JustificationApp ArchitectureCQRS (Independent Pools)Isolates critical high-traffic reads from compute-heavy writes.Token SourcingCounter + KGS (ZooKeeper)Complete avoidance of runtime hash collisions, guaranteeing $O(1)$ writes.Data StrategyNoSQL (DynamoDB / Cassandra)Horizontal scalability, automated auto-sharding, high performance for simple keys.Caching LayerDistributed Redis ClusterHolds daily active $20\%$ hot links in-memory (~38GB) for sub-10ms performance.Concurrency RuleConditional Writes / Mutex LocksFully blocks race conditions on custom creation and prevents cache stampedes on failure.
