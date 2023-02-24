# WebAssembly Reactive Streams (wasmRS) Protocol

WasmRS is an implementation of [reactive streams](https://www.reactive-streams.org) for WebAssembly modules using [RSocket](https://rsocket.io).

## See Also

- [waPC](https://wapc.io)
- [RSocket](https://rsocket.io)
- [Reactive Streams](https://www.reactive-streams.org)
- [Rust WasmRS How-To](./wasmrs-rust-howto.md)

## Protocol details

This implementation was inspired by the [waPC protocol](https://wapc.io/docs/spec/) and addresses key deficiencies like asynchronous calls and streaming.

The protocol differs in that, instead of using several host and guest calls to pass request and response buffers, 2 separate buffers are used for bidirectional communication (sending data from the host to guest and visa versa). Frames of data are sent over these buffers and are similar to those in other multiplexing protocols like HTTP/2. This implementation closely follows the [RSocket protocol](https://rsocket.io/about/protocol) when feasible.

Frames contain a stream ID, allowing a single WebAssembly module instance to handle multiple requests on a single thread. Frames also allow for streaming interaction models, which are incredibly useful in many backend processing scenarios (i.e. stream processing on querying databases).

The developer uses a Reactive Streams (RS) API that hides the details of the protocol so implementing core logic feels familiar.

## Function imports/exports

WebAssembly Module = `wasmrs`

*Host exports*
- `__init_buffers(guestBufferPtr: u32, hostBufferPtr: u32)`
- `__send(size: u32)`
- `__op_list(ptr: u32, len: u32)`

*Guest exports*
- `__wasmrs_init(guestBufferSize: u32, hostBufferSize: u32, maxHostFrameLen: u32): u32` - Initializes the host and guest buffers on the guest.
- `__wasmrs_op_list_request()`
- `__wasmrs_send(readUntil: u32)`

### Initialization

The host first calls `__wasmrs_init` with the desired size of the guest and host buffers. The guest allocates the two buffers and calls `__init_buffers` passing the pointers of the buffers.

If successful, `__wasmrs_init` returns `1` (or `true`). Both sides are now ready to communicate by sending frames.

### Sending frames

After initialization, one side of the protocol sends frames by copying data to the buffers (starting at offset 0).

Writing a frame to either the guest and host buffers is accomplished by writing 3 bytes for the length (Big Endian) followed by frame data itself. After copying the data, a "send" function is called to signal the host or guest to take action on the new data.

* When the host is sending a frame, the host calls `__wasmrs_send` with the offset the guest should read until.

* When the WebAssembly guest is sending a frame, the guest calls `wasmrs::__send` with how much data has been written.

The host and guest implement an event loop reading their buffer up to the position passed in the call above.

### Interaction models

Frames follow the [RSocket protocol](https://rsocket.io/about/protocol) and allow several requests to be processed concurrently (on a single Wasm thread) as well as concurrent streams of input and output data.

The RSocket protocol supports the following [interaction models](https://rsocket.io/about/motivations#interaction-models):

* **Request/Response** - each request receives a single response.
* **Fire-and-Forget** - the client will receive no response from the server.
* **Request Stream** - a single request may receive multiple responses.
* **Request Channel** - bidirectional streaming communication.

At the application level, the code makes use of reactive streams-style types, namely Mono and Flux. Mono represents the a single payload for Request/Response. Flux represents a stream of responses and returned by Request/Stream and Request Channel or a stream of input to a Request Channel.

### Binary Frame Format

WasmRS frames follow [RSocket binary format](https://rsocket.io/about/protocol#framing-format) unless specified.

### Operation Lists

WasmRS modules refer to their imported and exported functions by index and – without pre-sharing that list – there is no way to know what indexes map to what. To solve this, wasmRS hosts can call `__wasmrs_op_list_request` to request the list of operations from the guest. The guest then encodes its list of operations and calls `wasmrs::__op_list` with the memory pointer and length of the encoded data.

The format of the binary data includes a header and a list of operations.

*WasmRS operation list header*

```
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       |   |       |                           |
+-------+---+-------+---------------------------+
|"\0wrs"| v | # ops |    operations...          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

v (u16): Version
# Ops (u32): The number of operations to parse.
```

*WasmRS operation list*

```
Operations

 0                   1
 0 1 2 3 4 5 6 7 8 9 0 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| | |       |   |                   |   |                     |
+-+-+-------+---+-------------------+---+---------------------+
|A|B| Index | N |  Namespace[0..N]  | O |  OpLen[N+2..N+2+O]  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
A (u8): Operation type
B (u8): Direction (import/export)
Index (u32): The operation index to use when calling
N (u16): The length of the Namespace buffer
Namespace (u8[]): The Namespace String as UTF-8 bytes
O (u16): The length of the Operation buffer
Operation (u8[]): The Operation String as UTF-8 bytes
```
