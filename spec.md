# Globalized Secure Protocol (GSP)

**GSP Core Protocol Specification**  
**Version:** 1.0
**Status:** Experimental

![Status](https://img.shields.io/badge/status-experimental-orange)
![Specification](https://img.shields.io/badge/specification-GSP%2F1.0-blue)

---

## Abstract

The **Globalized Secure Protocol (GSP)** is a general-purpose protocol for
secure, low-latency communication between applications.

GSP provides a common protocol layer for applications that need more control
than a normal TCP connection provides, without requiring every application to
build its own networking stack on top of UDP.

The protocol combines:

- Secure communication
- Multiplexed streams
- Reliable and unreliable delivery
- Packet framing
- Acknowledgements
- Loss detection
- Flow control
- Congestion control
- Connection management
- Connection migration
- Optional compression
- Dynamic padding
- Extensibility
- Multiple transport profiles

GSP is designed to operate over transports such as UDP, TCP, and QUIC.

The protocol is currently experimental. The wire format, cryptographic
construction, and some transport behavior may change before a stable
version is released.

---

# 1. Introduction

There are already many networking protocols.

GSP is not intended to replace all of them.

The reason for GSP is the gap between a very simple transport and a complete
application protocol.

With TCP, an application gets a reliable ordered byte stream:

```text
Application
     |
     v
    TCP
     |
     v
   Network

With UDP, the application gets datagrams, but has to implement most of the features it needs itself:

Application
     |
     +-- Packet format
     +-- ACKs
     +-- Retransmission
     +-- Ordering
     +-- Encryption
     +-- Replay protection
     +-- Streams
     +-- Flow control
     +-- Congestion control
     |
     v
    UDP

GSP attempts to provide a common solution:

Application
     |
     v
+----------------------+
|         GSP          |
|----------------------|
| Security             |
| Streams              |
| Reliability          |
| Flow control         |
| Congestion control   |
| Connection handling  |
| Compression          |
+----------------------+
     |
     v
 Transport

The application is still free to decide how it uses the protocol.


---

2. Design Goals

GSP has several main goals.

2.1 Security by default

Applications should not need to invent their own encryption protocol.

GSP uses authenticated encryption and provides a defined place for authentication and key management.

2.2 Low latency

GSP is intended for applications where unnecessary round trips and protocol overhead matter.

The initial project targets are:

Metric	Target

Additional latency	2–5 ms
Protocol overhead	5–10%
Bandwidth overhead	5–10%


These are engineering targets, not protocol guarantees.

2.3 Multiple streams

One GSP connection can carry several independent logical streams.

Connection
|
+-- Stream 0
+-- Stream 1
+-- Stream 2
+-- Stream 3

2.4 Different delivery semantics

Not every packet needs to be retransmitted.

GSP therefore supports both:

Reliable
Unreliable

2.5 Transport independence

GSP should not require a single underlying transport.

The initial transport profiles are:

GSP/UDP
GSP/TCP
GSP/QUIC

2.6 Extensibility

Features that do not belong in the core protocol can be implemented as extensions.


---

3. Problems GSP Attempts to Solve

3.1 TCP is a byte stream

TCP does not preserve application message boundaries.

An application has to create its own framing:

[length][message][length][message]

GSP defines packet and message framing at the protocol level.


---

3.2 UDP leaves too much to the application

UDP provides datagrams, but does not provide:

Reliable delivery

ACKs

Retransmission

Streams

Connection management

Replay protection

Encryption

Flow control


GSP provides these mechanisms where required.


---

3.3 One delivery model is not suitable for everything

A file transfer wants reliable delivery.

A real-time game state update may not.

For example:

Player position:

100
101
102
103
104

If 101 is lost while 104 has already arrived, retransmitting 101 may be useless.

GSP allows applications to choose the appropriate delivery mode.


---

3.4 Custom UDP protocols repeat the same work

A typical custom UDP protocol eventually becomes:

UDP
 |
 +-- Custom packet format
 +-- Sequence numbers
 +-- ACKs
 +-- Retransmission
 +-- Encryption
 +-- Replay protection
 +-- Streams
 +-- Flow control
 +-- Congestion control

GSP provides a common protocol foundation instead.


---

3.5 Multiple logical connections

Applications often need several independent communication channels.

GSP allows them to share one connection:

GSP Connection
|
+-- Authentication
+-- Chat
+-- Game state
+-- File transfer
+-- Telemetry


---

3.6 Connection migration

Mobile devices can change networks.

For example:

Wi-Fi
  |
  | GSP connection
  |
  +----------------+
                   |
              Mobile network

GSP can keep the logical connection alive while the network path changes, provided the migration is authenticated.


---

4. What GSP Is Not

GSP does not attempt to replace:

IP

DNS

HTTP

TCP

UDP

QUIC

TLS

WebSocket


GSP is another protocol option.

A web application may still use HTTP.

A simple application may still use TCP.

An application that needs QUIC directly may use QUIC directly.

GSP exists for applications that want its particular combination of features.


---

5. Terminology

Endpoint
A device or process participating in GSP.

Client
The endpoint initiating a connection.

Server
The endpoint accepting a connection.

Peer
The other endpoint of a GSP connection.

Connection
A logical GSP session.

Stream
A logical channel inside a connection.

Packet
The basic GSP transmission unit.

Frame
A structure contained inside a GSP packet.

Transport
The protocol carrying GSP packets.

Path
A network route between two endpoints.

Connection ID
An opaque identifier associated with a GSP connection.


---

6. Normative Language

The words below are used in their usual protocol specification sense:

MUST — required

MUST NOT — prohibited

SHOULD — recommended

SHOULD NOT — generally discouraged

MAY — optional


These terms are intentionally used in the specification so that implementations can distinguish requirements from recommendations.


---

7. Protocol Architecture

GSP is divided into several logical parts.

+------------------------------------------------+
|                  Application                   |
+------------------------------------------------+
|               GSP Application Layer            |
+------------------------------------------------+
|                   GSP Core                     |
|------------------------------------------------|
| Streams | Frames | Reliability | Flow Control |
+------------------------------------------------+
|                GSP Security                    |
|------------------------------------------------|
| Key Exchange | AEAD | Replay | Padding        |
+------------------------------------------------+
|               GSP Transport                    |
+------------------------------------------------+
| UDP / TCP / QUIC / Future Transports           |
+------------------------------------------------+

The protocol may also be used as a bridge between GSP and existing application protocols.


---

8. GSP Protocol Identifier

The protocol version is identified as:

GSP/1.0

The URI scheme is:

gsp://

Example:

gsp://example.com
gsp://example.com:9000
gsp://example.com/game
gsp://192.0.2.10:9000/api

The URI identifies the GSP service.

It does not by itself determine whether UDP, TCP, or QUIC is used.


---

9. GSP Connection

A GSP connection has a lifecycle.

IDLE
 |
 v
CONNECTING
 |
 v
HANDSHAKING
 |
 v
ESTABLISHED
 |
 +--------> CLOSING
 |             |
 |             v
 |           CLOSED
 |
 +--------> ERROR

An implementation MUST NOT accept normal application data before the connection reaches ESTABLISHED.


---

10. Connection IDs

Every GSP connection has a Connection ID.

The experimental GSP/1.0 format uses a 64-bit Connection ID.

+------------------------------------------------+
|                 Connection ID                  |
|                    64 bits                     |
+------------------------------------------------+

Example:

import secrets

connection_id = secrets.randbits(64)

Connection IDs are opaque.

They MUST NOT be treated as authentication credentials.

An implementation SHOULD generate unpredictable Connection IDs.


---

11. Connection Establishment

A basic handshake is:

Client                         Server
  |                              |
  | -------- HELLO ------------> |
  |                              |
  | <------- HELLO_ACK --------- |
  |                              |
  | ----- KEY_EXCHANGE --------> |
  |                              |
  | <---- KEY_EXCHANGE --------- |
  |                              |
  |       Secure session         |

The actual cryptographic exchange depends on the selected security profile.


---

12. HELLO

HELLO starts capability negotiation.

It can advertise:

GSP version

Supported cipher suites

Supported compression

Transport capabilities

Maximum packet size

Maximum streams

Extensions

Padding capabilities


Conceptual representation:

hello = {
    "version": "GSP/1.0",
    "ciphers": [
        "CHACHA20-POLY1305"
    ],
    "compression": [
        "none",
        "lz4"
    ],
    "extensions": [],
    "max_packet_size": 65535,
    "max_streams": 1024
}

This representation is illustrative.

It is not the binary wire format.


---

13. HELLO_ACK

HELLO_ACK confirms the parameters selected by the server.

The server MUST NOT select an algorithm that was not advertised by the client.

The negotiated parameters apply to the connection unless a later protocol mechanism changes them.


---

14. Packet Format

The experimental GSP/1.0 packet header is:

0                   1                   2                   3
+-------------------+-------------------+-------------------+
| Version |  Type   |        Connection ID (high)          |
+-------------------+-------------------+-------------------+
|              Connection ID (low)                        |
+-------------------+-------------------+-------------------+
|                 Packet Number                         |
+-------------------+-------------------+-------------------+
|                Payload Length                        |
+-------------------+-------------------+-------------------+
|                    Payload ...                        |
+---------------------------------------------------------+

The fields are:

Field	Size

Version	1 byte
Type	1 byte
Connection ID	8 bytes
Packet Number	4 bytes
Payload Length	4 bytes
Payload	Variable


Total fixed header size:

18 bytes

All multi-byte integers use network byte order.


---

15. Packet Types

Current packet types:

Value	Name	Description

0x01	HELLO	Start negotiation
0x02	HELLO_ACK	Accept negotiation
0x03	KEY_EXCHANGE	Key establishment
0x04	DATA	General data
0x05	STREAM_DATA	Stream data
0x06	ACK	Acknowledgement
0x07	PING	Keepalive
0x08	PONG	Keepalive response
0x09	CLOSE	Graceful shutdown
0x0A	RESET	Immediate shutdown
0x0B	ERROR	Protocol error


Unknown packet types MUST NOT crash an implementation.


---

16. Frames

A packet MAY contain one or more logical frames.

Conceptually:

+-------------------+
| Packet Header     |
+-------------------+
| Frame             |
+-------------------+
| Frame             |
+-------------------+
| Frame             |
+-------------------+

Frames allow control information and application data to share packets.

A future stable wire format will define the exact frame encoding.


---

17. Packet Numbers

Every protected packet has a packet number.

Packet numbers are used for:

ACKs

Loss detection

Replay protection

Ordering information

Cryptographic nonce construction


A packet number MUST NOT be reused under the same cryptographic key and nonce construction.

Simple prototype:

class PacketCounter:
    def __init__(self):
        self.value = 0

    def next(self):
        value = self.value
        self.value += 1
        return value


---

18. Packet Encoding

Minimal experimental encoder:

import struct

GSP_VERSION = 1
HEADER_FORMAT = "!BBQII"
HEADER_SIZE = struct.calcsize(HEADER_FORMAT)


def encode_packet(packet_type, connection_id, packet_number, payload):
    header = struct.pack(
        HEADER_FORMAT,
        GSP_VERSION,
        packet_type,
        connection_id,
        packet_number,
        len(payload),
    )

    return header + payload

Decoder:

def decode_packet(data):
    if len(data) < HEADER_SIZE:
        raise ValueError("packet too short")

    (
        version,
        packet_type,
        connection_id,
        packet_number,
        payload_length,
    ) = struct.unpack(
        HEADER_FORMAT,
        data[:HEADER_SIZE],
    )

    if version != GSP_VERSION:
        raise ValueError("unsupported GSP version")

    if len(data) != HEADER_SIZE + payload_length:
        raise ValueError("invalid payload length")

    return {
        "version": version,
        "type": packet_type,
        "connection_id": connection_id,
        "packet_number": packet_number,
        "payload": data[HEADER_SIZE:],
    }


---

19. Streams

Streams are independent logical channels inside a GSP connection.

Connection
|
+-- Stream 0
|    +-- Message
|    +-- Message
|
+-- Stream 1
|    +-- Message
|    +-- Message
|
+-- Stream 2
     +-- Message

A stream has:

Stream ID

Sequence number

Delivery mode

State

Flow-control window



---

20. Stream IDs

Stream IDs are unsigned integers.

The exact stream-ID allocation rules are implementation-defined in this experimental release, but implementations SHOULD reserve separate ranges for locally and remotely created streams.

Stream IDs MUST NOT be reused inside the same connection.


---

21. Stream States

A stream can be represented as:

IDLE
 |
 v
OPEN
 |
 +------> HALF-CLOSED
 |
 v
CLOSED

A stream error MAY terminate only the affected stream rather than the entire connection.


---

22. Reliable Streams

Reliable streams guarantee that accepted data is delivered according to the stream's ordering rules, subject to connection failure.

A minimal implementation can track pending data:

class ReliableStream:
    def __init__(self, stream_id):
        self.stream_id = stream_id
        self.sequence = 0
        self.pending = {}

    def send(self, data):
        sequence = self.sequence
        self.sequence += 1
        self.pending[sequence] = data
        return sequence

    def acknowledge(self, sequence):
        self.pending.pop(sequence, None)

A production implementation additionally needs:

Loss detection

Retransmission timers

Duplicate handling

Ordering

Receive windows

Memory limits



---

23. Unreliable Streams

Unreliable streams do not require retransmission of lost data.

They are useful for information that becomes obsolete quickly.

Examples:

Game state
Sensor readings
Telemetry
Position updates
Live statistics

An application SHOULD use reliable delivery when losing a message would make the application state incorrect.


---

24. Message Boundaries

GSP preserves application message boundaries.

For example:

Message A
Message B
Message C

is not automatically converted into an undifferentiated byte stream.

Applications therefore do not need to reinvent basic message framing.


---

25. ACK

Reliable delivery uses acknowledgements.

A simple conceptual ACK:

ACK:
100
101
102
105

A range-based representation can reduce overhead:

100-102
105

The final binary ACK frame format is part of the experimental wire-format work.


---

26. Loss Detection

Loss detection is required for reliable GSP over transports that do not already provide reliable delivery.

A packet can be considered lost when:

A sufficiently later packet is acknowledged

A retransmission timer expires

Another loss-detection mechanism detects it


A fixed timeout is useful for a prototype:

import time


class PendingPacket:
    def __init__(self, data):
        self.data = data
        self.sent_at = time.monotonic()


def expired(packet, timeout=0.5):
    return time.monotonic() - packet.sent_at > timeout

Production implementations SHOULD estimate RTT dynamically.


---

27. Retransmission

A lost reliable packet MAY be retransmitted.

The implementation SHOULD avoid blindly retransmitting every packet after a fixed timeout.

A proper implementation should track:

RTT
RTO
Packet loss
ACK delay
Reordering
Congestion state

A packet number MUST remain unique even when the application data is retransmitted.


---

28. Head-of-Line Blocking

GSP streams are intended to isolate independent application flows.

For example:

Stream 0:
A -> B -> C

Stream 1:
X -> Y -> Z

Loss on Stream 0 should not unnecessarily prevent Stream 1 from making progress.

The exact degree of isolation depends on the transport profile.

TCP inherently imposes ordering at the transport layer, while UDP and QUIC can provide more independent stream behavior.


---

29. Flow Control

Flow control prevents a sender from exhausting receiver resources.

There are two levels:

Connection
|
+-- Connection window
|
+-- Stream 0 window
+-- Stream 1 window
+-- Stream 2 window

Example:

MAX_CONNECTION_DATA = 16 * 1024 * 1024
MAX_STREAM_DATA = 4 * 1024 * 1024

These values are examples.

An implementation SHOULD make limits configurable.


---

30. Congestion Control

Congestion control is separate from flow control.

Flow control protects the receiver.

Congestion control protects the network.

For GSP/TCP and GSP/QUIC, the underlying transport already provides congestion control.

For GSP/UDP, GSP must provide an appropriate congestion-control mechanism.

A GSP/UDP implementation MUST NOT continuously transmit at an unlimited rate.


---

31. Security Architecture

Security is part of the GSP architecture.

Application
     |
     v
GSP
     |
     +-- Authentication
     +-- Key exchange
     +-- Key derivation
     +-- AEAD
     +-- Replay protection
     +-- Padding
     |
     v
Transport

The initial recommended AEAD cipher is:

ChaCha20-Poly1305


---

32. Key Exchange

GSP uses ephemeral session keys.

The purpose is to avoid using a long-term secret directly as the encryption key for application packets.

Conceptually:

Client                              Server
  |                                   |
  | Client ephemeral key              |
  |---------------------------------->|
  |                                   |
  | Server ephemeral key              |
  |<----------------------------------|
  |                                   |
  |       Key derivation              |
  |<--------------------------------->|
  |                                   |
  |       Session keys                |

A production implementation MUST use a standardized, reviewed key exchange construction.

GSP MUST NOT invent cryptographic primitives.


---

33. Ephemeral Keys

Each connection SHOULD use fresh ephemeral key material.

This provides forward-secrecy properties when combined with an appropriate authenticated key-exchange protocol.

Long-term identity keys, when used, MUST be kept separate from ephemeral traffic keys.


---

34. Key Derivation

The key exchange output MUST NOT be used directly as an application encryption key.

A KDF is used to derive separate keys and contexts.

Conceptually:

Shared Secret
     |
     v
+-----------+
|    KDF    |
+-----------+
   /     \
  v       v
Client    Server
Key       Key

Separate directions SHOULD use separate keys.


---

35. AEAD Encryption

The recommended cipher suite is:

ChaCha20-Poly1305

Authenticated encryption protects:

Confidentiality
+
Integrity
+
Authentication of ciphertext

Example:

from cryptography.hazmat.primitives.ciphers.aead import ChaCha20Poly1305

key = ChaCha20Poly1305.generate_key()
cipher = ChaCha20Poly1305(key)

nonce = packet_number.to_bytes(12, "big")

ciphertext = cipher.encrypt(
    nonce,
    plaintext,
    associated_data,
)

plaintext = cipher.decrypt(
    nonce,
    ciphertext,
    associated_data,
)

This example is only an API demonstration.

It does not define the complete GSP key schedule or nonce construction.


---

36. PSIV

GSP reserves a protocol mechanism referred to as PSIV for protecting packet-specific cryptographic context.

PSIV is associated with the packet's cryptographic state rather than being treated as an ordinary application field.

An implementation MUST ensure that the final PSIV construction never causes nonce reuse with the same AEAD key.

The exact PSIV wire representation remains experimental in GSP/1.0.


---

37. Nonces

Nonce construction is critical.

The same nonce MUST NOT be reused with the same AEAD key.

The experimental design associates packet numbers with nonce generation.

Conceptually:

Traffic Key
     +
Packet Number
     |
     v
Nonce
     |
     v
ChaCha20-Poly1305

A production implementation MUST follow the finalized GSP nonce construction rather than simply copying the example code above.


---

38. Replay Protection

A receiver MUST reject packets that are outside its accepted replay window or have already been processed.

A prototype can use:

class ReplayWindow:
    def __init__(self):
        self.seen = set()

    def accept(self, packet_number):
        if packet_number in self.seen:
            return False

        self.seen.add(packet_number)
        return True

A production implementation SHOULD use a bounded sliding window.


---

39. Dynamic Padding

GSP supports optional dynamic packet padding.

The purpose is to make packet sizes less predictable.

Without padding:

Message A -> 42 bytes
Message B -> 43 bytes
Message C -> 200 bytes

With padding:

Message A -> 256 bytes
Message B -> 256 bytes
Message C -> 256 bytes

Padding MUST NOT be interpreted as application data.

Padding SHOULD be applied before encryption:

Application data
       |
       v
Compression
       |
       v
Dynamic padding
       |
       v
Encryption
       |
       v
Packet

Padding increases bandwidth usage and SHOULD therefore be configurable.


---

40. Compression

GSP supports optional compression.

The initial recommended algorithm is:

LZ4

Processing order:

Application data
       |
       v
Compression
       |
       v
Padding
       |
       v
Encryption
       |
       v
Transport

Encrypted data MUST NOT be compressed.

Example:

import lz4.frame

compressed = lz4.frame.compress(data)
restored = lz4.frame.decompress(compressed)

Implementations MUST place limits on decompressed data.


---

41. Compression Attacks

Compression can introduce information leaks when attacker-controlled data is compressed together with secret information.

Applications MUST avoid combining sensitive secrets and attacker-controlled data in a compression context without considering this risk.

Compression MAY be disabled per connection or stream.


---

42. GSPMTP

GSPMTP is the message/transport framing component of GSP.

Its purpose is to provide a consistent representation of:

Packet
  |
  +-- Header
  |
  +-- Frame
  |
  +-- Frame
  |
  +-- Authentication tag

GSPMTP separates protocol framing from the underlying transport.

Conceptually:

GSP Application
      |
      v
     PTA
      |
      v
    GSPMTP
      |
      v
 Transport Profile

The GSPMTP format is designed so that the same logical GSP messages can be handled over different transports.


---

43. PTA

GSP uses a Protocol Transport/Application (PTA) architecture.

The purpose of PTA is to keep application semantics separate from transport-specific behavior.

Application
     |
     v
+------------+
|    PTA     |
+------------+
     |
     v
+------------+
|   GSPMTP   |
+------------+
     |
     v
+------------+
| Transport  |
+------------+

This allows:

Application
     |
     v
GSP
     |
 +---+---+---+
 |   |   |   |
UDP TCP QUIC ...

without requiring the application protocol to be rewritten for every transport.


---

44. Transport Profiles

44.1 GSP/UDP

GSP/UDP provides the greatest control.

GSP is responsible for features that UDP does not provide:

ACK
Loss detection
Retransmission
Flow control
Congestion control
Replay protection

This profile is intended for applications where low latency and control matter.


---

44.2 GSP/TCP

GSP/TCP uses TCP as the underlying reliable transport.

TCP already handles:

Reliability

Ordering

Retransmission

Congestion control


GSP/TCP therefore SHOULD avoid implementing duplicate transport-level reliability where TCP already provides it.

GSP framing is still required because TCP is a byte stream.

GSP
 |
 v
GSPMTP
 |
 v
TCP
 |
 v
IP


---

44.3 GSP/QUIC

GSP/QUIC uses QUIC as the underlying transport.

QUIC already provides:

Encryption

Multiplexed streams

Reliability

Congestion control

Connection migration


GSP/QUIC SHOULD reuse those mechanisms rather than duplicating them.

Application
     |
     v
GSP
     |
     v
QUIC
     |
     v
UDP


---

45. Why GSP Can Run Over QUIC

GSP is not intended to compete with QUIC at exactly the same layer.

QUIC can be treated as a transport for GSP.

This lets an application use a common GSP application interface while selecting QUIC underneath it.

Application
     |
     v
GSP API
     |
     +-------- GSP/UDP
     |
     +-------- GSP/TCP
     |
     +-------- GSP/QUIC


---

46. Connection Migration

GSP connections MAY migrate to a different network path.

Example:

Path A:
Wi-Fi
   |
   v
GSP connection
   |
   X

Path B:
Cellular
   |
   v
Same logical GSP connection

Migration MUST be authenticated.

A peer MUST NOT accept an arbitrary packet from a new network path merely because it contains a valid Connection ID.

Path validation is required.


---

47. Path Validation

A simple path-validation exchange is:

Peer A                         Peer B
  |                              |
  | -------- PATH_CHALLENGE ---> |
  |                              |
  | <-------- PATH_RESPONSE ---- |
  |                              |
  |        Path validated        |

The challenge value MUST be unpredictable.


---

48. Keepalive

GSP defines:

PING
PONG

Example:

Client                    Server
  |                         |
  | -------- PING --------> |
  |                         |
  | <------- PONG --------- |

Keepalive packets can be used to:

Detect dead peers

Keep NAT mappings alive

Check a path


Implementations SHOULD avoid unnecessarily frequent keepalives.


---

49. Connection Close

Normal shutdown uses:

CLOSE

Immediate termination uses:

RESET

Example:

CLOSE = 0x09
RESET = 0x0A

connection.close(
    code=0,
    reason="normal shutdown"
)

After a connection is closed, application data MUST NOT be accepted.


---

50. Error Handling

Current error codes:

Code	Name

0x0001	PROTOCOL_ERROR
0x0002	INVALID_PACKET
0x0003	AUTHENTICATION_FAILED
0x0004	UNSUPPORTED_VERSION
0x0005	UNSUPPORTED_EXTENSION
0x0006	CONNECTION_TIMEOUT
0x0007	STREAM_ERROR
0x0008	FLOW_CONTROL_ERROR
0x0009	CRYPTO_ERROR
0x000A	DECOMPRESSION_ERROR
0x000B	PROTOCOL_VIOLATION
0x000C	RESOURCE_LIMIT


Malformed network input MUST NOT crash an implementation.


---

51. Version Negotiation

An endpoint MUST identify the GSP version it supports.

Example:

GSP/1.0

If a peer does not support the requested version, it SHOULD respond with a version negotiation or an appropriate error.

Experimental versions are not guaranteed to be wire-compatible with future versions.


---

52. Extensions

GSP has an extension mechanism.

Example:

EXT_LZ4 = 0x0001
EXT_MIGRATION = 0x0002
EXT_PADDING = 0x0003

An extension can define:

New frame types

New algorithms

New connection features

New stream behavior

New transport behavior


Unknown optional extensions SHOULD be safely ignored when possible.

Extensions MUST NOT silently change the meaning of existing core fields.


---

53. Extension Registry

A future stable version of GSP will maintain an extension registry.

Each extension should document:

Extension ID
Name
Version
Purpose
Required capabilities
Wire format
Security considerations
Compatibility

Experimental extensions SHOULD use explicitly experimental identifiers.


---

54. Resource Limits

Network input is untrusted.

Implementations MUST impose limits.

Example:

MAX_PACKET_SIZE = 64 * 1024
MAX_STREAMS = 1024
MAX_CONNECTIONS = 10_000
MAX_DECOMPRESSED_SIZE = 16 * 1024 * 1024
MAX_FRAME_SIZE = 64 * 1024

These are example defaults.

Applications MAY choose smaller limits.

An implementation MUST NOT allocate unlimited memory based on a peer's advertised value.


---

55. Rate Limiting

Servers SHOULD limit expensive operations during connection establishment.

Examples:

Too many HELLO packets
Too many invalid packets
Too many failed authentication attempts
Too many new connections

This helps prevent resource-exhaustion attacks.


---

56. Authentication

Encryption alone does not necessarily identify the peer.

GSP can support:

Anonymous encryption
Certificate-based identity
Pre-shared keys
Application authentication
Public-key identity

The selected authentication mechanism MUST be explicitly negotiated or defined by the application profile.


---

57. Security Model

GSP assumes that the network is hostile.

An attacker may:

Observe packets

Modify packets

Drop packets

Delay packets

Reorder packets

Duplicate packets

Inject packets

Attempt connection exhaustion


GSP security mechanisms are intended to protect against unauthorized modification, injection, and replay of protected packets.

GSP cannot prevent an attacker from physically dropping packets or disconnecting a network.


---

58. Traffic Analysis

Encryption does not hide everything.

An observer may still learn information such as:

Timing

Approximate packet sizes

Packet frequency

Connection duration

Network endpoints


Dynamic padding can reduce some packet-size information.

It cannot provide complete traffic-analysis resistance.


---

59. Minimal UDP Server

A minimal experimental server:

import socket
import struct

VERSION = 1

HELLO = 0x01
HELLO_ACK = 0x02

HEADER = "!BBQII"
HEADER_SIZE = struct.calcsize(HEADER)


def encode(packet_type, connection_id, number, payload):
    return struct.pack(
        HEADER,
        VERSION,
        packet_type,
        connection_id,
        number,
        len(payload),
    ) + payload


def decode(data):
    if len(data) < HEADER_SIZE:
        raise ValueError("packet too short")

    version, packet_type, connection_id, number, length = (
        struct.unpack(
            HEADER,
            data[:HEADER_SIZE],
        )
    )

    if version != VERSION:
        raise ValueError("unsupported version")

    if len(data) != HEADER_SIZE + length:
        raise ValueError("invalid packet length")

    return {
        "type": packet_type,
        "connection_id": connection_id,
        "number": number,
        "payload": data[HEADER_SIZE:],
    }


server = socket.socket(
    socket.AF_INET,
    socket.SOCK_DGRAM,
)

server.bind(("0.0.0.0", 9000))

print("GSP listening on UDP/9000")


while True:
    data, address = server.recvfrom(65535)

    try:
        packet = decode(data)
    except ValueError:
        continue

    if packet["type"] == HELLO:
        response = encode(
            HELLO_ACK,
            packet["connection_id"],
            0,
            b"GSP/1.0",
        )

        server.sendto(
            response,
            address,
        )

This is only a protocol demonstration.

It is NOT a production-secure GSP server.


---

60. Minimal UDP Client

import socket
import secrets
import struct


VERSION = 1
HELLO = 0x01

HEADER = "!BBQII"


def encode(packet_type, connection_id, number, payload):
    return struct.pack(
        HEADER,
        VERSION,
        packet_type,
        connection_id,
        number,
        len(payload),
    ) + payload


sock = socket.socket(
    socket.AF_INET,
    socket.SOCK_DGRAM,
)

connection_id = secrets.randbits(64)

packet = encode(
    HELLO,
    connection_id,
    0,
    b"GSP/1.0",
)

sock.sendto(
    packet,
    ("127.0.0.1", 9000),
)

data, address = sock.recvfrom(65535)

print("Server response:", data)


---

61. Minimal GSP API

A reference implementation can expose an API similar to:

connection = gsp.connect(
    "gsp://example.com:9000"
)

stream = connection.open_stream(
    reliable=True
)

stream.send(b"hello")

data = stream.receive()

stream.close()

connection.close()

For unreliable communication:

stream = connection.open_stream(
    reliable=False
)

stream.send(b"real-time state")

The API is not part of the wire protocol.

Different implementations may expose different programming interfaces.


---

62. GSP-HTTP Translator

GSP can be used as a transport for applications that still expose HTTP-like semantics.

The optional GSP-HTTP Translator maps HTTP operations onto GSP streams.

Conceptually:

HTTP Client
     |
     v
GSP-HTTP Translator
     |
     v
GSP
     |
     v
Network

Example:

HTTP:

GET /api/user
       |
       v
HTTP Response


GSP:

Stream 4
   |
   +-- REQUEST
   |
   +-- RESPONSE

The translator is optional.

GSP itself does not require HTTP.


---

63. Application Data

Application data SHOULD be carried in stream frames rather than creating a new packet type for every application feature.

For example:

Stream 5
|
+-- Application message
+-- Application message
+-- Application message

This keeps the GSP core independent of the application.


---

64. Packet Processing Order

For encrypted and compressed data, the recommended processing order is:

Application
    |
    v
Message framing
    |
    v
Compression
    |
    v
Dynamic padding
    |
    v
AEAD encryption
    |
    v
GSP packet
    |
    v
Transport

Receiving reverses the process:

Transport
    |
    v
GSP packet
    |
    v
AEAD verification/decryption
    |
    v
Remove padding
    |
    v
Decompression
    |
    v
Message processing

Authentication MUST occur before application data is trusted.


---

65. MTU and Packet Size

GSP implementations SHOULD avoid fragmentation whenever possible.

For UDP, the implementation SHOULD discover an appropriate path MTU or use a conservative packet size.

A packet larger than the supported transport or path size MUST NOT simply be transmitted indefinitely and assumed to work.

Example:

MAX_UDP_PAYLOAD = 1200

This is a conservative example, not a mandatory GSP value.


---

66. Fragmentation

Large application messages MAY be split into multiple frames or packets.

A fragmented message MUST contain enough information for the receiver to:

Identify the message

Identify its stream

Identify the fragment

Determine whether the message is complete


Implementations MUST enforce limits on incomplete messages.

A peer must not be able to make the receiver store unlimited partial data.


---

67. Ordering

Reliable streams normally provide ordered delivery.

Unreliable streams MAY deliver messages out of order depending on the transport profile.

Applications that require ordering SHOULD use reliable ordered streams.

Applications that only care about the newest state SHOULD consider unreliable delivery.


---

68. Stream Priorities

Implementations MAY assign priorities to streams.

Example:

Priority 0: Control
Priority 1: Interactive data
Priority 2: Game state
Priority 3: Bulk transfer

A bulk transfer SHOULD NOT starve latency-sensitive streams.

Stream priorities are advisory unless explicitly negotiated by an extension.


---

69. Session Resumption

Future GSP versions MAY support session resumption.

The intended purpose is to avoid repeating a complete handshake when a previously authenticated session can safely be resumed.

A resumption mechanism MUST NOT weaken forward secrecy or allow replay of application data.


---

70. 0-RTT

A future GSP profile MAY support 0-RTT application data.

0-RTT data has replay considerations.

Applications MUST NOT send replay-sensitive operations as unauthenticated 0-RTT data.

This feature is not considered stable in GSP/1.0.


---

71. NAT Traversal

GSP does not require a specific NAT traversal mechanism.

Future extensions MAY define:

STUN integration

Relay support

Hole punching

ICE-based negotiation


These are outside the GSP core.


---

72. IPv4 and IPv6

GSP implementations SHOULD support both:

IPv4
IPv6

The protocol itself does not depend on either address family.


---

73. TCP Framing

When GSP is carried over TCP, the receiver cannot assume that one recv() call contains one GSP packet.

For example:

send():

[GSP packet]

may arrive as:

recv():
[GSP hea]

recv():
[der + payload]

or:

recv():
[GSP packet A][GSP packet B]

The implementation MUST maintain a TCP receive buffer and parse packets according to the GSP length field.


---

74. QUIC Mapping

GSP/QUIC SHOULD map GSP streams to QUIC streams where possible.

A possible mapping is:

GSP Stream 0 <-> QUIC Stream 0
GSP Stream 1 <-> QUIC Stream 1
GSP Stream 2 <-> QUIC Stream 2

GSP should not implement a second reliability system on top of a reliable QUIC stream unless an application profile specifically requires it.


---

75. UDP Reliability Architecture

For GSP/UDP, a simplified reliable path looks like:

Application
    |
    v
Stream
    |
    v
Sequence Number
    |
    v
Packet
    |
    v
UDP
    |
    v
Network
    |
    v
ACK
    |
    v
Loss detection
    |
    v
Retransmission

The congestion controller MUST take retransmissions into account.


---

76. GSP Overhead

GSP attempts to keep its fixed overhead small.

The current experimental header is:

18 bytes

Additional overhead can come from:

Frame headers

AEAD authentication tags

Padding

Compression metadata

Stream identifiers

ACK information


Applications sending very small messages should consider batching where latency requirements permit it.


---

77. Batching

Multiple small frames MAY be placed in a single packet.

+-------------------+
| Packet Header     |
+-------------------+
| Frame A           |
+-------------------+
| Frame B           |
+-------------------+
| Frame C           |
+-------------------+

Batching reduces per-packet overhead.

It SHOULD NOT introduce unacceptable latency.


---

78. Zero-Copy Implementations

High-performance implementations MAY avoid copying payload data between every protocol layer.

A possible design:

Receive buffer
     |
     +---- Header view
     |
     +---- Frame view
     |
     +---- Payload view

This is an implementation optimization and does not change the wire format.


---

79. Error Isolation

A malformed stream frame SHOULD terminate only the affected stream when it is safe to do so.

A malformed cryptographic packet or fundamental protocol violation MAY require closing the entire connection.

For example:

Invalid application message
        |
        v
Stream error

while:

Invalid authenticated packet
        |
        v
Connection error


---

80. Security Checklist

A production implementation MUST:

Validate packet lengths.

Validate frame lengths.

Validate packet types.

Validate versions.

Validate stream IDs.

Authenticate protected packets.

Prevent AEAD nonce reuse.

Detect replayed packets.

Limit memory allocation.

Limit decompression output.

Limit incomplete messages.

Protect connection establishment.

Rate-limit expensive operations.

Validate migrated paths.

Avoid trusting unauthenticated input.



---

81. Implementation Requirements

A conforming GSP implementation MUST:

1. Reject malformed packets.


2. Enforce maximum packet sizes.


3. Enforce negotiated stream limits.


4. Correctly handle packet numbers.


5. Correctly handle connection states.


6. Never reuse an AEAD nonce with the same key.


7. Reject invalid authentication tags.


8. Protect against packet replay.


9. Avoid unbounded memory allocation.


10. Respect flow-control limits.



A secure implementation SHOULD additionally implement:

Dynamic congestion control

RTT estimation

Connection migration

Key rotation

Rate limiting

Resource quotas



---

82. Reference Constants

The following constants are used by the experimental specification:

GSP_VERSION = 1

HELLO = 0x01
HELLO_ACK = 0x02
KEY_EXCHANGE = 0x03
DATA = 0x04
STREAM_DATA = 0x05
ACK = 0x06
PING = 0x07
PONG = 0x08
CLOSE = 0x09
RESET = 0x0A
ERROR = 0x0B

Example limits:

MAX_PACKET_SIZE = 64 * 1024
MAX_STREAMS = 1024
MAX_FRAME_SIZE = 64 * 1024
MAX_DECOMPRESSED_SIZE = 16 * 1024 * 1024

These values may change before GSP/1.0 becomes stable.


---

83. Recommended Repository Structure

A reference implementation can use:

gsp/
├── README.md
├── SPEC.md
├── LICENSE
├── SECURITY.md
├── CONTRIBUTING.md
├── CHANGELOG.md
│
├── gsp/
│   ├── __init__.py
│   ├── packet.py
│   ├── frame.py
│   ├── connection.py
│   ├── stream.py
│   ├── crypto.py
│   ├── padding.py
│   ├── compression.py
│   ├── reliability.py
│   ├── congestion.py
│   ├── migration.py
│   │
│   └── transport/
│       ├── __init__.py
│       ├── udp.py
│       ├── tcp.py
│       └── quic.py
│
├── examples/
│   ├── client.py
│   ├── server.py
│   └── echo.py
│
└── tests/
    ├── test_packet.py
    ├── test_frame.py
    ├── test_crypto.py
    ├── test_stream.py
    ├── test_reliability.py
    └── test_transport.py


---

84. Testing

Implementations SHOULD have tests for:

Packet encoding
Packet decoding
Malformed packets
Version negotiation
Connection establishment
Stream creation
Stream closing
ACK processing
Packet loss
Packet reordering
Duplicate packets
Replay attacks
Encryption
Invalid authentication tags
Compression
Decompression limits
Padding
Flow control
Connection migration
Transport differences


---

85. Interoperability

Two GSP implementations are interoperable when they agree on:

Version
Packet format
Frame format
Handshake
Cryptographic profile
Stream behavior
Transport profile
Extensions

The reference implementation should include interoperability tests against other implementations whenever possible.


---

86. Example Connection

A complete conceptual connection:

Client                                      Server
  |                                           |
  |              HELLO                        |
  |------------------------------------------>|
  |                                           |
  |              HELLO_ACK                    |
  |<------------------------------------------|
  |                                           |
  |           KEY_EXCHANGE                    |
  |<----------------------------------------->|
  |                                           |
  |========= Secure GSP Session ==============|
  |                                           |
  | STREAM_DATA: Stream 1                     |
  |------------------------------------------>|
  |                                           |
  | ACK                                        |
  |<------------------------------------------|
  |                                           |
  | STREAM_DATA: Stream 2                     |
  |------------------------------------------>|
  |                                           |
  | PING                                       |
  |------------------------------------------>|
  |                                           |
  | PONG                                       |
  |<------------------------------------------|
  |                                           |
  | CLOSE                                      |
  |------------------------------------------>|
  |                                           |


---

87. Example Application Layout

A real application might use GSP like this:

Application
|
+-- Control Stream
|     reliable
|
+-- Authentication Stream
|     reliable
|
+-- Real-time State Stream
|     unreliable
|
+-- Chat Stream
|     reliable
|
+-- File Stream
      reliable

All of these can share one GSP connection.


---

88. Design Summary

The GSP architecture can be summarized as:

Application
                              |
                              v
                     +----------------+
                     |      PTA       |
                     +----------------+
                              |
                              v
                     +----------------+
                     |     GSPMTP     |
                     +----------------+
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
             Streams      Reliability    Security
                |             |             |
                +-------------+-------------+
                              |
                              v
                     +----------------+
                     |   Transport    |
                     +----------------+
                       /      |      \
                      /       |       \
                    UDP      TCP     QUIC


---

89. Current Limitations

GSP/1.0 is experimental.

The following parts are still subject to change:

Final packet header

Frame encoding

ACK range encoding

Complete handshake format

Key derivation

PSIV construction

Nonce construction

Key rotation

Authentication profiles

UDP congestion control

Migration frame format

Extension registry

0-RTT

Session resumption

NAT traversal

GSPMTP details

PTA details


This document describes the current direction of the protocol, not a guarantee that every experimental mechanism is frozen.


---

90. Future Work

Planned areas include:

Stable binary wire format

Complete cryptographic handshake

Formal key schedule

Key rotation

Better congestion control

More efficient ACK encoding

Connection resumption

0-RTT

NAT traversal

Multipath support

Better path discovery

Formal extension registry

Reference implementations

Cross-language implementations

Interoperability test suite

Fuzz testing

Performance benchmarks

Security audit



---

91. Experimental Status

GSP/1.0 is not a finished Internet standard.

The specification is being developed alongside implementations.

Breaking changes are expected.

A project using GSP should pin the protocol version instead of assuming future versions will remain wire-compatible.


---

92. Versioning Policy

Experimental versions use:

GSP/1.x

A future stable protocol may use a separate versioning policy.

Changes that alter the wire format MUST result in a new protocol version or an explicitly negotiated extension.


---

93. Security Considerations

GSP handles untrusted network input.

Implementations must assume that every packet can be malicious.

Particular attention should be given to:

Packet parsing

Integer overflow

Length validation

Memory allocation

Compression bombs

Replay attacks

Nonce reuse

Key reuse

Authentication failures

Connection exhaustion

Stream exhaustion

Path migration

Timing side channels

Traffic analysis


Cryptographic primitives should come from established cryptographic libraries.

GSP MUST NOT implement cryptographic primitives from scratch.


---

94. Minimal GSP Stack

The smallest useful GSP implementation can start with:

GSP
 |
 +-- Packet parser
 +-- Connection ID
 +-- Packet numbers
 +-- HELLO
 +-- HELLO_ACK
 +-- DATA
 +-- UDP transport

A more complete implementation adds:

+-- Streams
 +-- ACKs
 +-- Reliability
 +-- Flow control
 +-- Congestion control
 +-- Cryptography
 +-- Replay protection
 +-- Compression
 +-- Padding
 +-- Migration

This makes it possible to develop GSP incrementally instead of having to implement every feature before testing the protocol.


---

95. Example Minimal Server Architecture

+------------------+
                    |   Application    |
                    +------------------+
                             |
                             v
                    +------------------+
                    | GSP Connection   |
                    +------------------+
                             |
             +---------------+---------------+
             |               |               |
             v               v               v
          Stream 0        Stream 1        Stream 2
             |               |               |
             +---------------+---------------+
                             |
                             v
                    +------------------+
                    |     GSPMTP       |
                    +------------------+
                             |
                             v
                    +------------------+
                    |    GSP/UDP       |
                    +------------------+
                             |
                             v
                           UDP


---

96. Example Configuration

A hypothetical GSP server configuration:

gsp:
  version: "1.0"

  transport:
    udp:
      enabled: true
      port: 9000

    tcp:
      enabled: true
      port: 9000

    quic:
      enabled: false

  security:
    cipher: "CHACHA20-POLY1305"
    replay_protection: true
    dynamic_padding: true

  compression:
    enabled: true
    algorithm: "lz4"

  limits:
    max_packet_size: 65536
    max_streams: 1024
    max_connections: 10000

This configuration format is illustrative and is not part of the GSP wire protocol.


---

97. GSP in One Sentence

> GSP is a secure, extensible communication protocol that gives applications streams, message framing, delivery control, and transport flexibility without requiring them to rebuild the same networking layer from scratch.




---

98. Final Note

GSP is intentionally being developed as an experimental protocol.

The goal is not to claim that every existing protocol is obsolete.

The goal is to build a practical protocol that can sit between an application and different transports while keeping security, streams, delivery behavior, and connection management in one place.

Application
     |
     v
    GSP
     |
     +---------- UDP
     |
     +---------- TCP
     |
     +---------- QUIC
     |
     +---------- Future transports

Globalized Secure Protocol

GSP/1.0 — Experimental

gsp://