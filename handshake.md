# GSP Handshake Protocol Specification

**Globalized Secure Protocol (GSP)**
**Handshake Protocol Specification**
**Version:** Handshake 1.1 (FOAREVAMP) — FIRST OF ALL REVAMP
**Status:** Experimental / Draft
**URI Scheme:** `gsp://`

---

# 1. Abstract

The GSP Handshake establishes a cryptographically protected GSP session between two peers.

The handshake provides:

* Protocol version negotiation
* Capability negotiation
* Cryptographic suite negotiation
* Key exchange
* Session key derivation
* Peer authentication
* Responder authentication
* Initiator authentication
* Transcript integrity
* Downgrade protection
* Replay protection
* Forward secrecy
* Key confirmation
* Session identification
* Optional compression negotiation
* Optional session resumption
* Optional Handshake Context Cache (HCC)
* Optional key updates

GSP 1.1 uses an optimized handshake design in which the Initiator includes its ephemeral key exchange public key in the initial `HELLO` message and the Responder returns its ephemeral public key in `HELLO_ACK`.

This eliminates the separate `KEY_EXCHANGE` round trip used by the original handshake design.

GSP 1.1 additionally introduces the **Handshake Context Cache (HCC)**.

HCC allows previously negotiated handshake information to be referenced instead of retransmitted on subsequent connections.

The first connection may therefore carry more information.

Subsequent connections may use a compact cache reference and fresh cryptographic material.

The HCC compact control reference targets **18 bytes**.

The 18-byte reference is not itself a complete cryptographic handshake.

---

# 2. Security Model

A GSP handshake MUST establish a cryptographically protected session before normal authenticated application data is delivered.

A successful authenticated session provides:

* Confidentiality
* Integrity
* Authentication
* Forward Secrecy
* Replay Resistance
* Downgrade Resistance
* Key Separation
* Session Binding

The exact authentication guarantees depend on the selected authentication mode.

Anonymous mode does not provide authenticated peer identity.

The following invariant applies to authenticated sessions:

```text
NO VERIFIED RESPONDER
        |
        v
NO AUTHENTICATED APPLICATION DATA
```

The Initiator MUST verify the Responder's required authentication proof before sending authenticated application DATA in the optimized 1-RTT flight.

---

# 3. Normative Language

The following keywords are normative:

| Keyword    | Meaning      |
| ---------- | ------------ |
| MUST       | Required     |
| MUST NOT   | Prohibited   |
| REQUIRED   | Same as MUST |
| SHOULD     | Recommended  |
| SHOULD NOT | Discouraged  |
| MAY        | Optional     |

---

# 4. Terminology

| Term               | Definition                                              |
| ------------------ | ------------------------------------------------------- |
| Initiator          | Peer that starts the handshake                          |
| Responder          | Peer that receives the initial handshake                |
| Peer               | Either endpoint                                         |
| Session            | Established GSP connection                              |
| CID                | Connection Identifier                                   |
| SID                | Session Identifier                                      |
| KEX                | Key Exchange                                            |
| AEAD               | Authenticated Encryption with Associated Data           |
| PSK                | Pre-Shared Key                                          |
| KDF                | Key Derivation Function                                 |
| IV                 | Initialization Vector                                   |
| PSIV               | Protocol Session Initialization Vector                  |
| Transcript         | Ordered canonical representation of handshake messages  |
| RTT                | Round Trip Time                                         |
| Rekey              | Replacement of traffic keys                             |
| HCC                | Handshake Context Cache                                 |
| CACHE_ID           | Identifier referencing an HCC context                   |
| Cache Generation   | Version number of a cache context                       |
| Resumption Secret  | Secret used to authenticate and derive resumed sessions |
| Cold Handshake     | Full handshake without a valid HCC context              |
| Warm Handshake     | Handshake using a valid HCC context                     |
| Compact Resumption | HCC-based resumed handshake                             |

---

# 5. Handshake Versions

This specification defines:

```text
GSP/1.1
```

Implementations MAY support earlier versions.

A GSP implementation MUST NOT silently downgrade to an older version.

The selected version MUST be cryptographically bound to the handshake transcript.

---

# 6. Handshake Objectives

A successful handshake MUST establish:

1. A mutually supported protocol version.
2. A mutually supported cryptographic suite.
3. A mutually supported key-exchange algorithm.
4. A mutually supported authentication method.
5. Negotiated transport parameters.
6. Fresh ephemeral key material where forward secrecy is required.
7. A shared secret.
8. Derived traffic keys.
9. Authentication state where required.
10. A verified handshake transcript.
11. Key confirmation.
12. A unique session context.
13. A valid session identifier.
14. Fresh traffic-key state.
15. Appropriate replay protection.

---

# 7. Optimized Handshake

The original GSP handshake required:

```text
HELLO -> HELLO_ACK
KEY_EXCHANGE -> KEY_EXCHANGE_ACK
AUTH -> AUTH_ACK
FINISH -> FINISH_ACK
```

This resulted in four sequential request/response exchanges.

GSP 1.1 moves the ephemeral key exchange into the initial negotiation.

The optimized authenticated handshake is:

```text
Initiator                                      Responder

HELLO
+ X25519 public key
---------------------------------------------->

                         HELLO_ACK
                         + X25519 public key
                         + Responder authentication
                         <----------------------

Verify Responder authentication

Derive shared secret
Derive handshake keys

FINISH
+ Initiator authentication
+ optional encrypted DATA
---------------------------------------------->

                         FINISH_ACK
                         <----------------------

                 SESSION ESTABLISHED
```

The separate `KEY_EXCHANGE` and `KEY_EXCHANGE_ACK` messages are therefore NOT required when using the GSP 1.1 integrated KEX profile.

---

# 8. RTT Characteristics

The optimized handshake allows:

```text
HELLO -> HELLO_ACK
```

to provide the Initiator with:

* selected parameters;
* Responder random;
* Responder ephemeral public key;
* required Responder authentication proof.

After receiving and successfully validating `HELLO_ACK`, the Initiator can derive the required cryptographic state.

The Initiator may then send:

```text
FINISH + DATA
```

in the next flight.

Therefore authenticated first application DATA can travel after approximately:

```text
1 RTT
```

from the beginning of the connection.

The Responder MUST NOT deliver application DATA until the required Initiator authentication and FINISH verification succeed.

---

# 9. 1-RTT vs 0-RTT

GSP 1.1 1-RTT MUST NOT be confused with 0-RTT.

## 1-RTT

The Initiator receives:

```text
HELLO_ACK
```

before transmitting authenticated encrypted application DATA.

The Initiator MUST verify the Responder authentication contained in or bound to `HELLO_ACK` before transmitting such DATA.

## 0-RTT

The Initiator transmits application DATA before receiving the Responder's first response.

0-RTT requires a previously established resumption secret or equivalent mechanism.

0-RTT data is replay-sensitive and MUST NOT be enabled for arbitrary application operations.

---

# 10. Handshake Messages

GSP 1.1 defines:

| Type   | Name       |
| ------ | ---------- |
| `0x01` | HELLO      |
| `0x02` | HELLO_ACK  |
| `0x03` | AUTH       |
| `0x04` | AUTH_ACK   |
| `0x05` | FINISH     |
| `0x06` | FINISH_ACK |
| `0x07` | ALERT      |
| `0x08` | RETRY      |
| `0x09` | CLOSE      |
| `0x0A` | KEY_UPDATE |
| `0x0B` | RESUME     |
| `0x0C` | RESUME_ACK |

The former `KEY_EXCHANGE` messages are retained only as compatibility messages for profiles that explicitly require them.

A GSP/1.1 implementation using integrated KEX MUST NOT require the compatibility KEX exchange.

---

# 11. HELLO

**Direction:**

```text
Initiator -> Responder
```

`HELLO` is the first handshake message.

It advertises:

* Supported GSP versions
* Supported cipher suites
* Supported KEX algorithms
* Supported authentication methods
* Supported compression algorithms
* Capabilities
* Maximum frame size
* Maximum streams
* Random value
* Connection ID
* Ephemeral KEX public key
* Extensions

---

# 12. HELLO Structure

Logical representation:

```text
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
```

The actual binary representation is defined by the GSP canonical serialization rules.

---

# 13. HELLO Random

The Initiator MUST generate a fresh cryptographically secure random value.

Recommended size:

```text
32 bytes
```

The value MUST NOT be reused for independent handshakes.

---

# 14. HELLO_ACK

**Direction:**

```text
Responder -> Initiator
```

The Responder selects the parameters to be used for the session.

Logical structure:

```text
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

    responder_identity
    responder_authentication_proof

    extensions[]
}
```

When the selected authentication mode requires authenticated Responder identity, the following MUST be present:

```text
responder_identity
responder_authentication_proof
```

The Responder MUST generate fresh ephemeral KEX material before sending `HELLO_ACK`.

The authentication proof MUST be generated over a pre-authentication transcript that excludes the proof itself.

---

# 15. Version Negotiation

The Initiator advertises supported versions.

Example:

```text
supported_versions = [
    1.1,
    1.0
]
```

The Responder selects one mutually supported version.

If no compatible version exists:

```text
VERSION_UNSUPPORTED
```

MUST be returned.

The selected version MUST be included in the authenticated transcript.

---

# 16. Cryptographic Suite Negotiation

Recommended initial profile:

```text
GSP-CHACHA20-POLY1305-X25519
```

The recommended profile uses:

```text
KEX:
    X25519

Hash:
    SHA-256

KDF:
    HKDF-SHA-256

AEAD:
    ChaCha20-Poly1305
```

