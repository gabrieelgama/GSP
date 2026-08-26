GSP Handshake Specification

Globalized Secure Protocol (GSP)
Handshake Protocol Specification
Version: 1.0
Status: Experimental

---

1. Overview

The GSP handshake establishes a secure GSP connection between two endpoints before application data is exchanged.

The handshake is responsible for:

- Negotiating protocol versions
- Negotiating cryptographic algorithms
- Establishing shared session keys
- Authenticating endpoints
- Negotiating transport capabilities
- Negotiating optional features
- Providing replay protection
- Establishing connection identifiers
- Confirming that both endpoints derived the same keys

After a successful handshake, encrypted GSP packets can be exchanged.

Client                                      Server
  |                                           |
  | -------- CLIENT_HELLO -----------------> |
  |                                           |
  | <-------- SERVER_HELLO ----------------- |
  |                                           |
  | -------- CLIENT_KEY_EXCHANGE ----------> |
  |                                           |
  | <-------- SERVER_KEY_EXCHANGE ----------- |
  |                                           |
  | -------- CLIENT_FINISHED --------------> |
  |                                           |
  | <-------- SERVER_FINISHED --------------- |
  |                                           |
  | ========= Encrypted GSP Traffic ========= |

---

2. Handshake States

A GSP endpoint MUST maintain a handshake state.

IDLE
  |
  v
CLIENT_HELLO / SERVER_HELLO
  |
  v
KEY_EXCHANGE
  |
  v
AUTHENTICATION
  |
  v
FINISHED
  |
  v
ESTABLISHED

An endpoint MAY abort the handshake from any state.

If the handshake fails, the connection MUST NOT enter the "ESTABLISHED" state.

---

3. Handshake Message Types

GSP defines the following handshake message types.

Type| Name| Description
"0x01"| CLIENT_HELLO| Starts a client handshake
"0x02"| SERVER_HELLO| Server response
"0x03"| CLIENT_KEY_EXCHANGE| Client key exchange data
"0x04"| SERVER_KEY_EXCHANGE| Server key exchange data
"0x05"| CLIENT_AUTH| Client authentication
"0x06"| SERVER_AUTH| Server authentication
"0x07"| CLIENT_FINISHED| Client handshake confirmation
"0x08"| SERVER_FINISHED| Server handshake confirmation
"0x09"| HANDSHAKE_RETRY| Requests a new handshake
"0x0A"| HANDSHAKE_ERROR| Reports handshake failure
"0x0B"| HANDSHAKE_EXT| Negotiates extensions

Unknown handshake message types MUST cause a "HANDSHAKE_ERROR" unless explicitly marked as ignorable by an extension.

---

4. ClientHello

The client begins the handshake with "CLIENT_HELLO".

CLIENT_HELLO {
    version
    flags
    client_connection_id
    random
    supported_versions
    supported_ciphers
    supported_kdfs
    supported_compression
    supported_transports
    extensions
}

4.1 Version

The "version" field identifies the client's preferred GSP version.

Example:

GSP/1.0

The client MUST also provide "supported_versions".

Example:

supported_versions = [1.0, 0.9]

---

4.2 Client Connection ID

The client generates a random connection identifier.

client_connection_id = random(64 bits)

The identifier is used to associate handshake messages with the connection.

Connection IDs SHOULD be unpredictable.

---

4.3 Client Random

The client generates:

client_random = random(32 bytes)

The random value MUST be generated using a cryptographically secure random number generator.

It MUST NOT be reused between independent handshakes.

---

5. ServerHello

The server responds with "SERVER_HELLO".

SERVER_HELLO {
    version
    flags
    server_connection_id
    random
    selected_cipher
    selected_kdf
    selected_compression
    selected_transport
    extensions
}

The server generates:

server_random = random(32 bytes)

The server MUST select only algorithms advertised by the client.

---

6. Cryptographic Negotiation

GSP separates encryption, key derivation, and key exchange.

A typical configuration is:

Key Exchange:
    X25519

KDF:
    HKDF-SHA-256

AEAD:
    ChaCha20-Poly1305

