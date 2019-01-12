# Queues

Used to effectively manage requests in large-scale distributed systems. In
complex and large systems, writes can take an almost non-deterministically long
time. In such cases, achieving high performance and availability requires
components to work asynchronously.

All incoming requests are added to the queue, and as soon as any worker has the
capacity to process, they can pick up a task from the queue.

Queues are implemented on the asynchronous communication protocol, meaning when
a client submits a task to a queue they are no longer required to wait for the
results. Instead, they need only acknowledgement that the request was properly
received.

Also used for fault tolerance as they can provide some protection from service
outages and failures. For example, we can create a highly robust queue that can
retry service requests that have failed due to transient system failures. It is
preferable to use a queue to enforce quality-of-service guarantees than to
expose clients directly to intermittent service outages.
