# GSP Handshake Protocol Specification

**Globalized Secure Protocol (GSP)**
**Handshake Protocol Specification — Version 1.1**
**Codename:** FOAREVAMP (first of all revamp)
**Status:** Experimental / Draft
**URI Scheme:** `gsp://`

---

## 1. Abstract

The GSP Handshake establishes a cryptographically protected GSP session between two peers.

The handshake provides:

* Protocol version negotiation
* Capability negotiation
* Cryptographic suite negotiation
* Key exchange
* Peer authentication
* Session establishment
* Forward secrecy
* Replay protection
* Transcript binding
* Optional session resumption
* Optional 0-RTT data
* Handshake Context Cache (HCC)

GSP 1.1 introduces the **Handshake Context Cache (HCC)**.

HCC allows previously negotiated handshake information to be referenced instead of retransmitted on every connection.

The first connection may therefore be larger and slower.

Subsequent connections can use a compact cache reference and exchange only the cryptographic material required to establish a fresh session.

HCC is an optimization mechanism.

It is not an authentication mechanism by itself.

---

# 2. Design Goals

GSP Handshake 1.1 aims to provide:

1. Strong cryptographic protection.
2. Forward secrecy.
3. Fresh traffic keys for every session.
4. Authentication before authenticated application DATA.
5. Minimal handshake overhead for repeated connections.
6. Safe cache expiration.
7. Safe cache invalidation.
8. Protection against replay.
9. Protection against cache poisoning.
10. Graceful fallback when cached state is unavailable.
11. Compatibility with GSP/TCP, GSP/UDP and GSP/QUIC.
12. A compact control reference targeting approximately **18 bytes**.

The 18-byte target applies to the compact cache reference/control structure.

It does not imply that the complete cryptographic handshake is 18 bytes.

---

# 3. Non-Goals

GSP does not replace:

* IP
* DNS
* TCP
* UDP
* QUIC
* HTTP
* TLS
* WebSocket

GSP may operate over different transports.

The handshake is independent of the underlying transport.

---

# 4. Cryptographic Algorithms

The baseline GSP 1.1 profile uses:

| Function        | Algorithm         |
| --------------- | ----------------- |
| Key exchange    | X25519            |
| AEAD            | ChaCha20-Poly1305 |
| Hash            | SHA-256           |
| KDF             | HKDF-SHA-256      |
| Transcript hash | SHA-256           |
| Compression     | Optional LZ4      |

Implementations MUST NOT silently substitute cryptographic algorithms.

Future algorithm suites MUST be explicitly negotiated.

---

# 5. Security Terminology

### 5.1 Cold Connection

A connection without usable cached handshake context.

A cold connection performs the complete handshake.

### 5.2 Warm Connection

A connection using a valid HCC context.

### 5.3 Cache Context

A locally stored representation of previously negotiated handshake state.

### 5.4 CACHE_ID

An opaque identifier referencing a cache context.

`CACHE_ID` is not a secret and is not authentication.

### 5.5 Resumption Secret

Secret material associated with a cache context and used to authenticate a resumed connection and derive fresh session keys.

### 5.6 Context Hash

A hash identifying the exact canonical cached context.

```text
context_hash = SHA-256(canonical_context)
```

### 5.7 Full Handshake

A handshake in which required negotiation information is transmitted explicitly.

### 5.8 Compact Resumption

A handshake using HCC to avoid retransmitting cached negotiation state.

---

# 6. Security Invariants

The following invariants are mandatory:

```text
CACHE_ID != AUTHENTICATION

CACHE HIT != AUTHENTICATION

CACHE HIT != SESSION ESTABLISHMENT

EXPIRED CACHE != VALID CACHE

INVALID CACHE -> FULL HANDSHAKE

REUSED TRAFFIC KEY -> FORBIDDEN

REUSED X25519 EPHEMERAL KEY -> FORBIDDEN

UNVERIFIED RESPONDER -> NO AUTHENTICATED APPLICATION DATA

FAILED AUTHENTICATION -> NO APPLICATION DATA

FINAL TRANSCRIPT -> BINDS THE COMPLETE HANDSHAKE
```

---

# 7. Handshake Modes

GSP 1.1 defines:

```text
FULL
COMPACT
0-RTT
```

### FULL

Used when no valid HCC context exists.

### COMPACT

Used when a valid HCC context exists.

### 0-RTT

Optional.

0-RTT data is subject to additional replay restrictions and MUST NOT be treated as equivalent to normally authenticated post-handshake DATA.

---

# 8. Cold Handshake

A normal cold connection follows:

```text
Initiator                         Responder

HELLO -------------------------->

        <----------------------- HELLO_ACK

FINISH ------------------------->

        <----------------------- FINISH_ACK

ESTABLISHED
```

The exact contents depend on the negotiated authentication profile.

---

# 9. Compact Handshake

A warm connection follows:

```text
Initiator                         Responder

COMPACT_HELLO ------------------>

        <----------------------- COMPACT_ACK

FINISH ------------------------->

        <----------------------- FINISH_ACK

ESTABLISHED
```

The compact messages reference cached state.

They MUST NOT depend on the cache identifier alone for security.

---

# 10. Handshake Message Types

The following message types are reserved:

```text
0x01 HELLO
0x02 HELLO_ACK

0x03 KEY_EXCHANGE
0x04 KEY_EXCHANGE_ACK

0x05 AUTH
0x06 AUTH_ACK

0x07 FINISH
0x08 FINISH_ACK

0x09 CLOSE
0x0A ERROR

0x0B RESUME
0x0C RESUME_ACK

0x0D COMPACT_HELLO
0x0E COMPACT_ACK
```

Implementations MAY encode `RESUME`/`RESUME_ACK` as aliases of the compact resumption profile where wire compatibility is preserved.

---

# 11. HELLO

`HELLO` is sent by the Initiator.

A full `HELLO` may contain:

```text
version
random
capabilities
cipher_suites
key_exchange_suites
authentication_modes
compression
extensions
client_identity
client_ephemeral_public_key
```

The exact encoding is profile-dependent.

---

# 12. HELLO_ACK

`HELLO_ACK` is sent by the Responder.

A full response may contain:

```text
version
random
selected_cipher
selected_key_exchange
selected_authentication
selected_compression
capabilities
responder_identity
responder_ephemeral_public_key
responder_authentication_proof
extensions
```

When responder authentication is required, the responder authentication proof MUST be included.

---

# 13. Responder Authentication

Responder authentication MUST be bound to the handshake.

The responder proof MUST authenticate:

* protocol version
* selected cryptographic parameters
* Initiator random
* Responder random
* Initiator ephemeral public key
* Responder ephemeral public key
* relevant capabilities
* pre-authentication transcript

The proof MUST NOT authenticate itself.

---

# 14. Pre-Authentication Transcript

To avoid circular authentication, the responder proof is calculated over:

```text
T_pre_auth =
    "GSP-HANDSHAKE-RESPONDER-AUTH"
    ||
    version
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

The responder authentication proof is generated from this context.

---

# 15. Final Transcript

After the responder proof has been generated:

```text
T_final =
    "GSP-HANDSHAKE-FINAL"
    ||
    Encode(HELLO)
    ||
    Encode(HELLO_ACK)
    ||
    Encode(all authenticated handshake messages)
```

Then:

```text
transcript_hash = SHA-256(T_final)
```

The final transcript MUST include the responder authentication proof.

---

# 16. X25519

Each normal GSP session MUST use fresh X25519 ephemeral keys when the negotiated profile provides forward secrecy.

The following is forbidden:

```text
reuse previous X25519 private key
reuse previous X25519 public key
```

Caching an X25519 ephemeral private key solely to reduce handshake size is NOT permitted.

---

# 17. Key Derivation

The X25519 shared secret is:

```text
shared_secret = X25519(
    initiator_ephemeral_private,
    responder_ephemeral_public
)
```

The responder computes the equivalent operation.

HKDF-SHA-256 is then used.

The derivation MUST include:

```text
protocol identifier
protocol version
negotiated parameters
transcript hash
shared secret
```

Traffic keys MUST be directional.

For example:

```text
initiator_to_responder_key
responder_to_initiator_key
```

The same key MUST NOT be used in both directions.

---

# 18. Traffic Key Freshness

Every established session MUST derive new traffic keys.

A resumed session MUST NOT simply restore the previous traffic keys.

HCC stores state required to establish a new session.

It does not store active traffic keys for reuse.

---

# 19. FINISH

`FINISH` proves possession of the required handshake secrets and binds the handshake transcript.

The FINISH authenticator is derived from the handshake key material and:

```text
transcript_hash
```

A receiver MUST reject a FINISH whose transcript does not match.

---

# 20. Application DATA Gate

Application DATA is permitted only after the required authentication conditions have been satisfied.

For authenticated profiles:

```text
NO VERIFIED RESPONDER
        |
        v
NO APPLICATION DATA
```

The Initiator MUST NOT send authenticated application DATA before successful responder authentication.

---

# 21. Anonymous Mode

Anonymous mode provides confidentiality without peer identity authentication.

Under ANONYMOUS:

* responder identity is absent;
* responder authentication proof is absent;
* identity-bound features MUST NOT assume a verified peer;
* authenticated application semantics MUST NOT be inferred.

Application DATA MUST NOT be attached to the FINISH flight in anonymous mode.

---

# 22. Handshake Context Cache

HCC is a protocol-level cache containing previously established handshake context.

HCC exists to avoid retransmitting static or previously negotiated information.

The cache MAY be implemented using:

* memory
* local files
* databases
* secure platform storage
* hardware-backed storage

The wire protocol does not require a filesystem.

---

# 23. HCC Context Structure

A cache entry SHOULD contain:

```text
cache_id
cache_generation
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
server_name
application_profile
extension_set
resumption_policy
0rtt_policy
```

---

# 24. Context Hash

The cache context is canonicalized.

Then:

```text
context_hash =
    SHA-256(canonical_context)
