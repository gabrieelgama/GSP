

# Globalized Secure Protocol (GSP)

**GSP Core Protocol Specification**  
**Version:** 1.0  
**Status:** Experimental  
![Status](https://img.shields.io/badge/status-experimental-orange)
![Specification](https://img.shields.io/badge/specification-GSP%2F1.0-blue)

---

## Abstract

The Globalized Secure Protocol (GSP) is a general-purpose network protocol
for applications that need secure communication, multiplexed streams, and
control over how data is delivered.

GSP can run over different transports, including UDP, TCP, and QUIC.

The main idea is simple: applications should not have to build their own
packet format, acknowledgement system, encryption layer, stream handling,
and connection management every time they need something beyond a basic
TCP connection.

GSP provides those pieces as part of one protocol.

This specification describes the experimental GSP/1.0 protocol.

> GSP/1.0 is experimental. The wire format and some protocol details may
> change before the first stable release.

---

## 1. Why GSP?

There are already a lot of networking protocols. GSP is not meant to
replace all of them.

The problem is what happens when an application needs a combination of
features that does not fit neatly into one existing protocol.

For example:

- TCP gives you reliable ordered data, but only as a byte stream.
- UDP gives you datagrams, but leaves reliability and most transport
  mechanisms to the application.
- HTTP is great for request/response applications, but is not a general
  purpose transport for every kind of application.
- WebSocket provides persistent communication, but is tied to an HTTP
  handshake model.
- QUIC already solves many transport problems, but an application may still
  want its own protocol and message model.

A developer using UDP might end up writing something like this:

```text
Application
    |
    +-- Packet format
    +-- Packet IDs
    +-- ACKs
    +-- Retransmission
    +-- Replay protection
    +-- Encryption
    +-- Streams
    +-- Flow control
    +-- Compression
    |
    v
   UDP

GSP tries to provide a common implementation for these pieces.

It does not mean that every application has to use every GSP feature.


---

2. Goals

GSP has a few practical goals.

Security

Encrypted and authenticated communication should be the normal case, not something that applications have to bolt on afterwards.

Low overhead

GSP should avoid adding unnecessary bytes, round trips, and processing.

Multiple streams

A single connection should be able to carry multiple independent streams.

Different delivery modes

Not everything needs retransmission.

A file transfer and a player-position update have very different requirements.

GSP therefore supports both reliable and unreliable data.

Transport flexibility

GSP should not be permanently tied to one transport.

Extensibility

New features should be possible without changing the entire protocol.


---

3. Terminology

The following terms are used throughout this document.

Endpoint
A device or process participating in a GSP connection.

Client
The endpoint that starts a connection.

Server
The endpoint accepting a connection.

Peer
The other endpoint of a connection.

Connection
A logical GSP session between two endpoints.

Stream
A logical data channel inside a connection.

Packet
A GSP packet sent over the selected transport.

Frame
A structure carried inside a packet.

Transport
The protocol carrying GSP packets, such as UDP, TCP, or QUIC.


---

4. Protocol Overview

A simplified GSP connection looks like this:

Application
     |
     v
+------------+
| GSP Stream |
+------------+
     |
     v
+------------+
| GSP Packet |
+------------+
     |
     v
+------------+
|   Security |
+------------+
     |
     v
+------------+
|  Transport |
+------------+
     |
     +---- UDP
     +---- TCP
     +---- QUIC

A connection can contain multiple streams:

Connection
|
+-- Stream 0
+-- Stream 1
+-- Stream 2
+-- Stream 3
+-- ...

The exact use of each stream is decided by the application.


---

5. URI Scheme

GSP uses the gsp:// URI scheme.

Basic form:

gsp://host[:port][/path][?query]

Examples:

gsp://example.com
gsp://example.com:9000
gsp://example.com/game
gsp://192.0.2.10:9000/api

The URI identifies the GSP service.

It does not by itself select the underlying transport.


---

6. Connection Lifecycle

A normal connection goes through these states:

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
 +-------> CLOSING
 |            |
 |            v
 +--------> CLOSED
 |
 +-------> ERROR

A minimal implementation can represent this as:

from enum import Enum


class ConnectionState(Enum):
    IDLE = 0
    CONNECTING = 1
    HANDSHAKING = 2
    ESTABLISHED = 3
    CLOSING = 4
    CLOSED = 5
    ERROR = 6


---

7. Handshake

The initial handshake is intentionally small.

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

HELLO is used to advertise supported protocol features.

Depending on the implementation, it can contain:

GSP version

Supported cipher suites

Compression algorithms

Extensions

Maximum packet size

Stream limits


Example application representation:

hello = {
    "version": 1,
    "ciphers": [
        "CHACHA20-POLY1305"
    ],
    "compression": [
        "none",
        "lz4"
    ],
    "extensions": []
}

This dictionary is only an implementation example. It is not the wire encoding.


---

8. Packet Format

The initial experimental packet header is 18 bytes.

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
|                       Payload ...                     |
+---------------------------------------------------------+

For the experimental format, the fields are:

Field	Size

Version	1 byte
Type	1 byte
Connection ID	8 bytes
Packet Number	4 bytes
Payload Length	4 bytes
Payload	Variable


Multi-byte integers use network byte order (big-endian).


---

9. Packet Types

The current packet type values are:

Value	Name	Purpose

0x01	HELLO	Start negotiation
0x02	HELLO_ACK	Accept negotiation
0x03	KEY_EXCHANGE	Key exchange
0x04	DATA	General data
0x05	STREAM_DATA	Stream data
0x06	ACK	Acknowledgement
0x07	PING	Keepalive
0x08	PONG	Keepalive response
0x09	CLOSE	Graceful close
0x0A	RESET	Immediate close
0x0B	ERROR	Protocol error


Packet types not understood by an implementation MUST NOT cause a crash.


---

10. Minimal Packet Encoder

The following is enough to start experimenting with the packet format.

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

Example:

packet = encode_packet(
    packet_type=0x04,
    connection_id=12345,
    packet_number=0,
    payload=b"hello",
)

print(decode_packet(packet))


---

11. Connection IDs

Every connection has a Connection ID.

The ID is an opaque 64-bit value in the current experimental format.

Example:

import secrets

connection_id = secrets.randbits(64)

A Connection ID is not an authentication mechanism.

Implementations should generate IDs that are difficult to predict.


---

12. Packet Numbers

Packets have monotonically increasing packet numbers.

class PacketCounter:
    def __init__(self):
        self.value = 0

    def next(self):
        number = self.value
        self.value += 1
        return number

Packet numbers are useful for:

ACKs

Loss detection

Replay protection

Cryptographic nonce construction


A packet number MUST NOT be reused with the same cryptographic context.


---

13. Security

GSP uses authenticated encryption for protected application data.

The initial recommended cipher suite is:

ChaCha20-Poly1305

A production implementation must also define:

Key exchange

Key derivation

Nonce construction

Key rotation

Authentication

Replay protection


The protocol should never rely on the Connection ID as proof of identity.

Example using Python's cryptography package:

from cryptography.hazmat.primitives.ciphers.aead import ChaCha20Poly1305


key = ChaCha20Poly1305.generate_key()
cipher = ChaCha20Poly1305(key)

nonce = packet_number.to_bytes(12, "big")

encrypted = cipher.encrypt(
    nonce,
    plaintext,
    associated_data,
)

decrypted = cipher.decrypt(
    nonce,
    encrypted,
    associated_data,
)

The example is deliberately small. It does not define the complete GSP key schedule.


---

14. Replay Protection

A receiver must not blindly process the same authenticated packet twice.

A simple prototype can use:

class ReplayWindow:
    def __init__(self):
        self.seen = set()

    def accept(self, packet_number):
        if packet_number in self.seen:
            return False

        self.seen.add(packet_number)
        return True

A real implementation should use a bounded sliding window.


---

15. Streams

Streams allow multiple logical data flows to share one connection.

GSP Connection
|
+-- Stream 0
|    |
|    +-- data
|
+-- Stream 1
|    |
|    +-- data
|
+-- Stream 2
     |
     +-- data

A minimal stream object:

class Stream:
    def __init__(self, stream_id):
        self.id = stream_id
        self.sequence = 0

    def next_sequence(self):
        value = self.sequence
        self.sequence += 1
        return value


---

16. Reliable Data

Reliable streams keep track of data that has not been acknowledged.

class ReliableStream(Stream):
    def __init__(self, stream_id):
        super().__init__(stream_id)
        self.pending = {}

    def send(self, data):
        sequence = self.next_sequence()
        self.pending[sequence] = data
        return sequence

    def acknowledge(self, sequence):
        self.pending.pop(sequence, None)

This is only the basic mechanism.

A real implementation also needs:

Timers

Retransmission

Duplicate detection

Ordering

Loss detection



---

17. Unreliable Data

Some data is not worth retransmitting.

Consider a game sending:

Player position:
X=100
X=101
X=102
X=103

If the X=101 packet is lost, sending it again after X=103 usually does not help.

GSP therefore allows an application to use unreliable delivery where appropriate.

Typical examples:

Player positions

Sensor readings

Telemetry

Temporary state

Real-time updates


Reliable delivery is better for things such as:

File transfers

Authentication

RPC

Configuration

Important events



---

18. ACKs

Reliable packets are acknowledged using ACK.

A simple ACK could contain:

100
101
102
105

A more efficient representation uses ranges:

100-102
105

The exact binary ACK representation is part of the experimental wire format and may change.

Minimal implementation:

class Ack:
    def __init__(self, packets):
        self.packets = packets


---

19. Loss Detection

For transports such as UDP, GSP needs a way to detect lost packets.

A basic prototype can start with a timeout:

import time


class PendingPacket:
    def __init__(self, data):
        self.data = data
        self.sent_at = time.monotonic()


def is_lost(packet, timeout=0.5):
    return (
        time.monotonic() - packet.sent_at
        > timeout
    )

Production implementations should use a proper RTT estimator instead of a fixed timeout.

Packet reordering must also be taken into account.


---

20. Flow Control

Flow control prevents a sender from overwhelming a receiver.

There are two levels:

Connection
|
+-- Connection limit
|
+-- Stream 0 limit
+-- Stream 1 limit
+-- Stream 2 limit

Example limits:

MAX_CONNECTION_DATA = 16 * 1024 * 1024
MAX_STREAM_DATA = 4 * 1024 * 1024

A sender must respect the limits advertised by its peer.


---

21. Congestion Control

Flow control and congestion control are not the same thing.

Flow control asks:

> "Can the receiver handle this?"



Congestion control asks:

> "Can the network handle this?"



For GSP over TCP or QUIC, the transport already handles congestion control.

For direct UDP operation, GSP needs its own congestion-control mechanism.


---

22. Compression

GSP supports optional compression.

The initial recommended algorithm is:

LZ4

The order is:

Application data
       |
       v
Compression
       |
       v
Encryption
       |
       v
Packet

Do not compress encrypted data.

Example:

import lz4.frame

compressed = lz4.frame.compress(data)
data = lz4.frame.decompress(compressed)

Implementations must put a reasonable limit on decompressed data.


---

23. Keepalive

GSP provides PING and PONG.

Client                    Server
  |                         |
  | -------- PING --------> |
  |                         |
  | <------- PONG --------- |

Minimal handler:

PING = 0x07
PONG = 0x08


def handle_packet(packet):
    if packet["type"] == PING:
        return PONG

Keepalive should not be sent more frequently than necessary.


---

24. Connection Migration

A GSP connection may move between network paths.

For example:

Wi-Fi
  |
  |  GSP connection
  |
  +----------------+
                   |
              Mobile network

This is especially useful on devices that switch between Wi-Fi and cellular networks.

Migration must be authenticated.

The connection ID alone must not be treated as proof that a new network path belongs to the original peer.


---

25. Closing a Connection

Normal shutdown:

CLOSE

Immediate shutdown:

RESET

Example:

CLOSE = 0x09
RESET = 0x0A


connection.send(
    packet_type=CLOSE,
    payload=b"normal shutdown",
)

After a connection reaches CLOSED, no new application data should be accepted on that connection.


---

26. Error Codes

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


Malformed input must never be allowed to crash the server.


---

27. Transport Profiles

GSP can be carried by different transports.

Current profiles:

GSP/UDP
GSP/TCP
GSP/QUIC

UDP

Gives GSP the most control.

The GSP implementation may need to handle:

ACKs

Retransmission

Loss detection

Congestion control

Flow control


TCP

TCP already handles:

Reliability

Ordering

Retransmission

Congestion control


GSP therefore should not duplicate those mechanisms unnecessarily.

TCP is still a byte stream, so GSP framing is required.

QUIC

QUIC already provides:

Encryption

Streams

Reliability

Congestion control

Connection migration


GSP/QUIC should reuse these features where possible.


---

28. Why Not Just Use QUIC?

QUIC is one of the transports GSP can use.

GSP is aimed at a different layer of the problem.

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

An application can use GSP over QUIC without giving up QUIC's transport features.

For applications that want direct control over UDP, the GSP/UDP profile can provide a different implementation path.


---

29. Extensions

GSP has an extension mechanism for features that do not belong in the core.

Example:

EXT_LZ4 = 0x0001
EXT_MIGRATION = 0x0002

Possible extensions include:

New compression algorithms

Authentication methods

New packet types

Transport features

Application features


Extensions should be documented before being used in interoperable implementations.


---

30. Resource Limits

Network input is untrusted.

A GSP implementation should always have limits.

Example:

MAX_PACKET_SIZE = 64 * 1024
MAX_STREAMS = 1024
MAX_CONNECTIONS = 10_000
MAX_DECOMPRESSED_SIZE = 16 * 1024 * 1024

These are example values, not mandatory values for every implementation.

Do not allocate unlimited memory based on values received from the network.


---

31. Minimal UDP Server

The following is a small example of a GSP server.

It is intended for testing the protocol, not for production use.

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

server.bind(
    ("0.0.0.0", 9000)
)

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


---

32. Minimal UDP Client

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

33. Security Checklist

A production GSP implementation must:

Validate packet lengths.

Validate packet types.

Validate protocol versions.

Validate stream IDs.

Authenticate encrypted data.

Prevent nonce reuse.

Detect replayed packets.

Limit memory usage.

Limit decompression output.

Handle malformed packets safely.

Avoid trusting unauthenticated data.

Protect connection establishment against abuse.

Apply appropriate rate limits.


The examples in this document are intentionally incomplete.

They demonstrate the protocol structure, not a finished secure networking library.


---

34. Performance Targets

The original GSP design targets are:

Metric	Target

Additional latency	2–5 ms
Protocol overhead	5–10%
Bandwidth overhead	5–10%


These are targets, not guarantees.

Actual results depend heavily on the transport, network, hardware, implementation, packet size, encryption, compression, and workload.

Benchmarks should be published together with the environment in which they were measured.


---

35. Compatibility

An implementation should identify the exact GSP version it supports.

Example:

GSP/1.0
Transport: UDP
Encryption: ChaCha20-Poly1305
Compression: LZ4
Streams: Supported
Migration: Supported

Experimental versions should not assume wire compatibility with future versions.


---

36. Repository Layout

A reference implementation can be organized like this:

gsp/
├── README.md
├── SPEC.md
├── LICENSE
├── SECURITY.md
├── CONTRIBUTING.md
│
├── gsp/
│   ├── __init__.py
│   ├── packet.py
│   ├── connection.py
│   ├── stream.py
│   ├── crypto.py
│   ├── ack.py
│   ├── compression.py
│   │
│   └── transport/
│       ├── udp.py
│       ├── tcp.py
│       └── quic.py
│
├── examples/
│   ├── client.py
│   └── server.py
│
└── tests/
    ├── test_packet.py
    ├── test_crypto.py
    ├── test_stream.py
    └── test_transport.py


---

37. Future Work

The following areas are still being worked on:

Final binary wire format

Complete handshake format

Key derivation

Key rotation

UDP congestion control

Complete ACK range encoding

Connection migration details

Session resumption

0-RTT

NAT traversal

Interoperability tests

Reference implementation

Extension registry


Nothing in this section should be considered implemented unless it is defined elsewhere in the specification.


---

38. Experimental Status

GSP/1.0 is experimental.

The project is expected to change while implementations are being built and tested.

In particular, the following parts are not considered frozen:

Packet format

Handshake

Cryptographic key schedule

ACK encoding

Stream format

Extension identifiers

UDP transport behavior


Implementations should therefore expect breaking changes.


---

39. License

See LICENSE for licensing information.


---

Globalized Secure Protocol

GSP/1.0 — Experimental

gsp://