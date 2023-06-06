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
    - adding new servers is more expensive than upgrading new ones

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

### cache tools

- redis
  - stands for "remote dictionary server"
  - in-memory data store used as a key-value pair database
    - makes read / write incredibly fast
  - used as a primary database in some applications (e.g. twitter)
- memcached
