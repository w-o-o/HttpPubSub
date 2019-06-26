%%%
title = "An HTTP/2 extension for bidirectional messaging communication"
abbrev = "bidirectional messaging"
ipr= "trust200902"
area = "Internet"
workgroup = "httpbis Working Group"
submissiontype = "IETF"
keyword = ["Internet-Draft"]
#date = 2019-03-01T00:00:00Z

[seriesInfo]
name = "Internet-Draft"
value = "draft-xie-bidirectional-messaging-00"
stream = "IETF"
status = "standard"

[[author]]
initials = "G."
surname = "Xie"
fullname = "Guowu Xie"
#role = "editor"
organization = "Facebook Inc."
  [author.address]
  email = "woo@fb.com"
  [author.address.postal]
  street = "1 Hacker Way"
  city = "Menlo Park"
  country = "U.S.A."
  code = "CA 94025"
  
[[author]]
initials = "A."
surname = "Frindell"
fullname = "Alan Frindell"
#role = "editor"
organization = "Facebook Inc."
  [author.address]
  email = "afrind@fb.com"
%%%

.# Abstract
This draft proposes a http2 protocol extension, which enables
bidirectional messaging communication between client and server.

{mainmatter}

# Introduction

HTTP/2 is the de facto application protocol in Internet. The optimizations
developed in HTTP/2, like stream multiplexing, header compression, and binary
message framing are very generic. They can be useful in non web browsing
applications, for example, Publish/Subscribe, RPC. However, the request/response
from client to server communication pattern limits HTTP/2 from wider use in
these applications. This draft proposes a HTTP/2 protocol extension, which
enables bidirectional messaging between client and server.

The only mechanism HTTP/2 provides for server to client communication is
PUSH_PROMISE. While this satisfies some use-cases, it is unidirectional, i.g.
the client cannot respond. In this draft, a new frame is introduced which has
the routing properties of PUSH_PROMISE and the bi-directionality of HEADERS.
Further, clients are also able group streams together for routing purposes, such
that each individual stream does not need to carry additional routing
information.

# Conventions and Terminology

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL**, when
they appear in this document, are to be interpreted as described in [@!RFC2119].

All the terms defined in the Conventions and Terminology section in [@!RFC7540]
apply to this document.

# Solution Overview

## Routing Stream and ExStream

A routing stream (RStream) is a long lived HTTP/2 stream in nature. RStreams are
initiated by clients, and can be routed independently by any intermediaries.
Though an RStream is effectively a regular HTTP/2 stream, RStreams are
recommended for exchanging metadata, but not user data.

A new HTTP/2 stream called ExStream is introduced for exchanging user data.
ExStreams are recommended for short lived transactions, so intermediaries and
servers can gracefully shutdown ExStreams within a short time. The typical use
case can be a subscription or publish request/response in Publish/Subscribe use
case, or an RPC call between two endpoints.

An ExStream is opened by an EX_HEADERS frame, and continued by CONTINUATION and
DATA frames. An ExStream **MUST** be associated with an open RStream, and
**MUST NOT** be associated with any other ExStream. ExStreams are routed
according to their RStreams by intermediaries and servers. Effectively, all
ExStreams with the same RStream form a logical stream group, and are routed to
the same endpoint.

## Bidirectional Messaging Communication

With RStreams and ExStreams, HTTP/2 can be used for bidirectional messaging
communication. As shown in the follow diagrams, after an RStream is open from
client to server, either endpoint can initiate an ExStreams to its peer.

