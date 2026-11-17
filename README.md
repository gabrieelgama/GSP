# GSP — Globalized Secure Protocol

GSP is an experimental binary communication protocol designed to combine transport and application functionality into a single protocol, with encryption enabled by default.

Originally developed as CGP (Cockatiel G Protocol), the project explores an alternative to conventional network communication stacks, focusing on low latency, reduced overhead and peer-to-peer communication.

## Architecture

GSP provides two operating modes:

- `gsp://` — Native GSP communication.
- `gsp+http://` — Compatibility mode using an HTTP-GSP translator over HTTP/2, HTTP/3 or QUIC.

## Packet structure

| Type      | Description      |
|-----------|------------------|
| `0x00001` | Handshake        |
| `0x00002` | Data             |
| `0x00003` | Keep-alive       |
| `0x00004` | Error            |
| `0x00005` | Connection close |

GSP uses binary packet encoding, with support for encryption, compression and header obfuscation.

## Security

The proposed security architecture includes:

- X25519 ephemeral key exchange
- ChaCha20-Poly1305 authenticated encryption
- HMAC-SHA256 authentication tokens
- Handshake transcript binding
- Periodic key rotation
- Traffic padding and obfuscation
- Cryptographic challenges and rate limiting
- CRC32 for accidental corruption detection

## Network model

GSP is designed around direct peer-to-peer connections, with optional distributed proxies for routing and accessibility.

The protocol also explores dynamic port allocation and binary endpoint identifiers as an alternative to conventional DNS-based addressing.

## Features and development goals

- Native peer-to-peer communication
- Encrypted data streaming
- Connection multiplexing
- Device-based authentication
- Hardware-bound badges
- Encrypted network monitoring
- Multi-application support
- Global synchronized events

## Performance

The current performance targets are:

- Protocol overhead between 5% and 10%.
- Connection establishment below 50 ms where network conditions permit.
- Up to 20% lower bandwidth usage compared to equivalent HTTPS workloads.

These targets have not yet been validated through standardized benchmarks. Actual performance depends on network infrastructure, implementation and workload.

## Project status

GSP is currently experimental and under development. The protocol specification and implementation are subject to change.

## Contributing

Contributions, technical discussions and feedback are welcome.

## License

GSP is licensed under the Apache License 2.0.