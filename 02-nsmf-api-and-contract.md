# Nsmf API and Contract Behavior

## High: Duplicated Segment in Nsmf Location

The route prefix already includes the collection:

```text
/nsmf-event-exposure/v1/subscriptions
```

The Location builder appends another `/subscriptions/<id>`, producing:

```text
/nsmf-event-exposure/v1/subscriptions/subscriptions/<id>
```

Relevant code:

- `../smf/pkg/factory/config.go`
- `../smf/internal/sbi/api_eventexposure.go`

### Why the Prior Testbed Flow Still Passed

The current NWDAF extracts only the final path segment from `Location`, then
constructs its own Delete URI from the configured SMF endpoint.

Relevant code:

- `../5G_Infrastructure/NWDAF/NWDAF/internal/sbi/consumer/smf_service.go`

### Recommended Direction

Use the conventional split:

- service prefix: `/nsmf-event-exposure/v1`
- collection route: `/subscriptions`
- individual route: `/subscriptions/:subId`

Build the Location from the actual collection path exactly once.

## Medium: Route Prefix Includes a Resource Collection

Putting `/subscriptions` in the router group makes the service difficult to
extend with other TS 29.508 resources and caused the Location defect.

It also makes `GET /` under the group act as a collection-level health
endpoint, even though the same collection is used for subscription creation.

The normal free5GC routing pattern keeps the service root and resource paths
separate.

## Medium: Advertised PERIODIC-Only Capability Is Not Validated

The advertised initial capability says only `PERIODIC` is supported, but
Create does not validate `notifMethod`.

Current behavior:

- response echoes the requested `notifMethod`;
- UPF request always uses `PERIODIC`.

Therefore a consumer can request `ONE_TIME` and receive a response that appears
to accept it while the UPF is configured for periodic reporting.

Relevant code:

- `../smf/internal/sbi/api_eventexposure.go`

### Recommended Direction

Reject unsupported notification methods with `400 ProblemDetails`, or
implement an explicit mapping with documented semantics.

## Medium: Incomplete Input Validation

The current initial-capability validation does not fully validate:

- SUPI syntax;
- URI syntax for `notifUri` and `bundledEventNotifyUri`;
- negative `repPeriod`;
- duplicate measurement types;
- unknown JSON fields.

The current JSON binding ignores fields outside the local reduced request
structure.

## Medium: Validation Is Stricter Than "Contains"

The intended contract says the request must contain:

- an `UPF_EVENT`;
- a `USER_DATA_USAGE_MEASURES` UPF event.

The implementation rejects every additional event or UPF event type. A mixed
subscription containing the required supported event plus another event is
rejected in full.

This is acceptable as an explicit initial-capability limitation, but the
contract and ProblemDetails should clearly say "only" rather than "contains".

## Medium: Subscription Control Semantics Are Not Implemented

The reviewed Nsmf and Nupf schemas include lifecycle and reporting controls
that are not implemented by the current fixed flow:

- `expiry`;
- `maxReports`;
- `immediateFlag`;
- notification activation, deactivation, and retrieval flags;
- sampling and partition criteria;
- notification muting settings;
- subscription termination reporting;
- alternate notification addresses.

These are broader than additional event types or filters. Accepting one without
implementing its lifecycle effect could produce a subscription that behaves
differently from the consumer's request.

### Recommended Direction

Reject unsupported controls explicitly in the first upstream version. Add each
control only with corresponding validation, state, timer/lifecycle behavior,
Nupf mapping, and tests.

## Low: Response Does Not Preserve Full Accepted Event Data

The Create response includes only:

```json
{
  "eventSubs": [
    {
      "event": "UPF_EVENT"
    }
  ]
}
```

It does not return the accepted `upfEvents`, measurement types, granularity, or
bundled notification URI. This reduces resource representation fidelity and
will complicate future GET support.

## Low: Stale Lifecycle Comments

The Create handler still contains comments from an earlier implementation
stage stating that no UPF cascading occurs, although the handler now performs
UPF Create before storing local state.

This does not affect execution but creates maintenance risk.
