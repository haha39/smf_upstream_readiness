# Protocol Interoperability

## Scope

This document covers Nsmf-to-Nupf wire compatibility and feature negotiation.
Transport security and endpoint trust are covered separately.

The local schema review used:

- 3GPP TS 29.508 V19.4.0;
- 3GPP TS 29.564 V19.5.0;
- 3GPP TS 29.571 V19.5.0.

The final upstream baseline remains subject to maintainer guidance in
[free5gc/openapi#75](https://github.com/free5gc/openapi/issues/75).

## High: `ueIpAddress` Uses a Testbed-Specific JSON Shape

The reviewed TS 29.564 V19.5.0 schema defines `ueIpAddress` as the common
TS 29.571 V19.5.0 `IpAddr` object. An IPv4 value is therefore represented as:

```json
{
  "ueIpAddress": {
    "ipv4Addr": "192.0.2.1"
  }
}
```

The current handwritten Nupf request instead sends:

```json
{
  "ueIpAddress": "192.0.2.1"
}
```

Relevant code:

- `../smf/internal/sbi/nupf_eventexposure_client.go`

This shape works only when the peer accepts the same local contract. It should
not be treated as general TS 29.564 interoperability.

### Recommended Direction

Use the model generated from the release selected with free5GC maintainers.
Add an exact JSON assertion so the client cannot silently regress to the
testbed-specific string representation.

## Medium: `supportedFeatures` Is Not Negotiated

Both reviewed Event Exposure schemas include `supportedFeatures`, but the
current flow does not:

- accept or preserve the Nsmf feature bitmask;
- send supported features to the UPF;
- inspect the UPF's negotiated response;
- bind accepted behavior to the negotiated result.

Without negotiation, SMF assumes that every selected UPF supports the fixed
initial behavior.

### Recommended Direction

Define the exact feature bits required by the supported flow and reject or
downgrade requests when the peer does not advertise them. Store negotiated
features with the subscription linkage for later lifecycle operations.

## Medium: Successful UPF Response Data Is Ignored

On `201 Created`, the current client validates only the `Location` header. It
reads but does not decode `CreatedEventSubscription`, including:

- `subscriptionId`;
- the accepted or normalized `subscription`;
- optional immediate `reportList`;
- negotiated `supportedFeatures`.

Relevant code:

- `../smf/internal/sbi/nupf_eventexposure_client.go`

This is sufficient for the current testbed Delete flow, but it discards
protocol information needed for broader interoperability.

### Recommended Direction

Decode the typed success body, validate consistency between `Location` and
`subscriptionId`, and define how immediate reports are handled. Treat missing
or contradictory required response data as an upstream protocol error.

## Medium: UPF Error Semantics Collapse into Generic Bad Gateway

Most UPF failures are converted to `502 UPF_REQUEST_FAILED`, regardless of
whether the peer returned a permanent request error, missing session, conflict,
rate limit, or temporary service failure.

The current path does not preserve:

- typed TS 29.571 ProblemDetails;
- upstream `cause` and `invalidParams`;
- `Retry-After`;
- the distinction between retryable and non-retryable statuses.

### Recommended Direction

Return a typed consumer error carrying the upstream status, bounded
ProblemDetails, retry metadata, and wrapped transport error. The Nsmf boundary
should then apply an explicit mapping policy instead of exposing raw UPF bodies
or converting every case to the same response.

## Low: Nupf `nfId` Ownership Is Ambiguous

The Nupf subscription's `nfId` identifies the NF service consumer creating the
subscription. The implementation normally falls back to the SMF NF instance
ID, which is appropriate, but also allows an independent `nupfEeNfId`
override.

That name can be misread as the target UPF identity and permits a configured
identity that differs from the SMF's NRF identity.

### Recommended Direction

Use the registered SMF NF instance ID by default and require a documented,
standards-based reason before supporting an override. If an override remains,
rename and validate it as the Nupf consumer NF instance ID.
