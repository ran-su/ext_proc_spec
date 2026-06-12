# A93 ext_proc Edge Case Inventory

This document captures edge cases already defined by the upstream A93 proposal
and the docs it links. The local files in this repo are implementation notes,
not the source of truth.

Source material checked:

- A93 / gRPC PR 484, `A93-xds-ext-proc.md`.
- A102 / gRPC PR 510 for `GrpcService`, credentials, header
  representations, and header mutation rules.
- A106 / gRPC PR 520 and A103 for xDS CEL/request attributes.
- A39 for xDS HTTP filter integration and `typed_per_filter_config`.
- A81 and A83 for trusted xDS server behavior and filter-state retention.
- A92 as the related xDS ExtAuthz side-channel filter proposal.
- A60, A66, A79, and A89 for metrics/backend-service labels.
- Envoy PRs 38753 and 45509 where A93 links new ext_proc protocol fields.

It is intentionally conservative. If behavior is not defined in those sources,
this document says `Undefined in current upstream docs` instead of filling in
missing semantics.

## Configuration And Validation

Defined behavior:

- Missing top-level `grpc_service` invalidates the containing xDS resource.
- Missing top-level `processing_mode` invalidates the containing xDS resource.
- `request_body_mode` supports only `NONE` and `GRPC`; other modes invalidate
  the containing xDS resource.
- `response_body_mode` supports only `NONE` and `GRPC`; other modes invalidate
  the containing xDS resource.
- `response_body_mode=GRPC` requires `response_trailer_mode=SEND`; otherwise
  the containing xDS resource is invalid.
- In the linked Envoy processing-mode proto, request and response header modes
  default to `SEND`, request and response body modes default to `NONE`, and
  request and response trailer modes default to `SKIP`.
- `request_trailer_mode` is ignored because gRPC does not send request
  trailers.
- Unsupported attribute names are ignored.
- `deferred_close_timeout` must be a positive `google.protobuf.Duration`.
- If `deferred_close_timeout` is unset, the default is 5 seconds.
- Zero or negative `deferred_close_timeout` is rejected.
- `message_timeout` and `max_message_timeout` are ignored.
- `http_service`, `stat_prefix`, `filter_metadata`, `metadata_options`,
  `on_processing_response`, `disable_clear_route_cache`,
  `route_cache_action`, `send_body_without_waiting_for_header_response`,
  `allow_mode_override`, and `allowed_override_modes` are ignored.
- In `ExtProcPerRoute.overrides`, `processing_mode`, `grpc_service`,
  `request_attributes`, `response_attributes`, and `failure_mode_allow` are
  supported.
- In `ExtProcPerRoute.overrides`, `metadata_options` and
  `grpc_initial_metadata` are ignored.
- The generic `disabled` field in `HttpFilter` and route `FilterConfig`
  wrappers is honored.
- The `disabled` field directly inside `ExtProcPerRoute` is not used.
- The most-specific per-filter config wins; a route-level override can
  re-enable a less-specific disabled filter.
- Invalid ext_proc config follows normal xDS behavior and causes the
  containing resource to be invalid or NACKed.

Undefined in current upstream docs:

- The exact supported attribute name list and value definitions are not
  fully enumerated in A93; A93 delegates to A106/A103 CEL attribute behavior.

## Feature Guards

Defined behavior:

- Client-side support is gated by
  `GRPC_EXPERIMENTAL_XDS_EXT_PROC_ON_CLIENT`.
- Server-side support is gated by
  `GRPC_EXPERIMENTAL_XDS_EXT_PROC_ON_SERVER`.
- A102 says the appropriate feature guard must cover reading the
  `allowed_grpc_services` bootstrap field.
- A93 says the guards will be removed once the feature passes interop tests.

Undefined in current upstream docs:

- Whether a disabled guard rejects versus silently ignores config is described
  as implementation-style dependent, not fixed here.
- Whether every piece of ext_proc filter config parsing is guarded is not
  defined explicitly in A93.

## Compatibility And Versioning

Defined behavior:

- The upstream A93 proposal is still proposal-derived and dated because A93
  is not merged in the referenced status snapshot.
- Generated proto types are authoritative; hand-written field names should
  not be treated as authoritative.
- The draft flow-control diff contains spelling artifacts, so implementations
  should use final generated proto field names.
