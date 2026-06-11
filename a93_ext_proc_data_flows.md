# A93 xDS ext_proc Data Flows

Sources checked:

- gRPC proposal PR 484, `A93-xds-ext-proc.md`: https://github.com/grpc/proposal/pull/484
- Linked Envoy PR 38753 for `GRPC` body mode, `request_drain`, `end_of_stream_without_message`, and `grpc_message_compressed`: https://github.com/envoyproxy/envoy/pull/38753
- Linked Envoy PR 45509 for ext_proc flow control windows and window updates: https://github.com/envoyproxy/envoy/pull/45509
- Linked gRPC proposal PR 510 / A102 for `GrpcService`, `allowed_grpc_services`, credentials, and header mutation handling: https://github.com/grpc/proposal/pull/510
- Implementation-status spot check on 2026-06-11:
  - C++ / C-core draft implementation PR: https://github.com/grpc/grpc/pull/41704
  - Go merged setup PRs and open normal-mode PR: https://github.com/grpc/grpc-go/pull/9073, https://github.com/grpc/grpc-go/pull/9086, https://github.com/grpc/grpc-go/pull/9174
  - Java open client implementation PR: https://github.com/grpc/grpc-java/pull/12792

Assumptions:

- "Client app" means the gRPC application initiating the data-plane RPC.
- "Server app" means the gRPC application handling the data-plane RPC.
- "ext_proc filter" may run in the gRPC client stack or the gRPC server stack.
- "ext_proc callout server" is reached by a per-data-plane-RPC gRPC side stream created by the filter.
- "Control plane server" supplies xDS resources that configure the filter and its `GrpcService` side-channel target.

Implementation reality note:

- The flow diagrams below describe the target A93 behavior, not what every
  language has shipped. As of the 2026-06-11 spot check, visible C++, Go, and
  Java work is client-side first; server-side ext_proc is not implemented in
  those public PRs.
- Implementations are landing in layers. Config parsing and xDS registration
  appear first, then client filter/interceptor scaffolding, then normal-mode
  body/header behavior, then observability, metrics, channel retention, and
  eventually server-side support.
- The current Go normal-mode PR explicitly excludes channel retention,
  metrics, and observability mode. This is useful when reading these flows:
  those boxes are spec-required target behavior, but may still be future work
  for a specific language.
- The C++ draft PR currently has ext_proc config parsing and a filter class,
  but its call interception method is still a stub. Treat C++ data-plane flows
  as design intent until the runtime body is implemented.
- The Java open PR contains the richest visible client-side runtime shape:
  raw-message interception, side-stream handling, request/response mutation,
  fail-open behavior, observability-mode branches, client metrics, and a cached
  channel manager. It is still unmerged.

## 1. Configuration And Side-Channel Setup

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP->>Filter: xDS LDS/RDS route/filter config
    CP->>Filter: ExternalProcessor config: grpc_service, processing_mode, mutation_rules, observability_mode, failure_mode_allow
    CP->>Filter: typed_per_filter_config overrides or disabled flag
    Filter->>Filter: Validate GRPC/NONE body modes and supported fields
    Filter->>Filter: Resolve GrpcService using trusted_xds_server or bootstrap allowed_grpc_services
    Filter-->>EP: Retain or create side-channel gRPC channel when target or credentials change
    Client-->>Filter: Future data-plane RPCs enter configured filter
    Filter-->>Server: Future data-plane RPCs continue after filter decisions
