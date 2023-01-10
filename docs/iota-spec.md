# Iota

An Iota is an opinionated interface for dependencies. It's a language and platform independent way of defining dependencies so they can be swapped out, nested, or composed over, no matter if they are local libraries, WebAssembly, external microservices, or workers over a message queue.

Building towards the Iota spec lets you craft modular monoliths. Applications that are easy to build yet can scale from a single process monolith to a globally distributed application with little to no changes.

The Iota spec defines
- The interfaces of an Iota
- How to bundle and validate Iotas that can be transported and instantiated.
- The interfaces for connecting remote Iota instances.
- How an Iota host should execute Iota actions in sequences or pipelines.

## Inspiration

Iotas take inspiration from many languages, design patterns, and standards including

- JavaScript & node.js for its sandboxing, module system, and asynchronous nature.
- Rust for its safety and types.
- Runtimes and VMs like the JVM, .Net, and BEAM for their primitives and semantics.
- Functional programming for its composability and maintainability.
- Flow based programming for durable orchestration of independent processes.
- WebAssembly and the WebAssembly Component Model for design choices and compatibility.
- Protocols and existing technologies like Rx, Reactive Streams, gRPC, RSocket, COM, and ActiveX and so much more.

## Design choices

- **Async-only**. Iota interfaces must be asynchronous so any dependency can be replaced with another, regardless of where it exists or what it does.
- **Contract Driven & Self describing**. Iota bundles must be able to describe their interface and their configuration both statically (pre-execution) and as part of their exposed API while running.
- **Dependency Injection**. Iotas are passed their requirements from application configuration to maximize flexibility and reusability.
- **Isolation**. Every Iota is independent from one another and can not interact outside of the iota boundary.
- **As secure as possible**. Iota adapters and hosts limit access* to system resources like file systems, network ports, ENV variables, etc.
    - **as much as possible given the guest environment.*
- **Explicit vs implicit**. Iotas do not rely on any mechanism (e.g. namespace, registry, etc) to automatically link with one another. Every iota must be instantiated with its initial configuration and referred to by a unique instance ID.
- **Fault-tolerant**. Iotas propagate success and failure values in the same message so downstream iotas can optionally handle the error or the host can short-circuit execution.
- **Change-resistant**. Internal iota logic must operate on named input values (vs positional) to protect against basic refactoring causing major breakage and excess versioning.

## Definitions

### Iota hosts

Iotas can be used independently, but an Iota host can automate execution from Iota to Iota without additional integration or glue code. An Iota host is defined as a runtime environment that validates Iotas, their composition together, and how they execute in sequence.

### Iota bundles

Iotas can be anything that can respond to requests; native libraries, WebAssembly modules, remote services, workers over a message queue, etc. Some Iotas like WebAssembly & native binaries are distributable as Iota bundles to be fetched and unpacked like traditional dependencies.

### Guest calls

A call into an Iota.

### Host calls

A call from an Iota back to the Iota host or other host environment.

### `iota.yaml`

Iota bundle configuration is minimal. The configuration is a yaml file defined by the following apex definition.

```apexlang
type Configuration {
    "Iota identifier to uniquely refer to an ID across hosts. If omitted, defaults to ID specified in token."
    id: string

    "Version of this Iota. Defaults to version specified in token."
    version: string

    "The path to the Iota binary"
    main: string = "main.bin"

    "The path to the interface specification"
    interface?: string
}
```

For example:

```yaml
---
id: @candle/some_iota
version: 0.1.0-alpha
main: file.wasm
```

`id` and `version` should come from a token if present (see *Verifying authenticity*). If a token is present and the id and/or version do not match, an Iota host should throw an error during validation and refuse to load the module.

## Verifying authenticity

Iota bundles can be signed using PASETO public tokens using the following strategy.

1) Create a new or reuse an existing asymmetric keypair
2) Generate new claims with at least the following:
    - subject: id of the Iota
    - module_hash: sha256 hash of the Iota executable bytes (i.e. file pointed to by `main` or `main.bin` by default)
    - interface_hash: sha256 hash of the interface bytes (i.e. the file pointed to by `interface` in `iota.yaml` or `apex.axdl` by default )