```

The `context_hash` binds the cached parameters to the resumption state.

Changing security-sensitive cached parameters MUST result in a different context hash.

---

# 25. CACHE_ID

`CACHE_ID` is an opaque identifier.

The compact baseline representation uses:

```text
CACHE_ID = 64 bits
```

Implementations intended for large public deployments SHOULD use a larger identifier when the profile permits it.

`CACHE_ID` MUST NOT be treated as a password.

Knowledge of a `CACHE_ID` alone MUST NOT allow session establishment.

---

# 26. Cache Generation

Every cache entry has a generation value.

Example:

```text
cache_generation = 64 bits
```

Generation values allow an implementation to invalidate older state without immediately changing the cache identifier.

Example:

```text
CACHE_ID = 0x1234...
GENERATION = 7
```

After invalidation:

```text
GENERATION = 8
```

Older generation 7 entries MUST be rejected.

---

# 27. The 18-Byte Compact Reference

The minimum compact reference is:

```text
+--------+--------+----------------+------------------+
| Type   | Flags  | CACHE_ID       | Generation       |
| 1 byte | 1 byte | 8 bytes        | 8 bytes          |
+--------+--------+----------------+------------------+
```

Total:

```text
1 + 1 + 8 + 8 = 18 bytes
```

This is the **18-byte HCC control/reference structure**.

It is not a complete cryptographic handshake.

---

# 28. Meaning of the 18-Byte Target

The 18-byte structure allows GSP to represent:

```text
which cached context?
which generation?
which compact operation?
```

without retransmitting:

* capabilities
* cipher lists
* compression configuration
* identity metadata
* extension metadata
* previously negotiated parameters

Those values are recovered from the validated cache context.

---

# 29. Compact Cryptographic Payload

The cryptographic payload remains separate from the 18-byte reference.

A secure compact session may additionally require:

```text
fresh nonce
fresh X25519 public key
resumption authenticator
transcript binding
```

Therefore:

```text
18-byte reference
+
cryptographic payload
=
actual compact handshake
```

The complete handshake is therefore larger than 18 bytes.

---

# 30. Compact Handshake Target

GSP 1.1 defines the following engineering targets:

```text
18 bytes
    compact control/reference

~80–120 bytes
    aggressive compact cryptographic exchange

~120–180 bytes
    conservative compact exchange with stronger authentication material
```

Actual size depends on:

* authentication profile
* X25519 inclusion
* transport framing
* optional extensions
* authenticator encoding
* 0-RTT
* implementation encoding

These values are targets, not mandatory fixed packet sizes.

---

# 31. Resumption Secret

A successful full handshake creates a resumption secret.

The resumption secret MUST be cryptographically bound to:

```text
peer identity
protocol version
negotiated parameters
context_hash
handshake transcript
```

A conceptual derivation is:

```text
resumption_secret =
    HKDF-Expand(
        handshake_secret,
        "GSP-RESUMPTION" || context_hash,
        ...
    )
```

The exact output length is defined by the selected cryptographic profile.

---

# 32. Resumption Authenticator

A compact handshake MUST prove possession of the resumption secret.

Conceptually:

```text
resume_authenticator =
    MAC(
        resumption_secret,
        compact_transcript
    )
