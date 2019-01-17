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

Assumptions:
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
`500 hours  * 60 min * 50 MB ~= 1500 GB/min, 25 GB/sec`

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
- Uploader
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