```

## 2. Client-Side Normal Mode, Full Duplex Request And Response

In this flow, the ext_proc filter runs in the client stack. Header and trailer events are held until the ext_proc response arrives. Message bodies in `GRPC` mode are not passed through directly; the ext_proc callout server returns the serialized messages that should continue on the data-plane stream.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: Client-side ext_proc xDS config
    Client->>Filter: Client headers
    Filter->>EP: ProcessingRequest.request_headers + protocol_config + flow_control_init
    EP-->>Filter: ProcessingResponse.request_headers: CONTINUE + optional header_mutation
    Filter->>Server: Mutated client headers

    Client->>Filter: Client message M1
    Filter->>EP: ProcessingRequest.request_body body=M1
    EP-->>Filter: ProcessingResponse.request_body streamed_response body=output message
    Filter->>Server: Forward output request body stream chosen by ext_proc server

    Client->>Filter: Client half-close
    Filter->>EP: request_body end_of_stream or end_of_stream_without_message
    EP-->>Filter: streamed_response end_of_stream or end_of_stream_without_message
    Filter->>Server: Half-close when ext_proc response permits it

    Server-->>Filter: Server headers
    Filter->>EP: ProcessingRequest.response_headers
    EP-->>Filter: ProcessingResponse.response_headers: CONTINUE + optional header_mutation
    Filter-->>Client: Mutated server headers

    Server-->>Filter: Server message S1
    Filter->>EP: ProcessingRequest.response_body body=S1
    EP-->>Filter: ProcessingResponse.response_body streamed_response body=output message
    Filter-->>Client: Forward output response body stream chosen by ext_proc server

    Server-->>Filter: Server trailers
    Filter->>EP: ProcessingRequest.response_trailers
    EP-->>Filter: ProcessingResponse.response_trailers + optional header_mutation
    Filter-->>Client: Mutated server trailers
```

## 3. Server-Side Normal Mode, Full Duplex Request And Response

In this flow, the ext_proc filter runs in the server stack. The same protocol is used, but "downstream" is the transport/client side and "upstream" is the server application.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: Server-side ext_proc xDS config
    Client->>Filter: Client headers over data-plane RPC
    Filter->>EP: ProcessingRequest.request_headers + protocol_config + flow_control_init
    EP-->>Filter: ProcessingResponse.request_headers: CONTINUE + optional header_mutation
    Filter->>Server: Mutated client headers

    Client->>Filter: Client message M1
    Filter->>EP: ProcessingRequest.request_body body=M1
    EP-->>Filter: ProcessingResponse.request_body streamed_response body=output message
    Filter->>Server: Forward output request body stream chosen by ext_proc server

    Client->>Filter: Client half-close
    Filter->>EP: request_body end_of_stream or end_of_stream_without_message
    EP-->>Filter: streamed_response end_of_stream or end_of_stream_without_message
    Filter->>Server: Half-close when ext_proc response permits it

    Server-->>Filter: Server headers
    Filter->>EP: ProcessingRequest.response_headers
    EP-->>Filter: ProcessingResponse.response_headers: CONTINUE + optional header_mutation
    Filter-->>Client: Mutated server headers

    Server-->>Filter: Server message S1
    Filter->>EP: ProcessingRequest.response_body body=S1
    EP-->>Filter: ProcessingResponse.response_body streamed_response body=output message
    Filter-->>Client: Forward output response body stream chosen by ext_proc server

    Server-->>Filter: Server trailers
    Filter->>EP: ProcessingRequest.response_trailers
    EP-->>Filter: ProcessingResponse.response_trailers + optional header_mutation
    Filter-->>Client: Mutated server trailers
```

## 4. Observability Mode

In observability mode, events are copied to the ext_proc callout server, but the data-plane RPC does not wait for ext_proc responses and ext_proc responses are not expected to mutate the RPC. The filter still needs flow-control push-back before allowing a message to proceed if the copy to the ext_proc stream cannot pass normal stream flow control.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: xDS config with observability_mode=true
    Client->>Filter: Client headers/message/half-close
    par Copy request event for observation
        Filter->>EP: ProcessingRequest with observability_mode=true
    and Continue request data plane
        Filter->>Server: Same client event, no mutation wait
    end

    Server-->>Filter: Server headers/message/trailers
    par Copy response event for observation
        Filter->>EP: ProcessingRequest with observability_mode=true
    and Continue response data plane
        Filter-->>Client: Same server event, no mutation wait
    end

    Client-xServer: Data-plane RPC ends
    Filter-->>EP: Deferred close after data-plane destroy, default 5 seconds if unset
```

