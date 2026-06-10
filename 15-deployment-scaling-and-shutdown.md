# Deployment Scaling and Shutdown

## Scope

This document covers runtime behavior when SMF is replicated, drained, or
terminated. Process restart and orphan creation are also discussed in
`04-subscription-lifecycle-and-concurrency.md`.

## High: Process-Local State Does Not Support Multiple SMF Instances

Each SMF process owns an independent in-memory subscription map.

In a replicated deployment:

1. Nsmf Create may be handled by SMF-A.
2. The Nsmf subscription and UPF linkage exist only in SMF-A.
3. A later Delete may be routed to SMF-B.
4. SMF-B returns `404` and cannot clean the UPF subscription.

This failure does not require a crash or restart; ordinary load balancing is
enough to expose it.

### Recommended Direction

Choose and document one deployment model:

- a shared durable repository keyed by Nsmf subscription ID;
- explicit request affinity with a recovery/reconciliation mechanism; or
- a declared single-instance limitation for the first upstream version.

Request affinity alone is not sufficient for failover or restart, so it should
not be presented as durable lifecycle support.

## High: Graceful Shutdown Has No EES Drain Policy

The normal SMF shutdown sequence stops PFCP, deregisters from NRF, shuts down
the SBI server, and stops metrics. EES subscriptions are not part of that
sequence.

The current implementation does not explicitly:

- stop accepting new EES Create requests before deregistration;
- wait for in-flight Nupf Create/Delete operations;
- mark active subscriptions for reconciliation;
- attempt bounded cleanup of linked UPF subscriptions;
- report how many subscriptions remain.

As a result, graceful termination can create the same orphan conditions as an
unexpected crash.

### Recommended Direction

Give the EES service an application-owned lifecycle:

1. mark the service as draining;
2. reject new Create requests with an appropriate temporary-failure response;
3. allow bounded completion of in-flight operations;
4. persist cleanup intent or reconcile active linkage according to policy;
5. log a final non-sensitive subscription count;
6. then complete SBI shutdown and NRF deregistration in the agreed order.

The shutdown timeout must be bounded so an unavailable UPF cannot prevent SMF
termination indefinitely.

## Medium: In-Flight Operations Have No Explicit State Machine

The repository stores completed subscriptions but does not represent
transitional states such as:

- `creating`;
- `active`;
- `deleting`;
- `delete-pending`;
- `orphaned`.

This makes concurrent requests, shutdown, retry, and reconciliation difficult
to coordinate. For example, a concurrent Delete can race with another Delete,
and a shutdown cannot distinguish a fully linked subscription from a Create
that received `201` but was not persisted.

### Recommended Direction

Keep lifecycle states in the processor/repository boundary, not in the HTTP
handler. Define atomic transitions and make each transition idempotent where
possible.

## Medium: No Capacity or Admission Policy

The local store has no maximum subscription count, per-consumer quota, or
admission control. Each accepted request also performs a synchronous outbound
UPF operation.

At scale, a consumer can therefore grow process memory and occupy handler
goroutines even when expiry and cleanup are not available.

### Recommended Direction

Define operational limits independently from testbed values:

- maximum active and in-flight subscriptions;
- bounded request body size;
- per-target outbound concurrency;
- overload response and retry guidance;
- metrics for active, pending, rejected, and orphaned subscriptions.

Do not use SUPI, UE IP, notification URI, or subscription ID as metric labels.
