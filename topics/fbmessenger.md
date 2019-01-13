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
