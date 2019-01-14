# Long-Polling vs WebSockets vs Server-Sent Events

Popular communication protocols between a client (like a web browser) and web
server.

Lifetime of a standard HTTP web request:
1. Client opens a connection and requests data from server
2. Server calculates response
3. Server sends response back to client on opened request

## AJAX polling

Idea: client repeatedly polls (or requests) a server for data. The client makes
a request and waits for the server to respond with data. If no data is
available, an empty response is returned.

1. Client opens a connection and requests data from the server using regular
   HTTP
2. Requested webpage sends requests to the server at regular intervals (eg. 0.5
   seconds)
3. Server calculates response and sends it back, just like regular HTTP traffic
4. Client repeats above three steps periodically to get updates from server

Disadvantage: client keeps asking server for any new data. As a result, a lot
of responses are empty creating HTTP overhead.

## HTTP Long-Polling

Client requests information from server exactly as in normal polling, but
with expectation that the server may not respond immediately.

- If server does not have any data available for client, instead of sending an
  empty response, server holds request and waits until some data becomes
  available.
- Once data becomes available, a full response is sent to client. The client
  then immediately re-request information from the server so that the server
  will almost always have an available waiting request that it can use to
  deliver data in response to an event.
- Each Long-Poll request has a timeout. The client has to reconnect
    periodically after the connection is closed, due to timeouts.

## WebSockets

Provides _full duplex_ communication channels over a single TCP connection. It
provides a persistent connection between a client and a server that both
parties can use to start sending data at any time.

Client establishes a WebSocket connection through a process known as the
WebSocket handshake. If successful, the server and client can exhange data
bidirectionally in real-time. Made possible by providing a standardized way for
the server to send content to the browser without being asked by the client,
and allowing for messages to be passed back and forth while keeping the
connection open.

## Server-Sent Events

Client establishes a persistent and long-term connection with the server.
Server uses this connection to send data to a client. If client wants to send
data to the server, it would require the use of another tech or protocol.

1. Client requests data from a server using regular HTTP.
2. Requested webpage opens a connection to the server
3. Server sends the data to the client whenever there's new info available

Best when we need real-time traffic from the server to the client or if server
is generating data in a loop and will be sending multiple events to the client.