GSP SHOULD use established cryptographic primitives.

GSP MUST NOT require applications to implement cryptographic primitives themselves.

---

# 17. KEX Negotiation

The default GSP 1.1 KEX is:

```text
X25519
```

The Initiator generates:

```text
initiator_private_key
initiator_public_key
```

The Responder generates:

```text
responder_private_key
responder_public_key
```

Private keys MUST remain local.

Only public keys are transmitted.

---

# 18. Integrated X25519 Exchange

The Initiator places:

```text
initiator_public_key
```

inside `HELLO`.

The Responder places:

```text
responder_public_key
```

inside `HELLO_ACK`.

After receiving `HELLO_ACK`, both peers can independently derive:

```text
shared_secret
```

without another network exchange.

---

# 19. Shared Secret

For X25519:

```text
shared_secret =
    X25519(
        initiator_private_key,
        responder_public_key
    )
```

The Responder independently calculates:

```text
shared_secret =
    X25519(
        responder_private_key,
        initiator_public_key
    )
```

The results MUST be identical.

The shared secret MUST NEVER be transmitted.

---

# 20. Invalid KEX

The handshake MUST fail if:

* The public key has an invalid length.
* The public key is malformed.
* The selected KEX is unsupported.
* The KEX operation fails.
* The resulting shared secret is invalid according to the selected KEX profile.

The connection MUST enter the `FAILED` state.

---

# 21. Authentication Modes

GSP supports:

```text
ANONYMOUS
PSK
PUBLIC_KEY
CERTIFICATE
```

Authentication is negotiated during `HELLO`.

The selected authentication mode MUST be transcript-bound.

---

# 22. Anonymous Authentication

Anonymous mode provides cryptographic protection against passive observers but does not authenticate the identity of the peer.

Anonymous mode SHOULD NOT be used for security-sensitive applications.

Under anonymous mode:

* no authenticated peer identity is assumed;
* no responder identity proof is required;
* identity-bound operations MUST NOT assume peer authentication.

Application DATA MUST NOT be attached to the `FINISH` flight under anonymous mode unless a separate profile explicitly defines the security semantics.

---

# 23. PSK Authentication

PSK authentication uses a previously shared secret.

The PSK MUST NOT be transmitted.

Authentication MUST be bound to:

* protocol version;
* negotiated parameters;
* random values;
* ephemeral public keys;
* authentication mode;
* transcript;
* session context.

A raw password MUST NOT be used directly as a cryptographic PSK.

A password-derived key MUST use an appropriate password-based KDF outside the handshake's normal HKDF construction.

---

# 24. Public-Key Authentication

Public-key authentication allows a peer to prove possession of a private signing key.

The authentication signature MUST cover the current handshake context.

The signature MUST NOT be transferable to another handshake.

For authenticated 1-RTT, the Responder's public-key authentication proof MUST be included in `HELLO_ACK` or otherwise cryptographically bound to the `HELLO_ACK` response.

The Initiator MUST verify this proof before sending authenticated application DATA.

---

# 25. Certificate Authentication

Certificate mode MAY use a certificate chain.

Certificate validation is governed by the applicable GSP authentication profile.

Certificate authentication MUST verify:

* Certificate validity
* Signature chain
* Intended identity
* Key usage
* Expiration
* Revocation policy where applicable

The resulting authenticated identity MUST be bound to the handshake.

---

# 26. Responder Authentication Binding

Responder authentication is divided into two transcript stages.

This avoids circular authentication.

## Pre-Authentication Context

The Responder proof is calculated over:

```text
T_pre_auth =
    "GSP-HANDSHAKE-RESPONDER-AUTH"
    ||
    Encode(selected_version)
    ||
    Encode(HELLO)
    ||
    Encode(HELLO_ACK_without_responder_authentication_proof)
```

Then:

```text
pre_auth_transcript_hash =
    SHA-256(T_pre_auth)
```

The Responder authentication proof MUST cover this context.

The proof MUST authenticate, directly or indirectly:

* protocol version;
* selected cipher;
* selected KEX;
* selected authentication mode;
* selected compression;
* relevant capabilities;
* Initiator random;
* Responder random;
* Initiator ephemeral public key;
* Responder ephemeral public key;
* Responder identity;
* connection context.

The authentication proof MUST NOT include itself in the data it authenticates.

---

# 27. Final Transcript

After the Responder authentication proof exists, the complete handshake transcript is constructed.

Conceptually:

```text
T_final =
    "GSP-HANDSHAKE-FINAL"
    ||
    Encode(HELLO)
    ||
    Encode(HELLO_ACK)
    ||
    Encode(AUTH)
    ||
    Encode(AUTH_ACK)
    ||
    Encode(FINISH)
```

Messages not used by the selected profile are omitted.

The complete `HELLO_ACK`, including the Responder authentication proof, MUST be included in the final transcript.

---

# 28. Authentication Verification Gate

The Initiator MUST perform:

```text
Receive HELLO_ACK
        |
        v
Validate parameters
        |
        v
Validate Responder identity
        |
        v
Verify Responder authentication proof
        |
        +---- FAIL ----> HANDSHAKE FAILED
        |
        v
Responder authenticated
        |
        v
Application DATA may be sent
```

This gate is mandatory for authenticated 1-RTT profiles.

The Initiator MUST NOT bypass this gate because encryption keys have already been derived.

Possession of a valid encryption key is not equivalent to authentication of the Responder.

---

# 29. Initiator Authentication

The Initiator MAY authenticate itself through:

* `FINISH`;
* `AUTH`;
* PSK proof;
* public-key signature;
* certificate-based proof.

When Initiator authentication is required, the Responder MUST verify it before delivering application DATA to the application.

Therefore:

```text
Responder receives FINISH + DATA
        |
        v
Verify Initiator authentication
        |
        v
Verify FINISH
        |
        v
Decrypt DATA
        |
        v
Deliver DATA to application
```

---

# 30. AUTH

For authentication profiles requiring an explicit authentication message:

```text
AUTH {
    authentication_method
    identity
    credential
    signature_or_mac
}
```

The credential MUST be authenticated against the current handshake context.

---

# 31. AUTH_ACK

The Responder MAY send:

```text
AUTH_ACK {
    status
    identity
    credential
    signature_or_mac
}
```

If authentication succeeds:

```text
status = SUCCESS
```

Otherwise:

```text
AUTHENTICATION_FAILED
```

MUST be generated.

---

# 32. Integrated Authentication

GSP 1.1 MAY integrate Initiator authentication directly into `FINISH`.

The optimized profile may therefore use:

```text
FINISH {
    authentication_data
    verify_data
}
```

instead of:

```text
AUTH
AUTH_ACK
```

Responder authentication is different.

For authenticated 1-RTT, Responder authentication MUST be available to the Initiator before the Initiator transmits authenticated application DATA.

---

# 33. Transcript

The transcript is the canonical sequence of handshake messages.

It MUST include all security-relevant negotiation and authentication state.

---

# 34. Transcript Domain Separation

The transcript MUST use a GSP-specific context.

Example:

```text
"GSP-HANDSHAKE-FINAL"
```

and:

```text
"GSP-HANDSHAKE-RESPONDER-AUTH"
```

Different cryptographic purposes MUST use different domain labels.

---

# 35. Transcript Hash

The final transcript hash is:

```text
transcript_hash =
    SHA-256(T_final)
```

The hash MUST operate on canonical wire representations.

The transcript MUST NOT be calculated from language-level objects or compiler memory layouts.

---

# 36. Canonical Binary Encoding

All GSP handshake messages MUST have a deterministic binary representation.

The wire format MUST NOT depend on:

* compiler;
* CPU architecture;
* struct padding;
* ABI;
* pointer size;
* native endianness;
* programming language;
* memory alignment.

---

# 37. Integer Encoding

All multi-byte integers MUST use:

```text
Big-Endian
```

also known as:

```text
Network Byte Order
```

This includes:

```text
uint16
uint32
uint64
```

---

# 38. Explicit Integer Sizes

The wire specification MUST use explicit integer widths.

The following are forbidden in wire definitions:

```text
int
long
size_t
unsigned long
pointer
```

Instead:

```text
uint8
uint16
uint32
uint64
```

MUST be used.

---

# 39. Fixed-Length Fields

Cryptographic fields MUST have exact sizes.

Examples:

```text
X25519 public key:
    32 bytes

SHA-256:
    32 bytes

ChaCha20-Poly1305 authentication tag:
    16 bytes
```

Incorrect lengths MUST cause a parsing failure.

---

# 40. Variable-Length Fields

Variable-length fields MUST use explicit lengths.

Conceptually:

```text
+----------+----------------+
| Length   | Value          |
+----------+----------------+
```

The length itself MUST have a defined width and byte order.

---

# 41. No Compiler Padding

Wire serialization MUST NOT use:

```text
sizeof(struct)
```

or equivalent native memory serialization.

Every field MUST be explicitly encoded.

---

# 42. No Implicit Terminators

Binary strings and byte arrays MUST NOT require a trailing NUL byte.

The canonical representation is:

```text
length + bytes
```

unless a specific field explicitly defines another format.

---

# 43. Boolean Encoding

Boolean values MUST be:

```text
0x00 = false
0x01 = true
```

Other values MUST be rejected.

---

# 44. Enumeration Encoding

Algorithm identifiers and other enumerations MUST use explicitly assigned numeric IDs.

Unknown mandatory values MUST cause handshake failure.

---

# 45. Optional Fields

Optional fields MUST have an unambiguous representation.

An absent field and an empty field MUST only be considered equivalent if explicitly defined by that message.

