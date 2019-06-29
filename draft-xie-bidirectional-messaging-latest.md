---
title: An HTTP/2 Extension for bidirectional message communication
docname: draft-xie-bidirectional-messaging-latest
abbrev: HTTP-PUBSUB
ipr: trust200902
area: Internet
workgroup: httpbis Working Group
submissiontype: IETF
keyword: Internet-Draft
#date: 2019-03-01T00:00:00Z

seriesInfo:
 -
    name: Internet-Draft
    value: draft-xie-bidirectional-messaging-00
    stream: IETF
    status: standard

author:
 -
    ins: "G. Xie"
    name: "Guowu Xie"
    organization: "Facebook Inc."
    email: woo@fb.com
 -
    ins: "A. Frindell"
    name: "Alan Frindell"
    organization: "Facebook Inc."
    email: afrind@fb.com

--- abstract

This draft proposes an http2 protocol extension, which enables bidirectional
messaging communication between client and server.

--- middle

# Introduction

HTTP/2 is the de facto application protocol in Internet. The optimizations
developed in HTTP/2, like stream multiplexing, header compression, efficient
binary message framing, and graceful shutdown of open streams, are very generic.
They can be useful in non web browsing applications, for example,
Publish/Subscribe, RPC. However, the unidirectional from client to server
communication pattern limits HTTP/2 from wider use in these applications. This
draft proposes an HTTP/2 protocol extension, which enables bidirectional
communication between client and server.

So far, the only mechanism HTTP/2 provides for server to client communication is
server push. That is, servers can initiate unidirectional push promised streams
to clients, but clients cannot respond to them, except accepting or discarding
them silently. More interesting, the intermediaries may have different server
push policies, and may not forward to downstream at all. The best effort
mechanism is not reliable to deliver content from servers to clients. While this
satisfies some use-cases, its unidirectional property limits HTTP/2 for wider
use, for example, send messages and notifications from servers to clients at the
time they are available.

To work around this limitation, many techniques are developed, like long polling
{{!RFC6202}}, WebSocket {{!RFC8441}}, and tunneling. They are common at:
layering application protocol on top of HTTP/2 and using HTTP/2 streams as
transport connections. These solutions more or less defeat the optimizations
provided by HTTP/2. For example, first, if multiplexing multiple parallel
interactions into one HTTP/2 stream, the head of line block will be
re-introduced. Second, application meta data is encapsulated into DATA frame,
rather than HEADERS frame, so header compression is impossible. Third, user data
are framed more than once at different protocol layers, which offsets the wire
efficiency from HTTP/2 binary frame. Take WebSocket over HTTP/2 as an example,
user data is framed at application protocol, WebSocket, and HTTP/2 layers. This
not only introduces the byte overhead on the wire, but also complicates the data
processing. These techniques also pose new operational challenges to
intermediaries. Due to the traffic from the whole user session is encapsulated
into one HTTP/2 stream, this stream can last very long time. Intermediaries may
take long time to drain these streams. As a remind, GOAWAY can only signals the
remote endpoint to stop using the connection for new streams and does not help
the open streams. Moreover, intermediaries have no visibility to user
interactions, and lose the capability to collect telemetry metrics (e.g., time
to the first/last byte of request and response) for services.

In this draft, a new HTTP/2 frame is introduced which has the routing properties
of PUSH_PROMISE frame and the bi-directionality of HEADERS frame. The extension
provides several benefits: 1) after a HTTP/2 connection setup, a server can
initiate streams to the client anytime, and the client can respond to the
incoming streams accordingly. That is, the communication over HTTP/2 is
bidirectional and symmetric. 2) All the HTTP/2 optimizations still apply.
Intermediaries also have all the necessary meta data to accomplish their goals.
3) Further, clients are able to group streams together for routing purposes,
such that each individual stream group can be used for a different service,
within the same HTTP/2 connections.

# Conventions and Terminology

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL**, when
they appear in this document, are to be interpreted as described in
{{!RFC2119}}.

