# data

the coverage of the topics under the data section can vary heavily depending on what type of job you're applying to (e.g. if you're a data scientist, this may be drilled a lot harder than for me as a FED). you should have a baseline understanding of how databases work (i.e. at the level of like a first year databases student)

## databases

considerations for choosing between nosql and relational:

| relational                                                     | nosql                              |
| -------------------------------------------------------------- | ---------------------------------- |
| strucured data                                                 | semi structured data               |
| strict schema                                                  | dynamic or flexible schema         |
| relational data                                                | non relational data                |
| need for complex joins                                         | no need for complex joins          |
| transactions                                                   | store tons of data (i.e. TB or PB) |
| clear patterns for scaling                                     | very data intensive workload       |
| more establised: developers, community, code, tooling, etc ... | very high throughput for IOPS      |
| lookups by index are very fast                                 |                                    |

data that is well suited for nosql:

- rapid ingest of clickstream and log data
- leaderboard or scoring data
- temporary data such as a shopping cart
- frequently accessed (hot) tables
- metadata / lookup tables

### relational databse (RDBMS)

organizes a collection of data items into tables. typically accessed using a query language like SQL

**ACID**: set of properties that describe relational database transactions; all DBMSs guarantee ACID

- **atomicity**: each transaction is all or nothing
- **consistency**: any transaction will bring the databse from one valid state to another
- **isolation**: executing transactions concurrently has the seame results as if they were executed serially
- **duarbility**: once a transaction has been committed, it remains so

methods to scale a relational databse:

**master-slave replication**: see [availability patterns](./concepts.md#availability-patterns)

**master-master replication**: see [availability patterns](./concepts.md#availability-patterns)

**federation**: splits up databases by function

- example:
  - suppose for something similar to amazon
    - one database for forum discussions
    - one database for users
    - one database for products to be served
- cons:
  - not effective if your schema requires huge functions or tables
  - you need application logic to determine which database to read and write
  - joining data from two databases is more complex
  - adds more hardware and additional complexity

**sharding**: distributes data across different databases; each database only manages a subset of the data

- example:
  - a users databse is split up alphabetically by name
    - as number of users increases, more shards are added to the cluster
- pros:
  - results in less read and write traffic
  - less replication
  - index size is reduced
    - generally improves performance with faster queries
  - if one shard goes down, the other shards will remain operational
  - writes can happen in parallel
    - increases throughput
- cons:
  - need application logic to work with shards
    - can result in really complex SQL queries
  - data distribution can be come lopsided
    - examples:
      - a group of power users on one shard can result in increased load to that shard compared to others
      - in the users examples, the last shard [x-y] may receive really low traffic
  - joining data from shards becomes very complex
  - sharding adds more hardware and additional complexity

```
  users database is split up alphabetically in shards (one example of sharding strategy)

                  +-------------+
                  | application |
                  +-------------+
                         |
                         |
                         v
                 +---------------+
                 | load balancer |
                 +---------------+
                         |
                         |
    +------------+-------o-------+-----------+
    |            |               |           |
    |            |               |           |
    v            v               v           v
  +-------+   +-------+      +-------+   +-------+
  | user  |   | user  |      | user  |   | user  |
  | [a-c] |   | [d-f] |      | [g-i] |   | [x-y] |
  +-------+   +-------+      +-------+   +-------+

```

**denormalization**: creates redundant copies of data written in multiple tables to avoid expensive joins; improves read performance at the expect of some write performance

example:

suppose we have the following schema:

`table1`

| key          | type        |
| ------------ | ----------- |
| customer_id  | primary key |
| country      |             |
| city         |             |
| street       |             |
| house_number |             |

`table2`

| key                      | type        |
| ------------------------ | ----------- |
| product_id               | primary key |
| customer_id              | foreign key |
| product_storage_building |             |

`table3`

| key            | type        |
| -------------- | ----------- |
| product_id     | foreign key |
| product_name   |             |
| product_color  |             |
| product_origin |             |

- performing a `JOIN` on all 3 tables is really slow
- denormalization:
  - create a new table from `table1` and `table2` (all of `table1` but include `product_id` and `product_storage_building` from `table2`)
    - suppose this table is called `denormalized_table`
  - join `denormalized_table` with `table3`
    - `SELECT` operations from this table will be a lot faster since we don't need to `JOIN`
    - there is replicated data (`denormalized_table` has only duplicated data)

strategies of denormalization:

- join rows from different tables so you don't have to use queries with `JOIN`
- perform aggregate calculations (like `SUM()`, `COUNT()`, or `MAX()`) so you don't have to use queries with `GROUP BY`
- pre-calculate expensive calculations so you don't have to use queries with complex expressions in the `SELECT`

cons:

- data is duplicated
- constraints can help redundant copies of informations stay in sync
  - increases the complexity of the database design
- a denormalized database under heavy write load might perform worse than its normalized couterpart

**sql tuning**: techniques to optimize your database; this is a really broad topic so there are a lot of topics under this

common RDBMS tools:

- mysql
- postgres

---

### nosql

**nosql**: stores a collection of data in a key-value store, document store, wide column store, or graph database; (i.e. not using SQL to query data in tabular form)

key points:

- data is normalized
- joins are generally done in application code
- most nosql stores do not guarantee ACID
  - favours eventual consistency

properties of nosql systems (**BaSE**):

- **basically available**: system guarantees availability
- **soft state**: the state of the system may change over time, even without input
- **eventual consistency**: the system will become consistent over a period of time given that the system doesn't receive input during that period

types of nosql systems:

**key-value store**: stores values in key-value pairs like a hash table

- pros:
  - allows for `O(1)` reads and writes
    - really fast; often backed by memory or SSD
  - maintains keys in lexicographic order
    - allows for efficient retrieval of key ranges
  - best for systems that need very fast and high performance
- cons:
  - slower for joins
    - implementation is heavily tied to physical representation
    - better for situations that read a lot but do not need to write
- examples:
  - redis
  - mongodb

**document store**: key-value store but documents are stored as values

key points:

- centered around documents (e.g. XML, JSON, binary, etc ...)
  - document stores all information for a given object
- typically provides APIs or a query language based on the internal structure of the document itsefl
- organized by collections, tags, metadata, or directories

examples:

- mongodb
- couchdb
- dynamodb

**wide column store**: key-value store but value is a **column family**

example structure:

```

(row key) 1 ---> |    companies (super column family)     |
                 |----------------------------------------|
                 |address              |website           |
                 |---------------------|------------------|
                 |city   |san francisco|subdomain|www     |  each column family (e.g. address / website) has its own columns
                 |state  |california   |domain   |grio.com|
                 |street |kearny st    |protocol |http    |
```

- pros:
  - groups related data together
  - schemaless
    - can handle unstructured data
  - easier to scale up and replicate data
    - decentralized; can easily scale horizontally
  - use cases:
    - time series from an IoT device
    - historical records
    - high write but low read
    - very large data sets
- examples:
  - cassandra
  - bigtable
  - hbase

**graph database**: each node is a record and each arc is a relationship between nodes

key points:

- optimized to represent complex relationships with many foreign keys or many-to-many relationships
- offer high performance for data models with complex relationships (e.g. like a social network)
- relatively new and not widely used
  - more difficult to find development tools or resources
  - many can only be accessed with REST APIs

examples:

- neo4j
- flockdb (archived)