Hash:
    SHA-256

An implementation MAY support additional algorithms through registered GSP extensions.

---

7. Key Exchange

GSP recommends X25519 for ephemeral key exchange.

The client generates:

client_private_key
client_public_key

The server generates:

server_private_key
server_public_key

The shared secret is calculated as:

shared_secret =
    X25519(client_private_key, server_public_key)

and:

shared_secret =
    X25519(server_private_key, client_public_key)

Both sides MUST obtain the same shared secret.

Private keys MUST NOT be transmitted.

---

8. Key Derivation

GSP uses HKDF to derive independent cryptographic keys.

The handshake transcript is incorporated into key derivation.

handshake_hash =
    SHA256(all_handshake_messages)

The initial key material is derived from:

early_secret =
    HKDF-Extract(
        salt,
        shared_secret
    )

Session keys are then derived using domain-separated labels.

Example:

client_write_key =
    HKDF-Expand(
        early_secret,
        "GSP client key" || handshake_hash,
        key_length
    )

server_write_key =
    HKDF-Expand(
        early_secret,
        "GSP server key" || handshake_hash,
        key_length
    )

Implementations MUST use different keys for each traffic direction.

---

9. Nonces

GSP AEAD packets use unique nonces.

A packet nonce can be constructed from:

base_nonce XOR packet_number

The exact nonce construction MUST guarantee that the same key is never used with the same nonce more than once.

Packet numbers MUST NOT be reused with the same encryption key.

---

10. Authentication

Authentication is optional at the protocol level but MUST be supported by implementations that provide authenticated connections.

GSP supports:

Anonymous
PSK
Certificate
Public Key
Application-defined authentication

10.1 PSK Authentication

A pre-shared key may be configured between endpoints.

PSK_ID
PSK

The PSK MUST NOT be transmitted directly.

Instead, it participates in key derivation and authentication.

---

10.2 Public-Key Authentication

An endpoint MAY prove ownership of a private key by signing the handshake transcript.

signature = Sign(
    private_key,
    handshake_hash
)

The peer verifies:

Verify(
    public_key,
    handshake_hash,
    signature
)

---

11. Handshake Transcript

Every handshake message contributes to a transcript.

Conceptually:

transcript =
    CLIENT_HELLO ||
    SERVER_HELLO ||
    CLIENT_KEY_EXCHANGE ||
    SERVER_KEY_EXCHANGE ||
    CLIENT_AUTH ||
    SERVER_AUTH

The transcript hash is:

transcript_hash = SHA256(transcript)

The transcript prevents an attacker from modifying negotiation parameters without detection.

---

12. ClientFinished

After completing key exchange and authentication, the client sends "CLIENT_FINISHED".

CLIENT_FINISHED {
    verify_data
}

"verify_data" proves that the client successfully derived the handshake keys.

Example:

verify_data =
    HMAC(
        client_finished_key,
        transcript_hash
    )

The server MUST verify "verify_data".

If verification fails, the handshake MUST be aborted.

---

13. ServerFinished

The server responds with:

SERVER_FINISHED {
    verify_data
}

The client performs the same verification.

If verification succeeds:

connection_state = ESTABLISHED

Both endpoints may now exchange encrypted application packets.

---

14. Complete Handshake

A complete authenticated handshake looks like:

Client                                      Server
  |                                           |
  | CLIENT_HELLO                             |
  |------------------------------------------>|
  |                                           |
  |                    SERVER_HELLO           |
  |<------------------------------------------|
  |                                           |
  | CLIENT_KEY_EXCHANGE                       |
  |------------------------------------------>|
  |                                           |
  |                    SERVER_KEY_EXCHANGE    |
  |<------------------------------------------|
  |                                           |
  | CLIENT_AUTH                               |
  |------------------------------------------>|
  |                                           |
  |                    SERVER_AUTH             |
  |<------------------------------------------|
  |                                           |
  | CLIENT_FINISHED                           |
  |------------------------------------------>|
  |                                           |
  |                    SERVER_FINISHED         |
  |<------------------------------------------|
  |                                           |
  |          ENCRYPTED APPLICATION DATA       |
  |<=========================================>|

