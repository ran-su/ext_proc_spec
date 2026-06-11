# A93 ext_proc Filter Implementation Checklist

This checklist is for implementing the gRPC xDS `ext_proc` filter
described by A93. It is intended to be used together with:

- A93 / gRPC PR 484: https://github.com/grpc/proposal/pull/484
- Envoy PR 38753, adding `GRPC` body mode and related body/drain fields:
  https://github.com/envoyproxy/envoy/pull/38753
- Envoy PR 45509, adding ext_proc flow-control fields:
  https://github.com/envoyproxy/envoy/pull/45509
- A102 / gRPC PR 510, for `GrpcService`, credentials, headers, and header
  mutation rules: https://github.com/grpc/proposal/pull/510
- A39 for xDS HTTP filter integration, A83 for filter-state retention,
  A79/A66/A89/A60 for metrics/backend-service labeling, and A103/A106 for
  attributes/CEL integration. A93 links these proposals and relies on their
  existing mechanisms.
- Implementation-status spot check on 2026-06-11:
  - C++ / C-core draft: https://github.com/grpc/grpc/pull/41704
  - Go setup and normal-mode work: https://github.com/grpc/grpc-go/pull/9073,
    https://github.com/grpc/grpc-go/pull/9086,
    https://github.com/grpc/grpc-go/pull/9174
  - Go A102 dependency: https://github.com/grpc/grpc-go/pull/9175
  - Java ext_proc client implementation: https://github.com/grpc/grpc-java/pull/12792
  - Java GrpcService/header/channel dependencies:
    https://github.com/grpc/grpc-java/pull/12492,
    https://github.com/grpc/grpc-java/pull/12494,
    https://github.com/grpc/grpc-java/pull/12690

This document is implementation guidance. The source of truth remains A93
and the linked proto definitions. Use generated proto types instead of
hand-written field names.

## Implementation Status Snapshot

This section is non-normative and intentionally dated. Use it to understand
implementation sequencing, not to redefine A93 requirements.

As of 2026-06-11:

| Area | Merged work checked | Open or missing work checked | Spec-doc lesson |
| --- | --- | --- | --- |
| A93 proposal | None; proposal PR is not merged. | [grpc/proposal#484](https://github.com/grpc/proposal/pull/484) remains open. | Keep these docs explicitly proposal-derived and dated until A93 lands. |
| C++ / C-core | None inspected for runtime ext_proc. | [grpc/grpc#41704](https://github.com/grpc/grpc/pull/41704) is an open draft with generated protos, xDS config parsing, client-only filter registration, and an `ExtProcFilter` class. The actual call interception body is still a stub. | Config support can land before any real data-plane state transitions, so the flow/state docs describe target behavior, not current C++ runtime behavior. |
| Go | [grpc-go#9054](https://github.com/grpc/grpc-go/pull/9054), [grpc-go#9073](https://github.com/grpc/grpc-go/pull/9073), [grpc-go#9086](https://github.com/grpc/grpc-go/pull/9086), and [grpc-go#9163](https://github.com/grpc/grpc-go/pull/9163) are merged. | [grpc-go#9174](https://github.com/grpc/grpc-go/pull/9174) normal-mode `ClientStream` work and [grpc-go#9175](https://github.com/grpc/grpc-go/pull/9175) A102 `GrpcService` support are open. The normal-mode PR explicitly excludes channel retention, metrics, and observability mode. | The practical sequence is config parsing, filter scaffolding, normal mode, then deferred subsystems. Keep retention, metrics, observability, and flow control as distinct checklist items. |
| Java | [grpc-java#12492](https://github.com/grpc/grpc-java/pull/12492) `GrpcService` config and [grpc-java#12494](https://github.com/grpc/grpc-java/pull/12494) header mutation support are merged. | [grpc-java#12792](https://github.com/grpc/grpc-java/pull/12792) ext_proc client implementation and [grpc-java#12690](https://github.com/grpc/grpc-java/pull/12690) cached channel manager are open. | Java shows the richest visible client-side runtime shape: raw serialized messages, side-stream startup, fail-open paths, observability branches, metrics, and channel caching. |
| Cross-language gaps | None inspected. | No public C++, Go, or Java work inspected here implements server-side ext_proc or clearly completes the A93 protocol-level ext_proc flow-control window mechanism. | Treat server-side support and explicit ext_proc window updates as separate target-state sections, not implied by client-side normal mode or ordinary gRPC backpressure. |

Suggested implementation order:

- [ ] Land config parsing and validation first, guarded by client/server env
  vars.
- [ ] Land side-channel `GrpcService` creation separately from runtime body
  handling, because A102 support tends to be reusable across ext_authz and
  ext_proc.
- [ ] Land client normal mode before observability mode.
- [ ] Keep metrics, channel retention, observability deferred close, and
  protocol-level flow control as separate reviewable slices.
- [ ] Treat server-side support as a separate feature, not a trivial flip of
  client-side code.

## Scope

- [ ] Implement the xDS HTTP filter integration for `ext_proc` on the
  gRPC client side.
- [ ] Implement the xDS HTTP filter integration for `ext_proc` on the
  gRPC server side.
- [ ] Support one ext_proc side stream per data-plane RPC per ext_proc
  filter instance.
- [ ] Support the `GRPC` body send mode introduced for gRPC traffic.
- [ ] Support normal mode and observability mode.
- [ ] Support configured request headers, request messages, request
  half-close, response headers, response messages, and response trailers.
- [ ] Treat this checklist as per-filter-instance behavior. A93 does not
  define special behavior for multiple ext_proc filters on the same RPC.

## Feature Guards

- [ ] Gate client-side support with `GRPC_EXPERIMENTAL_XDS_EXT_PROC_ON_CLIENT`.
- [ ] Gate server-side support with `GRPC_EXPERIMENTAL_XDS_EXT_PROC_ON_SERVER`.
- [ ] Ensure the guard covers parsing of ext_proc filter config.
- [ ] Ensure the guard also covers bootstrap parsing needed by this feature,
  including A102 `allowed_grpc_services`.
- [ ] Add tests that disabled guards reject or ignore ext_proc config in the
  same style as other experimental xDS filters in the implementation.

## xDS Filter Integration

- [ ] Register the ext_proc HTTP filter name according to the existing xDS
  HTTP filter framework from A39.
- [ ] Support top-level HCM/LDS filter config.
- [ ] Support `typed_per_filter_config` overrides as described by A39.
- [ ] Honor the generic `disabled` field in `HttpFilter`.
- [ ] Honor the generic `disabled` field in the route `FilterConfig`
  wrapper used in `typed_per_filter_config`.
- [ ] Do not use the `disabled` field directly in `ExtProcPerRoute`.
- [ ] Use the most-specific per-filter config. For example, a route-level
  override can re-enable a filter that was disabled at virtual-host level.
- [ ] Preserve normal xDS validation behavior: invalid ext_proc config makes
  the containing xDS resource invalid/NACKed.

Tests:

- [ ] LDS top-level config enables the filter.
- [ ] Per-route override changes the processing mode.
- [ ] Per-route override changes the `GrpcService`.
- [ ] Most-specific override wins over less-specific override.
- [ ] Generic disabled wrapper disables the filter.
- [ ] More-specific config re-enables a less-specific disabled filter.

## Top-Level `ExternalProcessor` Config

Supported fields:

- [ ] `grpc_service` must be present.
- [ ] Validate `grpc_service` using A102 rules.
- [ ] `failure_mode_allow` controls whether non-OK ext_proc failure can be
  bypassed before any request or response body message has been sent to
  ext_proc.
- [ ] `processing_mode` is required.
- [ ] Support `request_header_mode`, `response_header_mode`, and
  `response_trailer_mode` according to the Envoy processing mode proto.
- [ ] For `request_body_mode`, support only `NONE` and `GRPC`.
- [ ] For `response_body_mode`, support only `NONE` and `GRPC`.
- [ ] Reject or invalidate unsupported body modes.
- [ ] If `response_body_mode=GRPC`, require `response_header_mode=SEND`.
- [ ] Ignore `request_trailer_mode` because gRPC does not send request
  trailers.
- [ ] Support `request_attributes` for client-to-server events.
- [ ] Support `response_attributes` for server-to-client events.
- [ ] Ignore unsupported attribute names.
- [ ] Support optional `mutation_rules` using A102 validation.
- [ ] Support `forward_rules.allowed_headers` and
  `forward_rules.disallowed_headers`.
- [ ] Support `disable_immediate_response`.
- [ ] Support `observability_mode`.
- [ ] Support `deferred_close_timeout` for observability mode.
- [ ] Use 5 seconds as default `deferred_close_timeout` when unset.
- [ ] Validate configured `deferred_close_timeout` as a positive
  `google.protobuf.Duration`.

Ignored fields:

- [ ] Ignore `message_timeout`.
- [ ] Ignore `max_message_timeout`.
- [ ] Ignore `http_service`.
- [ ] Ignore `stat_prefix`.
- [ ] Ignore `filter_metadata`.
- [ ] Ignore `metadata_options`.
- [ ] Ignore `on_processing_response`.
- [ ] Ignore `disable_clear_route_cache`.
- [ ] Ignore `route_cache_action`.
- [ ] Ignore `send_body_without_waiting_for_header_response`.
- [ ] Ignore `allow_mode_override`.
- [ ] Ignore `allowed_override_modes`.

Tests:

- [ ] Missing `grpc_service` invalidates the resource.
- [ ] Missing `processing_mode` invalidates the resource.
- [ ] Unsupported request body mode invalidates the resource.
- [ ] Unsupported response body mode invalidates the resource.
- [ ] `response_body_mode=GRPC` with response headers not sent invalidates
  the resource.
- [ ] Positive `deferred_close_timeout` is accepted.
- [ ] Zero or negative `deferred_close_timeout` is rejected.
- [ ] Ignored fields do not change runtime behavior.

## Override Config

In `ExtProcPerRoute.overrides`:

- [ ] Support `processing_mode` with the same validation as top-level config.
- [ ] Support `grpc_service` with the same validation as top-level config.
- [ ] Support `request_attributes`.
- [ ] Support `response_attributes`.
- [ ] Support `failure_mode_allow`.
- [ ] Ignore `metadata_options`.
- [ ] Ignore `grpc_initial_metadata`; use initial metadata from
  `grpc_service` instead.

Tests:

- [ ] Override changes body mode from `NONE` to `GRPC`.
- [ ] Override changes the side-channel target.
- [ ] Override changes `failure_mode_allow`.
- [ ] Ignored override fields do not change runtime behavior.

## A102 `GrpcService` And Side-Channel Setup

Validation:

- [ ] Require `google_grpc`.
- [ ] Reject `envoy_grpc`.
- [ ] Require `google_grpc.target_uri`.
- [ ] Validate target URI as a valid gRPC target URI.
- [ ] Check target URI against the resolver registry during xDS validation.
- [ ] Support `channel_credentials_plugin`.
- [ ] Support `call_credentials_plugin`.
- [ ] Ignore legacy `channel_credentials`, `call_credentials`,
  `credentials_factory_name`, and `config`.
- [ ] Ignore `stat_prefix`.
- [ ] Ignore `per_stream_buffer_limit_bytes`.
- [ ] Ignore `channel_args`.
- [ ] If `timeout` is present, validate it as a positive
  `google.protobuf.Duration`.
- [ ] Use `timeout` as the deadline for ext_proc side-stream RPCs.
- [ ] If `initial_metadata` is present, validate with A102 header rules.
- [ ] Send `initial_metadata` on ext_proc side-stream RPCs.
- [ ] Ignore `retry_policy`; use normal gRPC service config or xDS on the
  side-channel if retries are needed.

Trust and bootstrap:

- [ ] If the xDS server has `trusted_xds_server`, trust target URI and
  credentials from the `GrpcService` proto.
- [ ] If the xDS server is not trusted, look up the target URI in
  bootstrap `allowed_grpc_services`.
- [ ] If the untrusted target URI is not present in `allowed_grpc_services`,
  invalidate/NACK the xDS resource.
- [ ] If the untrusted target URI is present, ignore credentials in the
  `GrpcService` proto and use credentials from `allowed_grpc_services`.
- [ ] Require at least one supported channel credential in the selected
  credential source.
- [ ] Apply all supported call credentials from the selected credential
  source; ignore unsupported call credential types where A102 permits that.

Side-channel lifecycle:

- [ ] Create a gRPC channel to the ext_proc server from the parsed
  `GrpcService`.
- [ ] Retain/reuse the side-channel across LDS/RDS updates when target URI
  and channel credentials are unchanged.
- [ ] Recreate the side-channel when target URI or channel credentials
  change.
- [ ] Decide whether call credentials are attached to channel credentials or
  applied per call; if attached, recreate channel when call credentials
  change.
- [ ] Reference-count or otherwise retain the side-channel for in-flight
  data-plane RPCs when xDS resources change.
- [ ] Propagate trace context so the ext_proc RPC appears as a child span of
  the data-plane RPC.

Tests:

- [ ] Trusted xDS server uses proto target and credentials.
- [ ] Untrusted xDS server requires `allowed_grpc_services`.
- [ ] Missing allowed target invalidates resource.
- [ ] Allowed target uses bootstrap credentials, not proto credentials.
- [ ] Channel is reused across irrelevant xDS updates.
- [ ] Channel is recreated on target URI change.
- [ ] Channel is recreated or call credentials are updated on credential
  change.
- [ ] In-flight RPC keeps using a valid retained side-channel after config
  update.

## Per-RPC ExtProc Stream Lifecycle

- [ ] Do not create an ext_proc stream until the filter first needs to send
  an event for the data-plane RPC.
- [ ] Associate the ext_proc stream with exactly that data-plane RPC for this
  filter instance.
- [ ] Send all communication for that filter instance and data-plane RPC on
  that ext_proc stream.
- [ ] Populate `protocol_config` only on the first `ProcessingRequest`.
- [ ] Populate `protocol_config.request_body_mode` and
  `protocol_config.response_body_mode` from filter config.
- [ ] Never set `protocol_config.send_body_without_waiting_for_header_response`.
- [ ] In normal mode, populate `flow_control_init` on the first
  `ProcessingRequest`.
- [ ] Set all four initial window sizes in `flow_control_init`.
- [ ] Use a default initial window size around 65536 unless the
  implementation has a better memory-management value.
- [ ] Do not use ext_proc-level flow control in observability mode.

Tests:

- [ ] No side stream is created for an RPC whose config sends no events.
- [ ] First sent event includes `protocol_config`.
- [ ] Later sent events do not repeat `protocol_config`.
- [ ] First normal-mode event includes all flow-control initial windows.
- [ ] Observability-mode first event omits ext_proc-level flow-control init.

## Event Ordering

- [ ] Maintain client-to-server order: headers, zero or more messages, then
  half-close.
- [ ] Maintain server-to-client order: headers, zero or more messages, then
  trailers.
- [ ] Represent Trailers-Only response as `response_headers` with
  `end_of_stream=true`.
- [ ] Do not send a separate `response_trailers` event for a Trailers-Only
  response.
- [ ] Permit request-direction and response-direction events to interleave.
- [ ] Permit configured events to be absent from the ext_proc stream.
- [ ] If two events are sent, preserve their relative legal order.
- [ ] Send data-plane events to ext_proc as they occur, even if a previous
  event's response has not arrived, when the filter is configured to send
  that event.
- [ ] Treat out-of-order ext_proc responses as protocol errors and ext_proc
  non-OK failure.

Tests:

- [ ] Client headers before client body before half-close.
- [ ] Server headers before server body before trailers.
- [ ] Request and response events can interleave without protocol error.
- [ ] Body event can be sent while header response is pending.
- [ ] Response for a later event before an earlier required response causes
  ext_proc failure.
- [ ] Trailers-Only response produces only `response_headers` with EOS.

## ProcessingRequest Population

Headers:

- [ ] Use `request_headers` for client headers.
- [ ] Use `response_headers` for server headers.
- [ ] Include filtered headers according to `forward_rules`.
- [ ] `request_headers.end_of_stream` is always false.
- [ ] `response_headers.end_of_stream` is true only for Trailers-Only
  response.

Bodies:

- [ ] Use `request_body` for client messages.
- [ ] Use `response_body` for server messages.
- [ ] In `GRPC` mode, put exactly one complete serialized gRPC message in
  each body `ProcessingRequest` that carries a message.
- [ ] In gRPC implementations, send deframed serialized gRPC message bytes,
  not raw HTTP/2 DATA frames.
- [ ] For sent messages, run ext_proc before compression.
- [ ] For received messages, run ext_proc after decompression.
- [ ] Never set `HttpBody.grpc_message_compressed` from gRPC.
- [ ] For client half-close with last message, set
  `request_body.end_of_stream=true`.
- [ ] For client half-close without a message, set
  `request_body.end_of_stream_without_message=true`.
- [ ] For server messages, `response_body.end_of_stream` is always false.

Trailers and metadata:

- [ ] Use `response_trailers` for normal server trailers.
- [ ] Do not populate `request_trailers`.
- [ ] Do not populate `metadata_context`.
- [ ] Populate `attributes` according to configured request/response
  attributes.
- [ ] Use the fixed attributes key `envoy.filters.http.ext_proc`.
- [ ] Populate `observability_mode` with the config value on every
  `ProcessingRequest`.
- [ ] Send `client_window_update` when returning sidestream-to-upstream or
  sidestream-to-downstream flow-control credit.
- [ ] Allow `client_window_update` to be sent alone or attached to another
  `ProcessingRequest`.

Tests:

- [ ] Unary request with message and half-close sends body with EOS.
- [ ] Client-streaming half-close after last message sends
  `end_of_stream_without_message`.
- [ ] Empty gRPC message plus half-close is distinguishable from half-close
  without message.
- [ ] Server message never sets body EOS.
- [ ] Normal trailers use `response_trailers`.
- [ ] Trailers-Only uses `response_headers.end_of_stream=true`.
- [ ] Unsupported attributes are omitted.
- [ ] `observability_mode` bit matches config.

## Normal-Mode Behavior

Headers and trailers:

- [ ] When sending client headers, hold them until a valid
  `ProcessingResponse.request_headers` arrives.
- [ ] Apply allowed header mutations before forwarding client headers.
- [ ] When sending server headers, hold them until a valid
  `ProcessingResponse.response_headers` arrives.
- [ ] Apply allowed header mutations before forwarding server headers.
- [ ] When sending server trailers, hold them until a valid
  `ProcessingResponse.response_trailers` arrives.
- [ ] Apply allowed trailer mutations before forwarding server trailers.

Bodies:

- [ ] Do not forward original request body messages directly in `GRPC` body
  mode.
- [ ] Forward request body messages produced by
  `ProcessingResponse.request_body.response.body_mutation.streamed_response`.
- [ ] Do not forward original response body messages directly in `GRPC` body
  mode.
- [ ] Forward response body messages produced by
  `ProcessingResponse.response_body.response.body_mutation.streamed_response`.
- [ ] Do not assume one `ProcessingResponse` for each body
  `ProcessingRequest`.
- [ ] Allow the ext_proc server to drop, modify, pass through, replace, or
  change the count of body messages.
- [ ] For request-body streamed output, honor `end_of_stream`.
- [ ] For request-body streamed output, honor
  `end_of_stream_without_message`.
- [ ] If ext_proc sends request EOS before the client half-closes, forward
  request EOS and read/discard later client messages until the real
  half-close.
- [ ] For response-body streamed output, do not treat
  `streamed_response.end_of_stream` as response completion; A93 says this
  is honored only on client-to-server messages.

Tests:

- [ ] Header mutation blocks header forwarding until response arrives.
- [ ] Body output count 0, 1, and greater than 1 works.
- [ ] Request body output EOS half-closes upstream.
- [ ] Request body output EOS before real client half-close causes later
  client messages to be discarded.
- [ ] Response body output is forwarded before final server trailers.

## Observability Mode

- [ ] Send configured event contents to the ext_proc server.
- [ ] Do not wait for ext_proc modification responses.
- [ ] Do not apply mutations from ext_proc responses.
- [ ] Set `ProcessingRequest.observability_mode=true`.
- [ ] Do not use ext_proc-level flow control from PR 45509.
- [ ] Rely on normal HTTP/2 flow control for the ext_proc side stream.
- [ ] To avoid unbounded buffering, ensure the copy to ext_proc passes
  flow control before allowing the same message to proceed when required
  by the implementation's flow-control API.
- [ ] In C-core, wait for the ext_proc write to complete before allowing
  the message to continue.
- [ ] In Java, consider flow control available only after both ext_proc and
  data-plane paths pass flow control.
- [ ] In Go, rely on blocking writes and do not wait for an ext_proc
  response.
- [ ] On data-plane stream destruction, delay closing the ext_proc stream
  by `deferred_close_timeout`.

Tests:

- [ ] Headers proceed without waiting for ext_proc response.
- [ ] Body messages proceed without waiting for ext_proc response.
- [ ] Backpressure from ext_proc side stream prevents unbounded buffering.
- [ ] Deferred close gives ext_proc server time to read sent data.

## ProcessingResponse Handling

Headers:

- [ ] For `request_headers` and `response_headers`, require
  `CommonResponse.status=CONTINUE`.
- [ ] Treat `CONTINUE_AND_REPLACE` as unsupported and fail ext_proc.
- [ ] Apply `header_mutation` subject to validation and mutation rules.
- [ ] Ignore `body_mutation` in response to headers.
- [ ] Ignore `trailers` in response to headers.
- [ ] Ignore `clear_route_cache`.

Bodies:

- [ ] For `request_body` and `response_body`, require
  `CommonResponse.status=CONTINUE`.
- [ ] Treat `CONTINUE_AND_REPLACE` as unsupported and fail ext_proc.
- [ ] Use only `body_mutation.streamed_response` for GRPC body mode.
- [ ] Ignore `body_mutation.body`.
- [ ] Ignore `body_mutation.clear_body`.
- [ ] Ignore `header_mutation` in response to body.
- [ ] Ignore `trailers` in response to body.
- [ ] Ignore `clear_route_cache`.
- [ ] If `streamed_response.grpc_message_compressed=true`, cancel ext_proc
  and treat it as non-OK ext_proc failure.

Trailers:

- [ ] For `response_trailers`, apply `header_mutation` subject to validation
  and mutation rules.

Immediate response:

- [ ] Accept `immediate_response` in response to any data-plane event unless
  `disable_immediate_response=true`.
- [ ] If disabled, treat `immediate_response` as ext_proc non-OK failure.
- [ ] On client-side filter, fail the data-plane RPC as an out-of-band
  cancellation with `grpc_status` and `details`.
- [ ] On server-side filter, send trailers with `grpc_status` and `details`.
- [ ] If received in response to server trailers, set final status and
  optional headers in trailers.
- [ ] Use `headers` best effort according to event and filter side.
- [ ] Ignore HTTP `status` and `body` fields.
- [ ] The filter may cancel the ext_proc stream after accepting
  `immediate_response`.

Other response fields:

- [ ] Handle `request_drain` as described in the drain section.
- [ ] Handle `server_window_update` as described in the flow-control section.
- [ ] Ignore `override_message_timeout`.
- [ ] Ignore `dynamic_metadata`.
- [ ] Ignore `mode_override`.

Tests:

- [ ] Unsupported `CONTINUE_AND_REPLACE` fails ext_proc.
- [ ] Header response with body mutation ignores body mutation.
- [ ] Body response with header mutation ignores header mutation.
- [ ] `grpc_message_compressed=true` in streamed response fails ext_proc.
- [ ] `immediate_response` terminates RPC with gRPC status.
- [ ] Disabled immediate response is treated as ext_proc failure.
- [ ] Ignored fields do not alter behavior.

## Header Forwarding, Mutation, And Validation

Forwarding:

- [ ] Apply `forward_rules.allowed_headers`.
- [ ] Apply `forward_rules.disallowed_headers`.
- [ ] Include only configured/allowed headers in ext_proc header messages.

A102 header representation:

- [ ] Validate header names are non-empty, lower-case, valid gRPC header
  names, length <= 16384, not `host`, not starting with `:`, and not
  starting with `grpc-`.
- [ ] Validate header values are valid gRPC header values and length <= 16384.
- [ ] When writing `HeaderValue` for a `-bin` header, base64-encode value
  bytes.
- [ ] When reading `HeaderValue`, use `raw_value` if set, otherwise `value`.
- [ ] When writing `HeaderValue`, use `raw_value`, not `value`.
- [ ] Support `HeaderValueOption.append_action` enum values as A102
  describes.
- [ ] Do not support `keep_empty_value`; an empty resulting value removes the
  header key.
- [ ] Do not support deprecated `append`.

Mutation rules:

- [ ] Support `HeaderMutationRules.disallow_all`.
- [ ] Support `disallow_expression`; validate regex at resource validation
  time.
- [ ] Support `allow_expression`; validate regex at resource validation time.
- [ ] Support `disallow_is_error`.
- [ ] Ignore `allow_all_routing`.
- [ ] Ignore `disallow_system`, while still never allowing `:` or `host`
  modifications.
- [ ] Ignore `allow_envoy`.
- [ ] If side-channel header mutation validation fails, treat ext_proc stream
  as failed and apply `failure_mode_allow`.

Tests:

- [ ] Invalid header mutation from ext_proc fails ext_proc.
- [ ] Disallowed mutation is ignored when `disallow_is_error=false`.
- [ ] Disallowed mutation with `disallow_is_error=true` is treated as an
  ext_proc failure and then follows `failure_mode_allow` handling.
- [ ] `host`, pseudo headers, and `grpc-*` cannot be mutated.
- [ ] Binary headers are encoded/decoded correctly.

## Attributes

- [ ] For client-to-server events, use `request_attributes`.
- [ ] For server-to-client events, use `response_attributes`.
- [ ] Populate exactly one top-level `attributes` map entry with key
  `envoy.filters.http.ext_proc`.
- [ ] Put a `google.protobuf.Struct` value containing requested supported
  attributes.
- [ ] Use the same supported attribute set as xDS CEL expressions from
  A106/A103.
- [ ] Ignore unsupported attribute names.
- [ ] Attribute implementation does not have to use CEL internally, but must
  produce the same values for supported names.

Tests:

- [ ] `request.path` on request events contains the gRPC method path.
- [ ] Unsupported attribute name is ignored.
- [ ] Response attributes are not sent on request events and vice versa.

## Early Termination And Failure

Non-OK ext_proc termination:

- [ ] By default, fail the data-plane RPC with `INTERNAL`.
- [ ] If `failure_mode_allow=true` and no client or server body message has
  been sent to ext_proc yet, allow the data-plane RPC to continue.
- [ ] If any request or response body message has been sent to ext_proc,
  fail the data-plane RPC with `INTERNAL` even when
  `failure_mode_allow=true`.
- [ ] Once bypassed after allowed failure, take no further ext_proc action for
  that data-plane RPC.

OK ext_proc termination:

- [ ] Treat OK close as a signal that no more events need to be sent to
  ext_proc.
- [ ] Let all remaining data-plane events proceed unchanged.
- [ ] If request or response body messages are configured to be sent to
  ext_proc, require the ext_proc server to drain before OK termination.

Drain:

- [ ] On `request_drain=true`, half-close the ext_proc stream.
- [ ] Stop reading messages from the data-plane stream so normal flow-control
  push-back occurs.
- [ ] Continue forwarding message bodies received back from ext_proc.
- [ ] Expect the ext_proc server to echo all messages it received from the
  filter without modification until it observes the half-close.
- [ ] On ext_proc OK close after drain, resume reading data-plane messages.
- [ ] After successful drain/OK close, pass subsequent data-plane messages
  through unchanged.

Tests:

- [ ] Non-OK before any body, `failure_mode_allow=false`, fails INTERNAL.
- [ ] Non-OK before any body, `failure_mode_allow=true`, bypasses ext_proc.
- [ ] Non-OK after request body was sent fails INTERNAL.
- [ ] Non-OK after response body was sent fails INTERNAL.
- [ ] OK close without body processing bypasses later events.
- [ ] Drain path half-closes ext_proc, forwards echoed messages, then bypasses.

## Flow Control

Normal mode paths:

- [ ] Maintain independent flow control for downstream-to-sidestream request
  body.
- [ ] Maintain independent flow control for sidestream-to-upstream request
  body output.
- [ ] Maintain independent flow control for upstream-to-sidestream response
  body.
- [ ] Maintain independent flow control for sidestream-to-downstream response
  body output.
- [ ] Decrement the appropriate send window by bytes sent.
- [ ] Do not send more data when the window is less than or equal to the
  desired send amount, until a window update permits it.
- [ ] In `GRPC` body mode, if the window is greater than zero, allow one
  complete message even if it exceeds the window.
- [ ] After that oversized message, allow the window to go negative and block
  until it becomes positive.
- [ ] Apply positive and negative window updates immediately.
- [ ] On `server_window_update`, increase request-message send window by
  `window_increment_downstream_to_sidestream`.
- [ ] On `server_window_update`, increase response-message send window by
  `window_increment_upstream_to_sidestream`.
- [ ] Send `client_window_update.window_increment_sidesteram_to_upstream`
  when request-body output from ext_proc has been read/accepted.
- [ ] Send `client_window_update.window_increment_sidestream_to_downstream`
  when response-body output from ext_proc has been read/accepted.
- [ ] Only release data-plane read flow-control credit after the corresponding
  write path has passed flow control.

Compatibility details from PR 45509:

- [ ] Use generated proto field names; the draft diff contains spelling
  artifacts such as `sidesteram`.
- [ ] `client_window_update` and `server_window_update` may be attached to a
  message with a oneof payload or sent alone.
- [ ] If the final Envoy proto keeps `flow_control_init_timeout`, decide
  whether it applies to gRPC implementations; A93 says gRPC normal mode
  uses the ext_proc-level mechanism and observability mode does not.

Tests:

- [ ] Each of the four paths can block independently.
- [ ] Blocking sidestream-to-downstream does not deadlock
  downstream-to-sidestream.
- [ ] Negative window after oversized gRPC message blocks the next message.
- [ ] Positive window update unblocks the sender.
- [ ] Negative window update reduces available sending credit.
- [ ] Observability mode does not use ext_proc-level flow-control messages.

## Compression

- [ ] Run the ext_proc filter above compression.
- [ ] Send uncompressed serialized message bytes to the ext_proc server.
- [ ] For received messages, decompress before ext_proc processing.
- [ ] Never set `grpc_message_compressed` in gRPC-originated
  `ProcessingRequest` bodies.
- [ ] If a body streamed response sets `grpc_message_compressed=true`, cancel
  the ext_proc stream and treat it as non-OK failure.
- [ ] Disable compression for messages received back from the ext_proc server.

Tests:

- [ ] Compressed inbound data-plane message reaches ext_proc uncompressed.
- [ ] Outbound message returned by ext_proc is sent with compression disabled.
- [ ] `grpc_message_compressed=true` response follows non-OK failure handling.

## Metrics

Client-side labels:

- [ ] `grpc.target` is required.
- [ ] `grpc.lb.backend_service` is optional and populated from xDS cluster
  name/backend service when available.

Client-side histograms, unit seconds:

- [ ] `grpc.client_ext_proc.client_headers_duration`
- [ ] `grpc.client_ext_proc.client_half_close_duration`
- [ ] `grpc.client_ext_proc.server_headers_duration`
- [ ] `grpc.client_ext_proc.server_trailers_duration`

Server-side histograms, unit seconds:

- [ ] `grpc.server_ext_proc.client_headers_duration`
- [ ] `grpc.server_ext_proc.client_half_close_duration`
- [ ] `grpc.server_ext_proc.server_headers_duration`
- [ ] `grpc.server_ext_proc.server_trailers_duration`

Measurement:

- [ ] Measure from when the ext_proc filter sees the event until it allows
  the event to continue to the next filter.
- [ ] Do not add per-call metric architecture outside A79 patterns.

Tests:

- [ ] Each histogram records on the corresponding event path.
- [ ] Client metrics include `grpc.target`.
- [ ] Client metrics include backend service label when available.
- [ ] Server metrics have no client-only labels.

## Test Matrix Summary

Configuration:

- [ ] Valid minimal config.
- [ ] Invalid missing `grpc_service`.
- [ ] Invalid missing `processing_mode`.
- [ ] Invalid unsupported body mode.
- [ ] Invalid `response_body_mode=GRPC` without response headers.
- [ ] Top-level and per-route override combinations.
- [ ] Trusted and untrusted `GrpcService`.

Runtime happy paths:

- [ ] Unary request/response with headers, one request message, one response
  message, trailers.
- [ ] Client-streaming request with multiple request messages and half-close.
- [ ] Server-streaming response with multiple response messages.
- [ ] Full-duplex bidirectional streaming with interleaved directions.
- [ ] Trailers-Only response.
- [ ] Body modes `NONE`.
- [ ] Body modes `GRPC`.
- [ ] Observability mode.

Body transform:

- [ ] Drop all request body output.
- [ ] Replace request body messages.
- [ ] Expand request body messages.
- [ ] Drop all response body output.
- [ ] Replace response body messages.
- [ ] Expand response body messages.
- [ ] Request output EOS before client half-close.
- [ ] Half-close without message.
- [ ] Empty message with EOS.

Failure and protocol errors:

- [ ] ext_proc non-OK before body with failure mode allow.
- [ ] ext_proc non-OK after body with failure mode allow.
- [ ] ext_proc OK early close without body mode.
- [ ] ext_proc OK close after drain.
- [ ] Out-of-order response.
- [ ] Unsupported `CONTINUE_AND_REPLACE`.
- [ ] Invalid header mutation.
- [ ] `grpc_message_compressed=true`.
- [ ] `immediate_response`.
- [ ] `immediate_response` disabled.

Flow control:

- [ ] All four normal-mode paths.
- [ ] Observability-mode backpressure.
- [ ] Oversized gRPC message relative to window.
- [ ] Window updates alone and attached to other messages.

Resource updates:

- [ ] LDS/RDS update with no target or credential change reuses channel.
- [ ] LDS/RDS update with target change creates new channel.
- [ ] In-flight RPC survives config update by retaining old side-channel.

## Non-Normative Implementation Notes

- [ ] Prefer separate state machines for shared stream lifecycle, request
  direction, response direction, body transform, and flow control.
- [ ] Do not use a single "pending body response per body request" queue.
  Body streams are not one-to-one.
- [ ] Maintain a global `has_sent_any_body_to_ext_proc` flag for
  `failure_mode_allow`.
- [ ] Treat every protocol validation failure as ext_proc stream failure,
  then apply `failure_mode_allow`.
- [ ] Keep generated proto behavior authoritative, especially while linked
  Envoy PRs are still changing.