---

# 46. Extension Encoding

Recommended extension structure:

```text
extension {
    type
    flags
    length
    value
}
```

Recommended wire types:

```text
type:
    uint16

flags:
    uint16

length:
    uint32
```

All values use Big-Endian.

---

# 47. Extension Ordering

Extensions MUST appear in ascending numeric order unless an extension specification explicitly defines another ordering.

This ensures deterministic transcript generation.

---

# 48. Duplicate Extensions

Duplicate extensions MUST be rejected unless the extension specification explicitly permits multiple instances.

---

# 49. Unknown Extensions

Unknown optional extensions MAY be ignored.

Unknown mandatory extensions MUST produce:

```text
UNSUPPORTED_EXTENSION
```

---

# 50. Key Derivation

The raw X25519 shared secret MUST NOT directly become an AEAD key.

The recommended KDF is:

```text
HKDF-SHA-256
```

Conceptually:

```text
PRK =
    HKDF-Extract(
        salt,
        shared_secret
    )
```

followed by domain-separated expansion.

---

# 51. Session Salt

A session-specific salt SHOULD be derived from both random values.

Conceptually:

```text
salt =
    SHA-256(
        initiator_random ||
        responder_random
    )
```

---

# 52. Handshake Secret

The handshake secret is derived using a domain-separated label:

```text
"GSP/1.1 handshake"
```

The label MUST be included in the KDF context.

---

# 53. Traffic Keys

Separate traffic keys MUST be derived for each direction.

At minimum:

```text
initiator_write_key
responder_write_key

initiator_write_iv
responder_write_iv
```

---

# 54. Finished Keys

Finished verification keys MUST be separate from traffic encryption keys.

Example labels:

```text
"GSP/1.1 initiator finished"
"GSP/1.1 responder finished"
```

---

# 55. Key Separation

A single cryptographic key MUST NOT be reused for:

* handshake authentication;
* Finished verification;
* Initiator traffic;
* Responder traffic;
* resumption authentication;
* unrelated protocol purposes.

Each purpose MUST use independently derived material.

---

# 56. Role Separation

The cryptographic context MUST distinguish:

```text
INITIATOR
RESPONDER
```

Directional labels MUST be used during key derivation.

This prevents reflection attacks and cross-direction key confusion.

---

# 57. Session Identifier

The session identifier SHOULD be derived from the handshake context.

Conceptually:

```text
SID =
    SHA-256(
        "GSP/1.1 SID" ||
        initiator_random ||
        responder_random ||
        initiator_public_key ||
        responder_public_key ||
        negotiated_parameters
    )
```

The SID is an identifier and MUST NOT be treated as a secret.

---

# 58. AEAD Encryption

The recommended AEAD is:

```text
ChaCha20-Poly1305
```

Every encrypted record requires a unique nonce for its key.

---

# 59. AEAD Nonce Invariant

The following rule is absolute:

```text
A (Key, Nonce) pair MUST NEVER be reused.
```

Nonce reuse under ChaCha20-Poly1305 is catastrophic.

---

# 60. Sequence Numbers

Each traffic direction has its own sequence number:

```text
initiator_send_sequence
responder_send_sequence
```

The counters are independent.

---

# 61. Initial Sequence Number

The first encrypted record MAY use:

```text
sequence_number = 0
```

This is valid.

The security requirement is that the same sequence number MUST NOT be reused with the same traffic key.

---

# 62. Sequence Increment

For every new encrypted record:

```text
sequence_number += 1
```

The sequence number MUST be incremented monotonically under the current key.

---

# 63. Sequence Number Reset

A sequence number MUST NOT be reset while the same traffic key remains active.

A reset is permitted only after a successful key update.

Example:

```text
Key A
sequence 0..N
    |
    v
KEY_UPDATE
    |
    v
Key B
sequence 0
```

---

# 64. Sequence Number Width

GSP 1.1 defines:

```text
uint64
```

for encrypted-record sequence numbers.

Valid values:

```text
0 .. 2^64 - 1
```

---

# 65. Sequence Number Wrap

Sequence numbers MUST NOT wrap.

This is forbidden:

```text
2^64 - 1 -> 0
```

under the same key.

---

# 66. Rekey Before Exhaustion

Implementations SHOULD initiate `KEY_UPDATE` before sequence exhaustion.

If the final sequence value is reached and no safe key update can occur, the connection MUST be terminated.

The implementation MUST NOT wrap the counter.

---

# 67. Nonce Construction

For profiles using a static-IV construction:

```text
nonce =
    static_iv XOR sequence_number_encoded
```

For a 96-bit nonce:

```text
static_iv:
    96 bits

sequence_number:
    64 bits

encoded_sequence:
    96-bit zero-extended value
```

Then:

```text
96-bit static_iv
XOR
96-bit encoded sequence
=
96-bit nonce
```

---

# 68. Nonce Uniqueness

The implementation MUST guarantee that:

```text
same key + same sequence
```

never results in two independently generated encrypted records.

---

# 69. Retransmission

On unreliable transports, retransmission MUST NOT accidentally create nonce reuse.

A retransmitted logical record SHOULD reuse the same already-created ciphertext rather than independently encrypting the same plaintext with the same cryptographic state.

The implementation MUST distinguish:

```text
retransmission of existing ciphertext
```

from:

```text
new encrypted record
```

---

# 70. Receive Sequence Validation

For ordered transports, the receiver SHOULD require:

```text
received_sequence == expected_sequence
```

For unordered transports, GSP MAY use a replay window.

Old or already accepted sequence numbers MUST be rejected.

---

# 71. FINISH

`FINISH` confirms possession of the derived handshake keys.

Conceptually:

```text
FINISH {
    authentication_data
    verify_data
    optional_data
}
```

`optional_data` MUST only be used when the selected authentication profile permits application DATA in this flight.

---

# 72. Finished Verification

Conceptually:

```text
verify_data =
    HMAC(
        finished_key,
        transcript_hash
    )
```

The exact Finished construction MUST be defined by the selected cryptographic profile.

---

# 73. FINISH_ACK

The Responder returns:

```text
FINISH_ACK {
    verify_data
}
```

The Initiator MUST verify this value before considering the session fully established.

---

# 74. Application DATA Security Gate

The optimized GSP 1.1 profile MUST enforce the following sequence:

```text
HELLO
   |
   v
HELLO_ACK
   |
   v
Verify Responder
   |
   v
Derive/use handshake keys
   |
   v
FINISH + DATA
```

The following is prohibited:

```text
HELLO_ACK received
       |
       X
       |
send DATA without verifying Responder
```

Encryption alone is not sufficient to authenticate the Responder.

---

# 75. FINISH + DATA

The Initiator MAY attach encrypted application DATA to the same flight as `FINISH`.

This is the primary GSP 1-RTT latency optimization.

However, the Initiator MUST have successfully verified the Responder authentication required by the selected authentication mode before sending that DATA.

The Responder MUST NOT deliver the DATA to the application until:

1. Required Initiator authentication succeeds.
2. Finished verification succeeds.
3. Transcript verification succeeds.
4. AEAD authentication succeeds.
5. All relevant replay/state checks succeed.

---

# 76. Anonymous DATA

Under ANONYMOUS mode, the Initiator MUST NOT attach application DATA to `FINISH` unless an explicit anonymous-data profile defines the exact semantics.

The default anonymous profile therefore uses:

```text
HELLO
HELLO_ACK
FINISH
FINISH_ACK
DATA
```

This prevents anonymous mode from being confused with an authenticated 1-RTT profile.

---

# 77. Handshake State Machine

```text
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
                  VERIFY RESPONDER
                            |
                    +-------+-------+
                    |               |
                  FAIL            PASS
                    |               |
                    v               v
                 FAILED      +-------------+
                             | KEYS_DERIVED|
                             +------+------+
                                    |
                              FINISH/AUTH
                                    |
                                    v
                           +----------------+
                           | AUTHENTICATING |
                           +-------+--------+
                                   |
                              VERIFY FINISH
                                   |
                                   v
                           +----------------+
                           | KEY_CONFIRMATION|
                           +-------+--------+
                                   |
                              FINISH_ACK
                                   |
                                   v
                           +----------------+
                           |  ESTABLISHED   |
                           +----------------+
```

---

# 78. Initiator Algorithm

The Initiator MUST:

1. Generate a fresh random value.
2. Generate a fresh ephemeral KEX key pair.
3. Construct `HELLO`.
4. Include its ephemeral public key.
5. Send `HELLO`.
6. Receive `HELLO_ACK`.
7. Validate the selected version.
8. Validate the selected cipher.
9. Validate the selected KEX.
10. Validate the authentication method.
11. Validate extensions.
12. Validate the Responder public key.
13. Validate the Responder identity when required.
14. Verify the Responder authentication proof when required.
15. Reject the handshake if required Responder authentication fails.
16. Calculate the shared secret.
17. Construct the pre-authentication context where required.
18. Construct the final canonical transcript.
19. Derive handshake secrets.
20. Derive traffic keys.
21. Construct `FINISH`.
22. Attach encrypted DATA only if the authentication gate has passed.
23. Send `FINISH`.
24. Receive `FINISH_ACK`.
25. Verify `FINISH_ACK`.
26. Transition to `ESTABLISHED`.

---

# 79. Responder Algorithm

The Responder MUST:

1. Receive `HELLO`.
2. Validate the message.
3. Validate all lengths.
4. Validate the protocol version.
5. Select compatible parameters.
6. Generate a fresh random value.
7. Generate a fresh ephemeral KEX key pair.
8. Construct the `HELLO_ACK`.
9. Construct the Responder authentication proof when required.
10. Include the proof in `HELLO_ACK`.
11. Send `HELLO_ACK`.
12. Calculate the shared secret.
13. Construct the canonical transcript.
14. Derive handshake secrets.
15. Derive traffic keys.
16. Receive `FINISH`.
17. Verify Initiator authentication where required.
18. Verify Finished data.
19. Verify transcript state.
20. Authenticate/decrypt DATA.
21. Deliver DATA only after all required validation succeeds.
22. Send `FINISH_ACK`.
23. Transition to `ESTABLISHED`.

---

# 80. Handshake Timeout

Implementations MUST enforce a handshake timeout.

Recommended timers:

```text
HELLO_TIMEOUT
AUTH_TIMEOUT
FINISH_TIMEOUT
TOTAL_HANDSHAKE_TIMEOUT
```

A timeout MUST result in:

```text
HANDSHAKE_TIMEOUT
```

and the session MUST enter `FAILED`.

---

# 81. Retry

A Responder MAY send:

```text
RETRY
```

before performing expensive cryptographic work.

Example:

```text
Initiator -> HELLO
Responder -> RETRY
Initiator -> HELLO + retry_token
Responder -> HELLO_ACK
```

---

# 82. Retry Token

A retry token MAY contain:

* timestamp;
* client binding;
* original random;
* expiration;
* authentication tag.

The token MUST be integrity protected.

The token SHOULD be stateless from the server's perspective.

Tokens MUST expire.

---

# 83. Replay Protection

GSP uses:

* fresh random values;
* fresh ephemeral keys;
* transcript binding;
* session identifiers;
* sequence numbers;
* authentication.

High-security deployments MAY additionally maintain replay caches.

---

# 84. Downgrade Protection

The transcript MUST cover:

* supported versions;
* selected version;
* supported ciphers;
* selected cipher;
* supported KEX;
* selected KEX;
* supported authentication methods;
* selected authentication method;
* supported compression;
* selected compression.

An attacker MUST NOT be able to silently force a weaker negotiated configuration.

---

# 85. Capability Negotiation

Capabilities MAY include:

```text
MULTISTREAM
COMPRESSION
DATAGRAM
MIGRATION
LARGE_FRAMES
WIRELESS_DISPLAY
TERMINAL
FILE_TRANSFER
```

Capabilities MUST be explicitly negotiated.

A peer MUST NOT assume that a capability exists simply because it is implemented locally.

---

# 86. Compression

Supported compression algorithms MAY include:

```text
NONE
LZ4
```

Compression occurs before encryption:

```text
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
```

Encrypted data MUST NOT be compressed.

Handshake compression MUST be limited and MUST NOT be applied to unauthenticated attacker-controlled material when doing so could create a security problem.

---

# 87. Maximum Frame Size

Peers MAY negotiate:

```text
max_frame_size
```

The value MUST respect implementation and transport limits.

An oversized frame MUST be rejected.

---

# 88. Maximum Streams

For stream-capable GSP implementations:

```text
max_streams
```

MAY be negotiated.

The negotiated value becomes active after handshake establishment.

---

# 89. Maximum Handshake Size

Implementations MUST impose a maximum handshake size.

A recommended baseline is:

```text
MAX_HANDSHAKE_SIZE = 64 KiB
```

Implementations MAY choose a different limit.

Unbounded memory allocation based on remote length fields is forbidden.

---

# 90. Fragmentation

Handshake messages MAY be fragmented by the transport.

Fragments MUST be reconstructed before transcript processing.

Transport fragmentation MUST NOT change the logical handshake message.

---

# 91. Transport Independence

The handshake is designed to operate over:

```text
GSP/TCP
GSP/UDP
GSP/QUIC
```

and other GSP-compatible transports.

The handshake MUST NOT assume ordered reliable delivery unless the transport provides it.

---

# 92. Datagram Requirements

For unreliable datagram transports:

* Retransmission MUST be supported.
* Duplicate detection MUST be supported.
* Handshake timeouts MUST exist.
* Sequence state MUST be tracked.
* Fragmentation MUST be bounded.
* Replay protection MUST be enforced.

---

# 93. Duplicate Messages

A duplicate message MAY be accepted when it is a valid retransmission.

A conflicting duplicate MUST terminate the handshake.

The implementation MUST distinguish:

```text
valid retransmission
```

from:

```text
modified duplicate
```

---

# 94. Handshake Errors

GSP defines:

| Code   | Name                      |
| ------ | ------------------------- |
| `0x01` | UNKNOWN_ERROR             |
| `0x02` | INVALID_MESSAGE           |
| `0x03` | INVALID_VERSION           |
| `0x04` | VERSION_UNSUPPORTED       |
| `0x05` | INVALID_CIPHER            |
| `0x06` | CIPHER_UNSUPPORTED        |
| `0x07` | INVALID_KEX               |
| `0x08` | KEX_FAILED                |
| `0x09` | AUTHENTICATION_FAILED     |
| `0x0A` | INVALID_SIGNATURE         |
| `0x0B` | INVALID_MAC               |
| `0x0C` | TRANSCRIPT_MISMATCH       |
| `0x0D` | KEY_CONFIRMATION_FAILED   |
| `0x0E` | INVALID_EXTENSION         |
| `0x0F` | UNSUPPORTED_EXTENSION     |
| `0x10` | INVALID_STATE             |
| `0x11` | TIMEOUT                   |
| `0x12` | REPLAY_DETECTED           |
| `0x13` | DOWNGRADE_DETECTED        |
| `0x14` | FRAME_TOO_LARGE           |
| `0x15` | INVALID_LENGTH            |
| `0x16` | NONCE_REUSE               |
| `0x17` | SEQUENCE_EXHAUSTED        |
| `0x18` | INTERNAL_ERROR            |
| `0x19` | CACHE_MISS                |
| `0x1A` | CACHE_EXPIRED             |
| `0x1B` | CACHE_INVALID             |
| `0x1C` | CACHE_REVOKED             |
| `0x1D` | CACHE_GENERATION_MISMATCH |
| `0x1E` | CACHE_CONTEXT_MISMATCH    |
| `0x1F` | RESUMPTION_AUTH_FAILED    |
| `0x20` | CACHE_REPLAY_DETECTED     |

---

# 95. ALERT

Logical structure:

```text
ALERT {
    severity
    error_code
    diagnostic_data
}
```

Severity:

```text
WARNING
FATAL
```

Fatal handshake errors MUST terminate the handshake.

---

# 96. Diagnostic Data

Diagnostic data MUST NOT contain:

* private keys;
* shared secrets;
* PSKs;
* traffic keys;
* plaintext credentials;
* sensitive application data.

Production servers SHOULD avoid exposing detailed cryptographic failure information to unauthenticated clients.

---

# 97. Invalid State

Messages received in an invalid state MUST produce:

```text
INVALID_STATE
```

Examples:

```text
DATA before key confirmation
AUTH before KEX
FINISH before required key derivation
FINISH_ACK before FINISH
```

---

# 98. Denial-of-Service Protection

The Responder SHOULD perform cheap validation before expensive cryptographic operations.

Recommended order:

```text
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
```

---

# 99. Rate Limiting

Servers SHOULD rate-limit handshake attempts.

Possible limits include:

* source address;
* connection identifier;
* authentication identity;
* global server rate.

---

# 100. Forward Secrecy

The default X25519 profile uses ephemeral keys.

The private ephemeral keys SHOULD be securely erased after key establishment.

Compromise of a long-term authentication key SHOULD NOT expose previous sessions when ephemeral forward secrecy is correctly implemented.

An HCC implementation MUST NOT reuse old ephemeral X25519 private keys solely to reduce handshake size.

---

# 101. Secret Erasure

After key establishment, implementations SHOULD erase:

* ephemeral private key;
* raw shared secret;
* temporary KDF state;
* unused handshake secrets.

Only required session state should remain.

---

# 102. Key Update

GSP supports optional:

```text
KEY_UPDATE
```

for traffic key rotation.

A successful key update MUST produce new traffic keys.

The old keys MUST NOT be reused after retirement.

---

# 103. Key Update Sequence Reset

After a successful key update:

```text
old key
sequence = N

        ↓

KEY_UPDATE

        ↓

new key
sequence = 0
```

Resetting the sequence number is safe because the encryption key has changed.

---

# 104. Key Update Failure

If a safe key update cannot be completed before sequence exhaustion:

```text
CLOSE
```

MUST occur.

The implementation MUST NEVER allow sequence-number wrap.

---

# 105. Session Resumption

GSP MAY support session resumption.

A resumption ticket MUST NOT contain raw traffic keys.

The ticket SHOULD contain or represent protected resumption state.

Resumption MUST establish fresh session traffic keys.

---

# 106. RESUME

Conceptually:

```text
RESUME {
    ticket
    random
    supported_parameters
}
```

The Responder validates the ticket and establishes fresh session keys.

---

# 107. RESUME_ACK

Conceptually:

```text
RESUME_ACK {
    selected_parameters
    random
    ephemeral_key
    authentication_proof
}
```

The resumed session SHOULD use fresh ephemeral key material where forward secrecy is required.

---

# 108. 0-RTT Resumption

0-RTT MAY be implemented using a previously established resumption secret.

0-RTT MUST be explicitly enabled by the application/profile.

Replay-sensitive operations MUST NOT be sent using unrestricted 0-RTT.

0-RTT does not replace Responder authentication.

An application MUST understand that the first 0-RTT DATA flight occurs before the current Responder authentication response.