---

15. Handshake Encryption

Once the necessary handshake keys are available, sensitive handshake messages SHOULD be encrypted.

The following information SHOULD be protected:

- Authentication credentials
- Public-key authentication data
- Extensions containing sensitive information
- Application-specific handshake data

Initial negotiation information such as supported versions and cipher suites may remain visible.

---

16. Replay Protection

GSP handshakes MUST provide replay protection.

Endpoints SHOULD use:

client_random
server_random
connection_id
handshake_transcript

to ensure that an old handshake cannot simply be replayed.

PSK-based handshakes MUST additionally incorporate freshness into the authentication calculation.

---

17. Downgrade Protection

A malicious intermediary MUST NOT be able to force both endpoints to negotiate an older protocol version or weaker cryptographic algorithm without detection.

The selected parameters MUST be included in the handshake transcript.

Example:

transcript_hash =
    SHA256(
        CLIENT_HELLO ||
        SERVER_HELLO ||
        KEY_EXCHANGE ||
        AUTH
    )

The "FINISHED" messages authenticate this transcript.

---

18. Handshake Retry

A server MAY send:

HANDSHAKE_RETRY

when additional client information is required.

Example reasons:

RETRY_REQUIRED
INVALID_CLIENT_TOKEN
COOKIE_REQUIRED
RESOURCE_PROTECTION

The server SHOULD use a stateless retry token where possible.

This prevents attackers from forcing the server to allocate expensive handshake state.

---

19. Stateless Retry

A retry token may contain:

token = Encrypt(
    server_secret,
    client_ip_hash ||
    timestamp ||
    client_random
)

The server can validate the token without storing handshake state.

Tokens MUST expire.

Recommended lifetime:

30 seconds

Implementations MAY use a different lifetime according to deployment requirements.

---

20. Handshake Errors

Errors use:

HANDSHAKE_ERROR {
    error_code
    message
}

Recommended error codes:

Code| Name
"0x0001"| UNSUPPORTED_VERSION
"0x0002"| NO_COMMON_CIPHER
"0x0003"| NO_COMMON_KDF
"0x0004"| INVALID_KEY_EXCHANGE
"0x0005"| AUTHENTICATION_FAILED
"0x0006"| INVALID_TRANSCRIPT
"0x0007"| INVALID_FINISHED
"0x0008"| INVALID_MESSAGE
"0x0009"| REPLAY_DETECTED
"0x000A"| HANDSHAKE_TIMEOUT
"0x000B"| PROTOCOL_ERROR
"0x000C"| UNSUPPORTED_EXTENSION
"0x000D"| RESOURCE_LIMIT
"0x000E"| RETRY_REQUIRED

Error messages MUST NOT expose sensitive cryptographic material.

---

21. Handshake Timeout

A handshake MUST have a timeout.

Recommended default:

Handshake Timeout: 10 seconds

An implementation MAY configure a different timeout.

If the timeout expires:

HANDSHAKE_TIMEOUT

MUST be raised locally.

---

22. Key Confirmation

After the handshake completes, both endpoints possess:

client_write_key
server_write_key

The "FINISHED" messages confirm that both endpoints derived compatible key material.

A peer MUST NOT consider the connection established until the required "FINISHED" verification succeeds.

---

23. Session Secrets

After successful handshake establishment, the following logical secrets exist:

client_write_key
server_write_key
client_finished_key
server_finished_key

Implementations SHOULD erase temporary secrets when they are no longer required.

Examples:

ephemeral_private_key
shared_secret
early_secret

---

24. Key Update

Long-lived GSP connections SHOULD periodically rotate traffic keys.

A key update can derive:

new_client_key =
    HKDF-Expand(
        current_client_key,
        "GSP key update",
        key_length
    )

and similarly:

new_server_key =
    HKDF-Expand(
        current_server_key,
        "GSP key update",
        key_length
    )

Key updates MUST ensure that old packet numbers cannot collide with packets encrypted under the new key.

---

