# Instagram

## Requirements
Users: 
  - Should be able to upload/download/view photos
  - Perform searches based on photo/video titles
  - Can follow other users

- System should be able to generate and diisplay a user's News Feed consisting
  of top photos from all the people that the user follows

### Non-functional reqs
- High availability
- Latency `<= 200 ms` for News Feed generation
- Consistency can take a hit (in interest of availability), if user doesn't see
  a photo for a while, it should be fine
- Highly reliable, no uploaded content should be lost

Other things to consider: tagging photos, searching photos on tags, commenting
on photos, tagging users to photos, who to follow, etc.

## Design Considerations
- Read-heavy, retrieve photos quickly
- Upload as many photos as they would like -- efficient management of storage
  should be crucial
- Low latency for viewing photos
- 100% reliable data -- guarantees no losses

## Capacity Estimation and constraints
- 500M total users, 1M daily active users
- 2M new photos every day, 23 new photos every second
- Average photo file size => 200 KB
- Total space required for 1 day of photos: `2M photos * 200KB => 400 GB`
- Total space required over 10 years: `400GB * 365 * 10 ~= 1425TB`

## High level system design
- Support two scenarios:
    1. Upload photos
    2. View/search photos
- Need object storage servers to store photos and some database servers to
  store metadata information about the photos

## Database Schema
- Need to store data about users, their photos, and people they follow
- Photo table stores all data related to photo, index on (PhotoID,
  CreationDate) since we need to fetch recent photos first


|     | Photo                   |
| --- | ---                     |
| PK  | **PhotoID: int**        |
|     | UserID: int             |
|     | PhotoPath: varchar(256) |
|     | PhotoLatitude: int      |
|     | PhotoLongitude: int     |
|     | UserLatitude: int       |
|     | UserLongitude: int      |
|     | CreationDate: datetime  |

|     | User                   |
| --- | ---                    |
| PK  | **UserID: int**        |
|     | Name: varchar(20)      |
|     | Email: varchar(32)     |
|     | DateOfBirth: datetime  |
|     | CreationDate: datetime |
|     | LastLogin: datetime    |

|     | UserFollow       |
| --- | ---              |
| PK  | **UserID1: int** |
| PK  | **UserID2: int** |

- Store above schema in a distributed K-V store to enjoy NoSQL
- Metadata related to photos in one table (key: `PhotoID`, value: object
    containing `PhotoLocation`, `UserLocation`, etc.)

- Storing relationships between users and photos, and list of people a user
  follows → use wide-column datastore like Cassandra
- For `UserPhoto` table, the key would be `UserID`, and value would be the list
  of `PhotoID`s the user owns. Similar scheme for `UserFollow` table

## Data Size Estimation

**User**: Assuming each `int` and `datetime` is four bytes, each row in the
User's table will be of 68 bytes:
```
UserID (4 bytes) + Name (20 bytes) + Email (32 bytes) + DateOfBirth (4 bytes) +
CreationDate (4 bytes) + LastLogin (4 bytes) = 68 bytes
```

If we have 500 million users, we will need 32GB total storage.

**Photo**: Each row in Photo's table will be of 284 bytes:
```
PhotoID (4 bytes) + UserID (4 bytes) + PhotoPath (256 bytes) + PhotoLatitude (4
bytes) + PhotoLongitude (4 bytes) + UserLatitude (4 bytes) + UserLongitude (4
bytes) + CreationDate (4 bytes) = 284 bytes
```

If 2M new photos get uploaded every day, we will need 0.5GB of storage for one
day.

For 10 years, we will need 1.88TB of storage.

**UserFollow**: Each row in UserFollow table will be 8 bytes. If we have 500
million users and on average each user follows 500 users, we would need 1.82TB
of storage for the UserFollow table.

Total space required for all tables for 10 years will be **3.7TB**.

## Component Design

Photo uploads (or writes) can be slow as they have to go to disk, where as
reads will be faster, esp. if being served from cache.

Uploading users can consume all available connections, as uploading is a slow
process. Reads cannot be served if the system gets busy with all write
requests. Web servers have connection limits.

Assume a web server can have max 500 connections at any time, this would mean
it can't have more than 500 concurrent uploads or reads. To handle this
bottleneck: **split reads and writes into separate services**.

## Reliability and redundancy

Cannot lose files! Store multiple copies of each file, if one storage server
dies, we can retrieve photo from other present copy. If we want high
availability, we must have multiple replicas of services running in the system.

Creating redundancy removes single points of failure and provide backup or
spare functionality if needed in a crisis. If two instances of the same service
are running in production, and one fails or degrades, the system can failover
to the healthy copy (automatically or manually).

## Data sharding (metadata)

### **Partitioning based on UserID**
Keep all photos of a user on the same shard. If one DB shard is 1TB, we will
need four shards to store 3.7TB of data. For better performance and
scalability, we keep 10 shards.
  - Find the shard number by UserID % 10, and store the data there. To
      uniquely identify any photo in our system, we can append shard number
      with each PhotoID

**Generating PhotoIDs**: Each DB shard can have its own auto-increment sequence
for PhotoIDs, and since we append ShardID with each PhotoID, it will be unique
across system

**Issues with this partitioning scheme?**
1. How to handle hot users? Several people follow hot users, and a lot of other
   people see any photo they upload
