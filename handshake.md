GSP Handshake Protocol Specification

Globalized Secure Protocol (GSP)
Handshake Protocol Specification
Version: Handshake 1.1 (FOAREVAMP) - (FIRST OF ALL REVAMP)
status: Experimental / Draft
URI Scheme: "gsp://"

---

1. Abstract

The GSP Handshake establishes a cryptographically protected GSP session between two peers.

The handshake provides:

- Protocol version negotiation
- Capability negotiation
- Cryptographic suite negotiation
- Key exchange
- Session key derivation
- Peer authentication
- Transcript integrity
- Downgrade protection
- Replay protection
- Forward secrecy
- Key confirmation
- Session identification
- Optional compression negotiation
- Optional session resumption
- Optional key updates

GSP 1.1 uses an optimized handshake design in which the Initiator includes its ephemeral key exchange public key in the initial "HELLO" message and the Responder returns its ephemeral public key in "HELLO_ACK".

This eliminates the separate "KEY_EXCHANGE" round trip used by the original handshake design.

---

2. Security Model

A GSP handshake MUST establish a secure session before normal application data is delivered.

A successful authenticated session provides:

Confidentiality
Integrity
Authentication
Forward Secrecy
Replay Resistance
Downgrade Resistance
Key Separation
Session Binding

The exact authentication guarantees depend on the selected authentication mode.

Anonymous mode does not provide authenticated peer identity.

---

3. Normative Language

The following keywords are normative:

Keyword| Meaning
MUST| Required
MUST NOT| Prohibited
REQUIRED| Same as MUST
SHOULD| Recommended
SHOULD NOT| Discouraged
MAY| Optional

---

4. Terminology

Term| Definition
Initiator| Peer that starts the handshake
Responder| Peer that receives the initial handshake
Peer| Either endpoint
Session| Established GSP connection
CID| Connection Identifier
SID| Session Identifier
KEX| Key Exchange
AEAD| Authenticated Encryption with Associated Data
PSK| Pre-Shared Key
KDF| Key Derivation Function
IV| Initialization Vector
PSIV| Protocol Session Initialization Vector
Transcript| Ordered canonical representation of handshake messages
RTT| Round Trip Time
Rekey| Replacement of traffic keys

---

5. Handshake Versions

This specification defines:

GSP/1.1

Implementations MAY support earlier versions.

A GSP implementation MUST NOT silently downgrade to an older version.

The selected version MUST be cryptographically bound to the handshake transcript.

---

6. Handshake Objectives

A successful handshake MUST establish:

1. A mutually supported protocol version.
2. A mutually supported cryptographic suite.
3. A mutually supported key-exchange algorithm.
4. A mutually supported authentication method.
5. Negotiated transport parameters.
6. Fresh ephemeral key material.
7. A shared secret.
8. Derived traffic keys.
9. Authentication state where required.
10. A verified handshake transcript.
11. Key confirmation.
12. A unique session context.

---

7. Optimized Handshake

The original GSP handshake required:

HELLO -> HELLO_ACK
KEY_EXCHANGE -> KEY_EXCHANGE_ACK
AUTH -> AUTH_ACK
FINISH -> FINISH_ACK

This resulted in four sequential request/response exchanges.

GSP 1.1 moves the ephemeral key exchange into the initial negotiation.

The optimized handshake is:

Initiator                                      Responder

HELLO
+ X25519 public key
---------------------------------------------->

                         HELLO_ACK
                         + X25519 public key
                         <----------------------

Derive shared secret
Derive handshake keys

FINISH
+ authentication
+ optional encrypted DATA
---------------------------------------------->

                         FINISH_ACK
                         <----------------------

                 SESSION ESTABLISHED

The separate "KEY_EXCHANGE" and "KEY_EXCHANGE_ACK" messages are therefore NOT required when using the GSP 1.1 integrated KEX profile.

---

8. RTT Characteristics

The optimized handshake allows:

HELLO       -> HELLO_ACK

to establish enough cryptographic material for the Initiator to derive traffic keys.

The Initiator can then send:

FINISH + DATA

in the next flight.

Therefore the first application DATA can travel after approximately:

1 RTT

from the beginning of the connection.

The handshake itself may still require the Responder's "FINISH_ACK" before the session is considered fully established.

---

9. 1-RTT vs 0-RTT

GSP 1.1 1-RTT MUST NOT be confused with 0-RTT.

1-RTT

The Initiator receives:

HELLO_ACK

before transmitting encrypted application DATA.

0-RTT

The Initiator transmits encrypted application DATA before receiving the Responder's first response.

0-RTT requires a previously established resumption secret or equivalent mechanism.

0-RTT data is replay-sensitive and MUST NOT be enabled for arbitrary application operations.

---

10. Handshake Messages

GSP 1.1 defines:

Type| Name
"0x01"| HELLO
"0x02"| HELLO_ACK
"0x03"| AUTH
"0x04"| AUTH_ACK
"0x05"| FINISH
"0x06"| FINISH_ACK
"0x07"| ALERT
"0x08"| RETRY
"0x09"| CLOSE
"0x0A"| KEY_UPDATE
"0x0B"| RESUME
"0x0C"| RESUME_ACK

