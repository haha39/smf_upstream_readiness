# SMF Branch Conflict Report

## Executive Result

There are two materially different comparisons:

1. Local `main` / `origin/main` versus `ees-only`
2. Current `upstream/main` versus `ees-only`

The local branches have no conflict:

- local `main`: `df5818d`
- local `origin/main`: `df5818d`
- `ees-only`: `8668372`
- `main` is the exact merge base and a direct ancestor of `ees-only`
- divergence is `0` commits on `main` and `14` commits on `ees-only`

Merging local `main` into `ees-only` is therefore a no-op.

The current upstream comparison is different:

- `upstream/main`: `796a2de`, dated 2026-05-26
- merge base: `df5818d`, dated 2026-01-04
- upstream-only commits: `107`
- ees-only commits: `14`

Git can merge or rebase these branches without textual conflict. However, the
automatically merged tree does not compile until one test is adapted, and it
contains several behavioral integration hazards that Git cannot detect.

## Verification Method

The review used read-only branch inspection plus isolated repositories under
`/tmp`. The original `smf` worktree was not changed.

The following checks were performed:

- ancestry and merge-base inspection;
- three-way file comparison from `df5818d`;
- direct `upstream/main + ees-only` merge simulation;
- complete `ees-only` rebase simulation onto `upstream/main`;
- Go 1.26.2 package compilation in the merged tree;
- manual review of all files modified by both branches;
- review of upstream-only changes that alter EES assumptions.

## Conflict Classification

| Conflict class | Result |
| --- | --- |
| Local `main` to `ees-only` textual conflicts | None |
| `upstream/main` to `ees-only` direct-merge conflicts | None |
| Rebase conflicts across all 14 ees-only commits | None |
| Confirmed compile conflicts | One |
| Shared files requiring semantic review | Four |
| Runtime/API behavior conflicts | Multiple; manual decisions required |
| Patch hygiene issues | `git diff --check` fails on imported specs and `smfcfg.yaml` |

## Direct Merge and Rebase Results

The isolated direct merge completed with:

```text
Automatic merge went well; stopped before committing as requested
```

No unmerged paths or conflict markers were produced.

The isolated rebase replayed all 14 ees-only commits onto `upstream/main`.
It also completed without stopping for a conflict. The only three-way fallback
was `pkg/factory/config.go`, which Git auto-merged.

This proves only that the changed text regions are compatible. It does not
prove that the resulting APIs, tests, topology behavior, or configuration
semantics are compatible.

## Confirmed Compile Conflict

### High: Constructor Signature Mismatch

Upstream changed:

```go
NewUserPlaneInformation(...) *UserPlaneInformation
```

to:

```go
NewUserPlaneInformation(...) (*UserPlaneInformation, error)
```

The ees-only multi-AN test still expects one return value.

Affected file:

- `smf/internal/context/user_plane_information_test.go`

The exact Go 1.26.2 failure in the automatically merged tree is:

```text
internal/context/user_plane_information_test.go:361:26:
assignment mismatch: 1 variable but
smf_context.NewUserPlaneInformation returns 2 values
```

After adapting that one call in the isolated copy and asserting the returned
error, this compile-only command passed for every package:

```text
go test ./... -run '^$'
```

No other production-package compile conflict was found.

## Files Changed by Both Branches

### 1. `internal/context/pfcp_rules.go`

Change volume from the common base:

- upstream: `+115/-9`
- ees-only: `+4/-2`

Upstream substantially refactors URR ownership and lifecycle through the
`UrrTable` / `UrrEntry` model. Ees-only only guards volume threshold creation
so the threshold flag is set when the configured threshold is greater than
zero.

Git merges these changes cleanly because they touch different regions.

Required resolution:

- keep the upstream URR table and lifecycle implementation;
- retain the ees-only positive-threshold guard;
- run PFCP establishment, modification, release, and usage-report tests;
- do not restore any pre-upstream direct `URR.State` ownership assumptions.

Risk: medium. There is no current textual or compile conflict, but both changes
affect usage reporting behavior.

### 2. `internal/context/user_plane_information.go`

Change volume from the common base:

- upstream: `+72/-19`
- ees-only: `+116/-58`

Upstream:

- returns errors from `NewUserPlaneInformation`;
- makes dynamic topology creation transactional;
- isolates newly created UPFs until validation succeeds;
- avoids fatal process exits for invalid configuration.

Ees-only:

- adds `NupfEeApiRoot` to `UPNode`;
- loads and serializes that endpoint;
- changes path-source selection for multiple AN nodes.

#### High: Dynamic Topology Loses `NupfEeApiRoot`

The merged initial constructor copies:

```text
factory.UPNode.NupfEeApiRoot -> context.UPNode.NupfEeApiRoot
```

The upstream `UpNodesFromConfiguration` candidate-building path does not copy
the new field. A UPF added through OAM or dynamic topology reload can therefore
exist and carry traffic while EES resolution fails with
`ErrNoUpfEventApiRoot`.