25. Connection Migration

GSP supports connection migration when the selected transport permits it.

The connection identity is independent of the underlying network path.

Connection ID
      |
      +---- Wi-Fi
      |
      +---- Mobile Network
      |
      +---- Ethernet

A migration MUST NOT require a complete handshake when the existing cryptographic session remains valid.

Path validation SHOULD be performed before accepting traffic on a new path.

---

26. 0-RTT Data

GSP MAY support optional 0-RTT application data.

0-RTT data introduces replay considerations.

Therefore:

- 0-RTT MUST NOT be enabled by default.
- Applications MUST explicitly opt in.
- Replay-sensitive operations SHOULD NOT use 0-RTT.
- Servers MAY reject 0-RTT data.

Example:

CLIENT_HELLO
+ 0-RTT encrypted application data
---------------------------------------->

---

27. Handshake over GSP/UDP

When using UDP, handshake messages are carried inside GSP packets.

Example:

UDP
 |
 +-- GSP Packet
      |
      +-- Connection ID
      +-- Packet Number
      +-- Handshake Frame

Handshake reliability is handled by GSP retransmission mechanisms.

A lost "SERVER_HELLO" MUST be retransmitted when required.

---

28. Handshake over GSP/TCP

When using TCP, GSP handshake frames are transported over the reliable TCP stream.

TCP provides transport-level reliability, but GSP MUST still maintain handshake state and cryptographic validation.

---

29. Handshake over GSP/QUIC

When GSP runs over QUIC, GSP MAY reuse QUIC's transport security and connection-management facilities where explicitly defined by the transport profile.

The GSP application handshake MUST NOT create conflicting encryption layers unless the selected profile explicitly requires it.

---

30. Extension Negotiation

GSP supports extensions.

An extension has:

extension_id
extension_length
extension_data

Example:

extensions = [
    compression,
    multipath,
    datagrams,
    custom_auth
]

Unknown extensions MAY be ignored when marked optional.

Mandatory extensions MUST be rejected if unsupported.

---

31. Compression Negotiation

Compression is negotiated during the handshake.

Example:

supported_compression:
    none
    lz4

The client and server select one mutually supported method.

Compression SHOULD be disabled for sensitive data when compression side-channel attacks are a concern.

---

32. Recommended Default Profile

A conforming GSP implementation SHOULD support the following baseline profile:

Protocol:
    GSP/1.0

Key Exchange:
    X25519

KDF:
    HKDF-SHA-256

Hash:
    SHA-256

AEAD:
    ChaCha20-Poly1305

Compression:
    None

Connection ID:
    64-bit

Client Random:
    32 bytes

Server Random:
    32 bytes

AES-256-GCM MAY be supported as an alternative AEAD algorithm.

---

33. Security Requirements

Implementations MUST:

- Use a cryptographically secure random generator.
- Never reuse AEAD nonces with the same key.
- Verify the handshake transcript.
- Verify "FINISHED" messages.
- Reject invalid authentication.
- Reject unsupported mandatory extensions.
- Prevent handshake replay.
- Prevent cryptographic downgrade.
- Erase temporary private key material when possible.
- Abort connections when authentication or cryptographic verification fails.

Implementations MUST NOT:

- Transmit private keys.
- Reuse ephemeral keys across unrelated sessions.
- Accept unauthenticated "FINISHED" messages.
- Ignore transcript verification failures.
- silently downgrade to weaker cryptography.

---

34. Minimal Handshake

A minimal anonymous handshake can be implemented as:

Client                                      Server
  |                                           |
  | CLIENT_HELLO                              |
  |------------------------------------------>|
  |                                           |
  |                    SERVER_HELLO           |
  |<------------------------------------------|
  |                                           |
  | CLIENT_KEY_EXCHANGE                       |
  |------------------------------------------>|
  |                                           |
  |                    SERVER_KEY_EXCHANGE    |
  |<------------------------------------------|
  |                                           |
  | CLIENT_FINISHED                           |
  |------------------------------------------>|
  |                                           |
  |                    SERVER_FINISHED        |
  |<------------------------------------------|
  |                                           |
  |          Encrypted GSP Traffic            |