```

The authenticator MUST include:

```text
CACHE_ID
cache_generation
fresh connection nonce
context_hash
fresh ephemeral key material when used
protocol version
selected parameters
```

The exact MAC construction is defined by the selected cryptographic suite.

---

# 33. Cache Lookup

When receiving a compact connection request, the responder performs:

```text
1. Parse compact reference.
2. Locate CACHE_ID.
3. Check generation.
4. Check expiration.
5. Check revocation.
6. Check protocol compatibility.
7. Check parameter compatibility.
8. Validate context integrity.
9. Validate resumption authenticator.
10. Validate freshness/replay state.
11. Continue handshake.
```

A failure MUST NOT result in partial use of the cached context.

---

# 34. Cache Hit

A cache hit means only:

```text
CACHE_ID exists
```

It does NOT mean:

```text
peer authenticated
session authenticated
resumption authorized
```

Authentication occurs only after the cryptographic resumption proof succeeds.

---

# 35. Cache Miss

If the cache does not exist:

```text
CACHE_MISS
```

The connection falls back to the full handshake.

The responder MAY indicate this through an error/status message or simply require a full `HELLO`.

---

# 36. Cache Expiration

Each cache entry MUST have an expiration time.

Example:

```text
created_at
expires_at
```

An implementation MAY additionally enforce:

```text
idle_expires_at
```

When:

```text
current_time >= expires_at
```

the entry is expired.

Expired state MUST NOT be used for resumption.

---

# 37. Cache Expiration Response

A compact request referencing expired state may result in:

```text
CACHE_EXPIRED
```

The client then performs:

```text
FULL HANDSHAKE
```

A new cache entry MAY be created.

---

# 38. Cache Invalidation

A cache entry MUST be invalidated when required by security policy.

Examples:

* credential rotation
* peer identity change
* protocol version incompatibility
* cryptographic suite change
* authentication policy change
* explicit revocation
* detected compromise
* invalid generation
* administrator invalidation

---

# 39. Cache Poisoning Protection

Remote peers MUST NOT be allowed to directly insert arbitrary authenticated cache entries.

A cache entry MUST be created only after successful handshake establishment.

The cache context MUST be integrity protected.

Untrusted cache metadata MUST NOT overwrite an existing trusted entry without validation.

---

# 40. Cache Storage Security

Sensitive values such as:

```text
resumption_secret
peer authentication state
identity-bound metadata
```

SHOULD be protected at rest.

Implementations SHOULD use platform secure storage when available.

When an entry is securely invalidated, implementations SHOULD erase sensitive secret material.

---

# 41. Cache Lifetime

The implementation SHOULD define:

```text
maximum lifetime
maximum idle lifetime
maximum number of entries
maximum entry size
```

Example policy:

```text
MAX_CACHE_ENTRIES
MAX_CACHE_SIZE
MAX_CONTEXT_LIFETIME
MAX_IDLE_LIFETIME
```

Values are deployment-specific.

---

# 42. Cache DoS Protection

An implementation MUST NOT allocate unlimited storage because a remote peer requests new sessions.

Recommended protections:

* bounded cache size
* bounded number of entries
* rate limiting
* eviction policy
* authentication before expensive state creation
* randomized eviction where appropriate

---

# 43. Cache Eviction

When storage limits are reached, entries MAY be evicted.

Eviction MUST NOT be treated as a protocol error.

The next connection simply performs:

```text
FULL HANDSHAKE
```

---

# 44. Cache Enumeration

Because `CACHE_ID` is not authentication, implementations MUST consider enumeration.

`CACHE_ID` SHOULD be unpredictable enough to prevent practical enumeration.

A 64-bit identifier is the compact baseline.

Deployments requiring stronger anti-enumeration properties SHOULD use a larger identifier or a secret-bound ticket.

---

# 45. Cache Tickets

An implementation MAY replace server-side cache storage with an authenticated encrypted ticket.

In this model:

```text
client stores ticket
server validates ticket
```

The ticket MUST be:

* integrity protected
* authenticated
* bound to the intended context
* expiration controlled
* resistant to modification

A ticket is not equivalent to a plaintext `CACHE_ID`.

---

# 46. Stateless Resumption

A responder MAY use stateless resumption.

The responder stores only the secret required to validate tickets.

The ticket can contain encrypted/authenticated context.

The ticket MUST NOT expose sensitive information unnecessarily.

---

# 47. Freshness

Every compact connection MUST include fresh connection-specific material.

At minimum, the cryptographic resumption exchange MUST prevent an attacker from replaying a previous successful request as a new authenticated session.

Possible mechanisms include:

```text
fresh random nonce
server challenge
monotonic anti-replay state
authenticated timestamp
single-use token
```

The selected mechanism depends on the profile.

---

# 48. Replay Protection

A valid cache reference may be observed by an attacker.

Therefore:

```text
CACHE_ID alone MUST NOT authorize resumption.
```

A captured compact handshake MUST NOT be reusable to establish an equivalent authenticated session.

The resumption authenticator MUST bind fresh session state.

---

# 49. 0-RTT

0-RTT is optional.

0-RTT data is potentially replayable.

Applications MUST explicitly declare whether a request is replay-safe.

Operations such as:

```text
DELETE
payment
state-changing commands
credential modification
```

SHOULD NOT be accepted through unrestricted 0-RTT.

---

# 50. 0-RTT and HCC

HCC does not automatically enable 0-RTT.

The following are separate:

```text
HCC
RESUMPTION
0-RTT
```

A deployment may support:

```text
HCC + 1-RTT
```

without supporting:

```text
HCC + 0-RTT
```

This is the recommended default.

---

# 51. Compact PFS

The safest compact profile continues to use fresh X25519 ephemeral keys.

Therefore:

```text
previous session
        |
        X
        |
fresh session
        |
fresh X25519
        |
new traffic keys
```

HCC does not weaken forward secrecy merely to reduce bytes.

---

# 52. Why 18 Bytes Cannot Contain Everything

An X25519 public key alone requires 32 bytes.

A complete authenticated handshake also requires cryptographic authentication material.

Therefore the following claim is invalid:

```text
"the entire secure X25519 handshake is 18 bytes"
```

The correct claim is:

```text
"the HCC compact control reference is 18 bytes."
```

This distinction is mandatory in GSP documentation.

---

# 53. Compact Handshake Example

Conceptually:

```text
COMPACT_HELLO

18-byte HCC reference
+
fresh nonce
+
fresh X25519 public key
+
resumption authenticator
```

Responder:

```text
COMPACT_ACK

18-byte HCC reference
+
fresh nonce
+
fresh X25519 public key
+
resumption/authentication proof
```

Both peers then derive:

```text
new handshake keys
new traffic keys
```

---

# 54. Full Handshake Creates HCC

After a successful full handshake:

```text
FULL HANDSHAKE
       |
       v
AUTHENTICATED
       |
       v
CREATE HCC CONTEXT
       |
       +--> CACHE_ID
       +--> GENERATION
       +--> CONTEXT_HASH
       +--> RESUMPTION_SECRET
       +--> EXPIRATION
```

---

# 55. HCC Update

A successful resumed connection MAY refresh:

```text
last_used_at
expiration
generation
resumption state
```

A new resumption secret SHOULD be derived when security policy requires it.

Implementations MUST NOT indefinitely extend compromised state without reauthentication.

---

# 56. Credential Changes

When credentials change:

```text
old cache
    |
    X
    |
