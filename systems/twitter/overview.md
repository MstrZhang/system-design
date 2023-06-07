# twitter

design a social media site like twitter that provides:

- ability to make posts (i.e. tweet)
- profiles and ability to view other profiles
- ability to follow other users
- feed of posts

a lot of social media sites are really similar to twitter's design though some may have context-specific differences

## estimations

use cases:

- user can make a tweet
  - user's tweets go out into the feeds of all the people that follow them
  - user's tweets are viewable from their own profile
- user can make a profile
- user can view other people's profiles
  - profile shows tweets made by that specific user
- user can follow other users
- user has a feed that shows tweets
  - shows tweets from users that they follow
- user can search for specific tweets

out of scope:

- blocking / muting users
- muting keywords in tweets
- analytics
- recommended feed
- verified users feed

## constraints and assumptions

general:

- traffic is not evenly distributed
  - some users may tweet a lot
  - most users will read a lot
  - tweeting will depend on time of day / geographic location
- 100 million active users
- 500 million tweets per day (or 15 billion tweets a month)
  - each tweet on average needs to **fanout** to 10 deliveries
    - i.e. assume that on average each user has 10 followers
    - 5 billion total tweets delivered on fanout per day
    - 150 billion tweets delivered on fanout per month
- 250 billion read requests per month
  - about a 1:15 write to read ratio
- 10 billion searches per month

feed:

- viewing the timeline should be fast
- more users will be reading tweets than writing tweets
  - optimize for fast reads
- ingesting tweets is write heavy

search:

- searching should be fast
- searching is read heavy

## back of the envelope

- size per tweet:
  - `tweet_id`: shortlink for the tweet
    - suppose its 8 bytes
  - `user_id`: length of the username
    - suppose its 32 bytes
  - `text`: length of the tweet
    - historically this was 140 bytes
  - `media`: optional media (e.g. an image or a video) associated to the tweet
    - average 10 KB
  - **total**: around 10 KB per tweet
- tweets per month:
  - `10KB * 500 million tweets a day * 30 days per month = 150TB per month`
  - `1.8 PB` of tweet data per year
- read requests per second:
  - `250 billion read per month` => `100,000 reads per second`
- tweets (writes) per second:
  - `15 billion tweets per month` => `6,000 tweets per second`
- fanout per second:
  - `150 billion tweets per month` => `60,000 tweets per second`
- searches per second:
  - `10 billion searches per month` => `4,000 searches per second`

## algorithm details

fanout:

- motivation:
  - if on every read we need to perform a `JOIN` to serve the user's feed or a profile's tweets, this operation would be super slow
  - we want reads to be as fast as possible since this is the heaviest operation we support
- idea:
  - pre-compute all feeds so that on read, the read only has to serve the feed
  - reads do not need to perform any operations to generate the feed
  - add the tweet to all feeds that are subscribed to the tweeting user
    - i.e. the tweets "fan out" to all subscribers of the tweeter
- design considerations:
  - the fanout service is the main potential bottleneck for the system
  - if a user has a lot of followers, the fanout can be extremely slow
    - e.g. historically this has been katy perry, lady gaga, justin bieber
  - slow fanouts can lead to race conditions with replies to the tweet
    - e.g. fanout takes a long time but some users have already responded to the tweet; tweet order becomes messed up on serve
      - we can mitigate this by sorting the results on serve
- optimizations:
  - avoid fanning out tweets from highly-followed users
  - on feed serve:
    1. search for tweets from high follower users
    2. merge search results with the user's feed
    3. reorder results at serve time

## scaling considerations

scaling database:

- consistency is not a huge deal
  - if a user hits a replica with stale data and it takes a few seconds to update that's tolerable
    - twitter itself even asynchronously updates every once in a while
    - tweets don't need to come in "real-time" though they do come in "very roughly" realtime
- because of the immense amounts of reads to the application, a master-slave database even with replicas could be overwhelmed easily
- most obvious way of approaching this is through sharding
  - obvious sharding key is by `user_id`
    - when a user wants their feed, they only care about tweets from a subset of users
    - it's easy to determine which shards to query if they're all grouped by `user_id`
    - we can split the load across databases by not placing all high follower users on the same shard
  - why does sharding by `tweet_id` not work
    - sharding by `tweet_id` makes it difficult to determine what shard a given tweet is in for serving
      - a user's feed may contain tweets from tons of different shards making it super difficult to make feeds
- data may not necessarily have to be in a relational store
  - e.g. user data is graph related so we could potentially move the users table into a graph database (e.g. neo4j)
- search service should be tied to some sort of search cluster
  - e.g. something like elasticsearch or lucene
  - re-indexing can happen asynchronously but will have to happen semi-frequently to produce current results
    - maybe every few hours

scaling memory cache:

- we want to store tweets in memory for fast retrieval
- only store active users (e.g. users who have tweeted sometime in the past month)
  - keeping inactive users is inefficient since their feeds / tweets may never be served
  - if these users become active again, they should be able to tolerate the latency in re-generating their feeds from SQL
- keep maybe a month of tweets in the service
  - this amounts to roughly `5 tweets per day * 30 days a month = 150 tweets`
  - most users who look at a profile might look at, at most 20 tweets
  - if a user scrolls more, they should be able to tolerate some additional latency
