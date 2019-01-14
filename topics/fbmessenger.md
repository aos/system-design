# Facebook Messenger

Design an instant messaging service like FB messenger (or Whatsapp).

## Requirements and Goals

**Functional:**
1. Support 1-1 conversations between users
2. Keep track of online/offline statusest/tttttttttTtbeatbeat?

of its users
3. Support persistent storage of chat history

**Non-functional:**
1. Have real-time chat experience with minimal lag
2. Highly consistent -- users should be able to see the same chat history on
   all devices
3. High availability is desirable, can tolerate lower availability in interest
   of consistency

**Extended requirements:**
- Group chats: support multiple people talking to each other in a group
- Push notifications: be able to notify users of new messages when they are
  offline

## Capacity estimation and constraints

500 million daily active users, on average each user sends 40 messages daily.
This gives us 20 billion messages per day.

**Storage Estimation**: On average, a message is 100 bytes, so to store all the
messages for one day we would need 2TB of storage!

Just for estimation, to save five years of chat history, we would need 3.6
petabytes of storage: `2 TB * 365 days * 5 years ~= 3.6 PB`

Also need to store users' information, messages' metadata (ID, timestamp,
etc.). Above calculations don't keep data compression and replication in
consideration.

**Bandwidth estimation**: If service is getting 2TB of data every day, this
will give us 25MB of incoming data for each second.

Since each incoming message needs to go out to another user, we will need the
same amount of bandwidth `25MB/s` for both upload and download.

## High Level Design

At high level, we will need a chat server that is the central piece
orchestrating all communication between users. When a user wants to send a
message to another user, they will connect to the chat server and send the
message to the server; the server then passes that message to the other user
and also stores it in the DB.

**Detailed workflow:**
1. User-A sends a message to User-B thru chat server
2. Server receives message and sends ACK to User-A
3. Server stores message in DB and sends message to User-B
4. User-B receives message and sends ACK to server
5. Server notifies User-A that message has been delivered successfully to
   User-B

## Detailed Component Design

System needs to handle following use cases:
1. Receive incoming messages and deliver outgoing messages
2. Store and retrieve messages from DB
3. Keep a record of which user is online or has gone offline, and notify all
   relevant users about these status changes

### Message handling

**How to efficiently send and receive messages?** To send a message, a user
needs to connect to the server and post messages for the other users. To get a
message from the server, the user has two options:
1. **Pull model**: Users can periodically ask the server if there are any new
   messages for them
2. **Push model**: Users can keep a connection open with the server and can
   depend upon the server to notify them whenever there are new messages

If first approach, server needs to keep track of messages that are still
waiting to be delivered, and as soon as receiving user connects to the server
to ask for any new message, the server can return all pending messages. To
minimize latency for user, they have to check server quite frequently, most of
the time getting empty responses with no pending messages -- not an efficient
solution.

If second approach, all active users keep a connection open with the server,
then as soon as server receives a message it can immediately pass the message
to the intended user. This way, the server does not need to keep track of
pending messages and minimal latency.

**How will clients maintain an open connection with the server?** Use HTTP
_Long Polling_ or _Web Sockets_.

**How can server keep track of all the opened connects to redirect messages to
the users efficiently?** Server maintains a hash table, where 'key' would be
UserID and 'value' would be the connection object. Whenever server receives a
message for a user, it looks up that user in the table and finds the connection
object to send on the open request.

**What will happen when the server receives a message for a user who has gone
offline?** If receiver has disconnected, server can notify sender about
delivery failure. If temporary (long-poll request timeout), we expect a
reconnect from the user. In that case, we can ask the sender to retry sending
the message. Could be embedded in the client's logic. Server can also store
message for a while and retry sending it once receiver reconnects.

**How many chat servers do we need?** Plan for 500 million connects at any
time. Modern servers can handle 50K concurrent connections at any time → need
10K such servers.

**How to know which server holds the connection to which user?** Software
load balancer in front of chat servers that maps UserID to server.

**How should the server process a 'deliver message' request?** Server needs to
do the following things upon receiving a new message:
1. Store message in DB
2. Send message to receiver
3. Send ACK to sender

Chat server then:
1. Finds server that holds connection for the receiver
2. Pass message to that server to send it to receiver
3. Chat server sends ACK To sender
4. In no particular order, save message to DB

**How does the messenger maintain the sequencing of the messages?** Store a
timestamp with each message, which would be the time when the message is
received at the server. Not ensure correct ordering of messages for clients.
Scenario where server timestamp cannot determine exact order of messages would
be:
1. User-1 sends a message M1 to server for User-2
2. Server receives M1 at T1
3. User-2 sends a message M2 to server for User-1
4. Server receives message M2 at T2, such that T2 > T1
5. Server sends message M1 to User-2 and M2 to User-1