3) Create a new public token by signing the claims with your private key generated in **(1)**.
4) Save the token in a file named `iota.token` in the root of your Iota bundle.

Asserting an Iota bundle is valid requires verifying the token's claims using the the public key from **(1)** and asserting the `module_hash` and `interface_hash` claims match the hashes from the files in the bundle.

*Note: How to retrieve the public keys is left to the implementer.*

## Configuration

Iotas take a single JSONlike object (i.e. `Map<string, any>`) used to initialize its state.

How to instantiate and configure an Iota is specific to the guest implementation.

## Interfaces

Iotas are collections of named interfaces which are, in turn, collections of named actions (i.e. functions). Actions return a message that contains either a successful value or an error (referred to in this spec as a `Result<T>` where `T` is the type of the successful value).

Actions can be defined as one of four variations.

- Fire and Forget (i.e. `(input: Map<string, any>) -> Result<void>`): An action that returns immediately with a generic success/failure.
- Request->Response (i.e. `(input: Map<string, any>) -> Future<Result<T>>`): An action that returns a promise/future.
- Request->Stream (i.e. `(input: Map<string, any>) -> Stream<Result<T>>`): An action that returns a stream of `Result`s.
- Channel (i.e. `(input: Stream<Result<I>>) -> Stream<Result<O>>`): An action that takes in a single stream of `Result`s and returns a stream of `Result`s.

## Health checks

Dependencies can fail and iotas must expose a Request->Response action (`health() -> boolean`) that can be called to determine if an iota is healthy.

## Execution and composition rules

- An action with no named input arguments should be run at any point a host sees fit (e.g. ASAP or whenever a downstream action requires a value from the inputless action).
- Any action with a non-stream input that composes over a stream should be executed on every successful message resulting in a new stream of the action's output type (e.g. `req_stream(...).map(req_res)`).
- Any action with a stream input that composes over a future value should treat the future value as a stream that produces a single message (e.g. `req_channel(stream_from_future(req_res(...)))`.
- An Iota's action should only be executed with a success value. That is, a message that contains an error should cause an action to be skipped and the error propagated to the next action.
    - *Exception*: An action that accepts a `Result` should be passed the unprocessed `Result` to manage its own error handling.

## Implementations

*Note: Some of these implementations are partial, existing works and will adapt to the spec as it matures.*

### WebAssembly Module Iotas

**Protocol**: wasmRS (RSocket + MessagePack)

**Execution details**: WebAssembly module instantiated with application-configured security context (e.g. WASI config, et al)

**Interface & Configuration Definition**

- Static: *proposal* WebAssembly tarball with an `apex.axdl` file in the root.
- Dynamic: *proposal*  an exported ReqRes action named `iota::spec()` that returns a terse, binary representation of the interface.

### WebAssembly Module Iotas

WIP

### Native Iota Binaries

See the [wasmRS](./wasmrs.md) document for more information on the wasmRS protocol.

**Protocol**: RSocket

**Execution details**: Architecture-specific native binary spawned with configuration that connects back to iota host via RSocket

**Interface & Configuration Definition**

- Static: *proposal* Tarball or directory with an `apex.axdl` at the root of the structure
- Dynamic: *proposal* an exposed `ReqRes` action named `spec()`

### RSocket endpoints

**Protocol**: [RSocket](https://rsocket.io)

**Execution details**: Externally managed processes. Considered a single instance.

**Interface & Configuration Definition**

- Static: Externally managed.
- Dynamic: *proposal* an exposed `ReqRes` action named `spec()`

### Iotas as configuration

**Protocol**: *host-specific* implementation detail.

**Execution details**: Configuration that describes functionality as iotas and connected actions.

**Interface & Configuration Definition**

- Static: *proposal* TBD, mix of inference and configuration.
- Dynamic: *proposal* host exposes a `ReqRes` action named `spec` that returns the iota's interface.
