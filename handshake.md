GSP Handshake Protocol Specification

Globalized Secure Protocol (GSP)
Handshake Specification
Version: 1.0
Status: Experimental / Draft
Scheme: "gsp://"

---

1. Abstract

The GSP Handshake establishes a secure GSP session between two peers before application data is transmitted.

The handshake provides:

- Protocol version negotiation
- Capability negotiation
- Cryptographic algorithm negotiation
- Ephemeral key exchange
- Session key derivation
- Peer authentication
- Transcript integrity
- Downgrade protection
- Replay protection
- Session establishment
- Key confirmation
- Connection parameter negotiation
- Optional compression negotiation
- Optional application-layer negotiation

After successful completion, both peers possess the same cryptographic session state and can exchange encrypted GSP DATA frames.

---

2. Terminology

Term| Meaning
Initiator| Peer that starts the GSP handshake
Responder| Peer that receives the initial handshake
Client| Application initiating a GSP connection
Server| Application accepting a GSP connection
Peer| Either endpoint
Session| Cryptographically established GSP connection
Handshake| Protocol phase used to establish the session
Transcript| Ordered cryptographic record of handshake messages
PSIV| Protocol Session Initialization Vector
KEX| Key Exchange
AEAD| Authenticated Encryption with Associated Data
MTU| Maximum Transmission Unit
CID| Connection Identifier
SID| Session Identifier

---

3. Handshake Goals

A successful GSP handshake MUST establish:

1. A mutually supported GSP version.
2. A mutually supported cryptographic suite.
3. A mutually supported transport configuration.
4. Fresh ephemeral key material.
5. A shared secret.
6. Derived traffic keys.
7. Peer authentication when authentication is enabled.
8. A cryptographically verified transcript.
9. A unique session identifier.
10. Key confirmation from both peers.

No application DATA MUST be transmitted before the handshake reaches the "ESTABLISHED" state.

---

4. Handshake Overview

The canonical handshake is:

Initiator                                      Responder
   |                                               |
   |--------------- HELLO ------------------------>|
   |                                               |
   |<-------------- HELLO_ACK --------------------|
   |                                               |
   |--------------- KEY_EXCHANGE ----------------->|
   |                                               |
   |<-------------- KEY_EXCHANGE_ACK -------------|
   |                                               |
   |--------------- AUTH ------------------------->|
   |                                               |
   |<-------------- AUTH_ACK ----------------------|
   |                                               |
   |--------------- FINISH ----------------------->|
   |                                               |
   |<-------------- FINISH_ACK --------------------|
   |                                               |
   |============== SECURE SESSION ===============>|
   |                                               |
   |--------------- DATA ------------------------->|
   |<--------------- DATA -------------------------|

The exact presence of "AUTH" depends on the authentication mode.

---

5. Handshake States

An implementation MUST maintain a handshake state.

IDLE
 |
 v
HELLO_SENT
 |
 v
HELLO_RECEIVED
 |
 v
PARAMETERS_NEGOTIATED
 |
 v
KEY_EXCHANGE
 |
 v
KEYS_DERIVED
 |
 v
AUTHENTICATING
 |
 v
KEY_CONFIRMATION
 |
 v
ESTABLISHED

Failure from any state enters:

FAILED

The session MUST NOT return from "FAILED" to "ESTABLISHED".

---

6. State Definitions

6.1 IDLE

Initial state.

No GSP session exists.

Allowed operation:

Send HELLO
Receive HELLO

---

6.2 HELLO_SENT

The Initiator has transmitted "HELLO".

The Initiator waits for:

HELLO_ACK

A timeout MUST eventually terminate the handshake.

---

6.3 HELLO_RECEIVED

The Responder has received and validated "HELLO".

The Responder MUST:

1. Validate the GSP version.
2. Validate capabilities.
3. Select cryptographic parameters.
4. Generate fresh ephemeral key material.
5. Generate handshake randomness.
6. Send "HELLO_ACK".

---

6.4 PARAMETERS_NEGOTIATED

Both peers have selected compatible parameters.

The selected parameters become immutable for the remainder of the handshake.

Any attempt to change negotiated parameters MUST cause handshake failure.

---

6.5 KEY_EXCHANGE

The peers exchange ephemeral public keys.

The resulting shared secret MUST NOT be transmitted directly.

---

6.6 KEYS_DERIVED

Both peers derive the handshake and application traffic keys.

The raw key exchange secret MUST NOT be used directly as an AEAD key.

A KDF MUST be used.

---

6.7 AUTHENTICATING

The optional authentication phase verifies peer identity.

Authentication MAY use:

- Pre-shared keys
- Public-key signatures
- Certificates
- Application-defined authentication
- Anonymous mode

Anonymous mode SHOULD be disabled by default for applications requiring peer identity.

---

6.8 KEY_CONFIRMATION

Both sides prove possession of the negotiated session keys.

This prevents an endpoint from believing that the handshake succeeded when the peer derived different keys.

---

6.9 ESTABLISHED

The secure session is active.

Encrypted GSP DATA frames MAY now be transmitted.

---

7. Message Types

The handshake defines the following message types:

Type| Name
"0x01"| HELLO
"0x02"| HELLO_ACK
"0x03"| KEY_EXCHANGE
"0x04"| KEY_EXCHANGE_ACK
"0x05"| AUTH
"0x06"| AUTH_ACK
"0x07"| FINISH
"0x08"| FINISH_ACK
"0x09"| ALERT
"0x0A"| RETRY
"0x0B"| CLOSE

Values above "0x7F" are reserved for experimental/private extensions.

---

8. Generic Handshake Frame