---

# 109. Handshake Context Cache

GSP 1.1 introduces the:

```text
Handshake Context Cache
```

or:

```text
HCC
```

HCC stores previously negotiated and authenticated handshake context.

Its purpose is to avoid retransmitting information that both peers already possess.

The first connection may therefore be larger.

Subsequent connections may use a compact reference.

---

# 110. HCC Design Principle

The fundamental HCC rule is:

```text
DO NOT SEND AGAIN
WHAT BOTH SIDES ALREADY KNOW
```

A cold connection:

```text
negotiate
authenticate
establish
cache
```

A warm connection:

```text
reference
prove possession
derive fresh keys
establish
```

HCC is an optimization layer.

It does not replace cryptographic authentication.

---

# 111. HCC Context

A cache entry SHOULD contain:

```text
cache_id
cache_generation

hcc_version

protocol_version

cipher_suite
key_exchange_suite
authentication_mode
compression_mode

capabilities

peer_identity
peer_identity_binding

context_hash

resumption_secret

created_at
last_used_at
expires_at
idle_expires_at

credential_generation
protocol_generation
```

Optional fields MAY include:

```text
transport_profile
application_profile
extension_set
resumption_policy
0rtt_policy
```

---

# 112. HCC Context Hash

The cache context MUST have a canonical representation.

Then:

```text
context_hash =
    SHA-256(
        canonical_context
    )
```

Security-sensitive context changes MUST produce a different context hash.

---

# 113. CACHE_ID

`CACHE_ID` is an opaque identifier.

The compact baseline representation is:

```text
64 bits
```

`CACHE_ID` MUST NOT be considered a secret.

Knowledge of `CACHE_ID` alone MUST NOT authorize resumption.

High-scale or high-security deployments MAY use larger identifiers.

---

# 114. Cache Generation

Every HCC entry has a generation value.

The compact baseline uses:

```text
64 bits
```

Example:

```text
CACHE_ID = A71C92E41F0082B3
GENERATION = 4
```

After invalidation or security-sensitive replacement:

```text
GENERATION = 5
```

Older generations MUST be rejected.

---

# 115. 18-Byte HCC Compact Reference

The GSP HCC compact profile defines a complete 18-byte control message.

The compact representation is:

```text
+--------+--------+----------------+------------------+----------+
| Type   | Flags  | CACHE_ID       | Generation       | Auth Tag |
| 1 byte | 1 byte | 6 bytes        | 4 bytes          | 6 bytes  |
+--------+--------+----------------+------------------+----------+

Total: 18 bytes
```

Field sizes:

```text
Type:
    1 byte

Flags:
    1 byte

CACHE_ID:
    6 bytes

Generation:
    4 bytes

Compact Authenticator:
    6 bytes
```

Total:

```text
1 + 1 + 6 + 4 + 6 = 18 bytes
```

This is the actual wire size of the compact HCC control message.

The 18-byte message does not contain an X25519 public key.

The compact profile therefore does not perform a fresh asymmetric key exchange on every resumed connection.

---

# 116. Compact HCC Security Model

The compact HCC profile uses a symmetric ratchet derived from previously established resumption state.

The basic model is:

```text
Full Handshake
      |
      v
Resumption Root
      |
      v
HCC Context
      |
      v
Symmetric Ratchet
      |
      +--> Generation 1
      +--> Generation 2
      +--> Generation 3
      +--> ...
```

Each generation produces fresh cryptographic session state.

The previous ratchet state SHOULD be securely erased after successful advancement.

The compact profile therefore provides:

* low handshake size;
* low computational overhead;
* replay resistance through generation state;
* proof of possession of the resumption secret;
* protection of previous ratchet states when securely erased.

The compact ratchet MUST NOT be described as equivalent to full per-session X25519 Forward Secrecy.

---

# 117. HCC Security Trade-Off

The compact HCC profile intentionally trades fresh asymmetric key exchange on every connection for a symmetric ratchet.

This produces the following security model:

```text
Past sessions:
    Protected when previous ratchet states are erased.

Current session:
    Protected by the current ratchet state.

Future sessions:
    Depend on the current ratchet state until reanchoring.
```

Therefore compromise of the current ratchet state may allow derivation of subsequent sessions until a successful asymmetric reanchor occurs.

Implementations requiring continuous per-session PFS MUST use the full/reanchor profile instead of the compact ratchet profile.

---

# 118. HCC Reanchoring

The compact ratchet MUST periodically be reanchored using a fresh asymmetric key exchange.

The recommended reanchor mechanism is:

```text
X25519
```

A reanchor establishes a new independent ratchet root.

Conceptually:

```text
Old Ratchet
     |
     v
Generation N
     |
     v
REANCHOR
     |
     +--> fresh X25519
     |
     v
New Ratchet Root
     |
     v
Generation 0
```

The old ratchet state MUST NOT be used after successful reanchoring.

---

# 119. Reanchor Policy

Implementations MUST define a reanchor policy.

A reanchor MAY be triggered by:

```text
maximum number of compact sessions
maximum ratchet age
administrative policy
credential rotation
security event
suspected compromise
explicit peer request
protocol policy
```

The exact threshold MAY be implementation-defined.

Implementations SHOULD avoid allowing a single ratchet chain to remain active indefinitely.

---

# 120. HCC Context

A cache entry SHOULD contain:

```text
cache_id
cache_generation

hcc_version

protocol_version

cipher_suite
key_exchange_suite
authentication_mode
compression_mode

capabilities

peer_identity
peer_identity_binding

context_hash

resumption_secret
ratchet_secret

created_at
last_used_at
expires_at
idle_expires_at

credential_generation
protocol_generation

reanchor_policy
```

The cache MUST NOT contain old traffic keys.

The cache MUST NOT contain old X25519 ephemeral private keys.

---

# 121. HCC Context Hash

The cache context MUST have a canonical representation.

Then:

```text
context_hash =
    SHA-256(
        canonical_context
    )
```

The context hash MUST be cryptographically bound to the resumption and ratchet state.

Security-sensitive context changes MUST produce a different context hash.

---

# 122. CACHE_ID

`CACHE_ID` is an opaque cache identifier.

The compact profile uses:

```text
48 bits
```

The resulting identifier space is:

```text
2^48
```

`CACHE_ID` MUST NOT be considered secret.

Knowledge of `CACHE_ID` alone MUST NOT authorize a resumed session.

The reduced 48-bit identifier is an engineering trade-off required by the 18-byte compact format.

Deployments requiring stronger identifier-space properties MAY use an extended HCC format.

---

# 123. Cache Generation

The compact profile uses:

```text
uint32
```

for the generation field.

The generation identifies the current ratchet position.

Example:

```text
CACHE_ID:
    A71C92E41F00

Generation:
    00000004
```

A successful compact connection advances the ratchet.

For example:

```text
Generation 4
     |
     v
Generation 5
```

An already-consumed generation MUST NOT be accepted again.

---

# 124. HCC Ratchet Root

The initial HCC ratchet root is derived from the resumption state created by the full handshake.

Conceptually:

```text
ratchet_secret =
    HKDF-Expand(
        resumption_secret,
        "GSP/1.1 HCC RATchet Root" ||
        context_hash,
        32
    )
```

The resulting value becomes the initial ratchet state.

The exact HKDF schedule MUST use domain separation.

---

# 125. Symmetric Ratchet Advancement

For each new compact session:

```text
next_ratchet_secret =
    HKDF-Expand(
        current_ratchet_secret,
        "GSP/1.1 HCC RATchet" ||
        Encode(generation),
        32
    )
```

The new ratchet secret MUST be used for the new generation.

After successful advancement:

```text
current_ratchet_secret
        |
        v
securely erase
```

The implementation SHOULD ensure that old ratchet state cannot be recovered from ordinary memory.

---

# 126. Session Secret Derivation

The new session secret is derived independently from the ratchet advancement.

Conceptually:

```text
session_secret =
    HKDF-Expand(
        next_ratchet_secret,
        "GSP/1.1 HCC Session" ||
        Encode(generation) ||
        context_hash,
        32
    )
```

The session secret MUST NOT be used directly as an AEAD key.

It MUST be expanded into purpose-specific key material.

---

# 127. Compact Authentication Tag

The compact HCC message contains a truncated authenticator.

The authenticator is:

```text
compact_authenticator =
    Truncate_48(
        HMAC-SHA-256(
            current_ratchet_secret,
            "GSP/1.1 HCC Compact" ||
            cache_id ||
            generation ||
            context_hash
        )
    )
```

Where:

```text
Truncate_48
```

returns the first 6 bytes of the HMAC result.

The compact authenticator exists primarily as an early validation gate.

It MUST NOT be considered equivalent to full Finished authentication.

---

# 128. Compact Authentication Purpose

The compact authenticator allows the Responder to reject unauthorized requests before performing expensive operations.

The intended processing order is:

```text
18-byte request
      |
      v
Parse
      |
      v
Cache lookup
      |
      v
Generation validation
      |
      v
Compact authenticator
      |
      +---- FAIL ---> DROP / REJECT
      |
      v
Ratchet processing
      |
      v
Session key derivation
      |
      v
FINISH
```

An attacker who knows only `CACHE_ID` MUST NOT be able to pass this gate.

---

# 129. Compact Authenticator Security Level

The compact authenticator has:

```text
48-bit truncated output
```

This provides approximately:

```text
1 / 2^48
```

probability of random forgery per independent attempt.

The value is intentionally truncated to satisfy the 18-byte wire budget.

The compact authenticator is therefore an **early anti-DoS and authorization filter**, not the final authentication mechanism.