## 5. Headers-Only Request Or Body Mode NONE

If request and response body modes are `NONE`, only configured header and trailer events are sent to ext_proc. This is also the case where `failure_mode_allow=true` can allow the RPC to continue after a non-OK ext_proc failure, because no body stream replacement has begun.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: processing_mode body modes NONE
    Client->>Filter: Client headers
    Filter->>EP: ProcessingRequest.request_headers
    EP-->>Filter: Header response with optional mutation
    Filter->>Server: Client headers

    Client->>Filter: Client messages and half-close
    Filter->>Server: Pass through directly, no request_body events

    Server-->>Filter: Server headers
    Filter->>EP: ProcessingRequest.response_headers
    EP-->>Filter: Header response with optional mutation
    Filter-->>Client: Server headers

    Server-->>Filter: Server messages
    Filter-->>Client: Pass through directly, no response_body events

    Server-->>Filter: Server trailers
    Filter->>EP: ProcessingRequest.response_trailers if configured
    EP-->>Filter: Trailer response with optional mutation
    Filter-->>Client: Server trailers
```

## 6. Trailers-Only Server Response

For a trailers-only response, server headers are represented on the ext_proc stream as `response_headers` with `end_of_stream=true`. No `response_trailers` event is sent for that trailers-only response.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: xDS config includes response_header_mode=SEND
    Client->>Filter: Request events
    Filter->>Server: Request events after any ext_proc handling
    Server-->>Filter: Trailers-only response
    Filter->>EP: ProcessingRequest.response_headers end_of_stream=true
    EP-->>Filter: ProcessingResponse.response_headers + optional header_mutation
    Filter-->>Client: Mutated trailers-only response
```

## 7. Immediate Response From ext_proc

The ext_proc server may send `immediate_response` in response to any data-plane event unless `disable_immediate_response=true`. If disabled, the filter treats the ext_proc stream as failed.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: xDS config with immediate responses enabled or disabled
    Client->>Filter: Any request or response event reaches filter
    Filter->>EP: ProcessingRequest for that event
    EP-->>Filter: ProcessingResponse.immediate_response grpc_status, details, optional headers

    alt Filter runs on client side
        Filter-->>Client: Fail RPC as out-of-band cancellation with grpc_status
        Filter-xServer: Cancel or stop forwarding data-plane RPC
    else Filter runs on server side
        Filter-->>Client: Send trailers with grpc_status
        Filter-xServer: Stop or cancel interaction with server app as needed
    else disable_immediate_response=true
        Filter-xEP: Treat as ext_proc non-OK failure
        Filter-->>Client: Apply failure_mode_allow behavior
    end
```

## 8. Non-OK ext_proc Failure

A non-OK ext_proc stream status normally fails the data-plane RPC with `INTERNAL`. With `failure_mode_allow=true`, the RPC may continue only if the filter has not started sending request or response messages to the ext_proc stream.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: xDS config with failure_mode_allow true or false
    Client->>Filter: Data-plane event
    Filter->>EP: ProcessingRequest
    EP--xFilter: ext_proc stream terminates non-OK or protocol error

    alt failure_mode_allow=false
        Filter-->>Client: Fail data-plane RPC with INTERNAL
        Filter-xServer: Stop forwarding
    else failure_mode_allow=true and no body events were sent to ext_proc
        Filter->>Server: Continue data-plane RPC without further ext_proc action
        Server-->>Client: Normal data-plane response path
    else failure_mode_allow=true but body events were sent to ext_proc
        Filter-->>Client: Fail data-plane RPC with INTERNAL
        Filter-xServer: Stop forwarding because body replacement stream is unsafe to bypass
    end
```