All GSP handshake messages are encapsulated in a GSP frame.

Conceptually:

+--------+--------+--------+--------+
| Magic  | Version| Type   | Flags  |
+--------+--------+--------+--------+
| Length                         |
+---------------------------------+
| Connection / Session ID         |
+---------------------------------+
| Message Payload                 |
+---------------------------------+
| Integrity / Authentication     |
+---------------------------------+

The exact outer GSP framing MUST follow the core GSP transport specification.

---

9. HELLO

"HELLO" starts a handshake.

Direction

Initiator -> Responder

Purpose

The message advertises:

- Supported GSP versions
- Supported cipher suites
- Supported KEX algorithms
- Supported authentication methods
- Supported compression
- Maximum frame size
- Capabilities
- Random nonce
- Initiator CID
- Optional extensions

---

10. HELLO Payload

Logical structure:

HELLO {
    protocol_version
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
    extensions[]
}

---

11. HELLO Random

"random" MUST contain cryptographically secure random bytes.

Recommended size:

32 bytes

The value MUST be freshly generated for every handshake.

It MUST NOT be reused between independent sessions.

---

12. Connection ID

The Initiator MAY provide a connection identifier.

The CID allows implementations to distinguish multiple simultaneous sessions.

The CID MUST NOT be interpreted as a secret.

---

13. Version Negotiation

The Initiator advertises:

minimum_version
protocol_version

The Responder selects one mutually supported version.

Example:

Initiator:
    minimum = 1.0
    maximum = 1.2

Responder:
    supported = 1.0, 1.1

Selected:
    1.1

If no compatible version exists:

VERSION_UNSUPPORTED

MUST be returned.

---

14. Cryptographic Suite Negotiation

A cipher suite identifies the cryptographic algorithms used by the session.

Example:

GSP_AES256_GCM
GSP_CHACHA20_POLY1305

The implementation MUST NOT silently substitute an unsupported algorithm.

The negotiated suite MUST be included in the transcript.

This prevents an attacker from modifying algorithm negotiation without detection.

---

15. Recommended Experimental Suite

The initial GSP experimental profile MAY define:

GSP-CHACHA20-POLY1305

with:

KEX: X25519
AEAD: ChaCha20-Poly1305
KDF: HKDF-SHA-256
Hash: SHA-256

This is preferable to inventing a new cryptographic primitive.

GSP MAY define protocol-specific constructions around established primitives, but MUST NOT replace standardized cryptographic primitives without extensive cryptographic review.

---

16. Key Exchange

The default experimental KEX is:

X25519

Each peer generates:

private_key
public_key

The private key MUST remain local.

The public key is transmitted.

---

17. KEY_EXCHANGE

Initiator -> Responder

Logical structure:

KEY_EXCHANGE {
    key_exchange_algorithm
    public_key
    key_exchange_extensions[]
}

The public key MUST correspond to the algorithm selected during negotiation.

---

18. KEY_EXCHANGE_ACK

Responder -> Initiator

Logical structure:

KEY_EXCHANGE_ACK {
    key_exchange_algorithm
    public_key
    key_exchange_extensions[]
}

After receiving this message, the Initiator can calculate the shared secret.

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

Both values MUST be identical.

The shared secret MUST NOT be transmitted.

---

20. Session Transcript

Every handshake message MUST contribute to a transcript.

Conceptually:

T =
    HELLO ||
    HELLO_ACK ||
    KEY_EXCHANGE ||
    KEY_EXCHANGE_ACK ||
    AUTH ||
    AUTH_ACK ||
    FINISH

Messages that were not sent are omitted according to the negotiated handshake mode.

The transcript MUST preserve message order.

---

21. Transcript Hash

The transcript hash is:

transcript_hash =
    SHA-256(T)

The transcript hash MUST be calculated over the canonical wire representation.

Implementations MUST NOT hash an implementation-specific object representation.

---

22. Canonical Encoding

Handshake structures MUST have deterministic serialization.

For example:

field ID
field length
field value

Implementations MUST:

- Use the specified byte order.
- Encode lengths consistently.
- Reject malformed lengths.
- Reject duplicate mandatory fields.
- Preserve extension ordering rules.
- Reject ambiguous encodings.

This is essential because both peers must calculate the same transcript.

---

23. Key Derivation

The shared secret MUST be processed through a KDF.

Recommended:

HKDF-SHA-256

Conceptual process:

PRK = HKDF-Extract(
    salt,
    shared_secret
)

handshake_secret =
    HKDF-Expand(
        PRK,
        "GSP handshake",
        ...
    )

---

24. Session Salt

A session-specific salt SHOULD be derived from the handshake random values.

Conceptually:

salt =
    H(
        initiator_random ||
        responder_random
    )

The random values MUST be included in the transcript.

---

25. Traffic Key Derivation

Separate keys MUST be derived for each direction.

For example:

client_write_key
server_write_key
client_write_iv
server_write_iv

Conceptually:

client_write_key =
    HKDF-Expand(
        handshake_secret,
        "GSP client write key",
        key_length
    )

server_write_key =
    HKDF-Expand(
        handshake_secret,
        "GSP server write key",
        key_length
    )

---

26. Direction Separation

A key MUST NOT be reused for both directions.

The protocol MUST distinguish:

Initiator -> Responder

from:

Responder -> Initiator

This prevents reflection and cross-direction key confusion.

---

27. Nonce / IV Derivation

AEAD encryption requires unique nonces.

GSP MUST guarantee nonce uniqueness for a given key.

A conceptual construction is:

nonce =
    IV XOR sequence_number

where:

IV = derived per-direction IV