Required resolution:

- copy and trim `NupfEeApiRoot` in both initial and dynamic UPF creation;
- preserve it during topology serialization;
- add a dynamic add/reload test that resolves an EES target afterward.

#### High: Multi-AN Algorithm Must Not Be Ported Blindly

Ees-only gathers candidates reachable from every configured AN and merges
them. The selection input does not identify the UE's actual ingress AN.

If two disconnected AN-UPF paths support the same DNN and S-NSSAI, the merged
candidate set can select a UPF unreachable from that UE. The existing testbed
avoids the failure because different S-NSSAIs disambiguate the paths.

Required resolution:

- retain upstream behavior until ingress identity can constrain path search;
- or add an explicit AN/access-path key to session establishment and UPF
  selection;
- do not treat the current `3d3f3bd` algorithm as a general free5GC fix.

Risk: high. The code auto-merges but can silently choose the wrong UPF.

### 3. `internal/context/user_plane_information_test.go`

Change volume from the common base:

- upstream: `+8/-4`
- ees-only: `+97/-0`

This file contains the confirmed constructor compile conflict.

Required resolution:

- consume the returned `error`;
- add a test where two disconnected paths have the same DNN and S-NSSAI;
- test dynamic topology loading of `NupfEeApiRoot`;
- avoid accepting the current S-NSSAI-disambiguated test as proof of correct
  ingress selection.

Risk: high until the compile error is fixed; medium afterward.

### 4. `pkg/factory/config.go`

Change volume from the common base:

- upstream: `+3/-2`
- ees-only: `+6/-1`

Upstream:

- changes `SmfCallbackUriPrefix` to `/nsmf-callback/v1`;
- permits `nsmf-callback` in `serviceNameList`.

Ees-only:

- changes the Event Exposure prefix;
- adds Nupf NF ID, timeout, retry, and per-UPF API root settings.

Git preserves both sets of edits, but the resulting Event Exposure prefix is
not the desired final design.

Required resolution:

- keep upstream `/nsmf-callback/v1` and callback service validation;
- retain optional EES configuration fields;
- use `/nsmf-event-exposure/v1` as the service base;
- register `/subscriptions` and `/subscriptions/:subId` as route paths;
- validate timeout, retry, and endpoint values explicitly.

Neither branch's Event Exposure constant should win unchanged:

- upstream still uses `/nsmf_event-exposure/v1`, with an underscore;
- ees-only embeds `/subscriptions` in the service prefix.

Risk: high because automatic merging preserves an incorrect public URI model.

## Upstream-Only Changes That Affect EES

### NRF Registration

Upstream corrected `ApiFullVersion` and SBI scheme handling in NF service
registration. Ees-only does not modify the same file, so a merge automatically
inherits the fix.

Decision:

- upstream implementation wins;
- do not restore the older NF profile behavior;
- verify that `nsmf-event-exposure` is advertised with the current SMF scheme,
  API prefix, and version.

### SBI Server and OAuth

Upstream changed server and callback authorization behavior. Ees-only replaces
the Event Exposure handler but not `server.go`, so the route remains under the
upstream OAuth middleware after merging.

Decision:

- preserve upstream middleware and token validation;
- test EES with OAuth both enabled and disabled;
- do not register a parallel router that bypasses normal SMF authorization.

### Go and OpenAPI Dependencies

Upstream requires:

- Go `1.26.2`;
- `github.com/free5gc/openapi v1.2.4`;
- newer free5GC and utility dependencies.

The EES production code compiles with these versions after the single test
signature fix. This is not currently a blocking dependency conflict.

Decision:

- upstream `go.mod` and `go.sum` win;
- do not bring older module versions from the testbed branch;
- replace local hand-written protocol types with generated models only when
  the upstream OpenAPI package exposes the required Nupf models.

### Transactional Topology Updates

Upstream now validates candidate UP nodes before committing them to live state.
The EES API root must participate in this path; otherwise initial startup and
dynamic reload have different behavior.

Decision:

- treat `NupfEeApiRoot` as part of UPF configuration state, not as a testbed
  side field;
- cover initial load, dynamic add, serialization, and reload.

## Non-Overlapping EES Files

The following files are additions from ees-only and do not have path-level
conflicts with upstream:

- `internal/context/nsmf_event_exposure_store.go`
- `internal/context/nupf_event_exposure_resolver.go`
- `internal/sbi/nupf_eventexposure_client.go`
- EES documentation and imported specification files
- root `smfcfg.yaml`

They still require semantic review.

### Resolver

The resolver compiles against upstream but selects a context by SUPI only and
rejects multiple sessions. It also reads mutable session fields without an
explicit session lifecycle lock.

Porting requirement:

- define session selection keys beyond SUPI;
- verify the session is active;
- read UE address and selected UPF under the appropriate context lock;
- keep UPF selection tied to the actual session, not a fresh topology search.

### Nupf Client

The direct HTTP helper compiles, but it bypasses normal free5GC consumer
abstractions, generated clients, OAuth token acquisition, and NRF service
discovery.

