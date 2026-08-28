# Riffle

Riffle is a transport-independent protocol and Rust ecosystem for discovering,
connecting to, describing, and invoking operations on devices.

Every transport produces the same asynchronous byte-stream abstraction, keeping
device identity and session semantics transport-independent across USB, UART,
TCP, Thread, and future transports.

The ecosystem covers transport-independent discovery, connection, sessions,
descriptors, and dynamic unary and streaming RPC invocation.

## Ecosystem

- [`shuffle`](https://github.com/riffle-foundation/shuffle): CLI argument
  parsing, presentation, invoking discovery and connection, and dynamic
  invocation through Riffle's public domain interfaces.
- [`core`](https://github.com/riffle-foundation/core): host orchestration,
  transport registry, discovery and connection semantics, sessions, and RPC
  dispatch.
- [`protocol`](https://github.com/riffle-foundation/protocol): framing,
  handshake semantics, call envelopes, multiplexing, streaming, and opaque
  application payload transport.
- [`schema`](https://github.com/riffle-foundation/schema): provider-neutral
  operation and schema domain types, validation, and payload encoding and
  decoding interfaces.
- [`schema-protobuf`](https://github.com/riffle-foundation/schema-protobuf): the
  Protobuf schema provider.
- [`resolver`](https://github.com/riffle-foundation/resolver): locating and
  verifying opaque descriptor artifacts.
- [`transport-usb`](https://github.com/riffle-foundation/transport-usb): USB
  discovery and connection.
- [`device`](https://github.com/riffle-foundation/device): device-facing Riffle
  integration for firmware.

## Design

Riffle is a protocol over a connected byte stream. Discovery and connection vary
by transport; sessions, descriptors, and RPC behavior remain
transport-independent.

```text
Shuffle CLI
    |
    v
Host orchestration and transport registry
    |                         |
    v                         v
USB provider            future providers
    |                         |
    +---- connected byte stream ----+
                                     |
                                     v
                           Riffle session and RPC
                                     |
                                     v
                            Riffle wire protocol
```

### `shuffle`

Command-line parsing and presentation. It consumes transport capabilities,
discovered devices, sessions, and schema operations through `core`'s public
domain interfaces.

Transport support is compiled in. Generic commands such as `discover` and
`connect` dispatch through the build-time provider registry.

### Core

- build-time registry of enabled transport providers
- orchestration across those providers
- selection of a provider from a locator scheme
- handshake after connection
- multiplexed session and per-call lifecycle
- unary and streaming RPC dispatch, cancellation, and flow control

Transport providers own USB-, UART-, and IP-specific connection logic. Core
consumes their common byte-stream interface, provider-neutral schema operations,
and opaque encoded payloads.

### Transport providers

A provider owns the facts necessary to turn one transport into connected bytes:

- discovery of candidates
- its locator scheme and locator parsing
- opening and configuring the underlying connection
- transport-level setup and errors
- returning a connected asynchronous byte stream

The session owns the transport-independent Riffle HELLO exchange, schema
negotiation, and RPC messages.

### Protocol

The protocol owns Riffle frame types, framing, handshake semantics, call
envelopes, and the multiplexed call lifecycle. It remains independent of Tokio
and Embassy and operates on buffers or slices rather than I/O traits.

Application request and response items cross the protocol boundary as opaque
payload bytes. The public protocol API is application-schema-neutral. Framing
and control messages are protocol-owned wire semantics independent of any
application-schema provider.

RPC streams are logical protocol streams multiplexed over this framing, with
identical behavior over USB, UART, and TCP.

### Schema and descriptors

- operations and their request and response shapes
- unary and streaming cardinality
- schema values and validation failures
- interfaces for encoding and decoding opaque application payloads

Protobuf is the initial schema provider. The Protobuf adapter loads a
`FileDescriptorSet`, translates it into schema-owned domain values, and
implements the schema encoding and decoding interfaces. Only that adapter may
depend on host Protobuf implementations such as Prost. Neither Prost reflection
types nor raw Protobuf descriptors cross its public boundary.

The resolver treats a descriptor as an opaque artifact plus identity and owns
artifact lookup and verification. The Protobuf adapter owns parsing. The
application composition root selects the concrete schema adapter and supplies it
through the schema-owned interface.

### Firmware

Firmware owns its transport adapter and Riffle session loop. Host and firmware
protocol implementations share Riffle wire semantics. Their schema adapters
share `.proto` application contracts. Each side selects its own Protobuf
runtime, executor, and I/O abstractions. Allocator-free generated or
hand-authored firmware message code contains the embedded Protobuf dependency;
the rest of the firmware consumes the resulting message types.

### Runtime boundary

The host uses Tokio. A connected host transport therefore exposes Tokio's
asynchronous read/write semantics. Firmware may use Embassy or another embedded
executor because Riffle's protocol contract is runtime-independent.