The former "KEY_EXCHANGE" messages are retained only as compatibility messages for profiles that explicitly require them.

---

11. HELLO

Direction

Initiator -> Responder

"HELLO" is the first handshake message.

It advertises:

- Supported GSP versions
- Supported cipher suites
- Supported KEX algorithms
- Supported authentication methods
- Supported compression algorithms
- Capabilities
- Maximum frame size
- Maximum streams
- Random value
- Connection ID
- Ephemeral KEX public key
- Extensions

---

12. HELLO Structure

Logical representation:

HELLO {
    supported_versions[]
    minimum_version

    random
    connection_id

    cipher_suites[]
    key_exchange[]
    authentication[]
    compression[]

    capabilities

    max_frame_size
    max_streams

    ephemeral_key

    extensions[]
}

The actual binary representation is defined by the GSP canonical serialization rules.

---

13. HELLO Random

The Initiator MUST generate a fresh cryptographically secure random value.

Recommended size:

32 bytes

The value MUST NOT be reused for independent handshakes.

---

14. HELLO_ACK

Direction

Responder -> Initiator

The Responder selects the parameters to be used for the session.

Logical structure:

HELLO_ACK {
    selected_version

    random
    connection_id

    selected_cipher
    selected_key_exchange
    selected_authentication
    selected_compression

    capabilities

    max_frame_size
    max_streams

    ephemeral_key

    extensions[]
}

The Responder MUST generate fresh ephemeral KEX material before sending "HELLO_ACK".

---

15. Version Negotiation

The Initiator advertises supported versions.

Example:

supported_versions = [
    1.1,
    1.0
]

The Responder selects one mutually supported version.

If no compatible version exists:

VERSION_UNSUPPORTED

MUST be returned.

The selected version MUST be included in the transcript.

---

16. Cryptographic Suite Negotiation

GSP cryptographic suites identify the algorithms used by the session.

Recommended initial profile:

GSP-CHACHA20-POLY1305-X25519

The recommended profile uses:

KEX:
    X25519

Hash:
    SHA-256

KDF:
    HKDF-SHA-256

AEAD:
    ChaCha20-Poly1305

GSP SHOULD use established cryptographic primitives.

GSP MUST NOT require applications to implement cryptographic primitives themselves.

---

17. KEX Negotiation

The default GSP 1.1 KEX is:

X25519

The Initiator generates:

initiator_private_key
initiator_public_key

The Responder generates:

responder_private_key
responder_public_key

Private keys MUST remain local.

Only public keys are transmitted.

---

18. Integrated X25519 Exchange

The Initiator places:

initiator_public_key

inside "HELLO".

The Responder places:

responder_public_key

inside "HELLO_ACK".

After receiving "HELLO_ACK", both peers can independently derive:

shared_secret

without another network exchange.

---

19. Shared Secret

For X25519:

shared_secret =
    X25519(
        initiator_private_key,
        responder_public_key
    )

The Responder independently calculates:

shared_secret =
    X25519(
        responder_private_key,
        initiator_public_key
    )

The results MUST be identical.

The shared secret MUST NEVER be transmitted.

---

20. Invalid KEX

The handshake MUST fail if:

- The public key has an invalid length.
- The public key is malformed.
- The selected KEX is unsupported.
- The KEX operation fails.
- The resulting shared secret is invalid according to the selected KEX.

The connection MUST enter the "FAILED" state.

---

21. Authentication Modes

GSP supports:

ANONYMOUS
PSK
PUBLIC_KEY
CERTIFICATE

Authentication is negotiated during "HELLO".

---

22. Anonymous Authentication

Anonymous mode provides cryptographic protection against passive observers but does not authenticate the identity of the peer.

It SHOULD NOT be used for security-sensitive applications.

---

23. PSK Authentication

PSK authentication uses a previously shared secret.

The PSK MUST NOT be transmitted.

Authentication MUST be bound to:

protocol version
negotiated parameters
random values
ephemeral public keys
transcript
session context

A raw password MUST NOT be used directly as a cryptographic PSK.

---

24. Public-Key Authentication

Public-key authentication allows a peer to prove possession of a private signing key.

The authentication signature MUST cover the current handshake context.

The signature MUST NOT be transferable to another handshake.

---

25. Certificate Authentication

Certificate mode MAY use a certificate chain.

Certificate validation is governed by the applicable GSP authentication profile.

Certificate authentication MUST verify:

- Certificate validity
- Signature chain
- Intended identity
- Key usage
- Expiration
- Revocation policy where applicable

---

26. Authentication Binding

Authentication MUST be bound to the current handshake.

The authentication context SHOULD include:

"GSP/1.1"
role
selected_version
selected_cipher
selected_kex
selected_authentication
initiator_random
responder_random
initiator_public_key
responder_public_key
transcript_hash

---

27. AUTH

For authentication profiles requiring an explicit authentication message:

AUTH {
    authentication_method
    identity
    credential
    signature_or_mac
}

The credential MUST be authenticated against the handshake transcript.

---

28. AUTH_ACK

The Responder MAY send:

AUTH_ACK {
    status
    identity
    credential
    signature_or_mac
}

If authentication succeeds:

