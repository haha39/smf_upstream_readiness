# Topology and Session Resolution

## Medium: Missing UE Is Handled Only at Create Time

The resolver returns `ErrNoActiveSession` when no SMContext exists for the
requested SUPI, and the Nsmf Create handler maps it to TS 29.571
ProblemDetails.

Relevant code:

- `../smf/internal/context/nupf_event_exposure_resolver.go`
- `../smf/internal/sbi/api_eventexposure.go`

That is a reasonable Create-time behavior. The limitation is that the check is
only a snapshot taken during subscription creation; later UE release, session
release, or UPF relocation is not tied back to the EES subscription lifecycle.
Those post-create issues are tracked in `10-session-change-lifecycle.md`.

## High: Multi-AN Selection Is Not Bound to Actual Ingress AN

The multi-AN change enumerates all configured AN nodes, gathers every reachable
anchor UPF, merges the candidates, and selects from the combined set.

Relevant code:

- `../smf/internal/context/user_plane_information.go`
- `../smf/internal/context/user_plane_information_test.go`

### Why It Works in the Current Testbed

The current topology separates UPFs by S-NSSAI:

- gNB1 -> UPF-EES -> `010203`
- gNB2 -> UPF-EES2 -> `112233`

The requested S-NSSAI leaves only one valid UPF candidate.

### General Deployment Failure Case

If two disconnected AN-UPF pairs support the same DNN and S-NSSAI, the
combined candidate set may select a UPF that is not reachable from the UE's
actual ingress AN.

`UPFSelectionParams` currently contains only:

- DNN;
- S-NSSAI;
- DNAI;
- requested PDU address.

It contains no ingress AN identity or AN address.

### Recommended Direction

Do not solve multi-AN selection by globally merging every AN's candidates.
Selection should be constrained by the UE's actual access path.

A general design should:

1. derive or retain the serving AN identity for the SMContext;
2. resolve that identity to a configured AN topology node;
3. search only paths reachable from that AN;
4. define an explicit fallback when the serving AN cannot be mapped.

## Medium: Resolver Rejects Multiple PDU Sessions for One SUPI

The EES resolver scans the global SMContext pool and requires exactly one
context for the SUPI.

Relevant code:

- `../smf/internal/context/nupf_event_exposure_resolver.go`

### Impact

A normal UE with two PDU sessions cannot create an EES subscription, even when
the request or future contract fields could disambiguate by:

- PDU session ID;
- DNN;
- S-NSSAI;
- UE IP.

The resolver returns `409 MULTIPLE_SESSIONS`.

### Recommended Direction

Introduce a typed session selector and resolve with the most specific supplied
keys. SUPI-only resolution may remain valid only when exactly one eligible
session exists.

## Medium: "Active Session" Means Only "Present in Pool"

The resolver does not check:

- SMContext state;
- UP connection state;
- whether release is in progress;
- whether the selected UPF remains associated.

A context that still exists during teardown can be selected.

## Medium: Resolver Reads Mutable SMContext State Without Locking

The resolver reads `PDUAddress` and `SelectedUPF` without acquiring
`SMContext.SMLock`.

Concurrent session release or path updates can produce an inconsistent
resolution result.

### Recommended Direction

Read the required fields under the SMContext lock and return an immutable
snapshot rather than returning mutable internal pointers.

## Medium: Chained Paths Resolve Only One UPF

The resolver uses `SMContext.SelectedUPF`, which is the anchor/PSA chosen during
UE IP allocation.

For:

```text
AN -> I-UPF -> PSA-UPF
```

the current behavior subscribes only the selected anchor. It does not enumerate
all UPFs carrying the session.

This is documented V0 behavior, not an implementation accident. It becomes a
problem only if usage measurements are expected from every UPF in a chained
path.

## Low: UPF EES Endpoint Is Stored on the Topology Node

Adding `nupfEeApiRoot` per UPF is practical and reasonably general. However,
the endpoint is static configuration rather than being obtained from:

- UPF NF profile discovery;
- a service registry;
- an association capability exchange.

The static endpoint should be treated as a supported deployment mode, not the
only future resolution mechanism.
