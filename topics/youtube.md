# Youtube

## Requirements and Goals

### Functional
Users should be able to:
- Upload videos
- Share and view videos
- Perform searches based on video titles
- Add/view comments on videos

System should also be able to record stats on videos (likes/dislikes, number
views, etc.)

### Non-functional
- Highly reliable, no lost videos
- Highly available (consistency can take a hit, in the interest of
  availability), if a user doesn't see a video for a while, it should be fine
- Users should have real time experience while watching videos, and should not
  feel any lag

## Capacity Estimation and Constraints

**Assumptions:**
- 1.5 billion total users
- 800 million daily active users
- Users view five videos per day
- Upload:view ratio 1:200 (200 views per upload)
- 500 hours of videos uploaded every minute
- One minute of video needs 50 MB of storage (multiple formats)
- Each upload takes 10 MB/min of bandwidth

**Total video-views per second**: `800 M * 5 / 86400 sec ~= 46K videos/sec`

**Total videos uploaded per second**: `46 K / 200 ~= 230 videos/sec`

**Storage estimates for uploads:**
`500 hours * 60 min * 50 MB ~= 1500 GB/min, 25 GB/sec`

**Bandwidth estimates for uploads:**
`500 hours * 60 min * 10 MB ~= 300 GB/min, 5 GB/sec`

## System APIs

```
uploadVideo(
  api_dev_key, video_title, video_description, tags[], category_id,
  default_language, recording_details, video_contents
)

# Note: video_contents requires a 'stream' object
```
**Returns**: Successful upload will return HTTP 202 (request accepted), and
once the video encoding is completed, the user is notified through email with a
link to access the video. We can also expose a queryable API to let users know
the current status of their uploaded video.

```
searchVideo(dev_key, search_query, user_location, max_vids, page_token)
```
**Returns**: JSON with list of video resources matching search query.

```
streamVideo(dev_key, video_id, offset, codec, resolution)

# Notes:
- offset (number): Stream video from any offset, time in seconds from beginning
of video. If play/pause from multiple devices is supported, store the offset on
the server. This will enable users to start watching a video on any device from
the same point they left.

- codec & resolution: Required to support play/pause from multiple devices. A
TV and phone will have different resolutions and use different codecs.
```
**Returns**: (Stream) - a media stream (video chunk) from given offset

## High Level Design
1. **Processing queue**: Each uploaded video will be pushed to a processing
   queue, to be de-queued later for encoding, thumbnail generation, and
   storage
2. **Encoder**: To encode each uploaded video into multiple formats
3. **Thumbnail generator:** Need few thumbnails for each video
4. **Video and thumbnail storage:** Store in some distributed file storage
5. **User database:** Store user's info
6. **Video metadata storage:** Metadata database for all information about
   videos: title, file path, uploading user, total views, video comments, etc.

## Database Schema

**Video metadata storage** (SQL DB)
- VideoID
- Title
- Description
- Size
- Thumbnail
- Uploader/User
- Total number of likes/dislikes/views

For each video comment:
- CommentID
- VideoID
- UserID
- Comment
- TimeOfCreation

**User data storage** (SQL)
- UserID, name, email, address, age, registration details, etc.

## Detailed Component Design

Service is going to be read-heavy, let's build a system that can retrieve
videos quickly. Our read:write ratio will be 200:1, for every video upload
there are 200 video views.

**Storing videos:** Distributed file system like HDFS or GlusterFS

**Managing read traffic efficiently:** Segregate read traffic from write. Since
we have multiple copies of each video, distribute the read traffic on different
servers. For metadata, use a master-slave configuration. Writes go to master
first and then replayed to all slaves.

This config can cause staleness in data -> when a new video is added, its
metadata would be inserted into master first, and before it gets replayed at
slave, our slaves would not be able to see it and will therefore return stale
results to the user.

**Storing thumbnails:** Way more thumbnails than videos. If we assume five
thumbnails per video, we will need to have a very efficient storage system that
can serve a huge read traffic. Two considerations for storage system:

1. Thumbnails are small files - max 5KB
2. Read traffic for thumbnails will be huge compared to videos. While watching
   one video, users might be looking at a page that has 20 thumbnails

We can't store these thumbnails on disk: we have a huge number of files and
reading these files we have to perform a lot of seeks to different locations on
the disk.

Try using _Bigtable_. It combines multiple files into one block to store on
disk, very efficient in reading small amount of data. Keeping hot thumbnails in
cache will also help improve latencies, and we can easily cache a large number
in memory.

**Video uploads:** Support resume uploads from the same point.

**Video encoding:** Newly uploaded videos are stored on the server, and a new
task is added to the processing queue to encode the video into multiple
formats. Once all the encoding is complete, uploader is notified and video is
made available for view/sharing.

## Metadata Sharding

Due to the scale of our system, we need to distribute our data over multiple
machines.

**Sharding based on UserID:** Store all data for a particular user on one
server. To search videos by title, we have to query all the servers, and each
server will return a set of videos. A centralized server will then aggregate
and rank these results before returning them.

Issues:
1. How to handle popular users? Lots of queries creates a performance
   bottleneck.
2. Some users upload many more videos than others. How to maintain uniform
   distribution of data?

**Sharding based on VideoID:** To find videos of a user, we will query all
servers and each server will return a set of videos. This approach solves our
problem of popular users but shifts it to popular videos.

## Video Deduplication

Since massive scale of video data, our service will have to deal with
widespread video duplication. Duplicate videos often differ in aspect ratios or
encodings, can contain overlays or additional borders, can be excerpts from a
longer original video. Duplicate videos will have lots of impact:

1. Data storage: Wasting storage space by keeping multiple copies
2. Caching: Degraded cache efficiency by taking up space that could be used for
   unique content
3. Network usage: Increasing the amount of data that must be sent over the
   network
4. Energy consumption: All of the above can result in energy wastage

For our service, deduplication makes most sense early, when a user is uploading
a video vs. post-processing time to find duplicate videos later. As soon as any
user starts uploading a video, our service can run a video matching algorithm
(eg. block matching, phase correlation, etc.) to find duplicates. If newly
uploaded video is a subpart of an existing video or vice versa, we can
intelligently divide the video into smaller chunks, so that we only upload
those parts that are missing.

## Load Balancing

Use _consistent hashing_ among our cache servers. We use a static hash-based
scheme to map videos to hostnames. It can lead to an uneven load on the logical
replicas due to popularity of each video. To resolve this issue, any busy
server in one location can redirect a client to a less busy server in the same
cache location. Use dynamic HTTP redirections for this scenario.

Use of redirection has its drawbacks. Since service tries to load balance
locally, it can lead to multiple redirects if host that receives the
redirection can't server the video. Each redirect requires an additional HTTP
request.

## Cache

Introduce a cache for metadata servers to cache hot database rows. Using
Memcache to cache data and application servers before hitting DB can quickly
check if the cache has the desired rows. Use LRU as a cache eviction policy.

**Intelligent cache:** 80-20 rule states that 20% of daily read volume for
videos is generation 80% of traffic. Certain videos are so popular that the
majority of people view them, let's try caching 20% of daily read volume of
videos and metadata.

## CDN

Move most popular videos to CDNs:
- CDNs replicate content in multiple places. Better chance of a video being
  closer to user, with fewer hops, friendlier network
- CDN machines make heavy use of caching and can mostly serve videos out of
  memory

## Fault Tolerance

Use _consistent hashing_ for distribution among database servers. It will help
in replacing a dead server and distributing load among servers.