invalid
```

A new full authentication handshake is required.

The new handshake creates a new cache context.

---

# 57. Parameter Changes

If any security-sensitive parameter changes, the old cache MUST NOT be silently reused.

Examples:

```text
cipher
authentication mode
key exchange
protocol version
identity
security policy
```

---

# 58. Application Binding

Applications MAY bind a cache context to an application profile.

Example:

```text
GSP Terminal
GSPID
GSPWD
GSPMAIL
```

A cache created for one application profile MUST NOT automatically authorize another profile unless explicitly permitted.

---

# 59. Transport Binding

HCC SHOULD be transport-independent.

A context MAY be reused across:

```text
GSP/TCP
GSP/UDP
GSP/QUIC
```

only when the cached context explicitly permits this.

Transport-specific state MUST NOT be assumed to be valid on another transport.

---

# 60. Migration

If GSP supports connection migration, the migration MUST NOT invalidate cryptographic session identity merely because the network path changes.

However, transport-specific state MAY require renegotiation.

---

# 61. Cache and Connection Identity

The cache context MAY contain:

```text
peer_identity
```

but the peer identity MUST be cryptographically bound to the resumption state.

The implementation MUST NOT simply trust a locally supplied identity string.

---

# 62. Responder Authentication During Resumption

For an authenticated profile, the responder MUST prove that it controls the state associated with the cached context.

The compact responder proof MUST bind:

```text
cached identity
context_hash
CACHE_ID
generation
fresh connection state
new key exchange
final transcript
```

---

# 63. Initiator Authentication During Resumption

If the original context requires Initiator authentication, the resumed handshake MUST preserve that requirement.

A cache hit MUST NOT downgrade:

```text
authenticated
```

to:

```text
anonymous
```

---

# 64. Authentication Downgrade Protection

The following transitions are prohibited unless explicitly authorized:

```text
PUBLIC_KEY -> ANONYMOUS
PSK        -> ANONYMOUS
AUTHENTICATED -> unauthenticated
```

The cache context MUST bind the authentication mode.

---

# 65. Cipher Downgrade Protection

A cached context MUST bind the negotiated cipher suite.

An attacker MUST NOT cause:

```text
ChaCha20-Poly1305
```

to become:

```text
weaker/unauthorized cipher
```

through cache manipulation.

---

# 66. Version Downgrade Protection

The protocol version MUST be bound to the cache context.

An expired or incompatible context MUST trigger a full handshake.

---

# 67. Extension Binding

Security-sensitive extensions MUST be included in:

```text
context_hash
```

and/or:

```text
transcript_hash
```

Extensions that alter authentication or key derivation MUST NOT be silently omitted during resumption.

---

# 68. Cache Context Canonicalization

The context MUST use a deterministic encoding.

Equivalent contexts MUST produce the same canonical representation.

Different security-sensitive contexts MUST NOT produce the same representation intentionally.

Canonicalization MUST be specified by the selected GSP encoding profile.

---

# 69. Cache Corruption

If local cache data is corrupted:

```text
CACHE_INVALID
```

The implementation MUST discard the invalid state and perform a full handshake.

It MUST NOT attempt to guess or repair security-sensitive values.

---

# 70. Clock Handling

Expiration SHOULD use a monotonic clock for local lifetime calculations when available.

Wall-clock timestamps MAY be stored for diagnostics.

Clock rollback MUST NOT cause expired credentials to become valid again.

---

# 71. Cache Revocation

A responder MAY revoke a cache context.

Revocation may be represented by:

```text
generation change
revocation list
ticket key rotation
explicit invalidation
```

The selected mechanism is deployment-specific.

---

# 72. Key Rotation

Resumption keys SHOULD be rotated periodically.

For stateless tickets, the server MAY maintain:

```text
current_ticket_key
previous_ticket_key
```

The previous key may remain valid for a limited overlap period.

---

# 73. Cache Secret Separation

The following MUST remain logically separated:

```text
traffic keys
handshake keys
resumption secret
ticket encryption keys
cache storage encryption keys
```

One secret MUST NOT be reused for unrelated purposes.

Domain-separated HKDF labels SHOULD be used.

---

# 74. Example Key Schedule

Conceptually:

```text
X25519
   |
   v
shared_secret
   |
   v
HKDF-Extract
   |
   +--> handshake_secret
   |
   +--> traffic_secret
   |
   +--> resumption_secret
```

Each derived value MUST use a distinct context/label.

---

# 75. Resumption Key Schedule

A resumed connection uses:

```text
resumption_secret
+
fresh connection state
+
fresh X25519 shared secret when enabled
+
context_hash
+
new transcript
```

to derive fresh session keys.

The previous traffic keys are never restored.

---

# 76. Failure Handling

Defined compact failure conditions include:

```text
CACHE_MISS
CACHE_EXPIRED
CACHE_INVALID
CACHE_REVOKED
CACHE_GENERATION_MISMATCH
CACHE_CONTEXT_MISMATCH
RESUMPTION_AUTH_FAILED
REPLAY_DETECTED
PARAMETER_MISMATCH
VERSION_MISMATCH
```

---

# 77. Fallback Rule

For recoverable cache failures:

```text
COMPACT
   |
   +--> failure
           |
           v
      FULL HANDSHAKE
```

The responder MUST NOT silently accept a partially valid cache.

---

# 78. Fatal Authentication Failure

If the cryptographic authenticator fails:

```text
RESUMPTION_AUTH_FAILED
```

the peer MUST NOT establish an authenticated session using that cache context.

Implementations MAY terminate immediately rather than fall back automatically when policy requires it.

This helps reduce online probing.

---

# 79. Anti-Probing Policy

Implementations MAY make cache failures intentionally indistinguishable.

For example:

```text
CACHE_MISS
CACHE_EXPIRED
CACHE_REVOKED
```

may all produce:

```text
RESUME_REJECTED
```

This prevents information leakage about cache state.

---

# 80. Compact State Machine

```text
              +----------------+
              |      START     |
              +-------+--------+
                      |
                      v
              +----------------+
              | CACHE LOOKUP   |
              +---+---------+--+
                  |         |
                miss       hit
                  |         |
                  v         v
              FULL       VALIDATE
           HANDSHAKE       |
                  |        v
                  |    AUTHENTICATE
                  |        |
                  |    +---+---+
                  |    |       |
                  |  fail     success
                  |    |       |
                  |    v       v
                  |  FULL    RESUME
                  |            |
                  +-----+------+
                        |
                        v
                  ESTABLISHED
