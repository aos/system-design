# Dropbox

## Requirements and goals

1. Users should be able to upload and download their files/photos from any
   device
2. Users should be able to share files or folders with other users
3. Service should support automatic synchronization between devices
4. System should support storing large files up to a GB
5. ACID-ity is required. Atomicity, Consistency, Isolation, and Durability for
   all file operations should be guaranteed.
6. System should support offline editing. Users should be able to
   add/delete/modify files offline, and as soon as they come online, all their
   changes should be synced to the remote servers and other online devices.
7. **Extended requirement**: System should support snapshotting of data, so
   that users can go back to any version of the files.

## Design considerations

- Huge read and write volumes
- Read-to-write ratio is expected to be nearly the same
- Internally, files can be stored in small parts/chunks (~4MB). Benefits: all
  failed operations only retried for smaller parts of a file.
    - If a user fails to upload a file, then only failing chunk will be
      retried.
    - Reduce amount of data exchange by transferring updated chunks only.
    - By removing duplicate chunks, we can save storage space and bandwidth
      usage.
    - Keeping local copy of metadata (file name, size, etc.) with client can
      save a lot of round trips to server.
    - For small changes, clients can intelligently upload diffs instead of
      whole chunks

## Capacity estimation

- 500M total users, 100M daily active users (DAU)
- Each user connects from `three` different devices
- On average, 200 files/photos => 100 billion total files
- Average file size is 100KB: `100B * 100KB = 10PB (petabytes)`
- 1 million active connections per minute

## High Level Design

- User specifies folder as workspace on device. Any file placed in this folder
  will be uploaded to cloud, and when modified deleted it will be reflected
  in same way in cloud storage.
- User can specify similar workspaces on all devides and any modification done
  on one device will be propagated to all other devices.
- Need to store files and metadata information: File Name, File Size,
  Directory, etc. and who the file is shared with.

We need a few things:
1. Servers that can help with client upload/download files to cloud storage
   (block)
    - Handles cloud and metadata storage
2. Servers that can facilitate updating metadata about files and users
   (metadata)
    - Handles metadata storage
3. Server that implements some mechanism to notify all clients whenever an
   update happens so they can synchronize their files (synchronization)

## Component Design

### Client

Client application monitors workspace folder on user's machine and syncs all
files/folders in it with remote cloud storage. Works with storage servers to
upload, download, and modify actual files to backend cloud storage. Also
interacts with the remote Synchronization Service to handle any file metadata
updates (eg. change in file name, size, modification date, etc.)

Essential operations of the client:
1. Upload and download files
2. Detect file changes in the workspace folder
3. Handle conflict due to offline or concurrent updates

**How to handle file transfer efficiently?** Break each file into smaller
chunks and transfer only modified chunks (and not the whole file).

Divide each file into fixed size chunks of 4MB. Statically calculated what
could be an optimal chunk size based on:
1. Storage devices we use in the cloud to optimize space utilization and IO
   operations per second (IOPS)
2. Network bandwidth
3. Average file size in the storage

In metadata, keep a record of each file and the chunks that constitute it.

**Should we keep a copy of metadata with client?** Keeping a a local copy of
metadata enables offline updates and saves a lot of round trips to update
remote metadata.

**How can clients efficiently listen to changes happening on other clients?**
Have clients periodically check with the server if there are any changes.
Disadvantage: delay in reflecting changes locally as clients will be
checking for changes periodically compared to server notifying whenever
there is some change. If client frequently checks the server for changes, it
will waste bandwidth and keep server busy.
  - **Solution**: HTTP long polling. The client requests information from the
    server with the expectation that the server may not respond
    immediately. If server has no new data for client when poll is
    received, instead of sending an empty response, the server holds the
    request open and waits for response information to become available.
    When available, server sends an HTTP/S response to the client --
    completing the request. Upon receipt, the client can immediately issue
    another server request for future updates.

Divide client into following parts:
1. **Internal metadata database**: keep track of all files, chunks, their
   versions, and location in file system
2. **Chunker**: split files into smaller pieces called chunks. Also responsible
   for reconstructing a file from its chunks. Chunking algorithm will detect
   parts of the files that have been modified by user and only transfer those
   parts to cloud storage.
3. **Watcher**: Monitor local workspace and notify the _Indexer_ of any action
   performed by the users: when users create, delete, or update files or
   folders. Watcher also listens to any changes happening on other clients that
   are broadcasted by Synchronization service
4. **Indexer**: Process events received from the _Watcher_ and update internal
   metadata database with info about chunks of the modified files. Once chunks
   are successfully submitted/downloaded to cloud storage, Indexer will
   communicate with the remote Synchronization service to broadcast changes to
   other clients and update remote metadata database.

**How should clients handle slow servers?** Clients should exponentially
back-off if the server is busy/not-responding. If server is too slow to
respond, clients should delay their retries.

**Should mobile clients sync remote changes immediately?** Unlike desktop or
web clients that check for file changes on a regular basis, mobile clients
usually sync on demand to save user's bandwidth and space.

### Metadata Database

The metadata database is responsible for maintaining the versioning and
metadata information about files/chunks, users, and workspaces. Can be
relational or NoSQL DB. Synchronization service should be able to provide a
consistent view of the files using a DB, especially if more than one user work
with the same file simultaneously. Since NoSQL data stores do not support ACID
properties in favor of scalability and performance, we need to incorporate
support for ACID properties programmatically in the logic of our
Synchronization service in case we opt for this kind of DB. Using a relational
DB makes the most sense here.

Should be storing information about following objects:
1. Chunks
2. Files
3. User
4. Devices
5. Workspace (sync folders)

### Synchronization Service

