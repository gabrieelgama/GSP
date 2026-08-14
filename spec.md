Globalized Secure Protocol (GSP)

«GSP Core Protocol Specification
Version 1.0 — Experimental
"draft-gsp-core-00"»

""Status: Experimental" (https://img.shields.io/badge/status-experimental-orange)" (https://github.com/)
""Specification" (https://img.shields.io/badge/specification-GSP%2F1.0-blue)" (https://github.com/)

---

Abstract

The Globalized Secure Protocol (GSP) is a general-purpose communication protocol designed for secure, low-latency, and bandwidth-efficient communication.

GSP provides a common framework for:

- Secure communication
- Packet framing
- Connection management
- Multiplexed streams
- Reliable and unreliable delivery
- Acknowledgements
- Flow control
- Congestion control
- Optional compression
- Connection migration
- Protocol extensions

GSP is designed to operate independently of a single underlying transport. Transport-specific profiles may define how GSP operates over UDP, TCP, QUIC, or other compatible transports.

The protocol uses the following URI scheme:

gsp://

---

Table of Contents

- "1. Introduction" (#1-introduction)
- "2. Conventions and Terminology" (#2-conventions-and-terminology)
- "3. Protocol Architecture" (#3-protocol-architecture)
- "4. Connection Model" (#4-connection-model)
- "5. Packet Format" (#5-packet-format)
- "6. Connection Establishment" (#6-connection-establishment)
- "7. Security" (#7-security)
- "8. Streams" (#8-streams)
- "9. Reliability" (#9-reliability)
- "10. Flow Control" (#10-flow-control)
- "11. Transport Profiles" (#11-transport-profiles)
- "12. Compression" (#12-compression)
- "13. Error Handling" (#13-error-handling)
- "14. Extensions" (#14-extensions)
- "15. Security Considerations" (#15-security-considerations)
- "16. Implementation Requirements" (#16-implementation-requirements)
- "17. IANA Considerations" (#17-iana-considerations)
- "18. Future Work" (#18-future-work)

---

1. Introduction

The Globalized Secure Protocol (GSP) is a network protocol designed for applications that require secure communication without unnecessary protocol overhead.

GSP combines transport-oriented functionality with application-oriented messaging.

The protocol is intended for applications such as:

- Real-time communication
- Multiplayer games
- APIs
- RPC systems
- File transfer
- Streaming
- Distributed applications
- Embedded systems
- IoT systems

GSP does not attempt to replace every existing networking protocol. Instead, it provides a common protocol layer that applications can use when they need control over security, delivery semantics, streams, and transport behavior.

---

2. Conventions and Terminology

The key words MUST, MUST NOT, REQUIRED, SHOULD, SHOULD NOT, and MAY in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Endpoint

A device or process participating in a GSP connection.

Client

The endpoint that initiates a connection.

Server

The endpoint that accepts a connection.

Connection

A logical communication session between two endpoints.

Stream

An independently identified flow of application data inside a connection.

Packet

The basic unit transmitted by GSP.

Frame

A logical data structure contained inside a packet.

Transport

The underlying protocol carrying GSP packets.

---

3. Protocol Architecture

GSP uses a layered architecture.

┌───────────────────────────────────────┐
│              Application              │
├───────────────────────────────────────┤
│          GSP Application Layer        │
│          Messages / RPC / Data        │
├───────────────────────────────────────┤
│             GSP Core Layer            │
│       Packets / Frames / Streams      │
├───────────────────────────────────────┤
│            GSP Security               │
│       Encryption / Authentication     │
├───────────────────────────────────────┤
│           GSP Transport               │
│ Reliability / ACK / Flow Control      │
├───────────────────────────────────────┤
│          Transport Profile            │
│          UDP / TCP / QUIC             │
└───────────────────────────────────────┘

The layers are logical. An implementation MAY combine multiple layers internally.

---

4. Connection Model

A GSP connection is a logical session between two endpoints.

A single connection MAY contain multiple streams.

GSP Connection
│
├── Stream 0 — Control
├── Stream 1 — Application
├── Stream 2 — File Transfer
├── Stream 3 — Real-Time Data
└── Stream 4 — RPC

Stream failures SHOULD remain isolated from the rest of the connection whenever possible.

---

5. URI Scheme

GSP defines the following URI syntax:

gsp://host[:port][/path][?query]

Examples:

gsp://example.com
gsp://example.com:443
gsp://example.com/api
gsp://192.0.2.10:9000/game

The URI identifies a GSP service.

The actual underlying transport is selected by the implementation and the applicable transport profile.

---

6. Packet Format

A GSP packet consists of a header followed by one or more frames.

Conceptually:

+-------------------------+
| GSP Header              |
+-------------------------+
| Extension Fields        |
+-------------------------+
| Frame(s)                |
+-------------------------+
| Authentication Tag      |
+-------------------------+

The exact binary representation is defined by the wire-format specification.

---

7. Packet Header

The GSP header contains the following logical fields:

Field| Description
Version| GSP protocol version
Flags| Packet-level flags
Type| Packet type
Connection ID| Identifies the connection
Packet Number| Identifies the packet
Payload Length| Payload size
Extensions| Optional features

Multi-byte integers MUST use network byte order unless otherwise specified.

---

8. Packet Types

GSP/1.0 defines the following packet types:

Type| Name| Purpose
"0x01"| "HELLO"| Start connection negotiation
"0x02"| "HELLO_ACK"| Accept connection negotiation
"0x03"| "KEY_EXCHANGE"| Establish session keys
"0x04"| "DATA"| Carry connection data
"0x05"| "STREAM_DATA"| Carry stream data
"0x06"| "ACK"| Acknowledge packets
"0x07"| "PING"| Test connectivity
"0x08"| "PONG"| Respond to "PING"
"0x09"| "CLOSE"| Gracefully close connection
"0x0A"| "RESET"| Immediately terminate connection
"0x0B"| "ERROR"| Report protocol errors

Unassigned packet types are reserved.

---

9. Connection Establishment

A typical GSP handshake is:

Client                         Server
  │                              │
  │ -------- HELLO ------------> │
  │                              │
  │ <------- HELLO_ACK --------- │
  │                              │
  │ ----- KEY_EXCHANGE --------> │
  │ <---- KEY_EXCHANGE --------- │
  │                              │
  │ ===== Secure Session ======= │

The handshake negotiates:

- GSP version
- Cryptographic capabilities
- Compression
- Transport capabilities
- Protocol extensions
- Connection parameters

Application data MUST NOT be treated as authenticated GSP application data before the required security handshake has completed.

---

10. Security

Security is a core requirement of GSP.

Normal application data MUST use authenticated encryption.

The initial recommended cipher suite is:

ChaCha20-Poly1305

Implementations MUST NOT reuse an AEAD nonce with the same encryption key.

Implementations SHOULD use ephemeral key exchange mechanisms that provide forward secrecy.

---

11. Session Keys

Each GSP connection SHOULD use connection-specific cryptographic keys.

Long-lived connections MAY perform key updates.

A static symmetric key MUST NOT be reused indefinitely for unrelated connections.

---

12. Replay Protection

GSP uses packet numbers and connection-specific cryptographic state to detect replayed packets.

An implementation MUST reject packets that violate the applicable replay policy.

---

13. Packet Padding

GSP MAY use dynamic packet padding.

Padding can make packet sizes less predictable.

Example:

Payload: 143 bytes
Padded:  160 bytes

Padding does not replace encryption and MUST NOT be considered a complete traffic-analysis defense.

---

14. Streams

GSP supports multiplexed streams.

Each stream has a unique Stream ID within its connection.

Connection #42
│
├── Stream 0
├── Stream 1
├── Stream 2
├── Stream 3
└── Stream 4

Applications MAY assign different purposes to different streams.

---

15. Reliable Delivery

A reliable stream provides:

- Delivery confirmation
- Retransmission
- Optional ordered delivery

Reliable delivery is suitable for:

- File transfers
- API requests
- RPC
- Authentication
- Configuration
- Important application events

---

16. Unreliable Delivery

GSP MAY provide unreliable messages.

Unreliable messages do not require retransmission.

This mode is useful for rapidly changing information such as:

- Player positions
- Telemetry
- Voice packets
- Sensor updates
- Temporary state

An implementation MUST preserve the reliability semantics selected by the application.

---

17. Ordering

A stream MAY be ordered or unordered.

Ordered stream:

1 → 2 → 3 → 4

Unordered stream:

1
3
2
4

Applications SHOULD use unordered delivery when old messages become useless after newer messages arrive.

---

18. Acknowledgements

Reliable packets are acknowledged using "ACK".

An ACK MAY acknowledge multiple packet ranges.

Example:

ACK

100-105
108
110-112

This reduces acknowledgement overhead.

---

19. Loss Detection

GSP implementations SHOULD use packet acknowledgements and timing information to detect loss.

Implementations SHOULD account for packet reordering before declaring a packet lost.

A packet SHOULD NOT be retransmitted solely because a later packet arrived first.

---

20. Flow Control

GSP supports both connection-level and stream-level flow control.

Example:

Connection Window: 16 MiB

Stream 1: 4 MiB
Stream 2: 8 MiB
Stream 3: 4 MiB

A sender MUST NOT exceed the currently advertised flow-control limit.

---

21. Congestion Control

When GSP is responsible for packet transmission over a congestion-sensitive network, congestion control MUST be implemented.

Transport profiles MAY delegate congestion control to the underlying transport.

Flow control and congestion control serve different purposes:

Flow Control      → protects the receiver
Congestion Control → protects the network

---

22. Compression

GSP supports optional compression.

The initial recommended compression algorithm is:

LZ4

The processing order MUST be:

Application Data
      ↓
Compression
      ↓
Encryption
      ↓
Transmission

Encrypted data MUST NOT be compressed.

Applications MAY disable compression for already-compressed data.

---

23. Connection Migration

A GSP connection MAY migrate between network paths.

For example:

Wi-Fi
  ↓
GSP Connection
  ↓
Mobile Network

Migration MUST be authenticated to prevent connection hijacking.

---

24. Keepalive

GSP defines:

PING
PONG

A peer MAY send "PING".

The receiving peer SHOULD respond with "PONG".

Applications SHOULD avoid unnecessary keepalive traffic.

---

25. Connection Termination

A graceful shutdown uses:

CLOSE

An immediate shutdown uses:

RESET

After a connection reaches the "CLOSED" state, application data MUST NOT be accepted for that connection.

---

26. Error Codes

Initial GSP error codes are:

Code| Name
"0x0001"| "PROTOCOL_ERROR"
"0x0002"| "INVALID_PACKET"
"0x0003"| "AUTHENTICATION_FAILED"
"0x0004"| "UNSUPPORTED_VERSION"
"0x0005"| "UNSUPPORTED_EXTENSION"
"0x0006"| "CONNECTION_TIMEOUT"
"0x0007"| "STREAM_ERROR"
"0x0008"| "FLOW_CONTROL_ERROR"
"0x0009"| "CRYPTO_ERROR"
"0x000A"| "DECOMPRESSION_ERROR"
"0x000B"| "PROTOCOL_VIOLATION"
"0x000C"| "RESOURCE_LIMIT"

Unknown error codes MUST be handled safely.

---

27. Transport Profiles

GSP is transport-independent.

The initial transport profiles are:

- GSP/UDP
- GSP/TCP
- GSP/QUIC

Each profile defines how GSP maps onto the underlying transport.

                 GSP
                  │
       ┌──────────┼──────────┐
       │          │          │
      UDP        TCP        QUIC

---

28. GSP over UDP

When operating directly over UDP, GSP is responsible for the transport features required by its selected delivery mode.

This can include:

- Reliability
- Retransmission
- Packet numbering
- Loss detection
- Flow control
- Congestion control

---

29. GSP over TCP

When operating over TCP, TCP provides:

- Reliable delivery
- Ordering
- Retransmission
- Congestion control

GSP MUST NOT unnecessarily duplicate TCP's transport functionality.

GSP framing remains necessary to separate application messages.

---

30. GSP over QUIC

When operating over QUIC, GSP SHOULD reuse functionality already provided by QUIC where appropriate.

A GSP/QUIC implementation MUST avoid creating conflicting or redundant transport mechanisms.

---

31. Application Payloads

GSP does not define a mandatory application serialization format.

Applications MAY use:

- JSON
- CBOR
- MessagePack
- Protocol Buffers
- Custom binary formats

The application protocol is responsible for interpreting the payload.

---

32. Extensions

GSP supports protocol extensions.

An extension MAY define:

- New packet types
- New frames
- New cryptographic suites
- New compression methods
- Authentication mechanisms
- Transport features
- Application features

Extensions MUST have unique identifiers.

Unknown optional extensions MAY be ignored.

Unknown mandatory extensions MUST cause the corresponding feature or connection to be rejected.

---

33. Security Considerations

Implementations MUST:

1. Validate packet lengths before parsing.
2. Validate extension lengths.
3. Authenticate encrypted data before exposing plaintext.
4. Prevent AEAD nonce reuse.
5. Implement replay protection.
6. Limit memory consumption.
7. Limit decompression resources.
8. Reject malformed packets safely.
9. Validate all externally supplied identifiers.
10. Avoid trusting unauthenticated application data.

Network-facing servers SHOULD additionally implement rate limiting and connection limits.

---

34. Denial-of-Service Protection

GSP implementations SHOULD assume that all network input is potentially malicious.

Servers SHOULD limit:

- Handshake rate
- Number of simultaneous connections
- Number of streams
- Maximum packet size
- Incomplete message memory
- Decompression resources
- Authentication attempts

An implementation SHOULD reject malformed packets as early as possible.

---

35. Implementation Requirements

A minimal GSP/1.0 implementation MUST support:

- Packet parsing
- Version negotiation
- Connection establishment
- Authenticated encryption
- Packet numbering
- Basic data transfer
- ACK processing
- Connection termination
- Error handling

The following features MAY be implemented separately:

- LZ4 compression
- Multiplexed streams
- Unreliable delivery
- Connection migration
- RPC
- HTTP translation
- Additional transport profiles

---

36. Conformance

An implementation claiming support for GSP/1.0 SHOULD document:

GSP Version:
Transport Profiles:
Packet Types:
Cryptographic Suites:
Compression:
Stream Modes:
Extensions:
Maximum Packet Size:

Implementations MUST NOT claim support for features that they do not actually implement.

---

37. IANA Considerations

This specification currently requires no IANA actions.

If GSP is submitted to a standards organization in the future, registries SHOULD be created for:

- Packet types
- Error codes
- Extension identifiers
- Cryptographic suites
- Compression algorithms
- Transport profiles

---

38. Performance Targets

GSP was designed around the following engineering targets:

Metric| Target
Additional latency| 2–5 ms
Protocol overhead| 5–10%
Bandwidth overhead| 5–10%

These values are targets, not guarantees.

Actual performance depends on the network, hardware, operating system, transport, packet size, implementation, congestion, encryption, and compression.

---

39. Example

A GSP application may expose:

gsp://api.example.com/v1

After establishing a secure connection, it may create multiple streams:

Stream 0 → Control
Stream 1 → Authentication
Stream 2 → API
Stream 3 → Notifications
Stream 4 → File Transfer

All streams can share the same GSP connection and security context.

---

40. Reference Connection Flow

Client                                  Server
  │                                       │
  │               HELLO                   │
  ├──────────────────────────────────────>│
  │                                       │
  │             HELLO_ACK                 │
  │<──────────────────────────────────────┤
  │                                       │
  │           KEY_EXCHANGE                │
  ├──────────────────────────────────────>│
  │                                       │
  │           KEY_EXCHANGE                │
  │<──────────────────────────────────────┤
  │                                       │
  │        Secure GSP Connection          │
  │=======================================│
  │                                       │
  │           STREAM_DATA                 │
  ├──────────────────────────────────────>│
  │                                       │
  │              ACK                      │
  │<──────────────────────────────────────┤
  │                                       │
  │              CLOSE                    │
  ├──────────────────────────────────────>│
  │                                       │

---

41. Future Work

Future revisions of the GSP specification may define:

- A complete binary wire format
- Standardized key exchange
- 0-RTT connection establishment
- Session resumption
- NAT traversal
- Standardized connection migration
- Priority scheduling
- Traffic shaping
- Additional cryptographic suites
- Additional transport profiles
- Formal extension registries
- Interoperability test vectors
- Reference implementations

Features listed in this section are not part of GSP/1.0 unless explicitly defined elsewhere.

---

42. Status

This specification is currently Experimental.

The protocol is expected to evolve as implementations are developed and interoperability testing is performed.

Breaking changes may occur before a stable specification is released.

---

License

Unless otherwise stated, the GSP specification is released under the license included in the root of this repository.

---

Globalized Secure Protocol (GSP)
"gsp://"
GSP/1.0 — Experimental