- If the final Envoy proto keeps `flow_control_init_timeout`, A93 does not
  define whether it applies to gRPC implementations.

Undefined in current upstream docs:

- Capability negotiation with older ext_proc servers that do not implement
  A93 `GRPC` body mode is not defined here.
- Behavior when one peer expects ext_proc flow-control windows and the other
  peer does not send the corresponding update messages is not defined here.
- Final behavior for `flow_control_init_timeout` is not defined in A93.

## A102 GrpcService And Side-Channel Setup

Defined behavior:

- `google_grpc` is required.
- `envoy_grpc` is rejected.
- `google_grpc.target_uri` is required.
- The target URI is validated as a gRPC target URI and checked against the
  resolver registry during xDS validation.
- `channel_credentials_plugin` and `call_credentials_plugin` are supported.
- Legacy `channel_credentials`, `call_credentials`,
  `credentials_factory_name`, and `config` are ignored.
- `stat_prefix`, `per_stream_buffer_limit_bytes`, and `channel_args` are
  ignored.
- If `timeout` is present, it must be a positive
  `google.protobuf.Duration`.
- `timeout` is used as the deadline for ext_proc side-stream RPCs.
- If `initial_metadata` is present, it is validated with A102 header rules
  and sent on ext_proc side-stream RPCs.
- `retry_policy` is ignored; normal gRPC service config or xDS on the
  side-channel is used if retries are needed.
- If the xDS server has `trusted_xds_server`, target URI and credentials from
  the `GrpcService` proto are trusted.
- If the xDS server is not trusted, the target URI must be present in
  bootstrap `allowed_grpc_services`.
- If an untrusted target URI is missing from `allowed_grpc_services`, the xDS
  resource is invalid or NACKed.
- For an allowed untrusted target URI, credentials in the `GrpcService` proto
  are ignored and bootstrap credentials are used.
- At least one supported channel credential is required in the selected
  credential source.
- Supported call credentials from the selected credential source are applied;
  unsupported call credential types are ignored where A102 permits.
- Side-channels are retained or reused across LDS/RDS updates when target URI
  and channel credentials are unchanged.
- Side-channels are recreated when target URI or channel credentials change.
- In-flight data-plane RPCs retain the side-channel they started with across
  xDS resource updates.
- Trace context is propagated so the ext_proc RPC appears as a child span of
  the data-plane RPC.

Undefined in current upstream docs:

- Whether call credentials are attached to channel credentials or applied per
  call is left as an implementation decision.
- If call credentials are attached to channel credentials, the docs require
  channel recreation when call credentials change; otherwise the exact update
  strategy is not defined here.
- Retry safety for an ext_proc side-stream after any ext_proc event has been
  sent is not defined here beyond ignoring `retry_policy`.

## Timeouts And Retries

Defined behavior:

- `message_timeout` is ignored.
- `max_message_timeout` is ignored.
- `override_message_timeout` is ignored.
- If `GrpcService.timeout` is present, it must be a positive
  `google.protobuf.Duration`.
- `GrpcService.timeout` is used as the deadline for ext_proc side-stream
  RPCs.
- `GrpcService.retry_policy` is ignored.
- Normal gRPC service config or xDS on the side-channel is used if retries are
  needed.

Undefined in current upstream docs:

- Per-blocking-event timeout behavior is not defined here.
- Retry safety after a side-stream has sent any `ProcessingRequest` is not
  defined here.
- Tie-breaking between a side-stream deadline and an in-flight data-plane
  cancellation is not defined here.

## Per-RPC ExtProc Stream Lifecycle

Defined behavior:

- The ext_proc side stream is not created until the filter first needs to send
  an event for the data-plane RPC.
- An RPC whose config sends no events creates no side stream.
- There is exactly one ext_proc side stream per data-plane RPC per ext_proc
  filter instance.
- All communication for that filter instance and data-plane RPC uses that
  side stream.
- `protocol_config` is populated only on the first `ProcessingRequest`.
- Later `ProcessingRequest` messages do not repeat `protocol_config`.
- `protocol_config.request_body_mode` and `protocol_config.response_body_mode`
  come from filter config.
- `protocol_config.send_body_without_waiting_for_header_response` is never
  set.