```

---

# 81. Compact Handshake State

A compact connection SHOULD use states equivalent to:

```text
START
CACHE_REFERENCED
CACHE_VALIDATED
KEY_EXCHANGE
RESUMPTION_AUTHENTICATED
FINISH_VALIDATED
ESTABLISHED
```

---

# 82. State Transition Rules

```text
START
 -> CACHE_REFERENCED

CACHE_REFERENCED
 -> CACHE_VALIDATED
 -> FULL_HANDSHAKE

CACHE_VALIDATED
 -> KEY_EXCHANGE
 -> FULL_HANDSHAKE

KEY_EXCHANGE
 -> RESUMPTION_AUTHENTICATED
 -> FAILED

RESUMPTION_AUTHENTICATED
 -> FINISH_VALIDATED
 -> FAILED

FINISH_VALIDATED
 -> ESTABLISHED
```

---

# 83. Cache Creation Rules

A new HCC entry MUST NOT be created merely because a `HELLO` was received.

Creation requires successful establishment of the required authenticated handshake.

---

# 84. Cache Refresh Rules

A cache MAY be refreshed after a successful authenticated session.

Refreshing MUST NOT bypass authentication.

---

# 85. Cache Size

The cache context SHOULD be significantly larger than the compact reference.

This is intentional.

Example:

```text
CACHE_ID               8 bytes
generation             8 bytes
protocol parameters    variable
identity               variable
context_hash           32 bytes
resumption_secret      32 bytes
timestamps             variable
```

The cache exists precisely so this information does not need to cross the wire repeatedly.

---

# 86. What Is Actually Saved

HCC may save information such as:

```text
"we already negotiated ChaCha20-Poly1305"
"we already negotiated X25519"
"this peer uses authentication profile X"
"these extensions were accepted"
"this identity was authenticated"
"this is context generation 4"
"this resumption secret belongs to this context"
```

It does NOT save:

```text
old traffic key for reuse
old X25519 ephemeral private key
old session nonce for reuse
```

---

# 87. First Connection Cost

The first connection intentionally carries more information.

Conceptually:

```text
FIRST CONNECTION

HELLO
  + capabilities
  + cipher list
  + key exchange
  + authentication
  + extensions
  + identity
  + cryptographic material

            ↓

HCC CREATED
```

This cost is paid once per cache lifetime.

---

# 88. Subsequent Connection

```text
SUBSEQUENT CONNECTION

18-byte HCC reference
        +
fresh cryptographic material
        +
authentication
```

This significantly reduces repeated metadata.

---

# 89. Expiration Behavior

Example:

```text
Connection #1
    |
    v
Create cache
    |
    v
Connection #2
    |
    v
CACHE HIT
    |
    v
Compact handshake
    |
    v
Connection #N
    |
    v
CACHE EXPIRED
    |
    v
Full handshake
    |
    v
New cache
```

---

# 90. Performance Model

HCC primarily reduces:

```text
bytes
serialization
negotiation repetition
metadata transmission
```

It does not inherently reduce:

```text
network RTT
speed of light
transport latency
```

A 1-RTT compact handshake remains approximately 1 RTT.

---

# 91. Latency Goal

For repeated connections, GSP SHOULD target:

```text
1 RTT
```

for the normal compact authenticated handshake.

0-RTT MAY be supported separately.

---

# 92. Bandwidth Goal

The compact reference targets:

```text
18 bytes
```

for the HCC control structure.

The complete cryptographic handshake SHOULD be optimized toward:

```text
~80–120 bytes
```

where the selected authentication and key-exchange profile permits it.

---

# 93. Conservative Target

Implementations that cannot safely fit the aggressive target SHOULD prioritize security over size.

A handshake of:

```text
120–180 bytes
```

is preferable to removing required cryptographic protections.

---

# 94. No Security-by-Compression

HCC MUST NOT rely on compression to make security material fit into 18 bytes.

The 18-byte target comes from referencing already known state.

Compression MAY be used separately.

---

# 95. No Secret in CACHE_ID

The design MUST NOT assume:

```text
CACHE_ID = secret
```

The cache ID is an identifier.

Security comes from:

```text
resumption_secret
authentication
freshness
transcript binding
```

---

# 96. Cache Reference Privacy

A CACHE_ID may reveal that a cache context exists if exposed directly.

Implementations concerned about metadata privacy MAY use encrypted or opaque tickets instead.

---

# 97. Application DATA After Resumption

Once:

```text
responder authenticated
initiator authenticated when required
FINISH validated
```

the session becomes:

```text
ESTABLISHED
```

Application DATA may then flow normally.

---

# 98. FINISH and DATA

The implementation MAY optimize the first application DATA transmission when all authentication requirements have already been satisfied.

However:

```text
NO VERIFIED RESPONDER
        |
        X
        |
APPLICATION DATA
```

remains mandatory.

---

# 99. Anonymous Resumption

Anonymous resumption MAY be implemented only when explicitly defined by an application profile.

A cache context MUST NOT accidentally turn an anonymous session into an authenticated identity.

---

# 100. Identity-Bound Resumption

For authenticated peers:

```text
cached identity
      |
      v
