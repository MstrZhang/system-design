# concepts

a list of common concepts you should know. these show up in the vast majority of system design questions. for a lot of these, you'll rarely get drilled on implementation details but for some of these it might be beneficial to know a little more than baseline high level knowledge (e.g. load balancers)

---

## scaling

scaling is the process of making your system more resilient to load. wikipedia describes it as _the property of a system to handle a growing amount of work_

with directional scaling you can do a little bit of both as well:

- you can add more servers (horizontal scaling) but also make them more powerful as well

### vertical scaling

adding more processing power to your servers (e.g. RAM, CPUs, storage)

- pros:
  - less complexity since larger numbers of elements increases management complexity
  - easier to deal with concurrency
    - the workload is handled solely on a single server so no need for concurrency management
- cons:
  - there is a limit to how much you can scale
    - constrained by costs
    - constrained by the limits of technology (e.g. how fast processors are, how much memory comes in RAM cards)
  - if you only have a single node and it goes down, your entire system goes down

### horizontal scaling

adding more servers (adding more nodes to a system)

- pros:
  - workload becomes distributed across multiple loads
    - each individual load does not need to be super powerful
  - easier from a hardware perspective
    - you can just add more machines to your pool (no need to check system specifications)
  - better resilience and fault tolerance
    - if a single machine goes down, the system can still stay running
- cons:
  - some sort of management / orchestration is required to maintain and run workloads concurrently
  - higher initial costs
    - adding new servers is more expensive than upgrading existing ones

---

## load balancers

load balancers distribute a set of tasks over a set of resources:

- splits the work across your network of machines
  - makes the overall processing more efficient as it splits the work across multiple loads
  - helps with horizontal scaling
- load balancers themselves can be bottlenecks if they aren't configured properly / if they don't have enough resources
  - a single load balancer is a singple point of failure
- analogy:
  - you can think of the load balancer as a "traffic cop" that handles the requests (like cars in a city) and directs where they can go (like the roads they point to)

load balancers require a policy in order to distribute the load efficiently. each policy has its own tradeoffs and depends on the context:

- **round robin**: requests are distributed sequentially across the group of servers
  - pros:
    - really easy to implement
    - every process gets a chance to read
    - CPU is shared between all processes
  - cons:
    - there is no way to assign priorities to any of the processes
      - if you have low / high priority processes, a big low priority process can hog resources and cause a lot of unnecessary waiting
      - can do this with weighted connections but policy becomes a little trickier to implement
- **least connections**: requests are sent to the server that is handling the least amount of load
  - there are some variations on this depending on properties to look out for:
    - **weighted least connection**: certain servers are weighted higher to take on load
    - **weighted response time**: averages the response time of each server and sends to the quickest server
    - **resource based**: looks for the server with the most resources (i.e. CPU and memory) to give requests to

### load balancers tools

- nginx
- haproxy

---

## caching

a cache is a sort of "memory" component that stores data so that future requests for that data can be served faster. a **cache hit** occurs when the requested data exists in the cache. a **cache miss** occurs when the data is not in the cache

how it works:

- when querying for a resource, the system first checks the cache
  - on cache hit, returns the cache result (doesn't hit the server saving resources)
  - on cache miss, hits the server and returns the server result
    - depending on the cache update policy, the cache might update its cache at this point
- caching popular items can ease the server's job at returning results (eliminating a bottleneck)

key points:

- caches are usually relatively small (otherwise they would not be super efficient)
- caches can be used in most layers of the application
  - examples:
    - web server
    - database
    - application layer

cons:

- need to maintain consistency between the cache and the database
  - done through cache replacement policies
- cache invalidation (i.e. cache replacement policies) are a difficult problem to solve
  - there are additional complexities depending on how you update this / when to update the cache
- application changes might be needed such as adding redis or memcached

### cache implementation strategies

**cache aside**: application interfaces with both cache and storage; cache does not interact with storage

- this is how memcached works

```
           1. check cache first for results

    +--------+         +-------+
    | client |<------->| cache |
    +--------+         +-------+
        ^
        |  2. on miss, query the database and update the cache
        |     (cache never communicates with the databse)
        |
        |              +----------+
        +------------->| database |
                       +----------+
```

process:

- client checks cache for result
  - on cache hit, cache returns the result
  - on cache miss
    - client calls the database
    - database resturns the result
    - result gets added to the cache (e.g. LIFO)
    - result is returned

cons:

- each cache miss results in 3 trips which is slow
  1. client calls cache and misses and cache returns cache miss
  2. client calls database and database returns result
  3. client calls cache again to update cache
- data can become stale if it is updated in the databse
  - e.g. a cached value can't be updated since it can't communicate with the database
  - this can be mitigated with a TTL (**time to live**) flag
- new nodes start off empty which increases the latency
  - higher level of cache misses

**write-through**: application uses cache as the main data store; cache is responsible for reading and writing to the database

```
      +--------+
      | client |
      +--------+
          ^
          |         1. write to the cache
          v
      +-------+     2. on cache hit, return to the user
      | cache |        on cache miss, write to the database
      +-------+
          ^
          |
          v
      +----------+
      | database |
      +----------+
```

pros:

- operation is overall pretty slow but subsequent reads of freshly written data is fast
- users are generally more tolerant of latency when updating data vs reading it
  - good if you are expecting your application to make more reads than writes
- data in the cache is never stale as its constantly being updated by the client
  - this is actually a pretty large pro; you get cache / database synchronization for free

cons:

- new nodes won't cache until the data is first added to the database
  - a lot of initial writes before performance gains start to happen
  - using cache-aside alongside this can help mitigate the issue
- most data written may never be read
  - this can be mitigated with a TTL

**write-back**: data is updated to the cache first; database is updated after some set period of time (with an event processor)

```
    +--------+
    | client |
    +--------+
        |  1. write to the cache
        v
    +-------+        +-------+        +-----------------+
    | cache |------->| queue |------->| event processor |
    +-------+        +-------+        +-----------------+
           2. add to event queue              |   4. after some time, asynchronously write to the database
           3. return to user                  v
                                        +----------+
                                        | database |
                                        +----------+
```

pros:

- since writing happens asynchronously, write performance is improved
  - the user doesn't have to wait for changes to be made to the database
- as long as the write time limit is not too long, the data will probably be synchronized to an acceptable level

cons:

- risk of data loss
  - if the system goes down, changes to the cache data may not have been saved to the database
- more complex to implement

**refresh ahead**: cache refreshes hot data before it expires

implementation details:

- suppose cache TTL is `60 seconds`
- suppose refresh ahead factor is `0.5`
  - i.e. `60 * 0.5 = 30 seconds`
- scenario 1:
  - cached data is accessed at `25 seconds`
  - nothing special happens; cache data is returned as normal
- scenario 2:
  - cached data is accessed after `30 seconds`
  - cache returns the data
  - sometime later, the cache asynchronously refreshes the data
    - data is still hot (being accessed) and cache updates with the database

pros:

- can lead to reduced latency compared to read-through if the cache can accurately predict which items are likely needed in the future
- data is refreshed periodically mitigating stale data

cons:

- poor prediction polices can lead to reduced performance
- can be more difficult to implement
- may incur an extra load on the cache if all keys are refreshed at the same time

### cache eviction policies

a **cache eviction policy** is a strategy to decide which elements to remove first from a cache and manage hot data in a cache

**first-in-first-out (FIFO)**: remove the element that was first added to the cache

pros:

- easy to implement (using a queue)
- does not need to modify cache data at all
  - does not care about timestamp; solely evicts based on entry order
- very predictable and no starvation
  - items will eventually be replaced even if they are infrequently used

cons:

- unfairness
  - treats all cache elements equally so it may evict cache elements that are of high use
  - doesn't consider elements that may be needed to be used again in the future

**most recently used (MRU)**: remove the most recently accessed item from the cache first

idea:

- if you've just seen an item, it's a good indicator that you're unlikely to see the same thing again soon
- analogy:
  - imagine you are at a bus stop watching buses
  - you see the #36 bus come to the stop, pick up passengers and leave
  - it is likely that you will not see the #36 bus soon in contrast to other buses that will stop there

cons:

- quite inefficient if elements tend to stay frequently accessed over time (e.g. a top hits list)
- less data efficient than some other caching policies
  - needs to store a timestamp

**least frequently used (LFU)**: remove the item that has the lowest number of accesses first

pros:

- more data efficient compared to MRU
  - stores only a counter and not an entire timestamp

cons:

- can lead to stale data if item is initially frequently used and then usage drops off

**least recently used (LRU)**: remove the item that was accessed last from the cache

pros:

- this is similar to FIFO but takes into account timestamp
  - FIFO solely accounts for entry order whereas LRU takes into account usage statistics (i.e. timestamp)
- uses memory more efficiently
  - replaces elements that haven't been used for a long time
  - pages that are rarely used or not important are more likely to be swapped out
- relatively simple to implement
  - decent balance between complexity and performance

cons:

- requires cache updating
  - after a cache hit, the element needs to be updated with a new timestamp
- performs poorly in some cases:
  - example:
    - some elements are accessed occasionally but consistently
    - some elements are accessed very frequently for a short period and then never again
      - this data can become stale

### cache tools

- redis
  - stands for "remote dictionary server"
  - in-memory data store used as a key-value pair database
    - makes read / write incredibly fast
  - used as a primary database in some applications (e.g. twitter)
- memcached

---

## availability vs consistency

a distributed system can only guarantee supporting two of the following 3 guidelines in the **CAP theorem**:

- **consistency**: every read receives the most recent write or an error
- **availability**: every read receives a response (non-error response) but no guarantee that it is the most recent version
- **partition tolerance**: the system continues to operate despite an arbitrary number of messages being dropped (or delayed) in the event of a network failure

we need to make a software tradeoff between consistency and availbility:

- **CP** (optimizing for consistency)

  - good choice if the system requires atomic reads and writes (i.e. order of operations matters)
  - waiting for a response might result in a timeout error
  - system going down is tolerable but out of date / out of sync data is not an option
  - examples:
    - banking operations (e.g. stripe, square, paypal)

- **AP** (optimizing for availibility)
  - good choice if business needs to allow for **eventual consistency**
  - good for when the system needs to continue to work despite external errors
  - typically measured as uptime or reliability
    - critical for systems that provide mission critical services or handle sensitive data
  - system going down is not an option but not up to date data is tolerable
  - examples:
    - social media (e.g. twitter, facebook reddit)

### consistency patterns

a **consistency pattern** is a set of techniques that can be used to achieve consistency in a distributed system

**weak consistency**: after a write, reads may or may not see it; best effort approach is taken

- tolerable in situations where data loss is not detrimental
- examples:
  - voip, video chat: we may drop some frames or lose a connection and not hear or see what's happening
  - multiplayer games: you might lag out and lose your connection

**eventual consistency**: after a write, reads will see it (typically within milliseconds); data is replicated async

- if no new updates are made to a given data item, eventually all accesses to that item will return the last updated value
- examples:
  - shopping application: a user may see an item that is not on sale and a sale gets added; after a refresh, the user will see the sale on the item
  - email: a user has some emails in their inbox; after an email gets sent, the user eventually gets the email in their inbox
  - dns: a DNS doesn't necessarily reflect latest values and it takes some time to replicate modified values to all DNS clients and servers

**strong consistency**: after a write all reads will see it; data is replicated synchronously

- works well in systems that require strict transactions
- examples:
  - RDBMS (**relational database management system**): transaction order is very important so the database will lock transactions from occurring when writes are occurring
  - payment platforms: transfer of funds is important and requires synchronous operation of atomic procedures

### availability patterns

an **availibility pattern** is a set of techniques that can be used to achieve availibility in a distributed system. there are two complementary patterns to support high availibility being **fail-over** and **replication**

**fail-over**: if the system detects one node has failed, it responds by passing its duties to a different active node

- two main types of fail-over:
  1. **active-passive**: each node sends its health (e.g. heartbeat) to a main server; if the heartbeat dies, a passive server will take over the active server's IP address and resume service
  - length of downtime is determined whether the passive server is **hot** or **cold**
    - **cold start**: time for a passive server to start up and configure itself before it can start receiving requests
  - only the active server will handle traffic; the passive server just waits until the event of a failure
  - sometimes referred to as **master-slave failover**
  2. **active-active**: both servers are managing traffic but spread the load amongst themselves
  - in the event of a failure, the failed servers responsibilities are passed to the other server
    - this eliminates the cold start problem since the other server is already hot
    - avoids the issue of needing expensive passive servers that are idle most of the time
- cons of fail-over:
  - adds more hardware and additional complexity
  - there is a potential loss for data if the active system fails before any newly written data can be replicated to the passive server

**replication**: act of creating and / or maintaining multiple copies of data (generally database data)

- two main types of replication:
  1. **master-slave**: master writes to one or more slaves; slaves only serve reads
  - in the event of master failing, the system transitions to read-only mode
    - one of the slaves is promoted to be the new master
  - slaves are replicated in a tree-like fashion
  - cons:
    - additional logic is required to promote a slave to be a master
  2. **master-master**: multiple masters serve both reads and writes; if either master goes down, the system can continue to operate on the other master
  - cons:
    - you need a load balancer or application logic changes to split the work across the system
    - most master-master systems are loosely consistent (violating ACID) or have increased write latency due to synchronization
    - conflict resolution becomes a bigger problem as more write nodes are added
- cons of replication:
  - there is a potential loss of data if master fails before any newly written data can be replicated
  - writes are replayed to read replicas
    - if there are a lot of writes, the read replicas can get slowed down
    - the more read slaves you have, the more you have to replicate (results in greater replication lag)
  - in some systems, writing to master can spawn multiple threads to write in parallel
  - adds additional hardware and additional complexity

availability is often quantified in uptime (or downtime) as a percentage of time the service is available. generally represented as a percentage with some number of 9s:

**99.9% - three 9s**

| duration  | acceptable downtime |
| --------- | ------------------- |
| per year  | 8h 45min 57s        |
| per month | 43m 49.7s           |
| per week  | 10m 4.8s            |
| per day   | 1m 26.4s            |

**99.99% - four 9s**

| duration  | acceptable downtime |
| --------- | ------------------- |
| per year  | 52 min 35.7s        |
| per month | 4m 23s              |
| per week  | 1m 5s               |
| per day   | 8.6s                |

---

## content delivery network

a **content delivery network** (CDN) is a globally distributed network of proxy servers that serve content from locations closer to the users

key points:

- typically serves static files
  - e.g. HTML, CSS, JS, photos, videos, ...
  - some CDNs (e.g. Amazon CloudFront) support dynamic content
- a site's DNS resolution will tell clients which server to contact

- pros:
  - users receive content from data centers close to them
    - increased speed
  - eases load on the server
    - the server doesn't need to serve requests that the CDN fulfills
- cons:
  - CDN cost could be significant depending on the traffic
  - content might be stale if it is updated before the TTL expires
  - CDNs require changing URLs for static content to point to the CDN
    - additional overhead

### types of CDNs

**push CDN**: receives new content whenever changes happen on the server

key points:

- good for sites with a small amount of traffic
- good for sites that do not need to update their content very often
  - minimizes traffic but maximizes storage
- content is placed on the CDN once it is made

- cons:
  - you take full responsibility for providing content, uploading to the CDN, and rewriting the URLs to point to the CDN
    - additional overhead

**pull CDN**: grabs new content from the server when the user first requests content

key points:

- content is left on the server and you rewrite URLs to point to the CDN
  - results in a slower request until the content gets cached on the CDN
  - will need to add TTL flags to ensure that the CDN flushes stale data regularly
  - minimizes storage but can create redundant traffic if files expires and are pulled before they have been changed
- good for sites with heavy traffic

  - traffic is spread out more evenly with only recently-requested content remaining on the CDN

- cons:
  - CDN costs could be significant depending on traffic
    - this is often outweighed over the costs you would incur if you weren't using a CDN at all
  - content might be stale if its updated before the TTL expires it