- In normal mode, `flow_control_init` is populated on the first
  `ProcessingRequest`.
- All four initial window sizes are set in `flow_control_init`.
- A default initial window size around 65536 is suggested unless the
  implementation has a better memory-management value.
- Observability mode omits ext_proc-level flow-control init.

Undefined in current upstream docs:

- Exact behavior for side-stream startup failure before the data-plane stream
  is committed is not defined separately from general non-OK ext_proc
  termination.
- Exact behavior for an ext_proc OK close while a required blocking header or
  trailer response is still pending is not explicitly defined here.
- Exact behavior for data-plane cancellation racing with a pending ext_proc
  response is not defined here.

## Event Ordering

Defined behavior:

- Client-to-server event order is headers, zero or more messages, then
  half-close.
- Server-to-client event order is headers, zero or more messages, then
  trailers.
- A trailers-only response is represented as `response_headers` with
  `end_of_stream=true`.
- A trailers-only response does not send a separate `response_trailers` event.
- Request-direction and response-direction events may interleave.
- Configured events may be absent from the ext_proc stream.
- If two events are sent, their relative legal order is preserved.
- Data-plane events are sent to ext_proc as they occur, even if a previous
  event's response has not arrived, when the filter is configured to send the
  event.
- Out-of-order ext_proc responses are protocol errors and are treated as
  ext_proc non-OK failure.
- A body event may be sent while a header response is pending.
- A response for a later event before an earlier required response fails
  ext_proc.

Undefined in current upstream docs:

- A total ordering policy across request and response directions is not
  defined; the docs define legal interleaving and per-direction ordering.

## ProcessingRequest Population

Defined behavior:

- Client headers use `request_headers`.
- Server headers use `response_headers`.
- Header messages include filtered headers according to `forward_rules`.
- `request_headers.end_of_stream` is always false.
- `response_headers.end_of_stream` is true only for trailers-only responses.
- Client messages use `request_body`.
- Server messages use `response_body`.
- In `GRPC` mode, each body `ProcessingRequest` that carries a message
  contains exactly one complete serialized gRPC message.
- gRPC implementations send deframed serialized gRPC message bytes, not raw
  HTTP/2 DATA frames.
- For sent messages, ext_proc runs before compression.
- For received messages, ext_proc runs after decompression.
- gRPC-originated `ProcessingRequest` bodies never set
  `HttpBody.grpc_message_compressed`.
- A client half-close with a last message is represented by
  `request_body.end_of_stream=true`.
- A client half-close without a message is represented by
  `request_body.end_of_stream_without_message=true`.
- An empty gRPC message plus half-close is distinguishable from half-close
  without message.
- Server `response_body.end_of_stream` is always false.
- `request_trailers` is not populated.
- `metadata_context` is not populated.
- `attributes` are populated according to configured request or response
  attributes.
- The fixed attributes key is `envoy.filters.http.ext_proc`.
- `observability_mode` is populated with the config value on every
  `ProcessingRequest`.
- `client_window_update` is sent when returning sidestream-to-upstream or
  sidestream-to-downstream flow-control credit.
- `client_window_update` may be sent alone or attached to another
  `ProcessingRequest`.

Undefined in current upstream docs:

- The exact serialization and compression API boundary is implementation
  specific, but the bytes sent to ext_proc are defined as serialized,
  uncompressed gRPC message bytes.

## Normal-Mode Header And Trailer Gates

Defined behavior:

- Client headers are held until a valid
  `ProcessingResponse.request_headers` arrives.
- Server headers are held until a valid
  `ProcessingResponse.response_headers` arrives.
- Server trailers are held until a valid
  `ProcessingResponse.response_trailers` arrives.
- Allowed header mutations are applied before forwarding client headers.
- Allowed header mutations are applied before forwarding server headers.
- Allowed trailer mutations are applied before forwarding server trailers.
- Header and body responses require `CommonResponse.status=CONTINUE` where
  A93 requires it.
- `CONTINUE_AND_REPLACE` is unsupported and fails ext_proc.

Undefined in current upstream docs:

- Timeout behavior for a header or trailer gate that never receives a
  response is not defined here, except that `message_timeout` and
  `max_message_timeout` are ignored and `GrpcService.timeout` is used as the
  side-stream deadline.