The exact nonce construction MUST be fixed by the selected cipher profile.

An implementation MUST NEVER randomly generate nonces if that can result in reuse under the same key.

---

28. PSIV

GSP MAY use a Protocol Session Initialization Vector (PSIV).

The PSIV is session-specific material used as part of nonce/session initialization.

It MUST:

- Be unique enough for the selected security construction.
- Be included in authenticated handshake state.
- Never be treated as a replacement for the cryptographic key.
- Never be reused with the same key in a way that violates the AEAD nonce requirements.

---

29. Authentication Modes

GSP supports multiple authentication modes.

Mode 0 — Anonymous

ANONYMOUS

No peer identity is cryptographically verified.

Useful for:

- Discovery
- Local experimental connections
- Public services

Not recommended for sensitive applications.

---

Mode 1 — PSK

PRE_SHARED_KEY

Both peers possess a shared secret.

The PSK MUST NOT be transmitted.

Authentication is performed using a transcript-bound MAC or equivalent construction.

---

Mode 2 — Public-Key

PUBLIC_KEY

A peer proves possession of a private signing key.

The signature MUST cover the transcript and negotiated parameters.

---

Mode 3 — Certificate

CERTIFICATE

The peer presents a certificate chain or equivalent credential.

Validation rules are application/profile dependent.

---

30. AUTH

Initiator -> Responder

Logical structure:

AUTH {
    authentication_method
    identity
    credential
    signature_or_mac
}

The credential MUST be cryptographically bound to the current handshake.

---

31. Authentication Signature

For public-key authentication:

signature_input =
    context ||
    protocol_version ||
    negotiated_parameters ||
    initiator_random ||
    responder_random ||
    transcript_hash

The exact canonical representation MUST be specified by the authentication profile.

The signature MUST NOT cover mutable or unspecified data.

---

32. AUTH_ACK

Responder -> Initiator

The Responder confirms authentication.

Logical structure:

AUTH_ACK {
    status
    identity
    credential
    signature_or_mac
}

If authentication fails:

AUTHENTICATION_FAILED

MUST be returned.

---

33. Authentication Context Binding

Authentication MUST be bound to:

- GSP version
- Selected cipher
- Selected KEX
- Random values
- Public keys
- Session identifier
- Handshake transcript

This prevents credentials captured from one session from being reused in another.

---

34. Session Identifier

After sufficient handshake material exists, the peers derive a SID.

Conceptually:

SID =
    H(
        "GSP SID" ||
        initiator_random ||
        responder_random ||
        initiator_public_key ||
        responder_public_key ||
        negotiated_parameters
    )

The SID is an identifier, not a password.

---

35. FINISH

"FINISH" proves that the sender has successfully derived the session keys.

Initiator -> Responder

Logical structure:

FINISH {
    verify_data
}

---

36. FINISH Verify Data

Conceptually:

verify_data =
    HMAC(
        finished_key,
        transcript_hash
    )

The exact construction MUST be defined by the cryptographic profile.

The Responder verifies the value using its independently derived state.

---

37. FINISH_ACK

Responder -> Initiator

The Responder sends its own key-confirmation value.

FINISH_ACK {
    verify_data
}

The Initiator verifies it.

---

38. Handshake Completion

The handshake is complete only after:

1. Negotiation succeeds.
2. Key exchange succeeds.
3. Authentication succeeds when enabled.
4. Traffic keys are derived.
5. "FINISH" verifies.
6. "FINISH_ACK" verifies.

Then:

state = ESTABLISHED

---

39. Complete Canonical Exchange

Anonymous Mode

I                                            R
|                                            |
|------------- HELLO ---------------------->|
|                                            |
|<------------ HELLO_ACK -------------------|
|                                            |
|------------- KEY_EXCHANGE ---------------->|
|                                            |
|<------------ KEY_EXCHANGE_ACK ------------|
|                                            |
|------------- FINISH ---------------------->|
|                                            |
|<------------ FINISH_ACK ------------------|
|                                            |
|========== SESSION ESTABLISHED ============|

---

Authenticated Mode

I                                            R
|                                            |
|------------- HELLO ---------------------->|
|                                            |
|<------------ HELLO_ACK -------------------|
|                                            |
|------------- KEY_EXCHANGE ---------------->|
|                                            |
|<------------ KEY_EXCHANGE_ACK ------------|
|                                            |
|------------- AUTH ------------------------>|
|                                            |
|<------------ AUTH_ACK ---------------------|
|                                            |
|------------- FINISH ---------------------->|
|                                            |
|<------------ FINISH_ACK ------------------|
|                                            |
|========== SESSION ESTABLISHED ============|

---

40. Retry

A Responder MAY request a retry when additional anti-abuse or stateless validation is required.

RETRY

Example:

Initiator -> HELLO
Initiator <- RETRY
Initiator -> HELLO + retry_token

The retry token MUST be integrity protected.

The token SHOULD be stateless when possible.

---

41. Retry Token

A retry token MAY contain:

version
timestamp
client_address_binding
original_random
expiration
authentication_tag

The token MUST NOT expose secret server state.

Tokens MUST expire.

---

42. Replay Protection

GSP MUST protect handshake state against replay.

Replay protection is provided through:

- Fresh random values
- Ephemeral keys
- Transcript binding
- Session identifiers
- Authentication binding
- Optional retry tokens

A server MUST NOT accept a handshake solely because the packet format is valid.

---

43. Randomness Requirements

All security-sensitive randomness MUST originate from a cryptographically secure random number generator.

This includes:

- Ephemeral private keys
- Handshake random values
- PSIV material where applicable
- Authentication challenges
- Retry tokens where applicable