2. Some users have many more photos compared to others, making a non-uniform
   distribution of storage
3. What if we can't store all pix of a user on one shard? If we distribute
   photos of a user onto multiple shards, will it cause higher latencies?
4. Storing all photos of a user on one shard can cause issues like
   unavailability of all the user's data if that shard is down or higher
   latency if it is serving high load, etc.

### Partitioning based on PhotoID
Generate unique PhotoIDs first and then find a shard number through 'PhotoID %
10', this can solve above problems. No need to append ShardID with PhotoID in
this case as PhotoID will itself be unique throughout the system.

**How to generate PhotoIDs?** We cannot have an auto-incrementing sequence in
each shard to define PhotoID because we need to know PhotoID first to find the
shard where it will be stored. One solution could be that we dedicated a
separate database instance to generate auto-incrementing IDs. If our PhotoID
can fit into 64 bits, we can define a table containing only a 64 bit ID field.
Whenever we would like to add a photo into our system, we can insert a new row
in this table and take that ID to be our PhotoID of the new photo.

**Wouldn't this key generating DB be a single point of failure?** Yes.
Workaround: create two such databases, with one generating even-numbered IDs
and the other odd-numbered. For MySQL, the following script can define such
sequence:
```
KeyGeneratingServer1:
auto-increment-increment = 2
auto-increment-offset = 1

KeyGeneratingServer2
auto-increment-increment = 2
auto-increment-offset = 2
```

Put a load-balancer in front of both of these databases to round robin between
them and to deal with downtime. We can extend this design by defining separate
ID tables for Users, Photo-Comments, or other objects present in our system.

**Alternatively**, we can implement a 'key' generation scheme (service).

**Planning for future growth** We can have a large number of logical partitions
to accommodate future data growth, such that in the beginning, multiple logical
partitions reside on a single physical database server. Since each DB server
can have multiple DB instances on it, we can have separate databases for each
logical partition on any server. When a particular DB server has a lot of data,
we can migrate some logical partitions from it to another server. Maintain a
config file (or a separate DB) that can map logical partitions to DB servers.

## Ranking and News Feed Generation

To create the News Feed, we need to fetch the latest, most popular and relevant
photos of other people the user follows.

**Top 100 photos**:

App server:
    1. fetch list of people user follows
    2. fetch metadata info of latest 100 photos for each user
    3. Submit photos to ranking algorithm to determine top 100

Due to possible latency issues, pre-generate the News Feed and store it in a
separate table.

**Pre-generating the News Feed**: Dedicated servers that are continuously
generating users' News Feeds and storing them in a 'UserNewsFeed' table. Query
this table to get latest News Feed for user.

Whenever these servers need to generate the News Feed of a user, they will
first query the UserNewsFeed table to find the last time the News Feed was
generated for that user. Then new News Feed data will be generated from that
time onwards.

## Different approaches for sending News Feed contents to users

1. **Pull**: Clients can pull the News Feed contents from the server on a
   regular basis or manually whenever they need it. Possible problems: new data
   might not be shown to users until clients issue a pull request; most of the
   time, pull requests will result in an empty response if no new data
2. **Push**: Servers can push new data to the users as soon as it is available.
   To efficiently manage this, users have to maintain a Long Poll request with
   the server for receiving the updates. Possible problem: when a user has a
   lot of follows or a celebrity user who has millions of followers; in this
   case the server has to push updates quite frequently.
3. **Hybrid**: Move all users with high following to a pull based model and
   only push data to those users who have a few hundred (or thousand) follows.
   Another approach: server pushes updates to all the users not more than a
   certain frequency, letting users with a lot of follows/updates to regularly
   pull data.

## News Feed Creation with Sharded Data

One of those most important requirements for any given user is to fetch the
latest photos from all people the user follows. We need to have a mechanism to
sort photos on their time of creation. To do this efficiently, we can make
photo creation time part of the PhotoID! As we have a primary index on PhotoID,
it will be quick to find latest PhotoIDs.

Use epoch time for this. PhotoID has two parts:
1. Representing epoch time
2. Auto-incrementing sequence

To make a new PhotoID, we take current epoch time and append an
auto-incrementing ID from our key-generating DB. We can figure out shard number
from this PhotoID (PhotoID % 10) and store the photo there.

**What could be the size of our PhotoID?** If epoch time starts today, how many
bits would we need to store the number of seconds for next 50 years?
```
86400 sec/day * 365 (days/year) * 50 years => 1.6 billion seconds
```

Need 31 bits to store this number. Since we are expecting 23 new photos per
second, we can allocate 9 bits to store auto incremented sequence. So every
second we can store (2^9 => 512) new photos. We can reset our auto-incrementing
sequence every second.

## Cache and load balancing

Our service would need a massive-scale photo delivery system to serve the
globally distributed users. The service should push its content closer to the
user using a large number of geographically distributed photo cache servers and
use CDNs.

Introduce a cache for metadata servers to cache hot database rows. Use Memcache
to cache the data. App servers can quickly check if cache has desired rows
before hitting database. LRU can be a reasonable cache eviction policy for our
system.

**How can we build a more intelligent cache?** If we go with 80-20 rule (20% of
daily read volume for photos is generating 80% of traffic), certain photos are
so popular that the majority of people read them. This dictates we can try
caching 20% of daily read volume of photos and metadata.