context_hash
      |
      v
resumption_secret
      |
      v
new session
```

The new session MUST remain bound to the same authenticated identity unless a fresh authentication explicitly changes it.

---

# 101. GSPID Compatibility

If GSPID is used, its identity state MAY be stored in HCC.

However, GSPID MUST NOT treat:

```text
CACHE_ID
```

as proof of identity.

The identity remains valid only when the resumption cryptographic proof validates the identity-bound context.

---

# 102. Security Event Logging

Implementations SHOULD log:

```text
cache created
cache hit
cache miss
cache expired
cache revoked
cache authentication failure
replay detected
cache invalidated
```

Sensitive secret material MUST NOT be logged.

---

# 103. Diagnostics

Debug logs SHOULD identify:

```text
CACHE_ID
generation
result
```

but SHOULD avoid logging:

```text
resumption_secret
private keys
session keys
authentication secrets
```

---

# 104. Interoperability

Two implementations MUST agree on:

```text
protocol version
encoding
cryptographic profile
HCC profile
cache representation
authentication profile
```

An implementation MUST NOT assume that an arbitrary cache created by another implementation is compatible.

---

# 105. HCC Profile Identifier

A future GSP profile MAY define:

```text
HCC profile ID
```

to allow different cache formats.

The profile ID MUST be included in the context binding.

---

# 106. Cache Format Version

Cache storage MUST have a format version.

Example:

```text
hcc_version = 1
```

Unknown cache formats MUST be rejected safely.

---

# 107. Local Cache Isolation

Applications SHOULD NOT share HCC secrets unless explicitly authorized.

A cache created for one GSP application SHOULD be isolated from unrelated applications.

---

# 108. Multi-Device Operation

A cache context is normally device-specific.

Implementations MUST NOT copy resumption secrets between devices unless the deployment explicitly supports secure synchronization.

---

# 109. Backup

HCC secrets SHOULD NOT be included in ordinary backups unless encrypted and protected appropriately.

A copied cache secret may permit resumption until expiration or revocation.

---

# 110. Secure Deletion

When a cache is invalidated, implementations SHOULD erase:

```text
resumption_secret
ticket keys
identity-bound secret state
```

as securely as the platform permits.

---

# 111. HCC Security Invariants

The following are mandatory:

```text
CACHE_ID != AUTHENTICATION

CACHE HIT != AUTHENTICATION

EXPIRED CACHE != VALID CACHE

INVALID CACHE -> FULL HANDSHAKE

CACHE CONTEXT MUST BE INTEGRITY PROTECTED

RESUMPTION SECRET MUST BE PROTECTED

TRAFFIC KEYS MUST NOT BE REUSED

EPHEMERAL X25519 KEYS MUST NOT BE REUSED

RESUMPTION MUST USE FRESH SESSION STATE

FINAL TRANSCRIPT MUST INCLUDE AUTHENTICATED HANDSHAKE STATE

NO VERIFIED RESPONDER -> NO AUTHENTICATED APPLICATION DATA
```

---

# 112. Full Handshake Diagram

```text
Initiator                                      Responder

HELLO
  |
  |-------------------------------------------->
  |
  |       HELLO_ACK
  |<--------------------------------------------|
  |
  | verify responder authentication
  |
  | FINISH
  |-------------------------------------------->
  |
  |       FINISH_ACK
  |<--------------------------------------------|
  |
  v
ESTABLISHED

              |
              v
        CREATE HCC
```

---

# 113. Compact Handshake Diagram

```text
Initiator                                      Responder

18-byte HCC reference
+ fresh crypto
  |
  |-------------------------------------------->
  |
  |       COMPACT_ACK
  |       + fresh crypto
  |<--------------------------------------------|
  |
  | verify resumption authentication
  |
  | FINISH
  |-------------------------------------------->
  |
  |       FINISH_ACK
  |<--------------------------------------------|
  |
  v
ESTABLISHED
```

---

# 114. Cache Failure Diagram

```text
COMPACT_HELLO
      |
      v
CACHE LOOKUP
      |
      +---- HIT ----> VALIDATE
      |                  |
      |                  +---- SUCCESS ---> RESUME
      |                  |
      |                  +---- FAILURE ---> FULL
      |
      +---- MISS --------------------------> FULL
      |
      +---- EXPIRED -----------------------> FULL
      |
      +---- REVOKED -----------------------> FULL
```

---

# 115. 18-Byte Reference Diagram

```text
+--------+--------+----------------+------------------+
| TYPE   | FLAGS  | CACHE_ID       | GENERATION       |
| 1      | 1      | 8              | 8                |
+--------+--------+----------------+------------------+

                    18 BYTES
```

---

# 116. Example

Example conceptual cache:

```text
CACHE_ID:
    0xA71C92E41F0082B3

GENERATION:
    0x0000000000000004

PROTOCOL:
    GSP/1.1

CIPHER:
    ChaCha20-Poly1305

KEX:
    X25519

AUTH:
    PUBLIC_KEY

CONTEXT_HASH:
    SHA-256(...)

RESUMPTION_SECRET:
    secret

EXPIRES:
    timestamp
```

The wire does not need to retransmit all of this.

---

# 117. What the Responder Actually Checks

On compact resumption:

```text
Does CACHE_ID exist?
        |
        v