Porting requirement:

- place Nupf communication behind a consumer interface;
- inject a reusable HTTP/generated client;
- keep HTTP-to-ProblemDetails mapping at the Nsmf boundary;
- support discovered `ApiPrefix` where available;
- keep static `nupfEeApiRoot` only as an optional deployment override.

### In-Memory Store

The store is independent of upstream and auto-merges. Its restart, orphan, and
concurrent lifecycle limitations remain unchanged.

Porting requirement:

- add atomic lifecycle states or a single-operation ownership rule;
- define restart reconciliation before claiming durable support;
- add race tests for simultaneous Create/Delete and repeated Delete.

## Existing EES Behavior Defects Preserved by Auto-Merge

These are not Git conflicts, but a merge would carry them into current
upstream:

1. The Nsmf `Location` builder duplicates `/subscriptions`.
2. Collection routing depends on a prefix that already contains the collection.
3. The current go-upf integration uses a non-standard reporting-period field
   name on the receiving side.
4. Multiple sessions for one SUPI cannot be selected.
5. Dynamic topology updates omit `NupfEeApiRoot`.
6. Multi-AN selection is not bound to ingress AN.
7. UPF HTTP calls use deployment-specific direct endpoints and no OAuth.
8. Local subscription state cannot be reconciled after restart.

These findings are detailed in the other files in this directory.

## Patch Hygiene Conflict

`git diff --check upstream/main...ees-only` fails because:

- imported YAML specification files use CRLF/trailing whitespace;
- `smfcfg.yaml` contains several trailing spaces.

This does not prevent Git merge or Go compilation, but it can fail repository
quality gates and creates noisy future diffs.

Required resolution:

- decide whether imported 3GPP specifications are immutable vendor artifacts;
- if immutable, exclude them explicitly from whitespace checks;
- otherwise normalize them in a dedicated documentation-only commit;
- remove trailing whitespace from maintained configuration files.

## Commit-Level Porting Recommendation

Do not merge ees-only wholesale into a production branch merely because Git
reports no conflict. Create an integration branch from current
`upstream/main` and port behavior intentionally.

| Ees-only commit area | Recommendation |
| --- | --- |
| Imported specs and contracts | Port, then decide and document line-ending policy |
| Implementation plans and guides | Port after correcting paths and limitations |
| Initial Nsmf Create/Delete | Manually port with conventional route structure |
| Nupf Create cascade | Manually port behind a consumer interface |
| Nupf Delete cascade | Manually port with explicit orphan/retry policy |
| Interface stubs | Port only after route/API inventory is confirmed |
| Positive volume-threshold guard | Port onto upstream URR implementation |
| Event Exposure registration-path fix | Do not cherry-pick as-is |
| Testbed `smfcfg.yaml` | Keep as an example, not a generic upstream default |
| Multi-AN selection commit | Do not port as-is |
| README and operational docs | Port after implementation decisions are final |

## Recommended Integration Sequence

1. Branch from `796a2de` or the then-current upstream commit.
2. Add cleaned contracts/spec references without runtime changes.
3. Add EES configuration fields while preserving upstream callback and
   validation changes.
4. Add `NupfEeApiRoot` to initial and transactional dynamic topology paths.
5. Add session-bound resolver and in-memory store with focused unit tests.
6. Add a Nupf consumer interface and transport tests.
7. Add Nsmf routes under `/nsmf-event-exposure/v1/subscriptions`.
8. Add Create mapping and rollback tests.
9. Add Delete cascade and idempotency/orphan tests.
10. Reapply the positive volume-threshold guard to upstream URR code.
11. Leave multi-AN behavior unchanged until ingress-aware selection exists.
12. Run compile, lint, race, API, and testbed acceptance gates.

## Acceptance Gates

The integration should not be considered complete until all of these pass:

- `go test ./... -run '^$'`
- focused `internal/context` topology and resolver tests;
- focused Nsmf Create/Delete handler tests;
- Nupf request serialization tests using the outer `subscription` wrapper;
- exact Nsmf route and `Location` assertions;
- static startup and dynamic topology reload tests;
- one UPF/two UE subscription isolation;
- two UPF/two UE correct-UPF routing;
- one UE with I-UPF plus anchor-UPF behavior;
- multiple PDU sessions for one SUPI;
- OAuth-enabled NRF registration and outbound Nupf call behavior;
- concurrent Delete and restart/orphan scenarios;
- `go test -race` for the EES store and handler tests;
- `golangci-lint` with the repository configuration;
- an end-to-end NWDAF -> SMF -> UPF -> NWDAF run.

## Final Decision

The branch has no conventional Git conflict with either local `main` or the
current `upstream/main`. It does have one confirmed compile conflict and
multiple high-impact semantic conflicts.

The safest integration strategy is a controlled port onto current upstream,
not a blind merge. A rebase is mechanically possible, but it must still be
followed by the same manual corrections and acceptance tests.