status = SUCCESS

Otherwise:

AUTHENTICATION_FAILED

MUST be generated.

---

29. Integrated Authentication

GSP 1.1 MAY integrate authentication directly into "FINISH".

In the optimized profile:

FINISH {
    authentication_data
    verify_data
}

may replace the separate:

AUTH
AUTH_ACK

exchange.

The selected authentication profile MUST define which format is used.

---

30. Transcript

The transcript is the canonical sequence of handshake messages.

Conceptually:

T =
    Encode(HELLO)
    ||
    Encode(HELLO_ACK)
    ||
    Encode(AUTH)
    ||
    Encode(AUTH_ACK)
    ||
    Encode(FINISH)

Messages not used by the selected profile are omitted.

---

31. Transcript Domain Separation

The transcript MUST use a GSP-specific context.

Conceptually:

T =
    "GSP-HANDSHAKE"
    ||
    version
    ||
    encoded_messages

This prevents cross-protocol interpretation.

---

32. Transcript Hash

The transcript hash is:

transcript_hash =
    SHA-256(T)

The hash MUST operate on canonical wire representations.

The transcript MUST NOT be calculated from language-level objects or compiler memory layouts.

---

33. Canonical Binary Encoding

All GSP handshake messages MUST have a deterministic binary representation.

The wire format MUST NOT depend on:

compiler
CPU architecture
struct padding
ABI
pointer size
native endianness
programming language
memory alignment

---

34. Integer Encoding

All multi-byte integers MUST use:

Big-Endian

also known as:

Network Byte Order

This includes:

uint16
uint32
uint64

---

35. Explicit Integer Sizes

The wire specification MUST use explicit integer widths.

The following are forbidden in wire definitions:

int
long
size_t
unsigned long
pointer

Instead:

uint8
uint16
uint32
uint64

MUST be used.

---

36. Fixed-Length Fields

Cryptographic fields MUST have exact sizes.

Examples:

X25519 public key:
    32 bytes

SHA-256:
    32 bytes

ChaCha20-Poly1305 authentication tag:
    16 bytes

Incorrect lengths MUST cause a parsing failure.

---

37. Variable-Length Fields

Variable-length fields MUST use explicit lengths.

Conceptually:

+----------+----------------+
| Length   | Value          |
+----------+----------------+

The length itself MUST have a defined width and byte order.

Example:

uint16 length

means:

2-byte Big-Endian length

followed by exactly "length" bytes.

---

38. No Compiler Padding

Wire serialization MUST NOT use:

sizeof(struct)

or equivalent native memory serialization.

Every field MUST be explicitly encoded.

---

39. No Implicit Terminators

Binary strings and byte arrays MUST NOT require a trailing NUL byte.

The canonical representation is:

length + bytes

unless a specific field explicitly defines another format.

---

40. Boolean Encoding

Boolean values MUST be:

0x00 = false
0x01 = true

Other values MUST be rejected.

---

41. Enumeration Encoding

Algorithm identifiers and other enumerations MUST use explicitly assigned numeric IDs.

Example:

Cipher:

0x0001 = ChaCha20-Poly1305
0x0002 = AES-256-GCM

Unknown mandatory values MUST cause handshake failure.

---

42. Optional Fields

Optional fields MUST have an unambiguous representation.

An absent field and an empty field MUST only be considered equivalent if explicitly defined by that message.

---

43. Extension Encoding

Recommended extension structure:

extension {
    type
    flags
    length
    value
}

Recommended wire types:

type:
    uint16

flags:
    uint16

length:
    uint32

All values use Big-Endian.

---

44. Extension Ordering

Extensions MUST appear in ascending numeric order unless an extension specification explicitly defines another ordering.

This ensures deterministic transcript generation.

---

45. Duplicate Extensions

Duplicate extensions MUST be rejected unless the extension specification explicitly permits multiple instances.

---

46. Unknown Extensions

Unknown optional extensions MAY be ignored.

Unknown mandatory extensions MUST produce:

UNSUPPORTED_EXTENSION

---

47. Key Derivation

The raw X25519 shared secret MUST NOT directly become an AEAD key.

The recommended KDF is:

HKDF-SHA-256

Conceptually:

PRK =
    HKDF-Extract(
        salt,
        shared_secret
    )

followed by domain-separated expansion.

---

48. Session Salt

A session-specific salt SHOULD be derived from both random values.

Conceptually:

salt =
    SHA-256(
        initiator_random ||
        responder_random
    )

---

49. Handshake Secret

The handshake secret is derived using a domain-separated label:

"GSP/1.1 handshake"

The label MUST be included in the KDF context.

---

50. Traffic Keys

Separate traffic keys MUST be derived for each direction.

At minimum:

initiator_write_key
responder_write_key

initiator_write_iv
responder_write_iv

---

51. Finished Keys

Finished verification keys MUST be separate from traffic encryption keys.

Example labels:

"GSP/1.1 initiator finished"
"GSP/1.1 responder finished"

---

52. Key Separation

A single cryptographic key MUST NOT be reused for:

handshake authentication
finished verification
Initiator traffic
Responder traffic

Each purpose MUST use independent derived material.

---

53. Role Separation