## 9. OK Early Termination Without Drain

If the ext_proc stream ends with OK, the filter stops sending events to ext_proc and lets remaining data-plane events proceed unchanged. This is valid when message body modes are not being used, or when no undrained body messages can be dropped.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: xDS ext_proc config
    Client->>Filter: Initial data-plane event
    Filter->>EP: ProcessingRequest
    EP-->>Filter: ProcessingResponse, then OK stream close
    Filter->>Server: Current event after any accepted mutation
    Client->>Filter: Later request events
    Filter->>Server: Later request events pass through unchanged
    Server-->>Filter: Later response events
    Filter-->>Client: Later response events pass through unchanged
```

## 10. OK Early Termination With Drain For Body Mode GRPC

If request or response body events are being sent, the ext_proc callout server must drain before OK termination. It first asks for `request_drain`; the filter half-closes the ext_proc stream and stops reading from the data plane so normal flow-control push-back applies. The ext_proc server echoes all received messages, then closes OK. After that, future data-plane messages pass through unchanged.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: processing_mode request_body_mode=GRPC or response_body_mode=GRPC
    Client->>Filter: Body message
    Filter->>EP: ProcessingRequest.request_body
    EP-->>Filter: ProcessingResponse request_drain=true
    Filter-->>EP: Half-close ext_proc stream
    Filter-->>Client: Stop reading from data plane to apply push-back
    EP-->>Filter: Echo all already received body messages as streamed_response
    Filter->>Server: Echoed body messages continue on data plane
    EP-->>Filter: OK status close after observing ext_proc half-close
    Filter-->>Client: Resume reading data-plane messages
    Client->>Filter: Later request messages
    Filter->>Server: Later request messages pass through unchanged
    Server-->>Filter: Later response messages
    Filter-->>Client: Later response messages pass through unchanged
```

## 11. Client Half-Close Encodings

gRPC request trailers do not exist. A client half-close is represented in request body events.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: request_body_mode=GRPC

    alt Half-close with last message
        Client->>Filter: Last client message plus half-close
        Filter->>EP: request_body body=last_message, end_of_stream=true
        EP-->>Filter: streamed_response body=message, end_of_stream=true
        Filter->>Server: Message, then half-close
    else Half-close without a message
        Client->>Filter: Half-close only
        Filter->>EP: request_body end_of_stream_without_message=true
        EP-->>Filter: streamed_response end_of_stream_without_message=true
        Filter->>Server: Half-close only
    end
```

## 12. ext_proc Rewrites Request Message Count

In `GRPC` body mode, the ext_proc callout server owns the resulting body stream. It may drop, replace, or expand messages; the response message count does not need to match the original request message count.
The filter does not match individual `ProcessingResponse.request_body` messages back to specific `ProcessingRequest.request_body` messages.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: request_body_mode=GRPC
    Client->>Filter: Client message M1
    Filter->>EP: request_body body=M1
    Client->>Filter: Client message M2
    Filter->>EP: request_body body=M2

    alt Drop or buffer input
        EP-->>Filter: No output for M1 before later input or EOS
        Filter->>Server: Forward only ext_proc output, not original M1
    else Replace one message
        EP-->>Filter: streamed_response body=M1_rewritten
        Filter->>Server: M1_rewritten
    else Expand stream
        EP-->>Filter: streamed_response body=A
        EP-->>Filter: streamed_response body=B
        Filter->>Server: A
        Filter->>Server: B
    end
```

## 13. ext_proc Rewrites Response Message Count