Predictable PRNGs MUST NOT be used.

---

44. Downgrade Protection

An attacker MUST NOT be able to force peers into weaker parameters without detection.

The transcript MUST include:

supported_versions
selected_version
supported_ciphers
selected_cipher
supported_KEX
selected_KEX
supported_authentication
selected_authentication

The final authentication/key-confirmation data MUST depend on the transcript.

---

45. Unknown Extensions

Unknown optional extensions MUST be ignored only when explicitly marked as optional.

Unknown mandatory extensions MUST cause:

UNSUPPORTED_EXTENSION

The extension mechanism MUST distinguish:

OPTIONAL

from:

REQUIRED

---

46. Extension Structure

Conceptual format:

extension {
    type
    flags
    length
    value
}

Example flags:

0x00 = optional
0x01 = mandatory

Reserved flags MUST be rejected unless defined by the applicable specification.

---

47. Capability Negotiation

Capabilities MAY include:

MULTISTREAM
COMPRESSION
DATAGRAM
MIGRATION
0RTT
LARGE_FRAMES
WIRELESS_DISPLAY
TERMINAL
FILE_TRANSFER

A capability MUST NOT be assumed merely because the peer is running GSP.

It must be explicitly negotiated.

---

48. Compression Negotiation

Supported compression algorithms MAY include:

NONE
LZ4

The selected compression algorithm becomes part of the transcript.

Compression MUST occur before encryption:

Application Data
      |
      v
Compression
      |
      v
Framing
      |
      v
AEAD Encryption
      |
      v
Transport

Encrypted data MUST NOT be compressed.

---

49. Maximum Frame Size

Peers MAY negotiate:

max_frame_size

The selected value MUST NOT exceed the actual implementation or transport limits.

Malformed oversized frames MUST be rejected.

---

50. Maximum Streams

For stream-capable GSP implementations:

max_streams

may be negotiated.

The value becomes effective only after the handshake is established.

---

51. Handshake Timeout

Each handshake MUST have a maximum lifetime.

Recommended conceptual timers:

HELLO_TIMEOUT
KEY_EXCHANGE_TIMEOUT
AUTH_TIMEOUT
FINISH_TIMEOUT
TOTAL_HANDSHAKE_TIMEOUT

A timeout MUST cause:

HANDSHAKE_TIMEOUT

and transition to:

FAILED

---

52. Retransmission

For unreliable transports, handshake messages MUST support retransmission.

A retransmitted message MUST NOT create a new cryptographic session unless the protocol explicitly requires it.

Implementations SHOULD identify messages using:

handshake_sequence

and/or a unique session/connection identifier.

---

53. Duplicate Messages

A duplicate handshake message MAY be received.

Examples:

duplicate HELLO
duplicate KEY_EXCHANGE
duplicate FINISH

Implementations MUST distinguish:

- valid retransmission
- conflicting duplicate
- replayed message
- malformed message

A conflicting duplicate MUST terminate the handshake.

---

54. Handshake Sequence Number

Each handshake message MAY contain:

sequence_number

Example:

HELLO              = 0
HELLO_ACK          = 1
KEY_EXCHANGE       = 2
KEY_EXCHANGE_ACK   = 3
AUTH               = 4
AUTH_ACK           = 5
FINISH             = 6
FINISH_ACK         = 7

Unexpected sequence numbers MUST be rejected unless the negotiated transport explicitly permits reordering.

---

55. Error Handling

GSP defines handshake alerts.

Example values:

0x01 UNKNOWN_ERROR
0x02 INVALID_MESSAGE
0x03 INVALID_VERSION
0x04 VERSION_UNSUPPORTED
0x05 INVALID_CIPHER
0x06 CIPHER_UNSUPPORTED
0x07 INVALID_KEY_EXCHANGE
0x08 KEY_EXCHANGE_FAILED
0x09 AUTHENTICATION_FAILED
0x0A INVALID_SIGNATURE
0x0B INVALID_MAC
0x0C TRANSCRIPT_MISMATCH
0x0D KEY_CONFIRMATION_FAILED
0x0E INVALID_EXTENSION
0x0F UNSUPPORTED_EXTENSION
0x10 INVALID_STATE
0x11 TIMEOUT
0x12 REPLAY_DETECTED
0x13 DOWNGRADE_DETECTED
0x14 FRAME_TOO_LARGE
0x15 INTERNAL_ERROR

---

56. ALERT

Logical structure:

ALERT {
    severity
    error_code
    diagnostic_data
}

Severity:

WARNING
FATAL

Security-sensitive errors SHOULD use "FATAL".

---

57. Diagnostic Information

Diagnostic information MUST NOT expose:

- Private keys
- Session secrets
- PSKs
- Plaintext credentials
- Sensitive application data

Production implementations SHOULD avoid exposing detailed cryptographic failure reasons to unauthenticated remote peers.

---

58. Invalid State

If a message is received in an invalid state:

INVALID_STATE

MUST be generated.

Example:

DATA before FINISH_ACK

is invalid.

---

59. Handshake Integrity

Every security-sensitive handshake parameter MUST be authenticated by the final cryptographic transcript.

This includes:

version
cipher
KEX
authentication method
compression
capabilities
random values
public keys
extensions

---

60. MITM Resistance

Authenticated GSP handshakes MUST provide protection against active man-in-the-middle attacks.

Ephemeral key exchange alone does NOT authenticate the peer.

Therefore:

KEX + Authentication

is required for authenticated identity.

Anonymous GSP provides encryption against passive observation but does not inherently provide authenticated peer identity.

---

61. Forward Secrecy