All the terms defined in the Conventions and Terminology section in {{!RFC7540}}
apply to this document.

# Solution Overview

## RStream and XStream

A routing stream (RStream) is a regular HTTP/2 stream in nature. It is opened by
a HEADERS frame, and **MAY** be continued by CONTINUATION and DATA frames.
RStreams are initiated by clients to servers, and can be independently routed by
intermediaries on the network path. The main purpose for RStream is to
facilitate XStreams' intermediary traversal.

A new HTTP/2 stream called eXtended stream (XStream) is introduced for
exchanging user data bidirectionally. An XStream is opened by an XHEADERS frame,
and **MAY** be continued by CONTINUATION and DATA frames. XStreams can be
initiated by either clients or servers. Another difference from a regular
stream, an XStream **MUST** be associated with an open RStream. In this way,
XStreams can be routed according to their RStreams by intermediaries and
servers. XStream **MUST NOT** be associated with any other XStream, or any
closed RStream. Otherwise, it cannot be routed properly.

## Bidirectional Communication

With RStreams and XStreams, HTTP/2 can be used for bidirectional messaging
communication. As shown in the follow diagrams, as long as an RStream is open
from client to server, either endpoint can initiate an XStreams to its peer.

~~~
+--------+   RStream (5)   +---------+    RStream (1)   +--------+
| client |>--------------->|  proxy  |>---------------->| server |
+--------+                 +---------+                  +--------+
    v                        ^     v                        ^
    |    XStream(7, RS=5)    |     |    XStream(3, RS=1)    |
    +------------------------+     +------------------------+
