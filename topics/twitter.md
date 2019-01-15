# Twitter

## Requirements and Goals of the System

### Functional
Users should be able to:
- Post new tweets (can contain photos or videos)
- Mark tweets as favorites
- Follow other users

System should be able to:
- Create and display user's timeline consisting of top tweets from all the
  people the user follows

### Non-functional
- Highly available
- Acceptable latency is 200ms for timeline generation
- Consistency can take a hit (in interest of availability), if a user doesn't
  see a tweet for a while, it should be fine

### Extended Requirements
- Searching tweets
- Replying to tweets
- Trending topics - current hot topics/searches
- Tagging other users
- Tweet notification
- Who to follow? Suggestions?
- Moments

## Capacity Estimation and Constraints

1 billion total users, 200 million daily active users (DAU). 100 million new
tweets every day, and on average each user follows 200 people.

**How many favorites per day?** Each user favorites five tweets per day: `200M
users * 5 favorites => 1B favorites`

**How many total tweet-views our system will generate?** Average user visits
timeline two times a day, and visits five other people's pages. User sees 20
tweets on each page. `200M DAU * ((2 + 5) * 20 tweets) => 28 billion per day`

**Storage estimation**: Each tweet has 140 characters, need two bytes to store
a character without compression. Assume we need 30 bytes to store metadata with
each tweet (like ID, timestamp, user ID, etc.). Total storage we would need:
`100 M * (280 + 30) bytes => 30GB/day`

- For 5 years: `30GB/day * 365 * 5 ~= 55 TB`
- For favorites, assume each favorite requires 12 bytes of metadata (tweet ID)
    - `5 favorites per day * 12 = 30 bytes per day * 365 * 5 ~= 55 MB`
- For follows, assume same type of math

**Media**: Assume every 5th tweet has a photo, every tenth has a video. On
average, a photo is 200 KB, a video is 2 MB:
- `100 M / 5 photos * 200 KB + (100 M / 10 videos * 2 MB) ~= 24 TB / day`

**Bandwidth estimates:** Since total ingress is 24TB per day, this would
translate into 290 MB/sec.

Remembering that we have 28 B tweet views per day -- we must show the photo of
every tweet (if it has a photo), but also assume that the users watch every 3rd
video they see in their timeline. Total egress will be:
```
  (28 B * 280 bytes) / 86400s of text => 93 MB/sec
+ (28 B/5 * 200 KB) / 86400s of photos => 13 GB/sec
+ (28 B/10/3 * 2 MB) / 86400s of videos => 22 GB/sec

Total ~= 35 GB / sec
```

## System APIs

**Posting a new tweet**:  
- `tweet(api_dev_key, tweet_data, tweet_location, user_location, media_ids)`
- Returns `string` => URL to access tweet, otherwise error

## High Level System Design

We need a system that can efficiently store all the new tweets:
- Write: `100 M / 86400s ~= 1160 tweets per second`
- Read: `28 B / 86400s ~= 325K tweets per second`

This will be a read-heavy system.

At high level, we need multiple application servers to serve all these requests
with load balancers in front of them for traffic distributions. On the backend,
we need an efficient DB that can store all the new tweets and can support a
huge number of reads. We also need some file storage to store photos and
videos.

The traffic will not be distributed evenly though, at peak time we should
expect at least a few thousand write requests and around 1 M read requests per
second.

## Database Schema

This is similar to Instagram. We would need a Tweet table, a User table, a
UserFollow table, and a Favorite table. If we make the assumption that we are
using a NoSQL DB:
- UserFollow table would hold the UserIDs of the Follows of one User.
- Favorite table will hold the UserID and the TweetIDs of favorites.

## Data Sharding

**Sharding based on UserID:** Try storing all the data of a user on one server.
Pass the UserID to our hash function and get back the DB server that will hold
all of the user's tweets, favorites, follows, etc. While querying for
tweets/follows/favorites of a user, we can use the hash function to find data.

Disadvantages:
1. What if a user becomes hot? There could be a lot of queries on the server
   holding the user. This high load will affect the performance of our service.
2. Over time, some users can end up storing a lot of tweets or have a lot of
   follows compared to others. Maintaining uniform distribution of growing
   user's data will be challenging. 

**Sharding based on TweetID:** Map each TweetID to a random server where we
will store it. To search for tweets, we have to query all servers, and each
server will return a set of tweets.