The full `FINISH` verification remains mandatory.

High-security deployments MAY use an extended HCC format with a longer authenticator.

---

# 130. Compact HCC Wire Format

The complete compact request is:

```text
Compact HCC Request:

Type:
    1 byte

Flags:
    1 byte

CACHE_ID:
    6 bytes

Generation:
    4 bytes

Compact Authenticator:
    6 bytes
```

Total:

```text
18 bytes
```

No additional random field is required by the baseline compact profile.

Freshness is provided by the monotonically advancing generation and ratchet state.

Profiles requiring explicit per-session randomness MUST use an extended HCC format or reanchor handshake.

---

# 131. No X25519 in Baseline Compact Mode

The baseline compact HCC profile does not transmit:

```text
X25519 public key
```

in the 18-byte request.

This is intentional.

An X25519 public key requires:

```text
32 bytes
```

and therefore cannot fit inside the 18-byte budget.

The compact profile instead derives fresh session state from the symmetric HCC ratchet.

---

# 132. Freshness in Compact Mode

The compact profile obtains session freshness from:

```text
CACHE_ID
+
Generation
+
Ratchet state
```

Each generation MUST be used at most once.

A generation that has already been consumed MUST NOT be accepted again.

The same compact request therefore cannot establish an unlimited number of equivalent sessions.

---

# 133. Compact Replay Protection

The Responder MUST maintain sufficient state to determine whether a compact generation has already been consumed.

At minimum:

```text
current_generation
current_ratchet_secret
```

MUST be maintained.

A request with:

```text
generation < current_generation
```

MUST be rejected.

A request with:

```text
generation == current_generation
```

MUST only be accepted if the protocol state explicitly permits that generation to be consumed.

A request with:

```text
generation > current_generation
```

MUST NOT be accepted blindly.

Implementations MAY permit only:

```text
generation == current_generation + 1
```

for sequential ratchet advancement.

---

# 134. Atomic Ratchet Advancement

Ratchet advancement MUST be atomic with respect to connection state.

The implementation MUST avoid:

```text
derive generation N+1
send response
crash
restore generation N
```

because this could allow generation reuse.

Implementations SHOULD persist or otherwise safely commit ratchet state when persistence is required.

---

# 135. Failed Compact Authentication

If the compact authenticator fails:

```text
RESUMPTION_AUTH_FAILED
```

MAY be returned.

For publicly exposed services, implementations SHOULD prefer silently dropping or rate-limiting invalid requests where revealing cache state would aid attackers.

The Responder MUST NOT:

* generate fresh X25519 keys;
* generate expensive signatures;
* create new cache entries;
* derive expensive session state;

before the compact authentication gate succeeds.

---

# 136. HCC Lookup Order

The Responder MUST process compact requests in the following order:

```text
1. Parse
2. Validate fixed length
3. Validate type
4. Validate flags
5. Lookup CACHE_ID
6. Validate cache integrity
7. Validate generation
8. Validate expiration
9. Validate revocation
10. Validate context compatibility
11. Validate compact authenticator
12. Advance ratchet state
13. Derive session keys
14. Continue handshake
```

Expensive asymmetric operations MUST NOT occur before step 11 in the baseline compact profile.

---

# 137. HCC Authentication Gate

A successful compact authenticator means:

```text
Peer possesses valid ratchet state
```

It does not by itself mean:

```text
Application authenticated
```

The complete sequence remains:

```text
Compact Authenticator
        |
        v
Ratchet Authentication
        |
        v
Session Key Derivation
        |
        v
Responder Authentication
        |
        v
FINISH
        |
        v
Application DATA
```

---

# 138. Responder Authentication During Compact Resumption

The Responder MUST authenticate the resumed session using the identity and authentication state stored in the HCC context.

The Responder authentication MUST be bound to:

```text
CACHE_ID
generation
context_hash
peer identity
new session state
protocol version
selected parameters
```

The Initiator MUST verify the Responder authentication before sending authenticated application DATA.

Therefore:

```text
COMPACT REQUEST
       |
       v
COMPACT ACK
       |
       v
VERIFY RESPONDER
       |
       v
FINISH + DATA
```

is the permitted authenticated flow.

---

# 139. Compact Finished Verification

The resumed session MUST perform a complete Finished exchange.

The Finished value MUST prove possession of the derived session state.

Conceptually:

```text
finished_key =
    HKDF-Expand(
        session_secret,
        "GSP/1.1 HCC Finished",
        32
    )
```

Then:

```text
verify_data =
    HMAC(
        finished_key,
        transcript_hash
    )
```

The exact Finished construction MUST be defined by the selected cryptographic profile.

---

# 140. Compact Resumption Flow

```text
Initiator                                      Responder

18-byte HCC request
---------------------------------------------->

                         Parse
                         Cache lookup
                         Generation check
                         Context validation
                         Compact authenticator

                         If invalid:
                         DROP / REJECT

                         If valid:
                         Advance ratchet
                         Derive session state
                         Authenticate responder

COMPACT_ACK
+ authenticated session context
<----------------------------------------------

Verify Responder authentication

Derive session keys

FINISH
+ Initiator authentication
+ Finished
+ optional encrypted DATA
---------------------------------------------->

                         Verify Initiator
                         Verify Finished
                         Verify transcript
                         Decrypt DATA

FINISH_ACK
<----------------------------------------------

Verify FINISH_ACK

             ESTABLISHED
```

---

# 141. Compact DATA Security Gate

The Initiator MUST NOT send authenticated application DATA until:

```text
Responder authentication
        +
required parameter validation
```

has succeeded.

The Responder MUST NOT deliver application DATA until:

```text
Initiator authentication
        +
Finished verification
        +
AEAD verification
        +
replay validation
```

have succeeded.

---

# 142. HCC Reanchor Handshake

When the reanchor policy requires fresh asymmetric key material, the peers perform a normal cryptographic handshake.

The reanchor MUST generate:

```text
fresh X25519 initiator key
fresh X25519 responder key
fresh handshake state
fresh resumption root
fresh ratchet root
```

The new root MUST NOT be derived solely from the old ratchet secret.

The purpose of reanchoring is to introduce new independent asymmetric entropy.

---

# 143. Reanchor Key Schedule

Conceptually:

```text
fresh X25519 shared_secret
          +
authenticated HCC context
          |
          v
HKDF
          |
          +--> new handshake_secret
          |
          +--> new traffic keys
          |
          +--> new resumption_secret
          |
          +--> new ratchet_secret
```

The new ratchet is therefore cryptographically reanchored.

---

# 144. Reanchor Security Property

If an attacker compromises the old ratchet state before reanchoring, successful reanchoring prevents the attacker from deriving the new ratchet solely from the compromised state.

The attacker would additionally require the new handshake's cryptographic secrets.

This restores the stronger Forward Secrecy properties of the X25519 profile.

---

# 145. Compact vs Reanchor Profiles

GSP defines two HCC operating profiles.

## Compact Profile

```text
18-byte control request
symmetric ratchet
no fresh X25519 per connection
very low overhead
```

Security:

```text
Past sessions:
    protected if old states erased

Future sessions:
    dependent on current ratchet until reanchor
```

## Reanchor Profile

```text
full cryptographic handshake
fresh X25519
new independent ratchet root
strong PFS restoration
larger handshake
```

The implementation MAY switch automatically according to policy.

---

# 146. Recommended HCC Lifecycle

```text
FULL HANDSHAKE
      |
      v
NEW RESUMPTION SECRET
      |
      v
NEW HCC RATCHET
      |
      v
COMPACT GENERATION 0
      |
      v
COMPACT GENERATION 1
      |
      v
COMPACT GENERATION 2
      |
      v
...
      |
      v
REANCHOR
      |
      v
NEW X25519
      |
      v
NEW RATCHET ROOT
```

---

# 147. HCC Expiration

Each cache entry MUST contain:

```text
created_at
expires_at
```

An implementation MAY additionally maintain:

```text
idle_expires_at
```

Expired cache state MUST NOT be used.

Expiration MUST result in:

```text
FULL HANDSHAKE
```

unless an independently valid recovery mechanism exists.

---

# 148. HCC Invalidation

A cache entry MUST be invalidated when required by security policy.

Examples:

```text
credential rotation
identity change
protocol incompatibility
cipher change
KEX policy change
authentication policy change
explicit revocation
suspected compromise
ratchet corruption
generation exhaustion
administrative deletion
```

---

# 149. Generation Exhaustion

Because the compact generation uses:

```text
uint32
```

the generation MUST NOT wrap.

Before reaching:

```text
2^32 - 1
```

the implementation MUST perform a reanchor or replace the cache context.

The following is forbidden:

```text
0xFFFFFFFF -> 0x00000000
```

under the same ratchet root.

---

# 150. Cache Poisoning Protection

Remote peers MUST NOT be allowed to overwrite trusted HCC state directly.

A cache entry MUST only be created or replaced after successful authentication.

The following is forbidden:

```text
remote CACHE_ID
      |
      v
overwrite trusted cache
```

without cryptographic authorization.

---

# 151. Cache Storage

HCC MAY be stored in:

```text
memory
local files
database
secure platform storage
hardware-backed storage
```

Filesystem storage is an implementation detail.

The GSP wire protocol does not define a required cache file format.

---

# 152. Cache Secret Protection

The following values are sensitive:

```text
resumption_secret
ratchet_secret
session derivation state
ticket encryption state
```

They SHOULD be protected at rest.

Implementations SHOULD use secure platform storage when available.

---

# 153. Cache State That MUST NOT Be Reused