Process file updates made by a client and applies these changes to other
subscribed clients. Also synchronizes clients' local DB with information stored
in the remote metadata DB. The most important part of this system due to its
critical role in managing the metadata and synchronizing users' files.

Desktop clients communicate with the service to either obtain updates from
cloud or send files and updates to the cloud and potentially other users. If a
client was offline for a period, it polls the system for new updates as soon as
it becomes online. When the service receives an update request, it checks with
the metadata DB for consistency and then proceeds with the update.
Subsequently, a notification is sent to all subscribed users or devices to
report the file update.

Should be designed in such a way to transmit less data between clients and the
cloud to achieve better response times. To meet this goal, the service can
employ a differencing algorithm to reduce amount of data that needs to be
sync'd. Instead of transmitting entire files from clients to server, or vice
versa, we can just transmit the diff between two versions of a file. 

Divide our files into 4MB chunks and transfer modified chunks only. Server and
clients can calculate a hash (eg. SHA-256) to see whether to update a local
copy of a chunk or not. On server if we already have a chunk with a similar
hash (even from another user) we don't need to create another copy, we can use
the same chunk.

Consider using communication middleware between clients and service. Messaging
middleware should provide scalable message queuing and change notification to
support high number of clients using pull or push strategies. This way,
multiple Synchronization service instances can receive requests from a global
request Queue, and the communication middleware will be able to balance the
load.

### Message Queuing Service

We require a messaging middleware that should be able to handle a substantial
amount of requests. A scalable messaging queuing service that supports
asynchronous message-based communication between clients and the
synchronization service instance best fits the requirements.

Implements two types of queues in our system:
1. **Request queue**: global queue, and all clients will share it. Clients'
   requests to update the metadata DB will be sent to the request queue first,
   from there the Synchronization service will take it to update metadata.
2. **Response queues**: correspond to individual subscribed clients,
   responsible for delivering the updated messages to each client. Since a
   message will be deleted from the queue once received by a client, we need to
   create separate response queues for each subscribed client to share update
   messages.

### Cloud/Block Storage

Cloud/Block storage stores chunks of files uploaded by the users. Clients
directly interact with the storage to send and received objects from it.
Separation of metadata from storage enables us to use any storage either in
cloud or in-house.

## File Processing Workflow

Assume Client A updates a file that is shared with Client B and C:
1. Client A uploads chunks to cloud storage
2. Client A updates metadata and commits changes
3. Client A gets confirmation, and notifications are sent to Clients B and C
   about the changes
4. Client B and C receive metadata changes and download updated chunks

## Data Deduplication

Data deduplication is a technique used for eliminating duplicate copies of data
to improve storage utilization. Also can be applied to network data transfers
to reduce number of bytes that must be sent. For each new incoming chunk, we
can calculate a hash of it and compare that hash with all the hashes of the
existing chunks to see if we already have same chunk present in our storage.

Implemented in two ways:

1. **Post-process deduplication**: New chunks are first stored on the storage
   device, and later some process analyzes the data looking for dupes. Benefit:
   clients will not need to wait for the hash calculation or lookup to complete
   before storing data, ensuring no degradation of storage performance.
   Drawbacks: unnecessarily be storing duplicate data (for a short time),
   duplicate data will be transferred consuming bandwidth.
2. **In-line deduplication**: Hash calculations can be done in real-time as
   clients are entering data on their device. If system identifies a chunk
   which it has already stored, only a reference to existing chunk will be
   added in the metadata, rather than full copy of chunk. Optimal network and
   storage usage.

## Metadata Partitioning

For scaling our metadata DB, we need to partition it so that it can store
information about millions of users and billions of files/chunks.

1. **Vertical partitioning**: Partition our DB so that we can store tables
   related to one particular feature on one server. For example, we can store
   all user related tables in one DB, and all files/chunks related tables in
   another DB. Straightforward, but some drawbacks:
    - Still scaling issues. Trillions of chunks to be stored and our DB cannot
      support to store such huge number of records
    - Joining two tables in two separate DBs can cause performance and
      consistency issues
2. **Range-based partitioning**: Store files/chunks in separate partitions
   based on the first letter of the file path. All files starting with 'A' in
   one partition, all files starting with 'B' in another, etc. Combine certain
   less frequency occurring letters into one partition. Disadvantage:
   unbalanced servers. Possible to have too many files starting with letter 'E'
   into a partition, to an extent that we cannot fit them into one partition
3. **Hash-based partitioning**: Take hash of object we are storing and based on
   this hash we figure out DB partition to which this object should go. Take
   hash of 'FileID' of file object to determine partition the file will be
   stored. Hashing function will randomly distribute objects into different
   partitions: map any ID to a number between [1..256]. This approach can lead
   to overloaded partitions, which can be solved with consistent hashing.

## Caching

Two kinds of cache in our system:

1. **Block storage**: to deal with hot files/chunks. Can use an off-the-shelf
   solution like Memcache, that can store whole chunks with their respective
   IDs/Hashes, and Block servers before hitting Block storage can quickly check
   if the cache has desired chunk. Based on clients' usage pattern, we can
   determine how many cache servers we need. A high-end commercial server can
   have up to 144GB memory, so one such server can cache 36K chunks.
2. **Metadata**

**What type of cache replacement policy?** Least-Recently-Used seems to be
best-fit.

## Load Balancer

Add load-balancing layer at two places in our system:

1. Between clients and block servers
2. Between clients and metadata servers

Simple round robin approach can be adopted. Disadvantage: does not take server
load into consideration. A more intelligent LB solution that periodically
queries backend server about their load and adjusts traffic based on that.

## Security, Permissions, and File Sharing

Handle storing permissions of each file in our metadata DB to reflect what
files are visible or modifiable by any user.