When ephemeral Diffie-Hellman is used:

X25519 ephemeral key

the session SHOULD provide forward secrecy.

Private ephemeral keys MUST be securely erased after they are no longer needed.

---

62. Key Lifetime

Traffic keys SHOULD have a bounded lifetime.

Keys MAY be rotated based on:

- Number of encrypted records
- Number of bytes transmitted
- Time
- Explicit rekey
- Security policy

---

63. Rekey

A future GSP extension MAY provide:

KEY_UPDATE

A key update MUST derive new traffic keys from existing authenticated session state.

Old keys MUST NOT be reused after retirement.

---

64. 0-RTT

0-RTT is OPTIONAL.

If implemented, it MUST be treated as replay-sensitive.

Applications MUST NOT place replay-sensitive operations into unauthenticated 0-RTT data.

Examples of unsafe 0-RTT operations:

Transfer money
Delete data
Create account
Change password

---

65. Session Resumption

GSP MAY support session resumption.

A resumption mechanism MUST use a cryptographically protected ticket or equivalent.

The ticket MUST NOT contain raw session keys.

Conceptually:

Session
   |
   v
RESUMPTION_TICKET
   |
   v
Future connection

---

66. Session Resumption Handshake

A resumed handshake MAY be:

Initiator                         Responder
   |                                 |
   |-------- RESUME ---------------->|
   |                                 |
   |<------- RESUME_ACK -------------|
   |                                 |
   |-------- FINISH ---------------->|
   |                                 |
   |<------- FINISH_ACK -------------|
   |                                 |
   |========= ESTABLISHED ===========|

The resumption mode MUST retain downgrade and replay protections.

---

67. Session Key Separation

The following values MUST NOT all reuse the same key:

Handshake authentication
Handshake encryption
Client traffic
Server traffic
Finished verification
Application encryption

Separate derivation labels MUST be used.

Example:

"gsp handshake"
"gsp client write"
"gsp server write"
"gsp client finished"
"gsp server finished"

---

68. Domain Separation

Every KDF label SHOULD contain a GSP-specific context.

Example:

GSP-v1 handshake
GSP-v1 client-write
GSP-v1 server-write
GSP-v1 finished

This prevents keys generated for one protocol purpose from being accidentally interpreted as another.

---

69. Secret Erasure

After successful establishment, implementations SHOULD erase:

ephemeral_private_key
raw_shared_secret
temporary KDF state
unused handshake secrets

Only required session state should remain.

---

70. Memory Safety

Implementations MUST validate:

- Integer overflow
- Length fields
- Buffer boundaries
- Extension lengths
- Public-key lengths
- Signature lengths
- Maximum handshake size

Network-provided lengths MUST never be trusted blindly.

---

71. Maximum Handshake Size

Implementations MUST define a maximum acceptable handshake size.

Example profile:

MAX_HANDSHAKE_SIZE = 64 KiB

An implementation MAY choose another value appropriate to its deployment.

Oversized handshakes MUST be rejected before unbounded memory allocation.

---

72. Fragmentation

A handshake message MAY be fragmented by the underlying transport.

Fragmentation MUST NOT alter the logical transcript.

The transcript is calculated over the reconstructed canonical handshake messages, not individual transport fragments.

---

73. Transport Independence

The handshake is designed to operate over different GSP transports.

Examples:

GSP/TCP
GSP/UDP
GSP/QUIC-like transport
GSP wireless transport
Unix/local transport

The handshake MUST NOT assume reliable ordered delivery unless the selected transport guarantees it.

---

74. TCP-like Transport

For reliable ordered transports:

HELLO
HELLO_ACK
KEY_EXCHANGE
KEY_EXCHANGE_ACK
AUTH
AUTH_ACK
FINISH
FINISH_ACK

can be transmitted sequentially.

Transport-level retransmission handles packet loss.

---

75. Datagram Transport

For datagram transports:

- Message retransmission is required.
- Duplicate detection is required.
- Message ordering must be tracked.
- Handshake timeout is required.
- Fragmentation must be handled carefully.

---

76. Handshake State Machine

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
                      KEY_EXCHANGE
                            |
                            v
                  +------------------+
                  | KEYS_DERIVED     |
                  +--------+---------+
                           |
                         AUTH
                           |
                           v
                  +------------------+
                  | AUTHENTICATING   |
                  +--------+---------+
                           |
                        FINISH
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

77. Full Handshake Algorithm — Initiator

The Initiator MUST:

1. Generate a fresh random value.
2. Generate a fresh ephemeral KEX key pair.
3. Construct "HELLO".
4. Send "HELLO".
5. Receive "HELLO_ACK".
6. Validate the selected version.
7. Validate the selected cipher.
8. Validate KEX selection.
9. Validate authentication mode.
10. Validate extensions.
11. Receive responder public key.
12. Calculate the shared secret.
13. Calculate the transcript.
14. Derive handshake secrets.
15. Derive traffic keys.
16. Authenticate the Responder when required.
17. Construct "AUTH" when required.
18. Send "AUTH".
19. Receive "AUTH_ACK".
20. Verify authentication.
21. Construct "FINISH".
22. Send "FINISH".
23. Receive "FINISH_ACK".
24. Verify "FINISH_ACK".
25. Transition to "ESTABLISHED".

---

78. Full Handshake Algorithm — Responder

The Responder MUST:

1. Receive "HELLO".
2. Validate the message.
3. Validate the supported GSP version.
4. Select a compatible version.
5. Select cryptographic parameters.
6. Generate fresh random data.
7. Generate a fresh ephemeral KEX key pair.
8. Construct "HELLO_ACK".
9. Send "HELLO_ACK".
10. Receive "KEY_EXCHANGE".
11. Validate the Initiator public key.
12. Calculate the shared secret.
13. Derive handshake secrets.
14. Derive traffic keys.
15. Authenticate the Initiator when required.
16. Send "AUTH_ACK" when appropriate.
17. Receive "AUTH".
18. Verify the Initiator authentication.
19. Receive "FINISH".
20. Verify "FINISH".
21. Send "FINISH_ACK".
22. Transition to "ESTABLISHED".

---

79. Recommended Message Ordering

Canonical ordering:

1. HELLO
2. HELLO_ACK
3. KEY_EXCHANGE
4. KEY_EXCHANGE_ACK
5. AUTH
6. AUTH_ACK
7. FINISH
8. FINISH_ACK

Messages MAY be omitted only where explicitly permitted by the negotiated authentication profile.

---

80. Illegal Ordering Examples

The following are invalid:

DATA -> before FINISH_ACK
AUTH -> before KEY_EXCHANGE
FINISH -> before keys are derived
KEY_EXCHANGE_ACK -> before KEY_EXCHANGE
FINISH_ACK -> before FINISH

These MUST result in "INVALID_STATE".

---

81. Handshake Transcript Example

Conceptually:

T =
    Encode(HELLO)
 || Encode(HELLO_ACK)
 || Encode(KEY_EXCHANGE)
 || Encode(KEY_EXCHANGE_ACK)
 || Encode(AUTH)
 || Encode(AUTH_ACK)
 || Encode(FINISH)

Then:

TH = SHA256(T)

The exact point at which "FINISH" is included MUST be defined consistently by the implementation profile.

A common construction is that "FINISH" authenticates the transcript up to the immediately preceding handshake message, while "FINISH_ACK" authenticates the corresponding completed transcript.

---

82. Cryptographic Context

The session cryptographic context SHOULD conceptually contain:

GSPContext {
    version
    cipher_suite
    kex_algorithm
    authentication_mode

    initiator_random
    responder_random

    initiator_public_key
    responder_public_key

    session_id
    transcript_hash

    handshake_secret

    initiator_write_key
    responder_write_key

    initiator_write_iv
    responder_write_iv

    sequence_number_send
    sequence_number_receive
}

Private ephemeral keys SHOULD be removed after key establishment.

---

83. Security Invariants

A compliant implementation MUST preserve these invariants:

Invariant 1

No plaintext application data before establishment.

Invariant 2

No traffic key reuse between directions.

Invariant 3

No nonce reuse under the same AEAD key.

Invariant 4

All negotiated cryptographic parameters are transcript-bound.

Invariant 5

Authentication is session-bound.

Invariant 6

Ephemeral private keys are never transmitted.

Invariant 7

Raw shared secrets are never transmitted.

Invariant 8

Failed handshakes never become established sessions.

Invariant 9

Invalid state transitions are rejected.

Invariant 10

Handshake secrets are never logged.

---

84. Logging Rules

Debug logs MAY contain:

GSP version
selected cipher
selected KEX
selected compression
connection ID
session ID
handshake state
error code
timing

Logs MUST NOT contain:

private keys
shared secrets
PSKs
session traffic keys
authentication secrets
raw credentials
AEAD nonces together with sufficient secret state to compromise security

---

85. Timing

Implementations SHOULD avoid exposing unnecessary information through timing differences.

Particularly:

- MAC verification
- Signature verification
- Finished verification
- PSK verification

SHOULD use constant-time operations where applicable.

---

86. Denial-of-Service Protection

A responder SHOULD avoid expensive cryptographic operations before basic validation.

Recommended ordering:

Parse
  ↓
Length validation
  ↓
Version validation
  ↓
Capability validation
  ↓
Rate limiting / retry
  ↓
Cryptographic processing

This reduces CPU exhaustion attacks.

---

87. Resource Limits

Implementations SHOULD limit:

maximum concurrent handshakes
maximum handshake size
maximum extension count
maximum authentication credential size
maximum retry attempts
maximum handshake duration

---

88. Rate Limiting

Servers SHOULD implement handshake rate limiting.

Possible dimensions:

source address
connection ID
authentication identity
application identity
global server rate

---

89. Cryptographic Failure

Any of the following SHOULD terminate the handshake:

invalid public key
invalid signature
invalid MAC
invalid transcript
invalid finished value
invalid negotiated suite
invalid KDF output

The implementation MUST NOT continue with potentially inconsistent cryptographic state.

---

90. Session Establishment Event

Once "FINISH_ACK" has been verified:

GSP_SESSION_ESTABLISHED

MAY be emitted internally.

The application MAY then receive:

SESSION_READY

---

91. Application Notification

A GSP implementation SHOULD expose a clear API event:

onHandshakeComplete(session)

The application MUST NOT be notified as established before cryptographic confirmation.

---

92. Session Properties Exposed to Applications

After establishment, an application MAY query:

session.id
session.version
session.cipher
session.kex
session.authentication
session.compression
session.peer_identity
session.max_frame_size
session.max_streams

Secrets MUST NOT be exposed through the normal application API.

---

93. Handshake Failure API

Applications MAY receive:

onHandshakeFailure(error)

Example:

HANDSHAKE_FAILURE {
    code: AUTHENTICATION_FAILED
}

Sensitive cryptographic details SHOULD remain internal.

---

94. Close After Failure

After a fatal handshake error:

ALERT

MAY be sent if it is safe to do so.

Then:

connection = CLOSED
state = FAILED

The implementation MUST NOT continue using the failed cryptographic state.

---

95. Security Model

GSP's handshake aims to provide:

Confidentiality
Integrity
Peer Authentication
Forward Secrecy
Replay Resistance
Downgrade Resistance
Key Separation
Session Binding

The exact guarantees depend on the selected authentication mode.

---

96. Anonymous Security Model

Anonymous mode provides:

Encryption
Integrity
Ephemeral Key Exchange
Forward Secrecy

but does not inherently provide:

Authenticated Peer Identity
MITM Protection

---

97. PSK Security Model

PSK mode provides authentication if:

- The PSK is secret.
- The PSK is sufficiently strong.
- The authentication computation is transcript-bound.
- The PSK is not transmitted.

Weak human passwords SHOULD NOT be used directly as PSKs.

A password-based KDF should be used where passwords are unavoidable.

---

98. Public-Key Security Model

Public-key authentication provides:

Peer Authentication
MITM Resistance
Forward Secrecy

when correctly combined with ephemeral KEX.

The identity-to-public-key binding MUST be trusted by the application or certificate infrastructure.

---

99. Perfect Forward Secrecy

With ephemeral X25519:

Long-term identity key
        +
Ephemeral KEX key
        |
        v
Session secret

Compromise of a long-term authentication key SHOULD NOT reveal previously established session traffic, assuming ephemeral secrets were securely erased.

---

100. Replay Model

An attacker may capture:

HELLO
KEY_EXCHANGE
AUTH
FINISH

and attempt to retransmit them.

The protocol prevents successful replay through fresh session material and transcript-bound authentication.

Servers MAY additionally maintain short-lived replay caches for high-security deployments.

---

101. Reflection Protection

The protocol MUST distinguish:

initiator -> responder

from:

responder -> initiator

through:

- Separate traffic keys.
- Direction-specific KDF labels.
- Explicit roles in the transcript.

---

102. Role Binding

The cryptographic context SHOULD contain:

role = initiator

or:

role = responder

Role information MUST be included in key derivation where needed.

---

103. Cross-Protocol Protection

GSP cryptographic labels MUST contain a GSP-specific domain.

Example:

"GSP/1 handshake"
"GSP/1 traffic"
"GSP/1 finished"

This prevents keys from accidentally being reused across unrelated protocols.

---

104. Version 1.0 Required Features

A GSP 1.0 compliant implementation SHOULD support:

HELLO
HELLO_ACK
KEY_EXCHANGE
KEY_EXCHANGE_ACK
FINISH
FINISH_ACK

and MUST support:

Transcript hashing
Ephemeral key exchange
Key derivation
Key confirmation
Handshake timeout
Handshake failure handling

Authenticated profiles MUST additionally support the applicable "AUTH" messages.

---

105. Recommended GSP 1.0 Cryptographic Profile

Protocol:
    GSP/1

KEX:
    X25519

Hash:
    SHA-256

KDF:
    HKDF-SHA-256

AEAD:
    ChaCha20-Poly1305

Transcript:
    SHA-256

Key confirmation:
    HMAC-SHA-256

Compression:
    None
    LZ4 (optional)

---

106. Example Negotiation

Initiator:

Versions:
    1.0

Cipher:
    CHACHA20-POLY1305

KEX:
    X25519

Auth:
    PUBLIC_KEY

Compression:
    LZ4

Responder selects:

Version:
    1.0

Cipher:
    CHACHA20-POLY1305

KEX:
    X25519

Auth:
    PUBLIC_KEY

Compression:
    LZ4

The resulting parameters become immutable.

---

107. Example Session Derivation

Conceptually:

shared_secret
      |
      v
HKDF-Extract
      |
      v
handshake_secret
      |
      +------------------+
      |                  |
      v                  v
client_write_key    server_write_key
      |                  |
      v                  v
client_write_iv     server_write_iv

Finished keys are derived separately.

---

108. Wire-Level Example

Illustrative only:

C0 01 01 ...

represents the beginning of a GSP handshake frame.

Implementations MUST use the exact binary encoding defined by the GSP Core Wire Specification.

Human-readable diagrams in this document are not themselves the wire format.

---

109. State Transition Table

Current State| Message| Result
"IDLE"| HELLO| "HELLO_SENT" / "HELLO_RECEIVED"
"HELLO_SENT"| HELLO_ACK| "PARAMETERS_NEGOTIATED"
"PARAMETERS_NEGOTIATED"| KEY_EXCHANGE| "KEYS_DERIVED"
"KEYS_DERIVED"| AUTH| "AUTHENTICATING"
"AUTHENTICATING"| AUTH_ACK| "KEY_CONFIRMATION"
"KEY_CONFIRMATION"| FINISH| verify
"KEY_CONFIRMATION"| FINISH_ACK| "ESTABLISHED"
Any| fatal ALERT| "FAILED"
Any| invalid message| "FAILED"

---

110. Mandatory Validation

Before accepting a handshake message, implementations MUST validate:

Message type
Message length
Protocol version
Sequence number
Connection/session ID
State
Required fields
Field lengths
Cryptographic algorithm identifiers
Extension flags
Extension lengths

---

111. Invalid Input Handling

Malformed input MUST NOT cause:

crash
memory corruption
unbounded allocation
secret disclosure
undefined state transitions

---

112. Interoperability

Two compliant implementations MUST produce identical cryptographic results when given identical:

parameters
random values
ephemeral keys
authentication credentials
transcript

Canonical serialization is therefore a mandatory interoperability requirement.

---

113. Test Vectors

The GSP specification SHOULD eventually include public test vectors containing:

Initiator random
Responder random
Initiator private key
Initiator public key
Responder private key
Responder public key
Shared secret
Transcript
Transcript hash
Handshake secret
Traffic keys
Finished values