HCC MUST NOT retain for future session encryption:

```text
old traffic keys
old traffic IVs
old traffic sequence numbers
old X25519 private keys
old Finished keys
```

Only state required to derive a new session MUST remain.

---

# 154. HCC Context Binding

The ratchet state MUST be bound to:

```text
protocol version
cipher suite
KEX policy
authentication mode
compression mode
peer identity
context_hash
```

A cached ratchet MUST NOT silently migrate to an incompatible security context.

---

# 155. HCC Parameter Change

If any security-sensitive parameter changes:

```text
cached context
      |
      X
      |
full/reanchor handshake
```

must occur.

Examples:

```text
cipher
authentication
protocol version
peer identity
security policy
KEX policy
```

---

# 156. HCC Identity Binding

The identity stored in the HCC context MUST be cryptographically bound to the resumption state.

A client-supplied identity string MUST NOT be accepted as proof of identity.

---

# 157. HCC and GSPID

GSPID identity MAY be associated with an HCC context.

However:

```text
CACHE_ID != GSPID identity
```

GSPID MUST only consider the identity authenticated after successful cryptographic verification.

---

# 158. HCC and Application Profiles

HCC MAY bind state to an application profile.

Examples:

```text
GSP Terminal
GSPID
GSPWD
GSPMAIL
```

A cache authorized for one application profile MUST NOT automatically authorize another security-sensitive profile.

---

# 159. HCC and Transport

HCC SHOULD remain transport-independent.

A cache MAY be reused across:

```text
GSP/TCP
GSP/UDP
GSP/QUIC
```

only when the cached policy explicitly permits it.

Transport-specific state MUST NOT be blindly reused across transports.

---

# 160. HCC Cache Miss

If no valid cache entry exists:

```text
CACHE_MISS
```

MAY be returned.

The implementation SHOULD fall back to:

```text
FULL HANDSHAKE
```

rather than treating cache failure as permanent connection failure.

---

# 161. HCC Failure Flow

```text
COMPACT REQUEST
      |
      v
CACHE LOOKUP
      |
      +---- MISS ------------+
      |                      |
      +---- EXPIRED ---------+
      |                      |
      +---- REVOKED ---------+
      |                      |
      +---- INVALID ---------+
      |                      |
      +---- BAD GENERATION --+
      |                      |
      +---- BAD AUTH --------+
      |                      |
      v                      v
VALID CACHE             FULL HANDSHAKE
      |
      v
RATCHET AUTH
      |
      +---- FAIL ---> REJECT
      |
      v
SESSION
```

---

# 162. HCC Replay Protection

A captured 18-byte compact request MUST NOT be sufficient to establish a new session.

Replay protection is based on:

```text
generation
ratchet state
compact authenticator
Finished verification
```

A consumed generation MUST NOT be accepted again.

A replayed compact request therefore fails before a second equivalent session can be established.

---

# 163. HCC Amplification Protection

The compact authenticator exists specifically to preserve the DoS protection requirement from Section 98.

The Responder MUST NOT perform expensive cryptographic work for an unauthenticated compact request.

The preferred order is:

```text
Parse
 ↓
Length
 ↓
Version
 ↓
Flags
 ↓
Cache lookup
 ↓
Generation
 ↓
Context
 ↓
Compact authenticator
 ↓
Rate limiting / policy
 ↓
Ratchet processing
 ↓
Expensive cryptography
```

The exact order of rate limiting and compact authentication MAY be implementation-defined, provided that expensive cryptographic operations remain behind cheap validation.

---

# 164. HCC Rate Limiting

Servers SHOULD rate-limit:

```text
invalid CACHE_ID
invalid generation
invalid compact authenticator
replayed generation
expired cache requests
```

The server MAY rate-limit independently of normal handshake traffic.

---

# 165. HCC Enumeration Protection

Because:

```text
CACHE_ID
```

is not secret, implementations MUST assume that an attacker may obtain or enumerate identifiers.

The compact authenticator prevents possession of the identifier from being sufficient to pass the cryptographic gate.

Deployments requiring stronger anti-enumeration properties MAY use:

```text
128-bit CACHE_ID
encrypted tickets
authenticated tickets
extended HCC format
```

---

# 166. HCC Stateless Tickets

An implementation MAY use stateless encrypted tickets instead of server-side cache storage.

A ticket MUST provide:

```text
integrity
confidentiality where required
expiration
context binding
authentication
```

The ticket itself is not equivalent to a plaintext `CACHE_ID`.

---

# 167. HCC Cache Refresh

A successful compact session MAY update:

```text
last_used_at
expiration
ratchet state
```

A refresh MUST NOT bypass authentication.

A cache MUST NOT be indefinitely extended without respecting reanchor and security policy.

---

# 168. HCC Credential Rotation

When long-term credentials change:

```text
old HCC
    |
    X
```

The implementation MUST perform a full or reanchor authentication process.

A new HCC context MAY then be created.

---

# 169. HCC Clock Handling

Expiration SHOULD use a monotonic clock for local duration calculations where available.

Wall-clock timestamps MAY be retained for diagnostics.

Clock rollback MUST NOT make expired security state valid again.

---

# 170. HCC State Machine

```text
                 +-------+
                 | START |
                 +---+---+
                     |
                     v
              +-------------+
              | CACHE LOOKUP|
              +------+------+
                     |
                +----+----+
                |         |
              MISS       HIT
                |         |
                v         v
             FULL      VALIDATE
          HANDSHAKE       |
                |         v
                |    COMPACT AUTH
                |         |
                |     +---+---+
                |     |       |
                |   FAIL     PASS
                |     |       |
                |     v       v
                |   REJECT   RATCHET
                |             |
                +-------------+
                       |
                       v
                    FINISH
                       |
                       v
                  ESTABLISHED
```

---

# 171. HCC State Rules

```text
START
 -> CACHE_LOOKUP

CACHE_LOOKUP
 -> CACHE_VALIDATED
 -> FULL_HANDSHAKE

CACHE_VALIDATED
 -> COMPACT_AUTHENTICATION
 -> FULL_HANDSHAKE

COMPACT_AUTHENTICATION
 -> RATCHET_ADVANCEMENT
 -> FAILED

RATCHET_ADVANCEMENT
 -> RESPONDER_AUTHENTICATION
 -> FAILED

RESPONDER_AUTHENTICATION
 -> FINISH
 -> FAILED

FINISH
 -> ESTABLISHED
 -> FAILED
```

---

# 172. Complete Compact Handshake — 18 Bytes

The baseline compact HCC handshake begins with an actual 18-byte request:

```text
Initiator                                      Responder

18-byte HCC Compact Request
+-------------------------------------------------------+
| Type | Flags | CACHE_ID | Generation | Authenticator |
+-------------------------------------------------------+
-------------------------------------------------------->

                         Parse
                         Length validation
                         Cache lookup
                         Generation validation
                         Context validation
                         Compact authenticator

                         If invalid:
                         DROP / REJECT

                         If valid:
                         Advance ratchet
                         Derive session state
                         Prepare authenticated response

COMPACT_ACK
+ responder authentication
+ session parameters
<--------------------------------------------------------

Verify Responder authentication

Derive traffic keys

FINISH
+ Initiator authentication
+ Finished
+ optional encrypted DATA
-------------------------------------------------------->

                         Verify Initiator
                         Verify Finished
                         Verify transcript
                         Verify AEAD
                         Deliver DATA

FINISH_ACK
<--------------------------------------------------------

Verify FINISH_ACK

                    ESTABLISHED
```

The first message is exactly:

```text
18 bytes
```

in the baseline compact profile.

---

# 173. Complete Reanchor Handshake

When reanchoring is required:

```text
Initiator                                      Responder

HELLO
+ fresh random
+ fresh X25519 public key
---------------------------------------------->

                         HELLO_ACK
                         + fresh random
                         + fresh X25519 public key
                         + responder authentication
                         <----------------------

Verify Responder

Calculate fresh X25519 shared secret

Derive new handshake state

FINISH
+ authentication
+ Finished
---------------------------------------------->

                         Verify FINISH
                         Verify Initiator
                         Verify transcript

                         FINISH_ACK
                         <----------------------

ESTABLISHED
        |
        v
NEW HCC RATCHET ROOT
```

The new ratchet root MUST be independent of the old ratchet state except for explicitly authenticated context binding.

---

# 174. Full vs Compact HCC

The GSP implementation SHOULD support:

```text
FULL
COMPACT
REANCHOR
```

Conceptually:

```text
FULL
 |
 +--> create HCC
       |
       +--> COMPACT
       |      |
       |      +--> COMPACT
       |      +--> COMPACT
       |      +--> ...
       |
       +--> REANCHOR
              |
              +--> new HCC root
```

---

# 175. HCC Security Properties

The compact HCC design provides:

```text
18-byte initial control message
symmetric key ratcheting
cheap authentication filtering
generation-based replay protection
fresh session-derived keys
periodic asymmetric reanchoring
```

It does not provide:

```text
fresh X25519 PFS for every compact connection
```

unless the connection uses the reanchor profile.

---

# 176. Security Comparison

The security properties can be summarized as:

```text
                 Compact HCC       Reanchor
------------------------------------------------
18-byte request       YES             NO
Symmetric ratchet     YES             YES
Fresh X25519          NO              YES
Past-state protection YES*            YES
Future-state PFS      NO*             YES
Low CPU cost          YES             NO
Periodic reanchor     REQUIRED        N/A
```

`*` assumes secure erasure of previous ratchet state and correct reanchor policy.

---

# 177. Required Negative Tests — HCC

