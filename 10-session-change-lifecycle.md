# Session-Change Lifecycle

## Scope

This document records EES behavior after a subscription has already been
accepted. Create-time session lookup issues are covered separately in
`03-topology-and-session-resolution.md`.

## High: UE Release Does Not Clean Up EES Subscriptions

The current EES state is independent of the PDU session release path.

If a UE is subscribed and later releases the PDU session:

- the local Nsmf subscription can remain until NWDAF sends Delete;
- the linked UPF subscription can remain active;
- UPF may continue notifying NWDAF for a UE session that no longer exists.

Relevant code:

- `../smf/internal/context/nsmf_event_exposure_store.go`
- `../smf/internal/sbi/api_eventexposure.go`

### Recommended Direction

Define a PDU-session-release hook for EES-owned subscriptions. The policy can
be best-effort in V0, but it must be explicit:

1. find EES subscriptions bound to the released session;
2. attempt linked UPF Delete;
3. remove or mark local EES state;
4. log bounded cleanup results without exposing notification URIs.

## High: UPF Relocation Does Not Migrate EES Subscriptions

The current implementation resolves the selected UPF only during Nsmf Create
and stores the resulting UPF Location.

If the UE moves from UPF1 to UPF2, or the session path is otherwise changed:

- SMF does not delete the old UPF subscription;
- SMF does not create a replacement subscription on the new UPF;
- Nsmf Delete still uses the originally stored UPF Location.

This is a general-deployment gap, not a current testbed blocker.

### Recommended Direction

Attach EES linkage to the session/path lifecycle, not only to the original
Create request. On path change, either:

1. delete and recreate the UPF subscription on the new selected UPF; or
2. reject/make unsupported any subscription mode that cannot survive path
   changes.

The selected policy should be covered by tests using a fake resolver and fake
Nupf consumer.

## Medium: Chained UPF Behavior Needs an Explicit Policy

For a path such as:

```text
AN -> I-UPF -> PSA-UPF
```

the current resolver subscribes only the selected anchor/PSA UPF. That may be
valid for V0 if the expected measurement source is the anchor UPF, but it is
not automatically valid for every user-data-usage deployment.

The implementation should document whether usage measurements are collected
from:

- only the PSA/anchor UPF;
- every UPF carrying the PDU session;
- a UPF selected by filter fields such as DNN, S-NSSAI, DNAI, or UE IP.

The same point is referenced in `03-topology-and-session-resolution.md`; this
file records the post-subscription lifecycle impact.