Private test-vector values MUST be clearly labeled as test-only.

---

114. Negative Test Vectors

Implementations SHOULD test:

Wrong version
Wrong cipher
Wrong KEX
Wrong public key
Modified HELLO
Modified HELLO_ACK
Modified transcript
Modified AUTH
Invalid signature
Invalid MAC
Wrong FINISH
Duplicate message
Replay
Invalid sequence
Oversized message
Unknown mandatory extension
Invalid length
Timeout

---

115. Fuzzing Requirements

The parser SHOULD be fuzz-tested with:

random message types
random lengths
truncated packets
oversized packets
invalid extensions
nested malformed structures
duplicate fields
invalid UTF-8 where applicable
random flags
random sequence numbers

The parser MUST remain memory-safe.

---

116. Cryptographic Library Requirements

GSP implementations SHOULD use mature cryptographic libraries.

The protocol MUST NOT require implementers to write:

ChaCha20
Poly1305
X25519
SHA-256
HKDF

from scratch.

Custom cryptography SHOULD NOT be introduced merely to make GSP "more secure."

---

117. Security Review

Before GSP is considered production-ready, the handshake SHOULD undergo:

Protocol review
Cryptographic review
Implementation audit
Fuzz testing
Interoperability testing
Replay testing
Downgrade testing
MITM testing
DoS testing

---

118. Handshake Security Summary

The intended security construction is:

              GSP HANDSHAKE
                    |
        +-----------+-----------+
        |                       |
   Negotiation              Identity
        |                       |
        v                       v
 Version/Cipher             AUTH
 KEX/Features                 |
        |                     |
        +----------+----------+
                   |
                   v
             Ephemeral KEX
                   |
                   v
             Shared Secret
                   |
                   v
                  KDF
                   |
        +----------+----------+
        |                     |
        v                     v
   Traffic Keys          Finished Keys
        |                     |
        +----------+----------+
                   |
                   v
            Key Confirmation
                   |
                   v
              ESTABLISHED

---

119. Complete Handshake in One Diagram

INITIATOR                                               RESPONDER
   |                                                        |
   | Generate random + ephemeral key                       |
   |                                                        |
   |---------------------- HELLO --------------------------->|
   |                                                        |
   |                                      Validate HELLO    |
   |                                      Select parameters |
   |                                      Generate random   |
   |                                      Generate KEX key  |
   |                                                        |
   |<-------------------- HELLO_ACK ------------------------|
   |                                                        |
   | Validate negotiation                                   |
   |                                                        |
   |------------------- KEY_EXCHANGE ---------------------->|
   |                                                        |
   |                                      Calculate secret  |
   |                                      Derive keys       |
   |                                                        |
   |<---------------- KEY_EXCHANGE_ACK ---------------------|
   |                                                        |
   | Calculate shared secret                                |
   | Derive handshake secret                                |
   | Derive traffic keys                                    |
   |                                                        |
   |------------------------ AUTH -------------------------->|
   |                                                        |
   |                                      Verify identity   |
   |                                                        |
   |<---------------------- AUTH_ACK -----------------------|
   |                                                        |
   | Calculate transcript hash                              |
   | Generate FINISH proof                                  |
   |                                                        |
   |----------------------- FINISH ------------------------>|
   |                                                        |
   |                                      Verify FINISH     |
   |                                      Generate proof    |
   |                                                        |
   |<-------------------- FINISH_ACK -----------------------|
   |                                                        |
   | Verify FINISH_ACK                                      |
   |                                                        |
   |============== SESSION ESTABLISHED ====================|
   |                                                        |
   |---------------------- ENCRYPTED DATA ----------------->|
   |<--------------------- ENCRYPTED DATA ------------------|

---

120. Normative Keywords

The following terminology is normative:

- MUST — required for compliance.
- MUST NOT — prohibited.
- SHOULD — recommended.
- SHOULD NOT — generally discouraged.
- MAY — optional.
- REQUIRED — equivalent to MUST.
- OPTIONAL — equivalent to MAY.

---

121. Final Handshake Contract

A GSP implementation MUST conceptually perform:

DISCOVER
   ↓
NEGOTIATE
   ↓
EXCHANGE EPHEMERAL KEYS
   ↓
DERIVE SHARED SECRET
   ↓
DERIVE SESSION KEYS
   ↓
AUTHENTICATE
   ↓
VERIFY TRANSCRIPT
   ↓
CONFIRM KEYS
   ↓
ESTABLISH SESSION
   ↓
ENCRYPTED DATA

The fundamental rule is:

NO VERIFIED HANDSHAKE
        =
NO SECURE SESSION
        =
NO APPLICATION DATA

---

122. Reference Handshake Profile

For the initial GSP implementation, the recommended baseline is:

┌─────────────────────────────────────────┐
│              GSP/1 HANDSHAKE            │
├─────────────────────────────────────────┤
│ Version Negotiation                     │
│ Cipher Negotiation                      │
│ Capability Negotiation                  │
│ X25519 Ephemeral Key Exchange           │
│ HKDF-SHA-256 Key Derivation             │
│ SHA-256 Transcript                      │
│ ChaCha20-Poly1305 Traffic Encryption    │
│ HMAC-Based Key Confirmation              │
│ Optional Public-Key / PSK Authentication │
│ Optional LZ4 Compression                │
│ Replay Protection                       │
│ Downgrade Protection                    │
│ Forward Secrecy                         │
│ Session ID                              │
│ Directional Traffic Keys                │
└─────────────────────────────────────────┘

End of GSP Handshake Specification (noooo)