## Normal-Mode Body Ownership

Defined behavior:

- In `GRPC` request body mode, original request body messages are not
  forwarded directly.
- Request body output comes only from
  `ProcessingResponse.request_body.response.body_mutation.streamed_response`.
- In `GRPC` response body mode, original response body messages are not
  forwarded directly.
- Response body output comes only from
  `ProcessingResponse.response_body.response.body_mutation.streamed_response`.
- The filter does not assume one `ProcessingResponse` per
  `ProcessingRequest`.
- The ext_proc server may drop, modify, pass through, replace, or change the
  count of body messages.
- Request-body streamed output honors `end_of_stream`.
- Request-body streamed output honors `end_of_stream_without_message`.
- If ext_proc sends request EOS before the client half-closes, the filter
  forwards request EOS and reads/discards later client messages until the real
  client half-close arrives.
- For response-body streamed output, `streamed_response.end_of_stream` is not
  treated as response completion; A93 says that EOS is honored only on
  client-to-server messages.
- Final response completion is driven by server trailers, not by
  `response_body.end_of_stream`.

Undefined in current upstream docs:

- Maximum ext_proc output expansion, queue size, and memory limits are not
  defined here.
- Exact handling for an ext_proc server that produces no response-body output
  before final server trailers is not separately specified beyond body
  ownership and trailer-driven completion.

## ProcessingResponse Validation

Defined behavior:

- `server_window_update` is applied if present and flow control is negotiated.
- `immediate_response` is handled when `disable_immediate_response=false`.
- `immediate_response` is treated as ext_proc failure when
  `disable_immediate_response=true`.
- `request_drain=true` causes the filter to half-close the ext_proc stream,
  continue sending message bodies received from ext_proc until OK termination,
  and then pass later data-plane messages through as-is.
- `CommonResponse.status=CONTINUE` is validated where required.
- `CONTINUE_AND_REPLACE` is rejected.
- Header mutations and mutation rules are validated.
- `grpc_message_compressed=true` in a body streamed response is rejected.
- Responses must come back in the same order as the events sent by the
  filter.
- A response to a later event before an earlier event is a protocol error.
- Protocol errors are treated as if the ext_proc stream failed with non-OK
  status.

Undefined in current upstream docs:

- The docs do not define recovery from a protocol validation failure; they
  define it as ext_proc failure followed by `failure_mode_allow` handling.

## Expected Response Matching

Defined behavior:

- `ProcessingResponse.request_headers` is sent in response to client headers.
- `ProcessingResponse.response_headers` is sent in response to server headers.
- `ProcessingResponse.request_body` is sent in response to client messages.
- `ProcessingResponse.response_body` is sent in response to server messages.
- `ProcessingResponse.response_trailers` is sent in response to server
  trailers.
- `ProcessingResponse.immediate_response` may be sent in response to any
  data-plane message unless immediate responses are disabled.
- `ProcessingResponse.request_drain` and `server_window_update` are additional
  response fields defined by the linked Envoy PRs.

Undefined in current upstream docs:

- A93 does not define implementation variables such as
  `expected_blocking_response`; those are implementation details.
- The docs do not define a one-to-one body request/response matching scheme;
  body cardinality is explicitly not one-to-one.

## Header, Body, Trailer, And Other Response Fields

Defined behavior:

- For `request_headers` and `response_headers`, `CommonResponse.status` must
  be `CONTINUE`.
- Header responses apply `header_mutation` subject to validation and mutation
  rules.
- Header responses ignore `body_mutation`, `trailers`, and
  `clear_route_cache`.
- For `request_body` and `response_body`, `CommonResponse.status` must be
  `CONTINUE`.
- Body responses use only `body_mutation.streamed_response` in `GRPC` body
  mode.
- Body responses ignore `body_mutation.body`,
  `body_mutation.clear_body`, `header_mutation`, `trailers`, and
  `clear_route_cache`.
- For `response_trailers`, `header_mutation` is applied subject to
  validation and mutation rules.
- `override_message_timeout`, `dynamic_metadata`, and `mode_override` are
  ignored.

Undefined in current upstream docs:

- Semantics for ignored fields beyond "do not alter behavior" are not
  defined here.

## Immediate Response

Defined behavior:

- `immediate_response` is accepted in response to any data-plane event unless
  `disable_immediate_response=true`.
- If immediate responses are disabled, `immediate_response` is treated as
  ext_proc non-OK failure.
- On the client-side filter, accepted `immediate_response` fails the
  data-plane RPC as an out-of-band cancellation with `grpc_status` and
  `details`.
- On the server-side filter, accepted `immediate_response` sends trailers to
  the client with `grpc_status` and `details`.
- If accepted in response to server trailers, it sets final status and
  optional trailer headers.
- Headers are used best effort according to event and filter side.
- HTTP `status` and `body` fields are ignored for gRPC.
- The filter may cancel the ext_proc stream after accepting
  `immediate_response`.

Undefined in current upstream docs:

- Exact cancellation propagation details to an already-started upstream or
  server application are not defined here.
- Tie-breaking when `immediate_response` races with data-plane cancellation
  or ext_proc stream close is not defined here.

## Failure Mode Allow And Non-OK Failure

Defined behavior:

- By default, non-OK ext_proc termination fails the data-plane RPC with
  `INTERNAL`.
- If `failure_mode_allow=true` and the filter is in observability mode, the
  data-plane RPC continues after non-OK ext_proc termination.
- If `failure_mode_allow=true` and the filter is not in observability mode but
  has not yet started sending client or server messages to ext_proc, the
  data-plane RPC continues after non-OK ext_proc termination.
- If any request or response body message has been sent to ext_proc, the
  data-plane RPC fails with `INTERNAL` even when `failure_mode_allow=true`,
  unless the filter is in observability mode.
- Outside observability mode, the `failure_mode_allow` bypass condition is
  global across request and response body directions.
- Once bypassed after an allowed failure, the filter takes no further
  ext_proc action for that data-plane RPC.
- Protocol errors, invalid mutation fields, unsupported responses,
  out-of-order responses, disabled immediate responses, and
  `grpc_message_compressed=true` in streamed output are treated as ext_proc
  failure and then follow `failure_mode_allow`.

Undefined in current upstream docs:

- Failure handling after non-body events but before their blocking responses
  is defined only by observability mode or the global "no body sent" bypass
  condition; the docs do not separately define per-event rollback semantics.

## OK Termination And Drain

Defined behavior:

- OK ext_proc close means no more events need to be sent to ext_proc.
- After OK close, remaining data-plane events proceed unchanged.
- OK close without body processing bypasses later events.
- If request or response body messages are configured to be sent to ext_proc,
  the ext_proc server must drain before OK termination.
- Drain starts when the filter receives `request_drain=true`.
- On `request_drain=true`, the filter half-closes the ext_proc stream.
- During drain, the filter stops reading data-plane messages so normal
  flow-control push-back occurs.
- During drain, the filter continues forwarding message bodies received back
  from ext_proc.
- During drain, the ext_proc server is expected to echo all messages it
  received from the filter without modification until it observes the
  half-close.
- On ext_proc OK close after drain, the filter resumes reading data-plane
  messages.
- After successful drain and OK close, subsequent data-plane messages pass
  through unchanged.

Undefined in current upstream docs:

- Exact behavior when the ext_proc server violates the drain echo expectation
  is not separately defined here, except through general protocol/failure
  handling.
- Exact behavior when drain is requested while a blocking header or trailer
  gate is pending is not explicitly defined here.

## Observability Mode

Defined behavior:

- Configured event contents are copied to the ext_proc server.
- Data-plane events do not wait for ext_proc modification responses.
- Mutations from ext_proc responses are not applied.
- `ProcessingRequest.observability_mode=true` is set.
- Ext_proc-level flow control from PR 45509 is not used.
- Normal HTTP/2 flow control is used for the ext_proc side stream.
- To avoid unbounded buffering, the copy to ext_proc must pass flow control
  before allowing the same message to proceed when required by the
  implementation's flow-control API.
- In C-core, the ext_proc write completes before the message continues.
- In Java, flow control is considered available only after both ext_proc and
  data-plane paths pass flow control.
- In Go, blocking writes are used and the implementation does not wait for an
  ext_proc response.
- On data-plane stream destruction, the ext_proc stream close is delayed by
  `deferred_close_timeout`.
- Deferred close gives the ext_proc server time to read sent data.

