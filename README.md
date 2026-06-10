# SMF Event Exposure Upstream Readiness

This directory prepares the SMF Event Exposure implementation for an upstream
free5GC contribution. It records findings from the validated `smf/ees-only`
prototype, its integration with local NWDAF and go-upf checkouts, and the work
required to build a clean PR branch from current `upstream/main`.

## Review Snapshot

- Review date: 2026-06-06
- Closure review update: 2026-06-09
- SMF: `8668372` (`ees-only`)
- NWDAF: `2be6137` (`master`)
- go-upf: `76e4149` (`dev/ees-next-version`)
- SMF upstream reference: `796a2de` (`upstream/main`)
- SMF divergence at review time: 107 commits behind, 14 commits ahead

## Categories

1. [Integration compatibility](01-integration-compatibility.md)
2. [Nsmf API and contract behavior](02-nsmf-api-and-contract.md)
3. [Topology and session resolution](03-topology-and-session-resolution.md)
4. [Subscription lifecycle and concurrency](04-subscription-lifecycle-and-concurrency.md)
5. [Architecture and generalization](05-architecture-and-generalization.md)
6. [Testing and upstream integration](06-testing-and-upstream-integration.md)
7. [Branch conflict report](07-branch-conflict-report.md)
8. [Source architecture and generation alignment](08-source-architecture-and-generation.md)
9. [free5GC development and PR readiness](09-free5gc-development-and-pr-readiness.md)
10. [Session-change lifecycle](10-session-change-lifecycle.md)
11. [Logging category alignment](11-logging-category-alignment.md)
12. [Community and upstream contribution](12-community-and-upstream-contribution.md)

## Priority Summary

### Critical

- No currently confirmed issue blocks the tested Create -> Notify flow.

### High

- Nsmf `Location` contains a duplicated `/subscriptions` segment.
- SMF sends the standard `repPeriod`, but the current go-upf implementation
  reads the non-standard `reportPeriod`.
- Multi-AN UPF selection merges candidates from disconnected access networks
  without binding selection to the UE's actual ingress AN.
- Rebasing or merging onto current upstream has no textual conflict, but the
  merged multi-AN test does not compile with upstream's constructor signature.
- Dynamic topology updates do not copy the EES API root into newly added UPFs.
- The generated Event Exposure skeleton is used as a handwritten implementation
  file, but no reproducible OpenAPI regeneration workflow protects the changes.
- UE release and UPF relocation after subscription are not tied back to EES
  cleanup or migration.
- The current 14-commit prototype mixes runtime code, copied specifications,
  design history, and testbed configuration and is not suitable as-is for an
  upstream PR.
- Immediate Nupf retries can amplify a UPF outage under concurrent requests.

### Medium

- The resolver cannot disambiguate multiple PDU sessions for one SUPI.
- Subscription state is process-local and can leave orphan UPF subscriptions.
- The Nupf client is tightly coupled to configuration and lacks the normal
  free5GC consumer abstractions.
- Direct Nupf HTTP calls bypass standard free5GC outbound SBI metrics.
- Outbound Nupf operation results are logged under the inbound SBI category.
- EES state has no explicit application lifecycle owner.
- Runtime comments include patch-history details that should not be carried
  into an upstream PR branch.
- Cross-repository ownership, release baseline, and PR order require public
  maintainer alignment before generated protocol work is finalized.
- UPF response bodies are read without a size limit and can be returned and
  logged without sanitization.
- Handwritten EES identifiers do not consistently follow existing SMF telecom
  initialism conventions.
- Validation does not fully enforce the advertised V0 capability.

## Scope

These documents describe findings, contribution strategy, and recommended
directions only. They do not contain code changes, and this readiness directory
does not modify SMF, NWDAF, or go-upf behavior.
