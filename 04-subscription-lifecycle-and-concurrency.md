# Subscription Lifecycle and Concurrency

## Medium: Nsmf and Nupf Lifecycle Coupling Is Correct but Best-Effort

The current flow couples Nsmf Create/Delete to Nupf Create/Delete:

1. Nsmf Create validates the request and resolves the active UE session.
2. SMF creates the UPF subscription.
3. SMF stores the Nsmf-to-Nupf linkage only after UPF Create succeeds.
4. Nsmf Delete attempts the linked UPF Delete and then removes local state.

This is consistent with the Task2 semantics: UPF notifications go directly to
NWDAF, while SMF coordinates the subscription lifecycle. The issue is not the
coupling itself, but that the current implementation is synchronous,
process-local, and best-effort.

For a free5GC-quality implementation, keep the coupling but move the policy out
of the HTTP handler and into a processor/service layer with explicit rollback,
retry, and orphan handling.

## Medium: Process-Local State Is Lost on Restart

Nsmf-to-Nupf linkage is stored in a package-level in-memory map.

Relevant code:

- `../smf/internal/context/nsmf_event_exposure_store.go`

After an SMF restart:

- consumers still hold Nsmf subscription IDs;
- UPF subscriptions may remain active;
- SMF cannot cascade Delete because linkage is gone;
- UPF may continue notifying NWDAF indefinitely.

This is acceptable only if documented as volatile V0 behavior. It should not be
presented as durable subscription support.

## Medium: Orphan Window After UPF Create

The ordering correctly avoids storing local state before UPF Create succeeds.
However, there is still a crash window:

1. UPF returns `201`.
2. SMF crashes before storing local linkage.

The result is an UPF subscription with no SMF record.

This cannot be fully solved with only an in-memory map.

## Medium: Local Cleanup Continues After UPF Delete Failure

Current Delete semantics are deterministic:

1. attempt UPF Delete;
2. log a warning on failure;
3. always remove local state;
4. return `204`.

This is operationally simple but converts every upstream failure into a
potential permanent orphan because SMF discards the data required for retry.

### Recommended Direction

Keep the current behavior only as an explicit best-effort V0 policy.
For a general implementation, consider:

- a `delete-pending` state;
- bounded asynchronous retry;
- operator-visible reconciliation;
- a durable tombstone containing UPF Location.

## Medium: POST Retry Can Create Duplicate UPF Subscriptions

The SMF retries UPF Create on transport errors and 5xx responses.

If UPF creates the subscription but the response is lost, SMF sends another
POST with no idempotency key. The UPF may create a duplicate subscription.

Relevant code:

- `../smf/internal/sbi/nupf_eventexposure_client.go`

### Recommended Direction

Avoid automatic POST retry unless the UPF supports an idempotency mechanism.
At minimum, separate connection failures before request transmission from
ambiguous failures after transmission.

## Medium: Concurrent Delete Can Duplicate Upstream Calls

The handler performs:

1. map lookup;
2. UPF Delete;
3. map deletion.

Two concurrent requests can both obtain the state and both call UPF Delete.
Both may then return `204`.

An atomic state transition such as `active -> deleting` would prevent duplicate
upstream work.

## Low: Store Returns Mutable Pointers

The mutex protects map operations but not the contents of returned state
pointers. Callers can mutate state after the lock is released.

Store immutable values or return copies to make the synchronization boundary
meaningful.

## Low: No Secondary Index or Duplicate Policy

There is no defined policy for duplicate:

- `notifId`;
- SUPI;
- SUPI plus event parameters.

Multiple local and UPF subscriptions can be created for equivalent requests.
This may be valid, but it should be explicit and tested.

## Low: No Expiry or Garbage Collection

The request model can evolve to include expiry, but the current store has no:

- expiration timer;
- maximum lifetime;
- stale subscription scan;
- UPF reconciliation process.