Undefined in current upstream docs:

- In observability mode, `failure_mode_allow=true` allows the data-plane RPC
  to continue after non-OK ext_proc termination.
- Exact behavior for an observability side-stream failure after data-plane
  events have already proceeded when `failure_mode_allow=false` is not
  separately defined here.
- Exact policy for dropping observations versus blocking indefinitely under
  sustained observability side-stream backpressure is not defined here beyond
  relying on normal flow-control APIs.

## Header Forwarding, Mutation, And Validation

Defined behavior:

- `forward_rules.allowed_headers` and `forward_rules.disallowed_headers` are
  applied.
- Only configured and allowed headers are included in ext_proc header
  messages.
- If `forward_rules` is unset, all headers are forwarded to the ext_proc
  server.
- If neither `allowed_headers` nor `disallowed_headers` is set, all headers
  are forwarded.
- If both are set, only headers in `allowed_headers` and not in
  `disallowed_headers` are forwarded.
- If only `allowed_headers` is set, only headers in `allowed_headers` are
  forwarded.
- If only `disallowed_headers` is set, all headers except those in
  `disallowed_headers` are forwarded.
- `disallowed_headers` overrides `allowed_headers` when both match.
- Header names must be non-empty, lower-case, valid gRPC header names, length
  `<= 16384`, not `host`, not starting with `:`, and not starting with
  `grpc-`.
- Header values must be valid gRPC header values and length `<= 16384`.
- When writing `HeaderValue` for a `-bin` header, value bytes are
  base64-encoded.
- When reading `HeaderValue`, `raw_value` is used if set; otherwise `value`
  is used.
- When writing `HeaderValue`, `raw_value` is used instead of `value`.
- `HeaderValueOption.append_action` values are supported as A102 describes.
- `keep_empty_value` is not supported; gRPC keeps headers even if mutations
  result in an empty value, regardless of this field.
- Deprecated `append` is not supported.
- `HeaderMutationRules.disallow_all` is supported.
- `disallow_expression` and `allow_expression` are supported and their regexes
  are validated at resource validation time.
- `disallow_is_error` is supported.
- `allow_all_routing`, `disallow_system`, and `allow_envoy` are ignored.
- Even while `disallow_system` is ignored, `:`, `host`, and `grpc-*`
  modifications are never allowed.
- If side-channel header mutation validation fails, the ext_proc stream is
  treated as failed and `failure_mode_allow` is applied.
- Disallowed mutation is ignored when `disallow_is_error=false`.
- Disallowed mutation fails the data-plane RPC when `disallow_is_error=true`.
- Binary headers are encoded and decoded according to the documented
  `HeaderValue` rules.

Undefined in current upstream docs:

- Whether sensitive metadata such as authorization headers, cookies, or trace
  context are forwarded by default is not defined here beyond applying
  `forward_rules`.
- Exact collision behavior for multiple header mutations targeting the same
  key is not defined here beyond A102 append-action support.

## Attributes

Defined behavior:

- Client-to-server events use `request_attributes`.
- Server-to-client events use `response_attributes`.
- A93 says request attributes are populated only on the first
  client-to-server event, and response attributes are populated only on the
  first server-to-client event.
- Exactly one top-level `attributes` map entry is populated with key
  `envoy.filters.http.ext_proc`.
- The value is a `google.protobuf.Struct` containing requested supported
  attributes.
- The supported attribute set is the same as xDS CEL expressions from
  A106/A103.
- A106's initial CEL variable set supports the `request` variable, including
  request path, URL path, host, scheme, method, headers, referer, user agent,
  time, request ID, protocol, and query.
- A103 adds `xds.cluster_metadata.filter_metadata` on both client and server
  sides.
- A103 adds server-side-only source/connection attributes:
  `source.address`, `source.port`, `connection.requested_server_name`,
  `connection.tls_version`, and `connection.sha256_peer_certificate_digest`.
- Unsupported attribute names are ignored.
- Response attributes are not sent on request events, and request attributes
  are not sent on response events.
- `request.path` on request events contains the gRPC method path.

Undefined in current upstream docs:

- A103 notes that supporting attributes not available at client initial
  metadata time, such as response attributes, may need a more complex
  structure; A93 does not add more detail here.

## Flow Control