The cryptographic context MUST distinguish:

INITIATOR
RESPONDER

Directional labels MUST be used during key derivation.

This prevents reflection attacks and cross-direction key confusion.

---

54. Session Identifier

The session identifier SHOULD be derived from the handshake context.

Conceptually:

SID =
    SHA-256(
        "GSP/1.1 SID" ||
        initiator_random ||
        responder_random ||
        initiator_public_key ||
        responder_public_key ||
        negotiated_parameters
    )

The SID is an identifier and MUST NOT be treated as a secret.

---

55. AEAD Encryption

The recommended AEAD is:

ChaCha20-Poly1305

Every encrypted record requires a unique nonce for its key.

---

56. AEAD Nonce Invariant

The following rule is absolute:

A (Key, Nonce) pair MUST NEVER be reused.

Nonce reuse under ChaCha20-Poly1305 is catastrophic.

---

57. Sequence Numbers

Each traffic direction has its own sequence number:

initiator_send_sequence
responder_send_sequence

The counters are independent.

---

58. Initial Sequence Number

The first encrypted record MAY use:

sequence_number = 0

This is valid.

The security requirement is not that the sequence number must start at a non-zero value.

The requirement is:

The same sequence number MUST NOT be reused
with the same traffic key.

---

59. Sequence Increment

For every new encrypted record:

sequence_number += 1

The sequence number MUST be incremented monotonically under the current key.

---

60. Sequence Number Reset

A sequence number MUST NOT be reset while the same traffic key remains active.

This is forbidden:

Key A
sequence 0
sequence 1
sequence 2
...
sequence 0

A reset is permitted only after a successful key update:

Key A
sequence 0..N
        |
        v
KEY_UPDATE
        |
        v
Key B
sequence 0..N

---

61. Sequence Number Width

GSP 1.1 defines:

uint64

for encrypted-record sequence numbers.

Valid values:

0 .. 2^64 - 1

---

62. Sequence Number Wrap

Sequence numbers MUST NOT wrap.

This is forbidden:

2^64 - 1 -> 0

under the same key.

---

63. Rekey Before Exhaustion

Implementations SHOULD initiate "KEY_UPDATE" before sequence exhaustion.

If the final sequence value is reached and no safe key update can occur, the connection MUST be terminated.

The implementation MUST NOT wrap the counter.

---

64. Nonce Construction

For profiles using a static-IV construction:

nonce =
    static_iv XOR sequence_number_encoded

The sequence number MUST be encoded into the exact nonce width defined by the AEAD profile.

For a 96-bit nonce:

static_iv:
    96 bits

sequence_number:
    64 bits

encoded_sequence:
    96-bit zero-extended value

The XOR operation is then:

96-bit static_iv
XOR
96-bit encoded sequence number
=
96-bit nonce

---

65. Nonce Uniqueness

The implementation MUST guarantee that:

same key + same sequence

never results in two independently generated encrypted records.

---

66. Retransmission

On unreliable transports, retransmission MUST NOT accidentally create nonce reuse.

A retransmitted logical record SHOULD reuse the same already-created ciphertext rather than independently encrypting the same plaintext with ambiguous nonce state.

The protocol implementation MUST distinguish:

retransmission of existing ciphertext

from:

new encrypted record

---

67. Receive Sequence Validation

For ordered transports, the receiver SHOULD require:

received_sequence == expected_sequence

For unordered transports, GSP MAY use a replay window.

Old or already accepted sequence numbers MUST be rejected.

---

68. FINISH

"FINISH" confirms possession of the derived handshake keys.

Conceptually:

FINISH {
    authentication_data
    verify_data
}

---

69. Finished Verification

Conceptually:

verify_data =
    HMAC(
        finished_key,
        transcript_hash
    )

The exact Finished construction MUST be defined by the selected cryptographic profile.

---

70. FINISH_ACK

The Responder returns:

FINISH_ACK {
    verify_data
}

The Initiator MUST verify this value before considering the session fully established.

---

71. Application DATA

The Initiator MAY attach encrypted application DATA to the same flight as "FINISH".

Example:

FINISH + encrypted DATA

This is the primary GSP 1-RTT latency optimization.

The Responder MUST NOT deliver the DATA to the application until all required authentication and handshake validation succeeds.

---

72. Handshake State Machine

                         +------+
                         | IDLE |
                         +--+---+
                            |
                          HELLO
                            |
                            v
                    +---------------+
                    | HELLO_SENT    |
                    +-------+-------+
                            |
                       HELLO_ACK
                            |
                            v
                +-----------------------+
                | PARAMETERS_NEGOTIATED |
                +-----------+-----------+
                            |
                     DERIVE KEYS
                            |
                            v
                  +------------------+
                  | KEYS_DERIVED     |
                  +--------+---------+
                           |
                    AUTH / FINISH
                           |
                           v
                  +------------------+
                  | AUTHENTICATING   |
                  +--------+---------+
                           |
                      VERIFY FINISH
                           |
                           v
                  +------------------+
                  | KEY_CONFIRMATION |
                  +--------+---------+
                           |
                     FINISH_ACK
                           |
                           v
                  +------------------+
                  |   ESTABLISHED    |
                  +------------------+