Timeline generation example:
1. App server finds all people the user follows
2. App server will send the query to all database servers to find tweets from
   these people
3. Each DB server will find the tweets for each user, sort them by recency and
   return the top tweets
4. App server will merge all the results and sort them again to return the top
   results to the user

We have to query all DB partitions to find tweets of a user, which can result
in higher latencies.

**Sharding on Tweet creation time:** Storing tweets based on creation time will
give us the advantage of fetching all the top tweets quickly, and we only have
to query a very small set of servers. But the traffic load will not be
distributed, while writing, all new tweets will be going to one server, and
remaining servers will be sitting idle. Similarly while reading, the server
holding latest data will have a very high load as compared to servers holding
old data.

**Combine sharding by TweetID and Tweet creation time:** Store tweet creation
time and use TweetID to reflect that, we get benefits of both approaches. We
must make each TweetID universally unique in our system, and each TweetID
should contain a timestamp too.

TweetID will have two parts:
- First part is epoch seconds
- Second part is auto-incrementing sequence

Size of TweetID, if epoch time starts today, how many bits would we need to
store number of seconds for the next 50 years?

`86400 sec/day * 365 * 50 ~= 1.6 billion` -- we would need 31 bits to store
this number.

Since we are expecting 1150 new tweets per second, we can allocate 17 bits to
store auto incremented sequence -- this will make our TweetID 48 bits long.
Every second we can store (2 ^ 17 = 130K) new tweets. We can reset our sequence
every second. For fault tolerance and better performance, we can have two DB
servers to generate auto-incrementing keys for us, one generating even numbered
keys, and the other generating odd numbered keys.

In this approach, we still have to query all servers for timeline generation,
but our reads (and writes) will be substantially quicker.
1. We don't have any secondary index (on creation time) which reduces write
   latency.
2. While reading, we don't need to filter on creation time as our primary key
   has epoch time included in it.

## Cache

**Cache replacement policy to use?** LRU is best fit.

**Intelligent caching:** 80-20 rule -> 20% of tweets are generating 80% of read
traffic, which means that certain tweets are so popular that a majority of
people read them. Try to cache 20% of daily read volume from each shard.

**What if we cache the latest data?** If 80% of our users see tweets from past
3 days only, cache all of those tweets. A dedicated cache server that caches
all tweets from all users for the past 3 days: `100 million new tweets ~ 30GB
of new data every day * 3 = 90GB cache memory`. 

Cache could be like a hash table, key = `OwnerID` and value = doubly-linked
list containing all tweets from that user in the past 3 days. Since we want to
retrieve most recent data first, we can always insert new tweets at the head of
the linked list, which means all older tweets will be near the tail of the
list. We can remove tweets from the tail to make space for newer tweets.

## Timeline Generation

Refer to Designing FB Newsfeed.

## Replication and Fault Tolerance

Secondary servers will be used for read traffic only. All writes go to primary
server and then will be replicated to secondary servers. Also gives us fault
tolerance.

## Load Balancing

At three places:
1. Between clients and application servers
2. Between application servers and DB replication servers
3. Between aggregation servers and cache servers

Initially, simple Round Robin can be adopted. Then eventually adopt a more
intelligent LB solution which periodically queries backend server about their
load and adjusts traffic based on that.

## Monitoring

Collect following metrics/counters to understand performance:
1. New tweets per day/second -- daily peak
2. Timeline delivery stats, how many tweets per day/second our service is
   delivering
3. Average latency seen by user to refresh timeline

## Extended Requirements

**How to serve feeds?** Get all latest tweets from people someone follows and
merge/sort them by time. Use pagination to fetch/show tweets. Only fetch top N
tweets from all the people someone follows.

**Retweet:** Store ID of original Tweet and not store any contents on retweet
object.

**Trending Topics:** Cache most frequently occurring hashtags or search queries
in the last N seconds and keep updating them after every M seconds. Rank
trending topics based on frequency of tweets or search queries or retweets or
likes.

**Who to follow? Suggestions?** Suggest friends of people someone follows. Go
two or three levels down. Use ML?

**Moments:** Get top news for different websites for past 1-2 hours, figure out
related tweets, prioritize them, categorize them using ML.

**Search:** Search involves Indexing, Ranking, and Retrieval of tweets.