{#fig-client-to-server}
~~~
+--------+   RStream (5)   +---------+    RStream (1)   +--------+
| client |>--------------->|  proxy  |>---------------->| server |
+--------+                 +---------+                  +--------+
    v                        ^     v                        ^
    |    ExStream(7, RS=5)   |     |    ExStream(3, RS=1)   |
    +------------------------+     +------------------------+
~~~
Figure: Client initiates the ExStream to server, after an RStream is open.

{#fig-server-to-client}
~~~
+--------+   RStream (5)   +---------+    RStream (1)   +--------+
| client |>--------------->|  proxy  |>---------------->| server |
+--------+                 +---------+                  +--------+
     ^                        v     ^                        v
     |    ExStream(4, RS=5)   |     |    ExStream(2, RS=1)   |
     +------------------------+     +------------------------+
~~~
Figure: Server initiates the ExStream to client, after an RStream is open.

Beyond that, a client can multiplex RStreams, ExStreams and regular HTTP/2
streams into one single HTTP/2 connection. This enables clients to access 
different services without initiating new transport layer connections. This 
saves the latency of setting up new connections. This is more desirable for 
mobile devices because they usually have longer latency network connectivity 
and tighten battery constraints. Multiplexing these services also allows them 
to share a single transport connection congestion control context. It could 
open new optimization opportunities, like prioritizing interactive streams 
over static content fetching streams.

As shown in the following diagram, the client can exchange data with PubSub, 
RPC and CDN three different services within one HTTP/2 connection.

{#fig-multiplex}
~~~
+--------+   RStream (5)   +---------+    RStream (1)   +----------+
| client |>--------------->|  proxy  |>---------------->|  PUBSUB  |
+--------+                 +---------+                  +----------+
  v   v                     ^ ^  v  v
  |   |     RStream (7)    /  |  |   \    RStream (5)   +----------+
  |   +-------------------+   |  |    +---------------->|    RPC   |
  |                           |  |                      +----------+
  |                           |  |  
  |         Stream (9)        |  |      Stream (7)      +----------+
  +---------------------------+  +--------------------->|    CDN   |
                                                        +----------+
~~~
Figure: Client opens multiple RStreams and a HTTP/2 stream within one HTTP/2
connection.

## States of RStream and ExStream

RStreams are regular HTTP/2 streams that follow the stream lifecycle in
[@!RFC7540], section 5.1. ExStreams use the same lifecycle as regular HTTP/2
streams, but have extra dependence on their RStreams. If a RStream is reset, endpoints
**MUST** reset the ExStreams associated with that RStream. If the RStream is
closed, endpoints **SHOULD** allow the existing ExStreams to complete normally. The RStream
**SHOULD** remain open while communication is ongoing. Endpoints **SHOULD** refresh any
timeout on the RStream while its associated ExStreams are open.

A sender **MUST NOT** initiate new ExStreams if on an RStream that is in the
open or half closed (remote) state.

Endpoints process new ExStreams only when the RStream is open or half closed
(local) state. If an endpoint receives an EX_HEADERS frame specifying an RStream
in the closed or haf closed (remote) state, it **MUST** respond with a connection
error of type ROUTING_STREAM_ERROR. 

## Negotiate the Extension through SETTINGS frame

The extension **SHOULD** be disabled by default. As suggested in [@!RFC7540], 
section 5.5, the unknown ENABLE_EX_HEADERS setting and EX_HEADERS frame 
**MUST** be ignored by HTTP/2 compliant implementations, which have supported 
this extension yet.

Endpoints can negotiate the use of the extension through the SETTINGS frame. 
If an implementation supports the extension, it is **RECOMMENDED** to include 
the ENABLE_EX_HEADERS setting in the initial SETTINGS frame, such that the 
remote endpoint can disover the support at the earliest time. Once enabled, 
this extension **MUST NOT** be disabled over the lifetime of the connection.

An endpoint can send EX_HEADERS frames immediately upon receiving a SETTINGS 
frame with ENABLE_EX_HEADERS=1. 

An endpoint **MUST NOT** send out EX_HEADERS before receiving a SETTINGS frame 
with the ENABLE_EX_HEADERS=1. If a remote endpoint does not support this 
extension yet, the EX_HEADERS will be ignored, making the header compression 
contexts inconsistent between sender and receiver.

If an endpoint supports this extension, but receives EX_HEADERS frames before
ENABLE_EX_HEADERS, it is **RECOMMENDED** to respond with a connection error
EX_HEADER_NOT_ENABLED_ERROR. This helps the remote endpoint to implement this
extension properly.

Intermediaries **SHOULD** send the ENABLE_EX_HEADERS setting to clients, only if
intermediaries and their upstream servers can support this extension. If an
intermediary receives an ExStream but discovers the destination endpoint does
not support the extension, it **MUST** reset the stream with
EX_HEADER_NOT_ENABLED_ERROR.


## Interaction with standard HTTP/2 features

The extension implementation **SHOULD** apply stream and connection level flow
control, maximum concurrent streams limit, GOAWAY logic to both RStreams and
ExStreams.

# HTTP/2 EX_HEADERS Frame

The EX_HEADERS frame (type=0xfb) has all the fields and frame header flags
defined by HEADERS frame in HEADERS [@!RFC7540], section 6.2. Moreover, 
a EX_HEADERS frame has one extra field, Routing Stream ID. It is used to 
open an ExStream, and additionally carries a header block fragment. EX_HEADERS 
frames can be sent on a stream in the "idle", "open", or "half-closed (remote)" 
state.

Like HEADERS, the CONTINUATION frame (type=0x9) is used to continue a sequence
of header block fragments, if the headers do not fit into one EX_HEADERS frame.

{#fig-ex-headers-frame}
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
Figure: EX_HEADERS Frame Payload

The RStream specified in a EX_HEADERS frame **MUST** be an open stream. The recipient
**MUST** respond with a connection error of type ROUTING_STREAM_ERROR PROTOCOL_ERROR, if the
specified RStream is missing; or is an ExStream rather than a regualr HTTP/2 stream; or is
closed or half-closed (remote). Otherwise, the states maintained for header
compression or flow control) may be out of sync.

# IANA Considerations

This document establishes a registry for a new frame type, setting, and error
code.

## FRAME TYPE Registry

The entry in the following table are registered by this document.
~~~
   +---------------+------+--------------+
   | Frame Type    | Code | Section      |
   +---------------+------+--------------+
   | EX_HEADERS    | 0xfb |              |
   +---------------+------+--------------+
~~~

## Settings Registry

The entry in the following table are registered by this document.
~~~
+------------------------+--------+---------------+---------------+
| Name                   | Code   | Initial Value | Specification |
+------------------------+--------+---------------+---------------+
| ENABLE_EX_HEADERS      | 0xfbfb | 0             |               |
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
| EX_HEADERS_NOT_      | 0xfc | EX_HEADERS is not |               |
| ENABLED_ERROR        |      | enabled yet       |               |
+----------------------+------+-------------------+---------------+
~~~

{backmatter}
