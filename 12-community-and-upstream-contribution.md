# Community and Upstream Contribution

## Scope

This document records the community-facing contribution strategy for SMF Event
Exposure. It is based on:

- the Linux Foundation LFS114 summary and contribution guidance;
- observed contribution patterns in official free5GC repositories;
- the public `free5gc/openapi` Issue opened to confirm the preferred direction.

Technical implementation and test requirements remain in
`09-free5gc-development-and-pr-readiness.md`.

## Community Alignment Comes Before Cross-Repository Implementation

LFS114 describes the normal contribution flow as:

```text
Identify an issue
  -> discuss the intended contribution
  -> fork and create a feature branch
  -> implement and test
  -> submit a PR
  -> participate in review
```

Event Exposure affects shared OpenAPI definitions, SMF behavior, UPF behavior,
and deployment configuration. It is therefore appropriate to ask maintainers
about ownership and PR order before finalizing generated protocol code.

The public Issue should establish:

- whether OpenAPI support is required before the SMF PR;
- whether OpenAPI and SMF PRs may be reviewed in parallel;
- the supported 3GPP release baseline;
- the specification source and generator version;
- generated package and shared-model placement;
- whether a temporary handwritten consumer is acceptable.

Internal names such as `Task2`, patch numbers, prototype branch names, and
testbed-specific terminology should not appear in public issues, production
code, commits, or upstream PR descriptions.

## Repository Ownership

free5GC separates repositories by responsibility. Event Exposure changes
should follow the same boundary:

| Repository | Intended ownership |
| --- | --- |
| `free5gc/openapi` | Shared 3GPP models and generated API clients |
| `free5gc/smf` | SMF handler, processor, resolver, repository, and consumer integration |
| `free5gc/go-upf` | UPF Nupf Event Exposure service implementation |
| `free5gc/free5gc` | Generic deployment and sample configuration |
| Local testbed repository | Laboratory addresses, topology, credentials, and experiment scripts |

Do not place copied specifications, internal design history, or deployment
specific values in the SMF PR merely to explain the implementation.

## Observed free5GC Precedents

The following merged PRs show that shared protocol support is commonly added
to `free5gc/openapi` and then consumed by an NF:

- [openapi#11](https://github.com/free5gc/openapi/pull/11) added Traffic
  Influence models and generated client files.
- [openapi#13](https://github.com/free5gc/openapi/pull/13) added Converged
  Charging models and client support before SMF charging integration.
- [openapi#42](https://github.com/free5gc/openapi/pull/42) added Release 17 BSF
  OpenAPI support before later NF integrations.
- [smf#94](https://github.com/free5gc/smf/pull/94) integrated Converged Charging
  in SMF using the OpenAPI package.
- [smf#176](https://github.com/free5gc/smf/pull/176) shows that a consumer may
  combine shared OpenAPI models with handwritten HTTP transport, although that
  is weaker than using a generated client.

These examples support an OpenAPI-first direction, but they do not constitute
a published rule. The maintainer response to the public Issue remains
authoritative for this contribution.

## Current Event Exposure Context

At the time of review:

- `free5gc/openapi` contains an `smf/EventExposure` generated client;
- its shared Nsmf models do not include the reviewed `UPF_EVENT`,
  `upfEvents`, and `bundledEventNotifyUri` fields;
- no generated `upf/EventExposure` package was found;
- [go-upf#95](https://github.com/free5gc/go-upf/pull/95) uses handwritten Nupf
  data models and remains open;
- [free5gc#1044](https://github.com/free5gc/free5gc/pull/1044) separately
  proposes deployment configuration related to that UPF work.

The separation of go-upf functionality and `free5gc` configuration supports
keeping testbed or deployment changes out of the SMF code PR.

Review feedback on [go-upf#79](https://github.com/free5gc/go-upf/pull/79)
also shows that maintainers are sensitive to experiment-specific YAML and
README changes that do not match existing project style.

## Recommended PR Sequence

The preferred sequence, pending maintainer confirmation, is:

1. Discuss ownership and release baseline in `free5gc/openapi`.
2. Submit an OpenAPI PR containing only the required models/client changes.
3. Obtain a usable OpenAPI revision or release for the dependent NF.
4. Submit the clean SMF implementation PR against current `upstream/main`.
5. Submit generic deployment configuration separately to `free5gc/free5gc`
   only when required.
6. Keep laboratory configuration and experiment documentation in the local
   testbed repository.

If maintainers prefer parallel PRs, the SMF PR must explicitly identify its
OpenAPI dependency and should not vendor generated files into SMF as a
shortcut.

If a temporary handwritten Nupf client is accepted, isolate it behind the SMF
consumer interface so it can later be replaced without changing processor or
handler behavior.

## Release and Dependency Discipline

LFS114 emphasizes coordinated releases across network functions, generic
packages, and deployment repositories. For this contribution:

- record the exact `free5gc/openapi` revision used by the SMF branch;
- do not rely on an unpublished local OpenAPI modification without declaring
  it in the PR;
- document whether the work follows the current free5GC Release 17 baseline or
  introduces selected Release 19 schemas;
- test the exact dependency version that will appear in `go.mod`;
- keep OpenAPI, SMF, UPF, and deployment PR links cross-referenced.

Compatibility must be verified rather than inferred solely from semantic
version numbers.

## Public PR Communication

An upstream PR description should explain:

- the standards-based behavior being added;
- supported operations and event types;
- explicit limitations and non-goals;
- dependency on related OpenAPI or UPF work;
- unit, integration, regression, and testbed results.

It should not depend on changes to the upstream README or on copied internal
documents to explain the feature. Internal rationale belongs in local design
documents; stable behavior and limitations belong in the PR description and
tests.

## Contribution Gate

Before implementation-specific PR submission:

```text
[ ] Public ownership and PR-order discussion is resolved
[ ] 3GPP release baseline is confirmed
[ ] Specification source and generator process are confirmed
[ ] OpenAPI dependency is available or explicitly linked
[ ] Branch starts from current upstream/main
[ ] Testbed configuration and copied specifications are excluded
[ ] README changes are excluded unless maintainers request them
[ ] Unrelated URR and multi-AN changes are separated
[ ] Unit, integration, regression, and E2E evidence is recorded
[ ] Related OpenAPI, SMF, UPF, and deployment PRs cross-reference each other
```

The `ees-only` branch remains the validated prototype and behavioral reference;
it is not the branch to submit upstream.