Is generation correct?
        |
        v
Is it expired?
        |
        v
Is it revoked?
        |
        v
Is context intact?
        |
        v
Does context match?
        |
        v
Does authenticator verify?
        |
        v
Is the session fresh?
        |
        v
Is the new key exchange valid?
        |
        v
ESTABLISH
```

---

# 118. What the Attacker Cannot Do

Knowing:

```text
CACHE_ID
generation
old packets
```

must not be sufficient to:

```text
authenticate
derive traffic keys
impersonate responder
impersonate initiator
reuse old traffic keys
```

---

# 119. What Happens When the Cache Expires

```text
OLD CACHE
    |
    X
    |
EXPIRED
    |
    v
FULL HANDSHAKE
    |
    v
NEW RESUMPTION SECRET
    |
    v
NEW CACHE GENERATION
```

---

# 120. Implementation Requirements

A compliant implementation SHOULD implement:

```text
[ ] Full handshake
[ ] X25519
[ ] ChaCha20-Poly1305
[ ] HKDF-SHA-256
[ ] SHA-256 transcript
[ ] Responder authentication
[ ] Fresh traffic keys
[ ] HCC
[ ] CACHE_ID
[ ] Cache generation
[ ] Cache expiration
[ ] Cache invalidation
[ ] Resumption authentication
[ ] Replay protection
[ ] Full-handshake fallback
```

Optional:

```text
[ ] 0-RTT
[ ] Stateless tickets
[ ] Larger CACHE_ID
[ ] Secure hardware storage
[ ] Application-bound caches
```

---

# 121. Negative Security Tests

An implementation MUST test at least:

```text
expired CACHE_ID
wrong generation
wrong context_hash
wrong resumption secret
modified cache
modified authenticator
replayed compact handshake
reused nonce
reused X25519 key
wrong peer identity
wrong protocol version
wrong cipher
wrong authentication mode
downgrade attempt
corrupted cache
unknown cache format
revoked cache
```

Every test MUST result in rejection or safe full-handshake fallback.

---

# 122. Recommended Default

The recommended GSP 1.1 deployment is:

```text
FULL HANDSHAKE
    |
    +--> create HCC
    |
    v
COMPACT 1-RTT RESUMPTION
    |
    +--> fresh X25519
    +--> fresh traffic keys
    +--> authenticated resumption
    |
    v
ESTABLISHED
```

0-RTT SHOULD remain disabled unless the application explicitly needs it.

---

# 123. Canonical Full Flow

```text
HELLO
    ↓
HELLO_ACK
    ↓
verify responder
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
```

---

# 124. Canonical Compact Flow

```text
18-byte HCC REFERENCE
    ↓
CACHE LOOKUP
    ↓
CACHE VALIDATION
    ↓
fresh X25519
    ↓
resumption authentication
    ↓
HKDF
    ↓
FINISH
    ↓
FINISH_ACK
    ↓
ESTABLISHED
```

---

# 125. Final GSP HCC Contract

GSP HCC is a mechanism for replacing repeated handshake metadata with a compact reference to previously validated state.

The system MUST satisfy:

```text
FIRST CONNECTION MAY BE LARGE

SUBSEQUENT CONNECTIONS MAY BE COMPACT

CACHE_ID IS NOT AUTHENTICATION

CACHE HIT IS NOT AUTHENTICATION

CACHE MUST EXPIRE

CACHE MUST BE INVALIDATABLE

CACHE MUST BE INTEGRITY PROTECTED

CACHE SECRETS MUST BE PROTECTED

INVALID CACHE MUST FALL BACK SAFELY

RESUMPTION MUST USE FRESH SESSION STATE

TRAFFIC KEYS MUST NEVER BE REUSED

X25519 EPHEMERAL KEYS MUST NOT BE REUSED

RESPONDER AUTHENTICATION MUST PRECEDE AUTHENTICATED APPLICATION DATA

FINAL TRANSCRIPT MUST BIND THE HANDSHAKE

18 BYTES REPRESENT THE COMPACT HCC CONTROL REFERENCE

18 BYTES DO NOT REPRESENT THE COMPLETE CRYPTOGRAPHIC HANDSHAKE
```

---

# 126. Final Size Target

The GSP 1.1 HCC design therefore targets:

```text
COLD CONNECTION
    Full handshake
    Larger initial cost
    Creates HCC

WARM CONNECTION
    18-byte HCC reference
    +
    cryptographic payload

TARGET:
    ~80–120 bytes
    for an aggressive secure compact handshake

CONSERVATIVE:
    ~120–180 bytes

CONTROL REFERENCE:
    exactly 18 bytes
```

Security MUST take priority over reaching the smallest possible byte count.

---

# 127. Final Principle

The fundamental optimization of HCC is:

```text
DO NOT SEND AGAIN
WHAT BOTH SIDES ALREADY KNOW
```

Instead:

```text
FIRST CONNECTION
    negotiate
    authenticate
    establish
    cache

NEXT CONNECTION
    reference
    prove possession
    establish fresh keys
```

The cache is therefore an optimization layer around the handshake, not a replacement for cryptographic authentication.

---

# 128. End

**GSP Handshake Protocol Specification 1.1 — HCC / Compact Resumption**

**Globalized Secure Protocol**

**Status:** Experimental / Draft

**URI:** `gsp://`