So... User-1 will see M1 first then M2, User-2 will see M2 first then M1.

To resolve, keep a sequence # with every message for each client. Sequence
number will determine exact ordering of messages for EACH user. Both clients
will see a different view of the message sequence, but this view will be
consistent for them on all devices.

### Storing and retrieving messages from DB

Two options for storing a new message:
1. Start a separate thread, work with DB to store message
2. Send async request to DB to store message

Certain things to keep in mind while designing DB:
1. Efficiently working with DB connection pool
2. Retrying failed requests
3. Log requests that failed even after some retries
4. Retrying logged requests when all issues resolved

**What storage system to use?** DB that can support very high rate of small
updates, and fetch a range of records quickly. Because: huge # of small
messages that need to be inserted, and while querying, a user is interested in
sequentially accessing the messages.

- Cannot use RDBMS like MySQL or NoSQL like MongoDB because cannot afford to
  read/write a row from the DB every time a user receives/sends a message.
- Requirements can be met with a wide-column DB like HBase. HBase is a
  column-oriented key-value NoSQL database that can store multiple values
  against one key into multiple columns. Modeled after Google's BigTable and
  runs on top of Hadoop Distributed File System (HDFS). It groups data together
  to store new data in a memory buffer, and once the buffer is full, it dumps
  the data to the disk.

**How should clients efficiently fetch data from the server?** Clients should
paginate while fetching data from the server. Page size could be different for
different clients, eg. cell phones have smaller screens, so we need a lesser
number of message/conversations in the viewport.

### Managing user's status

Since we maintain a connection object on the server for all active users, we
can easily figure out the user's current status from this. Optimizing:
1. Whenever a client starts app, it pulls current status of all users in
   friends' list
2. Whenever a user sends a message to another user that has gone offline, send
   a failure to the sender and update the status on the client
3. Whenever a user comes online, server can broadcast that status with a delay
   of a few seconds to see if the user does not go offline immediately
4. Clients can pull the status from the server about those users that are being
   shown on the user's viewport. Should not be frequent op, as server is
   broadcasting online status of users and we can live with stale offline
   status of users for a while
5. When a client starts a new chat with another user, pull the status at that
   time

**Design summary:**
- Clients open a connection to chat server to send a message
- Server passes message to requested user
- Active users keep a connection open with server to receive messages
- When a new message arrives, chat server pushes it to receiving user on the
  long-poll request
- Messages can be stored in HBase, which supports quick small updates, and
  ranged-based searches
- Servers can broadcast online status of a user to other relevant users
- Clients can pull updates for users who are visible in the client's viewport
  on a less frequent basis

## Data Partitioning

What is our partitioning scheme?

**Partitioning based on UserID:** If we partition based on hash of UserID, we
can keep all messages of a user on the same DB. If one DB shard is 4TB, then we
will have `3.6PB / 4TB ~= 1000` shards for five years. So to do look-ups, we can
find the shard number by `hash(UserID) % 1000`. In beginning, we can start with
fewer DB servers with multiple shards residing on 1 physical server.

Start with a big number of logical partitions, which would be mapped to fewer
physical servers, and as our storage demand increases, we can add more physical
servers to distribute our logical partitions.

## Cache

Cache few recent messages (~ last 15) in a few recent conversations that are
visible in the user's viewport. Since we decided to store all of a user's
messages on one shard, cache for a user should entirely reside on one machine
too.

## Load balancing

Load balancer in front of our chat servers, that maps each UserID to a server
that holds the connection for the user and then direct the request to that
server. Also need load balancer for our cache servers.

## Fault tolerance and replication

**What will happen when a chat server fails?** Our servers are holding
connections with the users. If a server goes down, do we devise a mechanism to
transfer those connections to some other server? (Extremely hard to failover
TCP connections to other servers.) A simple approach can be to have clients
automatically reconnect if the connection is lost.

**Do we store multiple copies of user messages?** Cannot have only one copy of
the user's data, because if server holding data crashes or is down permanently,
we don't have any mechanisms to recover that data. Either we have to store
multiple copies on different servers or use techniques like Reed-Solomon
encoding to distribute and replicate it.

## Extended Requirements

### Group chat

Hold separate group-chat objects in our system that can be stored on the chat
servers. Identified by GroupChatID and will also maintain a list of people who
are part of that chat. The LB can direct each group chat message based on
GroupChatID and server handling that group chat can iterate through all users
of the chat to find the server handling the connection of each user to deliver
message.

In DB, store all group chats in a separate table partitioned based on
GroupChatID.

### Push notifications

Enable our system to send messages to offline users.

Each user can opt-in from their device to get notifications whenever there is a
new message or event. We would need to set up a Notification server, which will
take the messages for offline users and send them to manufacturer's push
notification server, which will then send them to the user's device.