Authentication messages MAY be omitted when the selected profile permits anonymous connections.

---

35. Full Handshake Pseudocode

Client

generate client_random
generate client_connection_id
generate ephemeral X25519 key pair

send CLIENT_HELLO

receive SERVER_HELLO

validate selected parameters

generate shared_secret

derive handshake keys

send CLIENT_KEY_EXCHANGE

if authentication required:
    send CLIENT_AUTH

receive SERVER_AUTH

calculate transcript_hash

send CLIENT_FINISHED

receive SERVER_FINISHED

verify server_finished

enter ESTABLISHED state

Server

receive CLIENT_HELLO

validate supported versions

select cryptographic parameters

generate server_random
generate server_connection_id
generate ephemeral X25519 key pair

send SERVER_HELLO

calculate shared_secret

derive handshake keys

send SERVER_KEY_EXCHANGE

if authentication required:
    send SERVER_AUTH

receive CLIENT_AUTH

verify client authentication

receive CLIENT_FINISHED

verify client_finished

send SERVER_FINISHED

enter ESTABLISHED state

---

36. Security Model

The GSP handshake is designed to provide:

Confidentiality
      +
Integrity
      +
Authentication
      +
Forward Secrecy
      +
Replay Protection
      +
Downgrade Protection

When ephemeral X25519 keys are used correctly, compromise of a long-term authentication key does not reveal previously established session keys.

---

37. IANA-Style Registries

GSP maintains internal registries for:

Handshake Message Types
Cipher Suites
Key Exchange Algorithms
KDF Algorithms
Compression Algorithms
Transport Profiles
Extension IDs
Error Codes

New values MUST NOT conflict with existing registered values.

Experimental implementations MAY use private or experimental ranges.

---

38. Example Session

CLIENT
    |
    | CLIENT_HELLO
    | version = 1.0
    | cipher = ChaCha20-Poly1305
    | kex = X25519
    |
    v
SERVER
    |
    | SERVER_HELLO
    | version = 1.0
    | cipher = ChaCha20-Poly1305
    | kex = X25519
    |
    v
CLIENT
    |
    | CLIENT_KEY_EXCHANGE
    |
    v
SERVER
    |
    | SERVER_KEY_EXCHANGE
    |
    v
CLIENT
    |
    | CLIENT_FINISHED
    |
    v
SERVER
    |
    | SERVER_FINISHED
    |
    v
ESTABLISHED

After this point:

GSP Application Data
====================

Packet
 ├── Connection ID
 ├── Packet Number
 ├── Flags
 └── Encrypted Payload

---

39. Handshake Completion

A handshake is considered complete only when:

1. Version negotiation succeeds.
2. Cryptographic algorithms are successfully negotiated.
3. Key exchange succeeds.
4. Required authentication succeeds.
5. The transcript is valid.
6. "CLIENT_FINISHED" is successfully verified.
7. "SERVER_FINISHED" is successfully verified.

Only then may the connection transition to:

ESTABLISHED

---

40. Reference Handshake

The canonical GSP/1.0 handshake is:

CLIENT                                    SERVER
  |                                         |
  | ClientHello                             |
  |---------------------------------------->|
  |                                         |
  |                  ServerHello            |
  |<----------------------------------------|
  |                                         |
  | ClientKeyExchange                       |
  |---------------------------------------->|
  |                                         |
  |                  ServerKeyExchange      |
  |<----------------------------------------|
  |                                         |
  | ClientAuth                              |
  |---------------------------------------->|
  |                                         |
  |                  ServerAuth             |
  |<----------------------------------------|
  |                                         |
  | ClientFinished                          |
  |---------------------------------------->|
  |                                         |
  |                  ServerFinished         |
  |<----------------------------------------|
  |                                         |
  |====== SECURE GSP SESSION ===============|
  |                                         |

This sequence represents the reference handshake for GSP/1.0. Implementations MAY optimize the sequence through transport-specific profiles, provided that the security properties defined by this specification are preserved.