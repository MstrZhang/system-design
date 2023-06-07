# URL shortener

design an application that takes a URL and converts it into a shortened URL. using the shortened URL will redirect to the main URL

example:

```
https://long-link.com/long-link/super-long-link?query=long-query&another=another-query

turns into:

https://kevin.com/abc1234
```

services that do this:

- bit.ly
- twitter
- tinyurl
- pastebin
  - pastebin is a really similar application but it stores a user supplied document (i.e. the paste) instead of a URL

## estimations

use cases:

- user enters a URL and receives a shortened URL
  - expiration:
    - discuss whether or not there should be any expiration or not
    - default setting: no expiration
    - optional setting: set an expiration
      - setting an expiration could be a good idea to ensure there isn't stale data
      - if a URL isn't used frequently, a TTL can flush it out and save storage space
- using a shortened URL directs the user to the actual link
- user is anonymous
  - small issue here: a malicious user could use up all our URLs
  - mitigation strategies:
    - user can sign up for an account
      - API can detect malicious users and ban them
      - downside is that it adds barrier to entry for our app which make it less desirable
      - increased complexity (need to add a users table)
    - rate limiting
      - detect malicious users and slow down access / ban their IP from accessing the API
      - downside is added application level complexity
- service deletes expired URLs
- service has high availability
  - we don't care as much about consistency (it doesn't matter if the app fails or returns an error)
  - we do care if the service is down; if we are down, we don't operate
- some sort of analytics service for getting use statistics

out of scope:

- user registering for an account
- user can set the shortlink (i.e. vanity URLs)

## constraints and assumptions

- traffic will not be evenly distributed
  - not all parts of the world will access our application evenly
  - sometimes we will have spurts of high traffic and sometimes none at all
- following a shortlink should be fast
- links are all valid
- URL view analytics do not need to be realtime
- usage assumptions:
  - 10 million users
  - 10 million URL writes a month
  - 100 million reads a month
    - 10:1 read to write ratio

## algorithm design

basic idea:

- algorithm receives a URL
  - worst case max length should be somewhere around 2048 characters
  - we only have to generate the short link part of the URL
    - i.e. https://kevin.com/**ABC123**
    - the domain name will always be fixed
- use some sort of hashing algorithm to generate a unique hash
  - e.g. MD5 or SHA256 (for example MD5, this will result in a 128-bit hash)
- encode the URL
  - suppose we use base62
  - base62 encoding:
    - with a 6 letter long shortlink, we have `62^6 = about 56 billion strings`
      - at 10 million writes per month, this would be exhausted in 5,600 months
      - pretty challenging to run out but possible with more scale
    - with a 7 letter long shortlink, we have `62^7 = 3.5 trillion strings`
      - at 10 million writes per month, this would be exhausted in around 29,000 years
      - even at 1,000 keys per second, it would take 111 years to run out of namespace
        - good tradeoff between URL length and namespace at 7 chars
  - potential hash collisions with MD5 + base62
    - MD5 will hash to a 128-bit hash
    - base62 encode will encode to a 20-something long string
    - we need to truncate the string to 7 characters to use for our shortlink
      - this truncation process can potentially lead to hash collisions

dealing with hash collisions:

- solution 1: attempt to add to the database first and change some chars around / add a salt on collision
  - disadvantages:
    - adds additional latency to the write process
    - not all databases support this (you need something like `putIfAbsent()`)
- solution 2: use a counter that counts through the namespace; encode the counter variable and not the URL
  - solves the problem since every shortlink becomes guaranteed to be unique
  - we would be required to purge old results at some point when the counter reaches the end
  - disadvantages:
    - single point of failure
      - if the counter fails, the entire app goes down
    - if requests spike, the counter needs to be able to handle it
    - counter can cause the results to be sequential
      - potential security issue since a malicious user can predict new URLs
      - can add some entropy to this (like a salt) to mitigate
    - very difficult to scale this design
      - if we add more counters to become more resilient, it becomes difficult to coordinate / synchronize the counters
- solution 3: pre-calculate all 3.5 trillion hashes and use a distributor service to return hashes on request
  - solves synchronization issue with the counters
  - since hash is computed already, saves computation power
  - hashes can be distributed randomly across instances which solves serialization issue
  - distributor can easily be scaled horizontally
    - data can be sharded across the distributors
    - if a distributor goes down, it's fine because we can re-route traffic to a different distributor

## extra things to note

load balancer:

- easiest way to scale this is by scaling servers horizontally
  - i.e. adding more servers
- to do this, we need a load balancer to distribute load evenly across the servers
  - easiest way to do this would probably be a round-robin
    - may not be the best policy since some areas might get more traffic and make more requests (e.g. usa vs egypt)
  - least connections policy would be good for distributing load evenly across servers

caching:

- for the read API
  - API checks cache for result and returns on hit
  - on miss, calls the DB and updates the cache
- for the write API
  - API checks to see if the URL has been shortened recently
    - if it was, we can save a redundant call
  - the way the API is setup right now can lead to redundant URLs
    - probably would be good to purge stale entries with some sort of TTL flag potentially
- write-aside cache is probably sufficient for this use case
  - writes do not happen as frequently as reads (10x more reads than writes)
  - user will be more tolerant to latency during writes but not during reads

storage:

- problem lends itself really nicely to nosql
  - we are storing key-value pairs (i.e. short URL to original URL)
- federation of databases:
  - main database: stores the key-value pairs of URLs
  - analytics database: stores the analytics data
    - maybe something like cassandra (wide-column storage)
  - users database: if we wanted to incorporate a user registration of some sort
  - object store: if we are storing documents instead of URLs (like pastebin)

pastebin:

- if we have something like pastebin where we are storing content, we can use an object store
  - e.g. amazon s3
- pastebin needs to quickly retrieve documents
  - we can optimize this process using DNS routing and a CDN
    - since pastes are static content (i.e. essentially text documents)
- pastebin has some slightly different constraints
  - we need to think of storage assumptions
  - we could quickly run out of data storage depending on how big the paste content we are getting is
