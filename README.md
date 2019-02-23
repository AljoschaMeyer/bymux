# BYMUX

> Byte-oriented multiplexing

Bymux is an abstraction that extends the concept of reliable, bidirectional, byte-oriented streams to support backpressure, heartbeats, and the ability to create independent substreams. Backpressure is implemented with a credit-based scheme: A (sub-)stream can grant credit to the other endpoint, and may only write as many bytes as it has been granted credit, consuming the credit in the process. Any (sub-)stream can be sent a heartbeat ping and answer to pings with heartbeat pongs.

A bymux stream supports the following operations:

- Receive credit, adding it to the currently available credit. No more than 2^64 - 1 credit can be available at any point in time.
- Give credit to the other endpoint, up to a limit of 2^64 - 1.
- Write some bytes to the other endpoint. These bytes will arrive there in the exact same order, order across different writes is preserved as well. Writing bytes consumes one credit per byte, it is not possible to write if no more credit is available. The other endpoint may idicate that it will not read any further bytes.
- Signal that no more writes will be performed, no more substreams will be opened (substreams that are already open may continue to write data), and no more heartbeat pongs will be sent.
- Read some bytes from the other endpoint. The endpoint may signal that it will not write any more bytes as well as not open any more substreams.
- Signal that no more data will be read, no more credit will be given, no more heartbeat pings will be sent, and any substreams newly-opened by the other endpoint will not be given credit. Existing substreams however may still be read and given more credit. This endpoint may also still create new substreams and give them credit and read from them.
- Send a heartbeat ping, annotated with a natural number smaller than 2^64.
- Receive a heartbeat ping from the other endpoint, including the annotation.
- Send a heartbeat pong, annotated with a natural number smaller than 2^64 (the usual reaction to a ping is to answer with a pong annotated with the same number, unless the stream has already been closed).
- Receive a heartbeat poing from the other endpoint, including the annotation.
- Open a bymux substream, which provides the same API, except you can't open nested substreams. Heartbeats and backpressure of all substreams are fully independent, both of each other and of the main stream. Additonally, there are no ordering guarantees between writes on distinct (sub-)streams. There can be at most 2^64 - 2 open substreams at the same time.

## BYMUX-REL

Bymux-rel is a specification for implementing the bymux abstraction on top of a reliable, bidirectional, byte-oriented channel, e.g. tcp connections or unix domain sockets.

We consider the API of the underlying connection to offer the following operations:

- Write some bytes to the other endpoint. The endpoint may signal that it will not read any further bytes.
- Signal that no more further bytes will be read.
- Read some bytes from the other endpoint. The endpoint may signal that it will not write any further bytes.
- Signal that no more further bytes will be written.

The bymux abstraction can be implemented on top of this API by sending certain *packets* that wrap the actual data. All packets begin with a single *tag* byte. The first three bits of the tag indicate the type of the packet, the next three bits indicate how the (sub-)stream to which the packet applies is addressed, and the last two bits are packet-specific. The following packet types exist:

| Type | First three bits |
|------|------------------|
| Credit | 000 |
| Write | 001 |
| Ping | 010 |
| Pong | 011 |
| Close | 100 |
| StopRead | 101 |
| SubStream | 110 |

Each packet applies to exactly one (sub-)stream. Streams are identified by unsigned 64 bit integers and which endpoint created them. The top-level stream has id zero and creation information is ignored, the id of a substream is specified in the `SubStream` packet that created it. The forth bit of the tag specifies the creator of the target stream. If it is set to 1, the packet applies to a stream that has been created by the sender of the packet. If it's 0, the packet applies to a stream created by the receiver of the packet. The fifth and sixth bit form a two-bit integer `id_width`. Each tag is followed by `2^id_width` bytes containing the stream id as an unsigned, big-endian integer. If the stream id is zero (i.e. the top-level stream), the forth bit of the tag is ignored (and should be set to zero).

The remaining sections give the packet-specific details.

### Credit

The last two bits of a `Credit` tag form a two-bit integer `credit_width`. The stream id of a Credit packet is followed by `2^credit_width` bytes containing the amount of credit that is given as an unsigned, big-endian integer. The total amount of credit the other endpoint has available may never exceed 2^64 - 1. Upon receiving so much credit that the sum of the previously available credit and the new credit would exceed 2^64 - 1, the connection must be immediately terminated.

### Write

The last two bits of a `Write` tag form a two-bit integer `written_width`. The stream id of a Write packet is followed by `2^written_width` bytes containing the length of data that is written as an unsigned, big-endian integer. This integer is then followed by that many bytes of data to write.

Writing to a stream consumes the credit for the stream, you can write at most as many bytes as creit had been available. Upon receiving a write packet whose length exceeds the amount of credit the endpoint should have avaiable, the connection must be immediately terminated.

### Ping

The last two bits of a `Ping` tag form a two-bit integer `nonce_width`. The stream id of a Ping packet is followed by `2^nonce_width` bytes containing arbirary data. Pongs must answer with the same data, so it can be used to identify to which ping a pong corresponds. There are no hard constraints to actually keep nonces unique, some applications may in fact always send a ero byte because they don't need to distinguish between pings at all.

### Pong

The last two bits of a `Pong` tag form a two-bit integer `nonce_width`. The stream id of a Pong packet is followed by `2^nonce_width` bytes containing arbirary data. Pongs may only be sent in response to a previously received ping and must reply with the same data that was contained in the ping.

### Close

The last two bits of a `Close` tag are ignored and should be set to zeros. The Close packet does not carry any data beyond the stream id. The Close ticket indicates that no more Write, SubStream or Pong packets will be sent on this stream. After receiving a Close packet, the connection should be immediately terminated upon receiving any further Write, SubStream or Pong packets on the same stream.

### StopRead

The last two bits of a `StopRead` tag are ignored and should be set to zeros. The StopRead packet does not carry any data beyond the stream id. The StopRead packet indicates that no more data will be read, no more credit will be given, no more pings will be sent, and any further substreams opened by the other endpoint will immediately be sent a StopRead packet as well.

### SubStream

The last two bits of a `SubStream` tag form a two-bit integer `subid_width`. The stream id of a SubStream packet is followed by `2^subid_width` bytes containing the id of the substream as an unsigned, big-endian integer. The new id can be any nonzero unsigned 64-bit integer that is not the id of any other same-level substream.

Since small ids take up less space, it is a good idea to recycle unused ids. An id that has been previously used can be assigned again if the previous substream of this id has had `Close` and `StopRead` packets from both endpoints.