Any fatal error
       |
       v
   +---------+
   | FAILED  |
   +---------+

---

73. Initiator Algorithm

The Initiator MUST:

1. Generate a fresh random value.
2. Generate a fresh ephemeral KEX key pair.
3. Construct "HELLO".
4. Include its ephemeral public key.
5. Send "HELLO".
6. Receive "HELLO_ACK".
7. Validate the selected version.
8. Validate the selected cipher.
9. Validate the selected KEX.
10. Validate the authentication method.
11. Validate extensions.
12. Validate the Responder public key.
13. Calculate the shared secret.
14. Construct the canonical transcript.
15. Derive handshake secrets.
16. Derive traffic keys.
17. Authenticate the Responder where required.
18. Construct "FINISH".
19. Attach encrypted DATA if permitted.
20. Send "FINISH".
21. Receive "FINISH_ACK".
22. Verify "FINISH_ACK".
23. Transition to "ESTABLISHED".

---

74. Responder Algorithm

The Responder MUST:

1. Receive "HELLO".
2. Validate the message.
3. Validate all lengths.
4. Validate the protocol version.
5. Select compatible parameters.
6. Generate a fresh random value.
7. Generate a fresh ephemeral KEX key pair.
8. Construct "HELLO_ACK".
9. Include its ephemeral public key.
10. Send "HELLO_ACK".
11. Calculate the shared secret.
12. Construct the canonical transcript.
13. Derive handshake secrets.
14. Derive traffic keys.
15. Authenticate the Initiator where required.
16. Receive "FINISH".
17. Verify authentication.
18. Verify Finished data.
19. Decrypt permitted application DATA.
20. Send "FINISH_ACK".
21. Transition to "ESTABLISHED".

---

75. Handshake Timeout

Implementations MUST enforce a handshake timeout.

Recommended timers:

HELLO_TIMEOUT
AUTH_TIMEOUT
FINISH_TIMEOUT
TOTAL_HANDSHAKE_TIMEOUT

A timeout MUST result in:

HANDSHAKE_TIMEOUT

and the session MUST enter "FAILED".

---

76. Retry

A Responder MAY send:

RETRY

before performing expensive cryptographic work.

Example:

Initiator -> HELLO
Responder -> RETRY
Initiator -> HELLO + retry_token
Responder -> HELLO_ACK

---

77. Retry Token

A retry token MAY contain:

timestamp
client binding
original random
expiration
authentication tag

The token MUST be integrity protected.

The token SHOULD be stateless from the server's perspective.

Tokens MUST expire.

---

78. Replay Protection

GSP uses:

fresh random values
fresh ephemeral keys
transcript binding
session identifiers
sequence numbers
authentication

to resist replay.

High-security deployments MAY additionally maintain replay caches.

---

79. Downgrade Protection

The transcript MUST cover:

supported versions
selected version
supported ciphers
selected cipher
supported KEX
selected KEX
supported authentication methods
selected authentication method
supported compression
selected compression

An attacker MUST NOT be able to silently force a weaker negotiated configuration.

---

80. Capability Negotiation

Capabilities MAY include:

MULTISTREAM
COMPRESSION
DATAGRAM
MIGRATION
LARGE_FRAMES
WIRELESS_DISPLAY
TERMINAL
FILE_TRANSFER

Capabilities MUST be explicitly negotiated.

A peer MUST NOT assume that a capability exists simply because it is implemented locally.

---

81. Compression

Supported compression algorithms MAY include:

NONE
LZ4

Compression occurs before encryption:

Application Data
       |
       v
Compression
       |
       v
Framing
       |
       v
AEAD
       |
       v
Transport

Encrypted data MUST NOT be compressed.

---

82. Maximum Frame Size

Peers MAY negotiate:

max_frame_size

The value MUST respect implementation and transport limits.

An oversized frame MUST be rejected.

---

83. Maximum Streams

For stream-capable GSP implementations:

max_streams

MAY be negotiated.

The negotiated value becomes active after handshake establishment.

---

84. Maximum Handshake Size

Implementations MUST impose a maximum handshake size.

A recommended baseline is:

MAX_HANDSHAKE_SIZE = 64 KiB

Implementations MAY choose a different limit.

Unbounded memory allocation based on remote length fields is forbidden.

---

85. Fragmentation

Handshake messages MAY be fragmented by the transport.

Fragments MUST be reconstructed before transcript processing.

Transport fragmentation MUST NOT change the logical handshake message.

---

86. Transport Independence

The handshake is designed to operate over:

GSP/TCP
GSP/UDP
GSP wireless transport
Unix/local transport
other GSP-compatible transports

The handshake MUST NOT assume ordered reliable delivery unless the transport provides it.

---

87. Datagram Requirements

For unreliable datagram transports:

- Retransmission MUST be supported.
- Duplicate detection MUST be supported.
- Handshake timeouts MUST exist.
- Sequence state MUST be tracked.
- Fragmentation MUST be bounded.
- Replay protection MUST be enforced.

---

88. Duplicate Messages

A duplicate message MAY be accepted when it is a valid retransmission.

A conflicting duplicate MUST terminate the handshake.

The implementation MUST distinguish:

valid retransmission

from:

modified duplicate

---

89. Handshake Errors

GSP defines:

Code| Name
"0x01"| UNKNOWN_ERROR
"0x02"| INVALID_MESSAGE
"0x03"| INVALID_VERSION
"0x04"| VERSION_UNSUPPORTED
"0x05"| INVALID_CIPHER
"0x06"| CIPHER_UNSUPPORTED
"0x07"| INVALID_KEX
"0x08"| KEX_FAILED
"0x09"| AUTHENTICATION_FAILED
"0x0A"| INVALID_SIGNATURE
"0x0B"| INVALID_MAC
"0x0C"| TRANSCRIPT_MISMATCH
"0x0D"| KEY_CONFIRMATION_FAILED
"0x0E"| INVALID_EXTENSION
"0x0F"| UNSUPPORTED_EXTENSION
"0x10"| INVALID_STATE
"0x11"| TIMEOUT
"0x12"| REPLAY_DETECTED
"0x13"| DOWNGRADE_DETECTED
"0x14"| FRAME_TOO_LARGE
"0x15"| INVALID_LENGTH
"0x16"| NONCE_REUSE
"0x17"| SEQUENCE_EXHAUSTED
"0x18"| INTERNAL_ERROR

---

90. ALERT

Logical structure:

ALERT {
    severity
    error_code
    diagnostic_data
}

Severity:

WARNING
FATAL

Fatal handshake errors MUST terminate the handshake.

---

91. Diagnostic Data

Diagnostic data MUST NOT contain:

private keys
shared secrets
PSKs
traffic keys
plaintext credentials
sensitive application data

Production servers SHOULD avoid exposing detailed cryptographic failure information to unauthenticated clients.

---

92. Invalid State

Messages received in an invalid state MUST produce:

INVALID_STATE

Examples:

DATA before key confirmation
AUTH before KEX
FINISH before key derivation
FINISH_ACK before FINISH

---

93. Denial-of-Service Protection

The Responder SHOULD perform cheap validation before expensive cryptographic operations.

Recommended order:

Parse
  ↓
Length validation
  ↓
Version validation
  ↓
Capability validation
  ↓
Rate limiting / Retry
  ↓
Cryptographic processing

---

94. Rate Limiting

Servers SHOULD rate-limit handshake attempts.

Possible limits include:

source address
connection identifier
authentication identity
global server rate

---

95. Forward Secrecy

The default X25519 profile uses ephemeral keys.

The private ephemeral keys SHOULD be securely erased after key establishment.

Compromise of a long-term authentication key SHOULD NOT expose previous sessions when ephemeral forward secrecy is correctly implemented.

---

96. Secret Erasure

After key establishment, implementations SHOULD erase:

ephemeral private key
raw shared secret
temporary KDF state
unused handshake secrets

Only required session state should remain.

---

97. Key Update

GSP supports optional:

KEY_UPDATE

for traffic key rotation.

A successful key update MUST produce new traffic keys.

The old keys MUST NOT be reused after retirement.

---

98. Key Update Sequence Reset

After a successful key update:

old key
sequence = N

        ↓

KEY_UPDATE

        ↓

new key
sequence = 0

Resetting the sequence number is safe because the encryption key has changed.

---

99. Key Update Failure

If a safe key update cannot be completed before sequence exhaustion:

CLOSE

MUST occur.

The implementation MUST NEVER allow sequence-number wrap.

---

100. Session Resumption

GSP MAY support session resumption.

A resumption ticket MUST NOT contain raw traffic keys.

The ticket SHOULD contain or represent protected resumption state.

---

101. RESUME

Conceptually:

RESUME {
    ticket
    random
    supported_parameters
}

The Responder validates the ticket and establishes fresh session keys.

---

102. RESUME_ACK

RESUME_ACK {
    selected_parameters
    random
    ephemeral_key
}

The resumed session SHOULD still use fresh ephemeral key material where forward secrecy is required.

---

103. 0-RTT Resumption

0-RTT MAY be implemented using a previously established resumption secret.

0-RTT MUST be explicitly enabled by the application/profile.

Replay-sensitive operations MUST NOT be sent using unrestricted 0-RTT.

---

104. Cross-Protocol Protection

All cryptographic derivations MUST use GSP-specific domain labels.

Examples:

"GSP/1.1 handshake"
"GSP/1.1 initiator traffic"
"GSP/1.1 responder traffic"
"GSP/1.1 initiator finished"
"GSP/1.1 responder finished"
"GSP/1.1 resumption"

---

105. Memory Safety

Implementations MUST validate:

lengths
integer overflow
buffer boundaries
extension sizes
key sizes
signature sizes
frame sizes

Network-controlled lengths MUST never result in uncontrolled allocation.

---

106. Constant-Time Operations

Security-sensitive operations SHOULD use constant-time implementations where applicable.

This includes:

MAC comparison
Finished verification
secret comparison
cryptographic primitive operations

---

107. Logging

Debug logs MAY contain:

GSP version
cipher
KEX
authentication mode
compression
CID
SID
handshake state
error code
timing

Logs MUST NOT contain:

private keys
shared secrets
PSKs
traffic keys
authentication secrets

---

