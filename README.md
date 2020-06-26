# BYMUX

> Byte-oriented multiplexing

Bymux is a minimalistic protocol for multiplexing multiple logical data streams over a single actual data stream. The primary use case is to simulate multiple [tcp](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)-like connections over a single physical connection, but it works on arbitrary reliable, ordered, bidirectional, byte-oriented channels. The multiplexed streams also support independent heartbeats to check whether a particular stream is still alive.

**Status: Stable. Will not change anymore.**

## What, Why, and How?

Establishing reliable connections between programs over a network can be rather expensive. TCP requires a three-way handshake, setting up encryption requires additional roundtrips, and the whole thing might be routed through [tor](https://www.torproject.org). In addition to connection setup time, each tcp connection requires keeping track of a bunch of state, usually tracked by the operating system. For these reasons, it is often desirable to create at most one tcp connection between two programs.

Within such a single connection, there might however be data streams that are logically independent. If one of them is closed, the others should continue working. If one of them needs to be throttled because the receiving program can't keep up processing the data, that doesn't necessarily mean that the other data streams have to be throttled as well. And finally, sending large amounts of data along one data stream should not mean that the others become unusable until the data transfer is done. Bymux provides a generic approach for maintaining separate data streams that have independent flow control (backpressure), don't starve each other, and can be opened and closed without interfering.

The data streams maintained by bymux also provide a heartbeat mechanism for checking whether the other endpoint is still alive: a special "ping" can be sent down a (logical) stream, and the endpoint responds with a corresponding "pong" if it is still working properly. Heartbeats for distinct data streams are fully independent, this makes sense if for example handling of different data streams is delegated to separate processes.

The actual mechanism for providing these features is fairly simple. Each of the logical data streams has a numeric identifier. All data is encapsulated in packets with a light header, this header addresses the stream to which the data pertains. There are different packet types, one for each possible action: creating a stream, writing data to a stream, closing a stream, sending pings and pongs for a stream, and managing flow control for a stream.

Flow control works with a credit-based scheme: Initially, no data may be sent down a stream. An endpoint can give some *credit* to the other endpoint for a stream. The credit indicates how much data may be written, sending one byte consumes one credit. Once credit reaches zero, no more data may be sent until new credit has been received. For bymux to function correctly, it is mandatory that credit is only given for data that can be handled immediately. Since all streams are multiplexed over a single actual data stream, all received data must be processed immediately, lest it block some supposedly independent stream. The credit mechanism makes sure that the other endpoint never overwhelms the receiver's ability to handle the incoming data quickly enough as to not be delaying other streams. In practice, this usually means that the incoming data is simply copied somewhere else in memory for later processing, so that further packets on the underlying connection can be handled immediately. This is also vital for the correctness of the heartbeat mechanism, it ensures that pings are never stuck behind unprocessed data. A more detailed discussion can be found [here](http://250bpm.com/blog:22).

Besides issuing credit exclusively up to an amount of data that can actually be handled immediately, there is a second factor for maintaining correctness: the bymux packets must be small enough relative to the bandwidth of the underlying channel. Suppose the underlying connection only transferred one byte per second, but you wrote a packet of 1800 bytes to a stream. This would block all other information (such as heartbeat pings or smaller chunks of data sent to different streams) for half an hour! This example is obviously exaggerated, but the same applies in realistic situations. Writing a few contiguous gigabytes to a substream over a tcp connection will also delay the other streams. A bymux implementation must thus keep packets reasonably small.

Small packets alone don't guarantee fairness, sending a large number of small packets to the same stream could still starve out other streams. A bymux implementation is thus also required to fairly interleave packets that pertain to different streams, the simplest solution being round-robin. All of this might seem a bit intimidating, but the important part is that these correctness requirements can be handled transparently by the bymux implementation.

## Spec

All data that flows over a bymux channel is encapsulated in packets. Each packet starts out with a *header* byte, whose first three bits indicate which of the six types of packet follows. The fourth bit for all packet types is a flag that indicates whether the packet applies to the whole connection, or to a particular stream. The remaining four bits of the header describe the width of certain integers that appear after the header: a value of `00` indicates a width of one byte, `01` two bytes, `10` four bytes, and `11` eight bytes, or in other words: two to power of the two bits. In general, all ints should be represented in the smallest number of bytes necessary.

| Type | 1st, 2nd and 3rd bit | 4th bit | 5th and 6th bit| 7th and 8th bit |
|------|------------------|---------|-----------------|-----------------|
| Credit | 000 | global? | id width | credit width |
| Write | 001 | global? | id width | amount width |
| Ping | 010 | global? | id width | - |
| Pong | 011 | global? | id width | - |
| Close | 100 | global? | id width | - |
| StopRead | 101 | global? | id width | - |

The 5th and 6th bit work the same way for all packets: if the fourth bit is not set, the packet applies to a particular stream. The id of a stream is a 64 bit integer. The header byte of such a packet is directly followed by the stream id, with the width specified by the 5th and 6th bit.  
When the 4th bit is set to one, the 5th and 6th bit are completely ignored instead, and no stream id follows the header byte. Such a packet is referred to as a *global* packet.

Whenever a header with a 4th byte of zero is followed by an id that does not correspond to an active stream is received, the whole connection must be terminated immediately.

### Sending Data: `Credit` and `Write` Packets

Writing to a stream consumes as much credit as there are bytes written. Writing data without credit being available is forbidden. The maximum of credit for a single stream at a time is 2^64 - 2. Credit can also be set to an infinite amount (represented as 2^64 - 1).

Credit is given to a stream with a `Credit` packet whose fourth bit is zero. The stream id is followed by the amount of credit to be added for the stream, as a big-endian int of the width indicated by the 7th and 8th header byte. If the sum of the previously available credit and the new additional credit equals `2^64 - 1`, the credit is set to infinity, and no more credit is required for writing to the stream. If the sum of the previously available credit and the new additional credit is greater than `2^64 - 1`, the whole connection must be terminated immediately. Sending a `Credit` packet that gives zero bytes sets the credit to infinity instead. Receiving zero credit on a stream that already has infinite credit is ignored. Receiving a nonzero amount of credit on such a stream is an error, and the whole connection must be terminated immediately. This choice of semantics might seem odd at first, but it allows for efficient implementations: zero credit can be handled uniformly, and all overflows of a 64 bit integer are an error.

Note that giving infinite credit can not be undone, and does not relieve the endpoint of the obligation to be able to handle incoming data immediately. This either means allocating an unbounded amount of memory (*don't* do that unless your code runs on a machine with infinite memory, this is exactly the situation that finite credit is designed to allow you to avoid), or being able to handle all incoming data with a bounded amount of memory. An example of the latter would be receiving credit: You can directly add incoming credit to a counter and are done processing, and the counter can not grow without bounds because there is a maximal legal amount of credit. Thus credit for a higher-level protocol (e.g. limiting how many rpc calls may be issued) can be managed over a channel without backpressure.

Data is written to a stream with a `Write` packet whose fourth bit is zero. The stream id is followed by the amount of bytes to be written, encoded as a big-endian int of the width indicated by the 7th and 8th header byte, followed by that many bytes of raw data. Writing to a stream consumes one credit per byte - the metadata (header, stream id, length) does *not* cost any credit. Writes may not exceed the available credit. When receiving written data that hasn't been issued credit for, the whole connection must be terminated immediately. Data may always be written to a stream whose credit has been set to infinity.  
Endpoints should be careful to only send `Write` packets whose length is small enough that the transmission time does not block data from other streams too long.

### Heartbeats: `Ping` and `Pong` Packets

Heartbeat ping and pong packets are straightforward: A ping packet whose fourth bit is one is answered with a pong packet whose fourth bit is one - these packets have no body beyond the header byte. A ping packet whose fourth bit is zero is followed by a stream id, and is answered with a matching pong packet.

Ping and pong don't have clearly defined semantics, from the perspective of bymux either can be sent at any point, there is no invalid state that can be reached by issuing pings or pongs. In particular it is totally fine to send unsolicited pongs. The other endpoint is allowed to ignore them, or it might interpret them as malicious and terminate the connection, but terminating the connection is something it is allowed to do at any point anyways.

### Ending Streams: `Close` and `StopRead` Packets

Ending streams serves two purposes: it notifies the other endpoint that no more data will flow, and it indicates that the state associated with a stream can be forgotten and its id can be reused. Since streams are bidirectional, an endopint can both indicate that it won't send and/or that it won't read data anymore.

A `Close` packet whose fourth bit is zero indicates that the sending endpoint will not write data to the stream anymore. The receiving endpoint must answer with a `StopRead` packet for the same stream (unless it has already sent a `StopRead` packet for that stream). `StopRead` packets can also be sent proactively, in which case they must be answered by a `Close` packet for the same stream (unless the other endpoint has already sent a `Close` packet for that stream).

After sending a `Close` packet for a stream, no more `Write` or further `Close` packets may be sent to it by the same endpoint. After sending a `StopRead` packet for a stream, no more `Credit` or further `StopRead` packets may be sent to it. After having sent both a `Close` and a `StopRead` packet for a stream, no more `Ping` and `Pong` packets may be sent to it. Receiving any of these invalid packets means that the whole connection must be terminated immediately. The stream is however still considered alive until both a `Close` and a `StopRead` packet have been sent by both endpoints. At that point, the stream is fully ended and its id may be reused by further, newly created streams.

Global `Close` and `StopRead` packets (with the 4th bit set to one) refer to the ability to create and accept new streams. They do *not* restrict the actions that can be performed on streams that already exist. They essentially indicate that no new streams will be started, but the old ones can still finish their business. More precisely:

A `Close` packet whose fourth bit is one indicates that the sending endpoint will not create new streams anymore. The receiving endpoint must answer with a global `StopRead` packet (unless it has already sent a global `StopRead` packet). After sending a global `Close` packet, no more global `Write` or further global `Close` packets may be sent by the same endpoint.

A `StopRead` packet whose fourth bit is one indicates that the sending endpoint will not accept new streams anymore. The receiving endpoint must answer with a global `Close` packet (unless it has already sent a global `Close` packet). After sending a global `StopRead` packet, no more global `Credit` or further global `StopRead` packets may be sent by the same endpoint.

After having sent both a global `Close` and a global `StopRead` packet, no more global `Ping` and `Pong` packets may be sent. Receiving any of these invalid packets means that the whole connection must be terminated immediately.

### Creating Streams: Global `Credit` and `Write` Packets

The process of creating streams is also guided by backpressure, one point of global credit allows to create one stream. Global credit is given with a `Credit` packet whose fourth bit is one. Such a header is followed by the amount of global credit to be added, as a big-endian int of the width indicated by the 7th and 8th header byte. If the sum of the previously available global credit and the new additional credit exceeds `2^64 - 1`, the whole connection must be terminated immediately.

A new stream is created with a `Write` packet whose fourth bit is one. For such a packet, the 7th and 8th bit indicate the width of the id of the stream to be created. The header is followed by that id, as a big-endian int of the width indicated by the 7th and 8th header byte.

There are a few rules that govern which new ids can be chosen. First, the endpoint that initiated the connection - called the *proactive* endpoint - may only create streams with an even id, the other endpoint - the *reactive* endpoint - may only create streams with an odd id. If the connection has been established without a clear distinction which of the two processes initiated it, the two roles must be assigned in some other way, the exact mechanism may depend on the specific scenario and is out of scope for this spec.

The second restriction for new ids is that the id may not be in use for another stream. When receiving a packet trying to create a stream with an already active id, the whole connection must be terminated immediately. A stream id is considered active by an endpoint until it has both sent and received `Close` and `StopRead` packets for that stream.

When one endpoint creates a stream, the other endpoint has zero sending credit on that stream. The creation of the stream can be immediately followed up with a `Credit` packet, so the other endpoint can begin sending data virtually the same time it learns of the new stream's existence. If the creating endpoint also had to start out with zero credit, then it would have to wait for the other side to grant some credit before it could start writing data, costing a full roundtip time. Fortunately, as long as both endpoints agree on the same value, there is nothing stopping them from assigning a nonzero amount of starting credit to the creator of a stream. The particular value might be "hardcoded" into a protocol, or it could be communicated during connection setup. The two endpoints might differ in the amount of initial credit they get when creating a new stream. When no other specifics are given however, vanilla "bymux" denotes the setting with starting credit of zero for the creator of any new stream.

## Initial State

When setting up a bymux connection, the initial state can be arbitrary, as long as both sides agree on the same state. For example a protocol might start out with certain streams already available, possibly even with nonzero credit, to save a few roundtrips at startup. It might even begin in a state that pretends that global `Close` and `StopRead` packets had already been sent, thus statically determining the set of available streams. The initial state might be "hardcoded" into a protocol, or it could be negotiated during connection setup. When no other specifics are given however, vanilla "bymux" means that initially there are no streams, both endpoints have global credit of zero, and no `Close` and `StopRead` packets have been sent yet.

### Appendix A: State Summary

An endpoint needs to maintain the following global state:

- its role: proactive (creates streams of even id) or reactive (creates streams of odd id)
- whether it has sent and/or received global `Close` and `StopRead` packets
- how much global credit it currently has
- how much global credit the other endpoint currently has
- the amount of credit that this endpoint is automatically assigned when it creates a new stream
- the amount of credit the the other endpoint is automatically assigned when it creates a new stream

And for each stream the endpoint maintains:

- the stream id
- whether it has sent and/or received `Close` and `StopRead` packets
- how much credit it currently has
- how much credit the other endpoint currently has

The state for a stream can be discarded once `Close` and `StopRead` packets have been both sent and received on that stream.

### Appendix B: Efficiently Restoring Credit

Endpoints are free to grant credit however they see fit. Often they will keep credit below a maximum, filling it up as soon as some incoming data has been processed. Naively sending `Credit` packets whenever some credit becomes available can be rather inefficient. Waiting too long with granting credit however can cause hiccups. In general, the closer the other endpoint is to reaching zero available credit, the more "urgent" it becomes to send new credit, even small amounts. This can be captured by a very simple mechanism: Only send credit if the amount is greater or equal to the amount of credit the other endpoint currently has available (and at least one).