Defined behavior:

- Normal mode has four independent ext_proc flow-control windows:
  downstream-to-sidestream request body copy, sidestream-to-upstream request
  body output, upstream-to-sidestream response body copy, and
  sidestream-to-downstream response body output.
- The sender for each path tracks the available flow-control window.
- The appropriate send window is decremented by bytes sent.
- Data is not sent when the window is less than or equal to the desired send
  amount until a window update permits it.
- In `GRPC` body mode, if the window is greater than zero, one complete
  message may be sent even if it exceeds the window.
- After an oversized message, the window may go negative and the next message
  is blocked until the window becomes positive.
- Positive and negative window updates are applied immediately.
- `server_window_update.window_increment_downstream_to_sidestream` increases
  the request-message send window.
- `server_window_update.window_increment_upstream_to_sidestream` increases
  the response-message send window.
- `client_window_update.window_increment_sidesteram_to_upstream` is sent when
  request-body output from ext_proc has been read or accepted.
- `client_window_update.window_increment_sidestream_to_downstream` is sent
  when response-body output from ext_proc has been read or accepted.
- Data-plane read flow-control credit is released only after the corresponding
  write path has passed flow control.
- `client_window_update` and `server_window_update` may be attached to a
  message with a oneof payload or sent alone.
- Observability mode does not use ext_proc-level flow-control messages.

Undefined in current upstream docs:

- If the final Envoy proto keeps `flow_control_init_timeout`, whether it
  applies to gRPC implementations is not defined here.
- Exact blocked-queue size limits are not defined here.

## Compression

Defined behavior:

- The ext_proc filter runs above compression.
- Ext_proc receives uncompressed serialized message bytes.
- Received data-plane messages are decompressed before ext_proc processing.
- gRPC-originated `ProcessingRequest` bodies never set
  `grpc_message_compressed`.
- If a body streamed response sets `grpc_message_compressed=true`, the filter
  cancels the ext_proc stream and treats it as non-OK failure.
- Messages received back from ext_proc are sent with compression disabled.

Undefined in current upstream docs:

- No additional compression negotiation behavior is defined here.

## Metrics

Defined behavior:

- Client-side metrics require the `grpc.target` label.
- Client-side metrics optionally populate `grpc.lb.backend_service` when xDS
  cluster/backend-service data is available.
- Server-side metrics do not use client-only labels.
- Client-side histograms use unit seconds:
  `grpc.client_ext_proc.client_headers_duration`,
  `grpc.client_ext_proc.client_half_close_duration`,
  `grpc.client_ext_proc.server_headers_duration`, and
  `grpc.client_ext_proc.server_trailers_duration`.
- Server-side histograms use unit seconds:
  `grpc.server_ext_proc.client_headers_duration`,
  `grpc.server_ext_proc.client_half_close_duration`,
  `grpc.server_ext_proc.server_headers_duration`, and
  `grpc.server_ext_proc.server_trailers_duration`.
- Durations are measured from when the ext_proc filter sees the event until
  it allows the event to continue to the next filter.
- Metrics should follow A79 patterns and not add unrelated per-call metric
  architecture.

Undefined in current upstream docs:

- Whether failed, bypassed, or immediately terminated events record all
  histograms is not defined here.

## Server-Side Support

Defined behavior:

- Server-side ext_proc is in scope and has its own feature guard.
- Server-side support is treated as a separate feature, not a trivial flip of
  client-side code.
- Server-side implementation is expected to need separate filter-chain
  integration, metadata direction names, and metrics labels.
- In server-side normal mode, the same protocol is used, but downstream is
  the transport/client side and upstream is the server application.
- On server-side `immediate_response`, the filter sends trailers to the client
  with `grpc_status` and `details`.

Undefined in current upstream docs:

- Exact server application cancellation and backpressure integration details
  are not defined here.
- Public implementation status in the docs says server-side ext_proc was not
  implemented in the checked C++, Go, or Java PRs.

## Multiple Filters And Filter Composition

Defined behavior:

- The checklist is per-filter-instance behavior.
- A93 does not define special behavior for multiple ext_proc filters on the
  same RPC.

Undefined in current upstream docs:

- Cross-filter body ownership, cross-filter bypass interaction, and
  cross-filter failure composition are not defined here.