108. Application API

After establishment, an implementation MAY expose:

session.id
session.version
session.cipher
session.kex
session.authentication
session.compression
session.peer_identity
session.max_frame_size
session.max_streams

Raw cryptographic secrets MUST NOT be exposed through the normal API.

---

109. Handshake Completion Event

A GSP implementation MAY provide:

onHandshakeComplete(session)

This event MUST NOT occur until all required cryptographic verification succeeds.

---

110. Handshake Failure Event

An implementation MAY provide:

onHandshakeFailure(error)

The exposed error SHOULD avoid leaking sensitive cryptographic information.

---

111. Security Invariants

A compliant implementation MUST preserve all of the following:

1. No application DATA before required handshake verification.

2. No (Key, Nonce) pair is ever reused.

3. Sequence numbers never wrap under the same key.

4. Sequence numbers are independent per direction.

5. Sequence reset occurs only after a key change.

6. All multi-byte integers use Big-Endian.

7. Wire structures contain no compiler-generated padding.

8. Variable-length fields have explicit lengths.

9. Cryptographic fields have fixed lengths.

10. Transcript encoding is canonical.

11. Negotiated parameters are transcript-bound.

12. Ephemeral X25519 keys are fresh for full handshakes.

13. Private keys are never transmitted.

14. Raw shared secrets are never transmitted.

15. Traffic keys are separated by direction.

16. Handshake keys and traffic keys are separated.

17. Authentication is bound to the current session.

18. Unknown mandatory extensions cause failure.

19. Failed handshakes cannot transition to ESTABLISHED.

20. Nonces remain unique for every AEAD key.

21. Sequence-number exhaustion cannot result in wraparound.

22. Cryptographic primitives use established algorithms.

23. Remote lengths cannot cause unbounded allocation.

24. Downgrade attempts are detected through transcript binding.

---

112. Complete Optimized Handshake

INITIATOR                                      RESPONDER
    |                                               |
    | Generate random                               |
    | Generate X25519 key pair                      |
    |                                               |
    |---------------- HELLO ----------------------->|
    |    version                                    |
    |    capabilities                               |
    |    cipher suites                              |
    |    KEX suites                                 |
    |    authentication                             |
    |    random                                     |
    |    X25519 public key                          |
    |                                               |
    |                         Validate HELLO         |
    |                         Select parameters     |
    |                         Generate random      |
    |                         Generate X25519 pair  |
    |                                               |
    |<--------------- HELLO_ACK --------------------|
    |    selected version                           |
    |    selected cipher                            |
    |    selected KEX                               |
    |    selected authentication                    |
    |    random                                     |
    |    X25519 public key                          |
    |                                               |
    | Calculate shared secret                       |
    | Derive handshake secret                       |
    | Derive traffic keys                           |
    |                                               |
    | Construct transcript                          |
    |                                               |
    |------------- FINISH + DATA ----------------->|
    |    authentication                             |
    |    finished verification                      |
    |    encrypted application data                 |
    |                                               |
    |                         Verify authentication |
    |                         Verify transcript     |
    |                         Verify FINISH         |
    |                         Decrypt DATA          |
    |                                               |
    |<--------------- FINISH_ACK -------------------|
    |                                               |
    | Verify FINISH_ACK                             |
    |                                               |
    |============ ESTABLISHED =====================|
    |                                               |
    |<============ ENCRYPTED DATA =================>|

---

113. Handshake State Table

State| Accepted| Action
"IDLE"| HELLO| Parse and negotiate
"HELLO_SENT"| HELLO_ACK| Validate negotiation
"PARAMETERS_NEGOTIATED"| —| Derive keys
"KEYS_DERIVED"| AUTH / FINISH| Authenticate
"AUTHENTICATING"| FINISH| Verify authentication
"KEY_CONFIRMATION"| FINISH_ACK| Establish
"ESTABLISHED"| DATA| Process encrypted data
Any| ALERT| Handle error
Any| invalid message| Fail

---

114. Recommended Cryptographic Profile

Protocol:
    GSP/1.1

Key Exchange:
    X25519

Hash:
    SHA-256

KDF:
    HKDF-SHA-256

AEAD:
    ChaCha20-Poly1305

Nonce:
    Per-direction static IV + sequence number

Sequence:
    uint64

Integer Encoding:
    Big-Endian

Authentication:
    Public Key / PSK / Certificate / Anonymous

Compression:
    None / LZ4

Forward Secrecy:
    Enabled through ephemeral X25519

Handshake:
    Optimized 1-RTT

Key Update:
    Supported

Session Resumption:
    Optional

---

115. Implementation Checklist

A GSP implementation is not considered compliant with this handshake profile until it has implemented and tested:

[ ] Version negotiation
[ ] Cipher negotiation
[ ] KEX negotiation
[ ] Authentication negotiation
[ ] Capability negotiation
[ ] Canonical serialization
[ ] Big-Endian integer encoding
[ ] Explicit field sizes
[ ] X25519 ephemeral KEX
[ ] HKDF-SHA-256
[ ] SHA-256 transcript
[ ] Directional key derivation
[ ] Finished verification
[ ] AEAD encryption
[ ] Unique nonce generation
[ ] uint64 sequence numbers
[ ] Sequence exhaustion handling
[ ] Key update
[ ] Replay protection
[ ] Downgrade protection
[ ] Handshake timeout
[ ] Duplicate handling
[ ] Retry support
[ ] Maximum handshake size
[ ] Extension validation
[ ] Error handling
[ ] Secret erasure
[ ] Fuzz testing