The same body-stream ownership applies to server-to-client messages when `response_body_mode=GRPC`. A valid config with response body mode `GRPC` requires `response_header_mode=SEND`.
The filter forwards the ext_proc output stream and does not require a one-to-one response for each input server message.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: response_header_mode=SEND and response_body_mode=GRPC
    Server-->>Filter: Server headers
    Filter->>EP: response_headers
    EP-->>Filter: response_headers CONTINUE
    Filter-->>Client: Server headers

    Server-->>Filter: Server message S1
    Filter->>EP: response_body body=S1

    alt Drop or buffer server message
        EP-->>Filter: No output for S1 before later input or trailers
        Filter-->>Client: Forward only ext_proc output, not original S1
    else Replace server message
        EP-->>Filter: streamed_response body=S1_rewritten
        Filter-->>Client: S1_rewritten
    else Expand response stream
        EP-->>Filter: streamed_response body=X
        EP-->>Filter: streamed_response body=Y
        Filter-->>Client: X
        Filter-->>Client: Y
    end
```

## 14. Protocol Error Or Unsupported Response Field

Several ext_proc responses become protocol failures in gRPC: out-of-order responses, `CONTINUE_AND_REPLACE`, `grpc_message_compressed=true` in a response, invalid header mutation, or other unsupported required semantics.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: xDS config with mutation_rules and failure_mode_allow
    Client->>Filter: Data-plane event E1
    Filter->>EP: ProcessingRequest for E1
    Client->>Filter: Later data-plane event E2
    Filter->>EP: ProcessingRequest for E2

    EP-->>Filter: Response for E2 before E1, or unsupported field/status
    Filter-xEP: Cancel ext_proc stream and treat as non-OK failure

    alt failure_mode_allow permits bypass
        Filter->>Server: Continue only if no body events were sent to ext_proc
        Server-->>Client: Continue data-plane RPC
    else bypass not allowed
        Filter-->>Client: Fail RPC with INTERNAL
        Filter-xServer: Stop forwarding
    end
```

## 15. Four Independent Flow-Control Paths In Normal Mode

The linked flow-control proto adds initial windows and update messages so the four body paths can be pushed back independently. This avoids the deadlock where both body directions would otherwise push back through one ext_proc HTTP/2 stream.

```mermaid
flowchart LR
    CP["control plane server"]
    Client["client app"]
    Filter["ext_proc filter"]
    EP["ext_proc callout server"]
    Server["server app"]

    CP -->|"xDS processing_mode + flow-control-capable protocol"| Filter

    Client -->|"1. downstream_to_sidestream: request body copy"| Filter
    Filter -->|"ProcessingRequest.request_body, window decremented"| EP
    EP -->|"server_window_update: downstream_to_sidestream"| Filter

    EP -->|"2. sidestream_to_upstream: rewritten request body"| Filter
    Filter -->|"request body to upstream, then client_window_update"| Server

    Server -->|"3. upstream_to_sidestream: response body copy"| Filter
    Filter -->|"ProcessingRequest.response_body, window decremented"| EP
    EP -->|"server_window_update: upstream_to_sidestream"| Filter

    EP -->|"4. sidestream_to_downstream: rewritten response body"| Filter
    Filter -->|"response body to downstream, then client_window_update"| Client
```

## 16. Compression Interaction

gRPC sends uncompressed serialized messages to ext_proc because the filter is above compression on sends and after decompression on receives. If an ext_proc response sets `grpc_message_compressed=true`, gRPC treats that as an ext_proc failure.

```mermaid
sequenceDiagram
    participant CP as control plane server
    participant Client as client app
    participant Filter as ext_proc filter
    participant EP as ext_proc callout server
    participant Server as server app

    CP-->>Filter: xDS ext_proc config
    Client->>Filter: Serialized message before transport compression
    Filter->>EP: request_body body=uncompressed message, grpc_message_compressed not set
    EP-->>Filter: streamed_response body=message, grpc_message_compressed=false
    Filter->>Server: Message continues, gRPC disables compression for messages returned by ext_proc

    EP-->>Filter: streamed_response grpc_message_compressed=true
    Filter-xEP: Cancel ext_proc stream and treat as non-OK failure
    Filter-->>Client: Apply failure_mode_allow behavior
```