Implementations SHOULD test:

```text
Unknown CACHE_ID
Invalid CACHE_ID
Expired CACHE_ID
Revoked CACHE_ID
Wrong generation
Old generation
Future generation
Generation wrap
Modified context
Modified context_hash
Invalid compact authenticator
Random authenticator forgery
Replayed compact request
Replayed generation
Ratchet state corruption
Ratchet state rollback
Cache poisoning
Identity substitution
Protocol downgrade
Cipher downgrade
Authentication downgrade
Invalid responder authentication
Invalid FINISH
Invalid FINISH_ACK
Old traffic key reuse
Old X25519 private key reuse
Reanchor failure
Reanchor rollback
```

---

# 178. HCC Test Vectors

HCC test vectors SHOULD contain:

```text
resumption_secret
context_hash
CACHE_ID
initial generation
initial ratchet_secret

generation N
current ratchet_secret
compact authenticator

generation N+1
next ratchet_secret
session_secret

Finished key
Finished value
```

Reanchor test vectors SHOULD additionally contain:

```text
Initiator X25519 private key
Initiator X25519 public key
Responder X25519 private key
Responder X25519 public key
Shared secret
New ratchet root
```

---

# 179. Updated Security Invariants

A compliant implementation MUST preserve:

```text
1. CACHE_ID is never authentication.

2. CACHE_ID alone cannot authorize resumption.

3. Compact authentication MUST occur before expensive cryptography.

4. Compact generations MUST NOT be reused.

5. Generation counters MUST NOT wrap.

6. Ratchet states MUST advance monotonically.

7. Previous ratchet states SHOULD be erased.

8. Old traffic keys MUST NOT be reused.

9. Old X25519 private keys MUST NOT be reused.

10. Compact Finished verification remains mandatory.

11. Responder authentication MUST precede authenticated
    application DATA.

12. Initiator authentication MUST precede application
    delivery when required.

13. Compact HCC MUST derive fresh session keys.

14. Compact HCC MUST NOT be described as per-session
    X25519 Forward Secrecy.

15. Reanchoring MUST introduce fresh asymmetric entropy.

16. Reanchoring MUST establish a new ratchet root.

17. Expired cache state MUST NOT be resumed.

18. Invalid cache state MUST NOT be partially accepted.

19. Cache state MUST be integrity protected.

20. A captured compact request MUST NOT establish an
    unlimited number of sessions.

21. Negotiated parameters MUST remain cryptographically bound.

22. AEAD key/nonce pairs MUST never repeat.
```

---

# 180. Recommended HCC Profile

```text
HCC:
    Enabled

Compact Reference:
    18 bytes

CACHE_ID:
    48 bits

Generation:
    uint32

Compact Authenticator:
    HMAC-SHA-256 truncated to 48 bits

Compact KDF:
    HKDF-SHA-256

Compact Ratchet:
    Symmetric HKDF ratchet

Per-Connection X25519:
    Not used in baseline compact mode

Reanchor:
    X25519

Traffic AEAD:
    ChaCha20-Poly1305

Transcript:
    SHA-256

Reanchor Policy:
    Implementation-defined

Replay Protection:
    Generation + ratchet state + Finished

Application DATA:
    Only after required Responder authentication
```

---

# 181. Updated Implementation Checklist

```text
[ ] HCC cache
[ ] 18-byte compact format
[ ] 48-bit CACHE_ID
[ ] uint32 generation
[ ] context_hash
[ ] resumption_secret
[ ] ratchet_secret
[ ] HKDF ratchet
[ ] generation advancement
[ ] generation replay protection
[ ] compact authenticator
[ ] 48-bit authenticator verification
[ ] cheap validation before expensive crypto
[ ] atomic ratchet advancement
[ ] secret erasure
[ ] cache expiration
[ ] cache invalidation
[ ] cache revocation
[ ] cache poisoning protection
[ ] cache integrity
[ ] full-handshake fallback
[ ] reanchor
[ ] fresh X25519 on reanchor
[ ] new ratchet root after reanchor
[ ] Responder authentication
[ ] Responder authentication gate
[ ] Finished verification
[ ] fresh traffic keys
[ ] key separation
[ ] replay protection
[ ] downgrade protection
```

---

# 182. Canonical Cold Flow

```text
HELLO
    ↓
HELLO_ACK + Responder Authentication
    ↓
VERIFY RESPONDER
    ↓
X25519
    ↓
HKDF
    ↓
FINISH
    ↓
FINISH_ACK
    ↓
ESTABLISHED
    ↓
CREATE HCC
    ↓
CREATE RATCHET ROOT
```

---

# 183. Canonical Warm Compact Flow

```text
18-BYTE HCC REQUEST
    ↓
CACHE LOOKUP
    ↓
GENERATION VALIDATION
    ↓
CONTEXT VALIDATION
    ↓
48-BIT COMPACT AUTHENTICATOR
    ↓
RATCHET ADVANCEMENT
    ↓
FRESH SESSION SECRET
    ↓
RESPONDER AUTHENTICATION
    ↓
VERIFY RESPONDER
    ↓
FINISH + optional DATA
    ↓
FINISH_ACK
    ↓
ESTABLISHED
```

---

# 184. Canonical Reanchor Flow

```text
COMPACT HCC
    ↓
REANCHOR POLICY
    ↓
FRESH X25519
    ↓
NEW SHARED SECRET
    ↓
NEW RESUMPTION SECRET
    ↓
NEW RATCHET ROOT
    ↓
GENERATION 0
    ↓
COMPACT HCC
```

---

# 185. Final HCC Contract

HCC has three distinct security states:

```text
COLD:
    Full cryptographic handshake.

WARM:
    18-byte symmetric-ratchet compact handshake.

REANCHOR:
    Fresh X25519 handshake establishing a new ratchet root.
```

The compact HCC message is exactly:

```text
TYPE        1 byte
FLAGS       1 byte
CACHE_ID    6 bytes
GENERATION  4 bytes
AUTH TAG    6 bytes
--------------------
TOTAL       18 bytes
```

The 18-byte format is therefore a real complete compact control request rather than:

```text
18 bytes + X25519 + random + authenticator
```

The price for this optimization is that the baseline compact profile does not perform a fresh X25519 exchange on every connection.

That property is restored periodically through reanchoring.

---

# 186. Final Optimized GSP 1.1 Model

The optimized GSP architecture is:

```text
                    FULL HANDSHAKE
                          |
                       X25519
                          |
                          v
                  RESUMPTION ROOT
                          |
                          v
                     HCC CACHE
                          |
                          v
                    RATCHET ROOT
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
           Gen 1       Gen 2       Gen 3
            18 B        18 B        18 B
              |           |           |
              +-----------+-----------+
                          |
                       REANCHOR
                          |
                       X25519
                          |
                          v
                   NEW RATCHET ROOT
```

This allows GSP to optimize the common reconnection path without removing the stronger asymmetric security mechanism entirely.

---

# 187. Final Security Contract

For normal authenticated GSP:

```text
NO VERIFIED RESPONDER
        |
        v
NO AUTHENTICATED APPLICATION DATA
```

For Responder-side delivery:

```text
NO VERIFIED INITIATOR
        |
        v
NO APPLICATION DELIVERY
```

For HCC:

```text
CACHE_ID != AUTHENTICATION

CACHE HIT != AUTHENTICATION

COMPACT AUTHENTICATOR
        |
        v
RATCHET AUTHENTICATION
        |
        v
FINISHED AUTHENTICATION
```

For the compact profile:

```text
18 BYTES
    =
COMPLETE COMPACT CONTROL REQUEST
```

For Forward Secrecy:

```text
COMPACT RATCHET
    =
protected previous state + limited future secrecy

X25519 REANCHOR
    =
fresh asymmetric entropy + restoration of strong PFS
```

For AEAD:

```text
ONE KEY
   +
ONE NONCE
   |
   v
ONE UNIQUE ENCRYPTED RECORD
```

No `(Key, Nonce)` pair may ever be reused.

---

# 188. Final Principle

GSP HCC follows four fundamental rules:

```text
FIRST CONNECTION:
    AUTHENTICATE + ESTABLISH + CACHE

NORMAL RECONNECTION:
    18-BYTE REFERENCE + RATCHET + PROVE

PERIODIC RECONNECTION:
    REANCHOR WITH FRESH X25519

AUTHENTICATED 1-RTT:
    VERIFY RESPONDER BEFORE DATA
```

The compact HCC mechanism exists to reduce:

```text
bytes
CPU cost
latency
repeated negotiation
```

It does not remove:

```text
authentication
Finished verification
replay protection
key separation
context binding
periodic asymmetric reanchoring
```

The 18-byte target is therefore achieved by changing the **key-establishment strategy**, not by incorrectly compressing a 32-byte X25519 public key.

---

# 189. End of Specification

**GSP Handshake Protocol Specification**

**Globalized Secure Protocol**

**Version 1.1 — FOAREVAMP**

**Status:** Experimental / Draft

**URI Scheme:** `gsp://`

**Optimized 1-RTT Handshake**

**Integrated X25519**

**ChaCha20-Poly1305**

**HKDF-SHA-256**

**Canonical Binary Encoding**

**Transcript Authentication**

**Responder Authentication Before 1-RTT Application DATA**

**Directional Key Separation**

**Replay Protection**

**Downgrade Protection**

**Forward Secrecy**

**Symmetric HCC Ratchet**

**Periodic X25519 Reanchoring**

**18-Byte Compact HCC**

**Session Resumption**

**Handshake Context Cache**

**END OF DOCUMENT**