~~~
{: #fig-client-to-server title="Client initiates an XStream to server."}

~~~
+--------+   RStream (5)   +---------+    RStream (1)   +--------+
| client |>--------------->|  proxy  |>---------------->| server |
+--------+                 +---------+                  +--------+
     ^                        v     ^                        v
     |    XStream(4, RS=5)    |     |    XStream(2, RS=1)    |
     +------------------------+     +------------------------+
~~~
{: #fig-server-to-client title="Server initiates an XStream to client."}

## XStream Grouping

A client can multiplex RStreams, XStreams and regular HTTP/2 streams into one
single HTTP/2 connection. Beyond that, all the XStreams associated with the same
RStream form a logical stream group, and are routed to the same endpoint. This
enables clients to access different services without initiating new connections.
As shown in the following diagram, the client can exchange data with PubSub, RPC
and CDN three different services within one HTTP/2 connection.

~~~
+--------+   RStream (5)   +---------+    RStream (1)   +----------+
| client |>--------------->|  proxy  |>---------------->|  PubSub  |
+--------+   XStream (7)   +---------+    XStream (3)   +----------+
  v   v                     ^ ^  v  v
  |   |     RStream (11)    /  |  |  \    RStream (5)   +----------+
  |   +-------------------+   |  |    +---------------->|    RPC   |
  |         XStream (13)      |  |        XStream (7)   +----------+
  |                           |  |
  |         Stream (21)       |  |      Stream (9)      +----------+
  +---------------------------+  +--------------------->|    CDN   |
                                                        +----------+
~~~
{: #fig-multiplex title="Client opens multiple RStreams, XStreams and an 
HTTP/2 stream within one HTTP/2 connection."}

Reusing one connection for different purposes saves the latency of setting up
new connections. This is desirable for mobile devices since they usually have
higher latency network connectivity and tighten battery constraints.
Multiplexing these services also allows them to share a single transport
connection congestion control context. It could open new optimization
opportunities, like prioritizing interactive streams over static content
fetching streams. Also, it reduces the number of connections on intermediaries
and servers.

## Recommended Usage
RStreams and XStreams are designed for different purposes. To get the more
benefit from this extension, RStreams are recommended for exchanging metadata
only, and **SHOULD** be kept long lived. Because once RStream is closed, the
routing information is gone. No XStream can be exchanged before RStream is
re-established. To maintain RStreams active, clients and servers should not send
out END_STREAM flag, and refresh the timeouts on RStreams if a new XStream is
exchanged. 

By contrast, XStreams are **RECOMMENDED** for exchanging user data, and
**SHOULD** be kept short lived. In long polling, WebSocket and tunneling
solutions, streams have to be kept long live because servers need those streams
for sending back updates in a future time. With this extension, servers are able
to initiate new XStreams as long as RStreams are still open.  Keeping many
XStreams alive for a long time is unnecessary, and costs system resources on
intermediaries and server. Morever, short lived XStreams make connection
graceful shutdown easier on intermediaries and servers. After exchanging GOAWAY
frames, open XStreams can be drained within a short time.

## States of RStream and XStream

RStreams are regular HTTP/2 streams that follow the stream lifecycle in
{{!RFC7540}}, section 5.1. XStreams use the same lifecycle as regular HTTP/2
streams, but have extra dependence on their RStreams. If an RStream is reset,
endpoints **MUST** reset the XStreams associated with that RStream. If the
RStream is closed, endpoints **SHOULD** allow the existing XStreams to complete
normally. The RStream **SHOULD** remain open while communication is ongoing.
Endpoints **SHOULD** refresh any timeout on the RStream while its associated
XStreams are open.

A sender **MUST NOT** initiate new XStreams if on an RStream that is in the open
or half closed (remote) state.

Endpoints process new XStreams only when the RStream is open or half closed
(local) state. If an endpoint receives an XHEADERS frame specifying an RStream
in the closed or haf closed (remote) state, it **MUST** respond with a
connection error of type ROUTING_STREAM_ERROR.

## Negotiate the Extension Through SETTINGS Frame

The extension **SHOULD** be disabled by default. As suggested in {{!RFC7540}},
section 5.5, the unknown ENABLE_XHEADERS setting and XHEADERS frame **MUST** be
ignored by HTTP/2 compliant implementations, which have supported this extension
yet.

Endpoints can negotiate the use of the extension through the SETTINGS frame. If
an implementation supports the extension, it is **RECOMMENDED** to include the
ENABLE_XHEADERS setting in the initial SETTINGS frame, such that the remote
endpoint can disover the support at the earliest time. Once enabled, this
extension **MUST NOT** be disabled over the lifetime of the connection.

An endpoint can send XHEADERS frames immediately upon receiving a SETTINGS frame
with ENABLE_XHEADERS=1.

An endpoint **MUST NOT** send out XHEADERS before receiving a SETTINGS frame
with the ENABLE_XHEADERS=1. If a remote endpoint does not support this extension
yet, the XHEADERS will be ignored, making the header compression contexts
inconsistent between sender and receiver.

If an endpoint supports this extension, but receives XHEADERS frames before
ENABLE_XHEADERS, it is **RECOMMENDED** to respond with a connection error
XHEADER_NOT_ENABLED_ERROR. This helps the remote endpoint to implement this
extension properly.

Intermediaries **SHOULD** send the ENABLE_XHEADERS setting to clients, only if
intermediaries and their upstream servers can support this extension. If an
intermediary receives an XStream but discovers the destination endpoint does not
support the extension, it **MUST** reset the stream with
XHEADER_NOT_ENABLED_ERROR.


## Interact with Standard HTTP/2 Features

The extension implementation **SHOULD** apply stream and connection level flow
control, maximum concurrent streams limit, GOAWAY logic to both RStreams and
XStreams.

# HTTP/2 XHEADERS Frame

The XHEADERS frame (type=0xfb) has all the fields and frame header flags defined
by HEADERS frame in HEADERS {{!RFC7540}}, section 6.2. Moreover, an XHEADERS
frame has one extra field, Routing Stream ID. It is used to open an XStream, and
additionally carries a header block fragment. XHEADERS frames can be sent on a
stream in the "idle", "open", or "half-closed (remote)" state.

Like HEADERS, the CONTINUATION frame (type=0x9) is used to continue a sequence
of header block fragments, if the headers do not fit into one XHEADERS frame.

## Definition

~~~
 +---------------+
 |Pad Length? (8)|
 +-+-------------+-----------------------------------------------+
 |E|                 Stream Dependency? (31)                     |
 +-+-------------+-----------------------------------------------+
 |  Weight? (8)  |
 +-+-------------+-----------------------------------------------+
 |R|                 Routing Stream ID (31)                      |
 +-+-------------+-----------------------------------------------+
 |                   Header Block Fragment (*)                 ...
 +---------------------------------------------------------------+
 |                           Padding (*)                       ...
 +---------------------------------------------------------------+
~~~
{: #fig-ex-headers-frame title="XHEADERS Frame Payload"}

The RStream specified in a XHEADERS frame **MUST** be an open stream. The
recipient **MUST** respond with a connection error of type ROUTING_STREAM_ERROR
PROTOCOL_ERROR, if the specified RStream is missing; or is an XStream rather
than a regualr HTTP/2 stream; or is closed or half-closed (remote). Otherwise,
the states maintained for header compression or flow control) may be out of
sync.

## Examples
This section shows HTTP/1.1 request and response messages are transmitted on 
RStream with regualar HEADERS frames, and on XStream with HTTP/2 XHAEDERS 
frames. 

~~~
  GET /login HTTP/1.1               HEADERS
  Host: example.org          ==>     - END_STREAM
                                     + END_HEADERS
                                       :method = GET
                                       :scheme = https
                                       :path = /login
                                       host = example.org

  {binary data .... }        ==>    DATA
                                     - END_STREAM
                                       {binary data ... }
~~~
{: #c2s-rstream-req title="The request message and HEADERS frame on an 
RStream"}

~~~
  HTTP/1.1 200 OK                   HEADERS
                            ==>      - END_STREAM
                                     + END_HEADERS
                                       :status = 200

  {binary data .... }       ==>     DATA
                                     - END_STREAM
                                       {binary data...}
~~~
{: #c2s-rstream-resp title="The response message and HEADERS frame on an 
RStream"}

The server initiate an XStream to this client.

~~~
  POST /new_msg HTTP/1.1            XHEADERS
                                       RStream_ID = 3
  Host: example.org          ==>     - END_STREAM
                                     + END_HEADERS
                                       :method = POST
                                       :scheme = https
                                       :path = /new_msg
                                       host = example.org
  {binary data}              ==>    DATA
                                     + END_STREAM
                                       {binary data}
~~~
{: #s2c-xstream-req title="The request message and XHEADERS frame on an 
XStream"}

~~~
  HTTP/1.1 200 OK                   XHEADERS
                                       RStream_ID = 3
                             ==>     + END_STREAM
                                     + END_HEADERS
                                       :status = 200
~~~
{: #s2c-xstream-resp title="The response message and XHEADERS frame on an 
XStream"}



# IANA Considerations

This document establishes a registry for a new frame type, setting, and error
code.

## FRAME TYPE Registry

The entry in the following table are registered by this document.

~~~
   +---------------+------+--------------+
   | Frame Type    | Code | Section      |
   +---------------+------+--------------+
   | XHEADERS      | 0xfb |              |
   +---------------+------+--------------+
~~~

## Settings Registry

The entry in the following table are registered by this document.

~~~
+------------------------+--------+---------------+---------------+
| Name                   | Code   | Initial Value | Specification |
+------------------------+--------+---------------+---------------+
| ENABLE_XHEADERS        | 0xfbfb | 0             |               |
+------------------------+--------+---------------+---------------+
~~~

## Error Code Registry

The entry in the following table are registered by this document.

~~~
+----------------------+------+-------------------+---------------+
| Name                 | Code | Description       | Specification |
+----------------------+------+-------------------+---------------+
| ROUTING_STREAM_ERROR | 0xfb | Routing stream is |               |
|                      |      | not open          |               |
| XHEADERS_NOT_        | 0xfc | XHEADERS is not   |               |
| ENABLED_ERROR        |      | enabled yet       |               |
+----------------------+------+-------------------+---------------+
~~~

--- back