---

116. Required Negative Tests

Implementations SHOULD test:

Invalid version
Unsupported version
Invalid cipher
Unsupported cipher
Invalid KEX
Invalid X25519 key
Modified HELLO
Modified HELLO_ACK
Modified random
Modified public key
Modified authentication
Invalid signature
Invalid PSK MAC
Modified transcript
Invalid FINISH
Invalid FINISH_ACK
Wrong sequence number
Duplicate sequence number
Sequence wrap
Nonce reuse
Unknown mandatory extension
Malformed extension
Oversized message
Integer overflow
Truncated message
Replay
Downgrade attempt
Timeout
Retry token expiration

---

117. Fuzzing

The handshake parser SHOULD be fuzz-tested with:

random message types
random lengths
truncated packets
oversized packets
invalid flags
invalid extensions
duplicate fields
invalid sequence numbers
invalid cryptographic identifiers
malformed public keys
malformed signatures

The implementation MUST remain memory-safe.

---

118. Test Vectors

The GSP specification SHOULD eventually publish deterministic test vectors containing:

Initiator random
Responder random
Initiator private key
Initiator public key
Responder private key
Responder public key
Shared secret
Canonical HELLO
Canonical HELLO_ACK
Transcript
Transcript hash
Handshake secret
Traffic keys
Finished values

These vectors are for interoperability testing only.

---

119. Security Review Requirements

Before production deployment, the GSP handshake SHOULD undergo:

Cryptographic review
Protocol review
Implementation audit
Fuzz testing
Interoperability testing
MITM testing
Replay testing
Downgrade testing
Nonce-reuse testing
DoS testing

---

120. Final Protocol Flow

The GSP 1.1 handshake is fundamentally:

                 GSP HANDSHAKE

HELLO
  |
  +-- version negotiation
  +-- cipher negotiation
  +-- KEX negotiation
  +-- authentication negotiation
  +-- capabilities
  +-- Initiator random
  +-- Initiator X25519 public key
  |
  v
HELLO_ACK
  |
  +-- selected parameters
  +-- Responder random
  +-- Responder X25519 public key
  |
  v
X25519
  |
  v
Shared Secret
  |
  v
HKDF
  |
  +----------------------+
  |                      |
  v                      v
Handshake Keys       Traffic Keys
  |                      |
  v                      |
Authentication            |
  |                      |
  +----------+-----------+
             |
             v
          FINISH
             |
             +-- authentication
             +-- transcript verification
             +-- key confirmation
             +-- optional encrypted DATA
             |
             v
        FINISH_ACK
             |
             v
        ESTABLISHED

---

121. Final Security Contract

The GSP handshake MUST guarantee the following fundamental contract:

NO VALID HANDSHAKE
        |
        v
NO ESTABLISHED SESSION
        |
        v
NO APPLICATION DATA

And for encrypted traffic:

ONE KEY
   +
ONE NONCE
   |
   v
ONE UNIQUE ENCRYPTED RECORD

A nonce MUST NEVER be reused with the same key.

A sequence number MUST NEVER wrap under the same key.

A sequence number MAY start at zero.

A sequence number MAY be reset after a successful key change.

---

122. Final GSP 1.1 Handshake

The canonical optimized GSP 1.1 handshake is:

1. Initiator generates random + ephemeral X25519 key.

2. Initiator sends:
       HELLO
       + X25519 public key

3. Responder validates HELLO.

4. Responder selects:
       version
       cipher
       KEX
       authentication
       compression
       capabilities

5. Responder generates random + ephemeral X25519 key.

6. Responder sends:
       HELLO_ACK
       + selected parameters
       + X25519 public key

7. Both peers calculate:
       X25519 shared secret.

8. Both peers derive:
       handshake keys
       traffic keys
       finished keys.

9. Both peers construct the canonical transcript.

10. Initiator sends:
       FINISH
       + authentication
       + optional encrypted DATA

11. Responder verifies:
       authentication
       transcript
       Finished value

12. Responder decrypts permitted DATA.

13. Responder sends:
       FINISH_ACK

14. Initiator verifies FINISH_ACK.

15. Both peers enter:
       ESTABLISHED

16. Encrypted GSP DATA traffic begins.

17. Traffic keys are rotated through:
       KEY_UPDATE

18. Sequence numbers MUST never wrap under a key.

19. When the session ends:
       CLOSE

---

123. End of Specification

GSP Handshake Protocol Specification — Version 1.1

GSP/1.1
Secure Handshake
Optimized 1-RTT Data Path
Ephemeral X25519
HKDF-SHA-256
ChaCha20-Poly1305
Canonical Binary Encoding
Big-Endian Wire Format
Transcript Authentication
Directional Key Separation
Replay Protection
Downgrade Protection
Forward Secrecy
Key Update
Session Resumption

END OF DOCUMENT