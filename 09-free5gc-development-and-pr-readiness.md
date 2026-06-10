# free5GC Development and PR Readiness

## Scope

This review applies the practices described in:

- the Linux Foundation LFS114 chapter on free5GC development basics;
- local SMF `main` at `df5818d`
- current local `upstream/main` at `796a2de`
- EES prototype branch at `8668372` (`ees-only`)

The objective is not only to preserve the NWDAF testbed behavior, but to shape
the implementation into a reviewable free5GC upstream contribution.

Issues already documented in the other reports are referenced rather than
repeated in full.

## Overall Assessment

The branch is suitable as an experimental implementation, but it should not be
submitted to free5GC as the current 14-commit series.

Positive results:

- all 14 commit subjects follow the Conventional Commits pattern;
- `go build ./...` passes on the branch's Go 1.25.5 baseline;
- the exact local workflow linter, golangci-lint `v2.7.2`, reports zero issues;
- outbound UPF requests use the incoming request context;
- the in-memory store protects its map with an `RWMutex`;
- the new EES code does not introduce unmanaged goroutines.

Upstream PR blockers:

- the branch must first be rebuilt or rebased onto current upstream;
- the rebased tree has a known test compilation conflict;
- approximately one thousand lines of EES runtime code have no direct tests;
- the diff mixes product code, copied specifications, design history, and
  testbed configuration;
- several concurrency and error-boundary behaviors need hardening;
- current upstream CI uses newer Go and linter versions than the branch.

## CI Baselines

### Branch Baseline

The local branch workflow uses:

```text
Go 1.25.5
golangci-lint 2.7.2
go build -v ./...
go test -v ./...
```

Observed results:

| Check | Result |
| --- | --- |
| `go build ./...` | Pass |
| golangci-lint `v2.7.2` | Pass, zero issues |
| Conventional Commit subjects | All pass the configured pattern |
| EES-focused tests | None exist |
| `go test ./...` | Not green in this local environment |

The complete test command fails in existing PFCP handler and UDP tests. The
same PFCP handler failure is reproducible from local `main`, so it is not
evidence of an EES regression. It does mean the branch cannot honestly be
reported as having a fully green local equivalent of the GitHub test job.

Do not modify unrelated PFCP tests inside the EES PR merely to hide this
baseline issue. Re-run the suite after porting to current upstream, where many
test and logging changes already exist.

### Current Upstream Baseline

Current `upstream/main` uses:

```text
Go 1.26.2
golangci-lint 2.11.4
actions/checkout v6
actions/setup-go v6
```

Therefore, passing the historical branch workflow is necessary but not
sufficient. The final integration branch must run the current upstream
versions.

The isolated upstream merge already identified one compile failure in the
the prototype branch's multi-AN test because upstream changed
`NewUserPlaneInformation` to return both a value and an error. See the branch
conflict report for the exact failure.

## High: No free5GC-Style Isolated Test Layer for EES

The absence of EES tests is already recorded in the testing report. The LFS114
workflow provides a more concrete implementation direction:

- `httptest` for inbound Nsmf HTTP requests and responses;
- `gock` or a local fake server for outbound Nupf HTTP behavior;
- `gomock` for application, resolver, repository, and consumer interfaces;
- `testify/require` for response and state assertions.

The SMF repository already depends on `gock` and `testify`, and existing
processor tests use `httptest`. The missing part is not test tooling; it is
testable EES boundaries.

### Recommended Test Shape

Create interfaces before adding tests:

```text
EventExposureRepository
EventExposureResolver
NupfEventExposureConsumer
```

Then split tests by responsibility:

1. Handler tests parse requests, invoke a mocked processor, and verify HTTP
   status, ProblemDetails, headers, and body.
2. Processor tests use mocked resolver, repository, and Nupf consumer
   dependencies to verify lifecycle decisions.
3. Consumer tests use `gock` or `httptest.Server` to verify exact Nupf method,
   path, body, timeout, retry, Location, and upstream error handling.
4. Repository tests verify duplicate, concurrent, and delete semantics.

Because `testpackage` is enabled, prefer external test packages where
practical. Test behavior through exported interfaces rather than making
production internals public solely for tests.

## High: Current PR Scope Is Too Broad

The final diff from local main contains:

```text
17 files
11,491 insertions
69 deletions
```

The three copied specification files account for 9,652 added lines. The branch
also includes:

- historical design and patch documents;
- a root testbed `smfcfg.yaml`;
- runtime implementation;
- topology selection changes;
- usage-reporting changes;
- a new repository README.

This scope makes it difficult for upstream reviewers to isolate the behavior
being proposed and to distinguish standard implementation from local testbed
material.

### Recommended Direction

Do not submit the existing branch history unchanged.

Create a clean branch from current upstream and separate contributions:

1. free5GC OpenAPI change, if TS 29.508/29.564 generated types and clients are
   required.
2. SMF EES server, processor, repository, and tests.
3. Nupf consumer integration and tests.
4. Small SMF configuration example or documentation accepted by the
   maintainers.

Keep deployment-specific NWDAF, UPF, addresses, UUIDs, and topology in the
testbed repository rather than the generic SMF default configuration.

Imported 3GPP specifications should not be added to SMF merely as reference
copies if the authoritative generation workflow belongs in
`github.com/free5gc/openapi`.

## High: Immediate Retries Amplify Failure Under Concurrency

Each inbound Nsmf Create runs synchronously in its HTTP handler goroutine. On a
transport error or UPF 5xx response, the Nupf client retries immediately with
no:

- delay;
- exponential backoff;
- jitter;
- per-UPF concurrency limit;
- circuit breaker;
- retry budget shared across requests.

This is acceptable in a small testbed, but it behaves poorly at control-plane
scale. During a UPF outage, every concurrent NWDAF request can hold an SMF
handler goroutine and issue the configured number of immediate retries,
increasing load on an already failing UPF.

### Recommended Direction

- keep retries bounded and context-aware;
- add exponential backoff with jitter;
- do not retry non-idempotent Create automatically unless an idempotency or
  reconciliation strategy prevents duplicate subscriptions;
- add per-target concurrency or failure controls if retries remain enabled;
- return promptly when the request context is cancelled;
- expose retry count and final failure through metrics.

For the initial capability, the safer generic default is zero automatic
Create retries.

## Medium: UPF Response Bodies Are Unbounded and Re-exposed

The Nupf client calls `io.ReadAll` for Create and Delete responses. On failure,
the complete response body is:

- appended to an SMF-generated ProblemDetails detail;
- returned to the Nsmf consumer;
- logged by the handler;
- stored in the `upf_body` log field during Delete.

The variable named `bodySummary` is not actually summarized or length-limited.

This violates the training chapter's goal of contextual but non-noisy boundary
logging. A malformed or hostile UPF can return a large response, consume
memory, inflate logs, and expose internal upstream details to NWDAF.

### Recommended Direction

- limit response reads with `io.LimitReader`;
- parse `application/problem+json` into typed ProblemDetails when possible;
- preserve status, cause, and a sanitized bounded detail;
- do not return arbitrary raw UPF bodies to the Nsmf consumer;
- truncate log detail and record whether truncation occurred;
- avoid logging notification URIs, authorization data, or large payloads.

The consumer layer should return a typed error. The Nsmf boundary should make
the final ProblemDetails and logging decision.

## Medium: Handwritten Telecom Initialisms Are Inconsistent

The training material recommends consistent telecom terminology. Existing SMF
handwritten code commonly uses names such as:

```text
SelectedUPF
PDUAddress
UEIPPool
APIClient
NFManagement
```

New handwritten EES types use:

```text
UpfEventExposureTarget
UeIpAddress
SelectedUpf
UpfApiRoot
SubId
NotifId
```

JSON field names must remain exactly aligned with 3GPP, but Go identifiers do
not need to copy JSON capitalization.

### Recommended Direction

For handwritten domain code, prefer consistent identifiers such as:

```text
UPFEventExposureTarget
UEIPAddress
SelectedUPF
UPFAPIRoot
SubscriptionID
NotificationID
```

Keep wire tags unchanged:

```go
UEIPAddress string `json:"ueIpAddress"`
```

Do not rename generated OpenAPI operation or model identifiers inside generated
files. Translate naming only at the generated-to-domain boundary.

## Medium: Error Types Do Not Preserve the Failure Chain

The training chapter recommends wrapping lower-level errors and logging once at
the appropriate boundary.

The Nupf helper does avoid logging internally, which is correct. However, it
converts transport, body-read, and upstream response failures directly into
HTTP ProblemDetails. This loses a normal Go error chain and couples the
transport layer to the Nsmf response policy.

This architectural issue is also described in the generalization report.

### Recommended Direction

Return a typed error carrying:

```text
operation
target service
upstream status
retry count
bounded upstream ProblemDetails
wrapped transport or decode error
```

Use `%w` for underlying Go errors so `errors.Is` and `errors.As` remain useful.
The processor decides rollback/lifecycle behavior, and the Nsmf handler maps
the final error to TS 29.571 ProblemDetails and logs it once.

## Lint Generated-Code Clarification

`api_eventexposure.go` still claims it was generated by OpenAPI Generator even
though the prototype branch adds roughly 599 lines of handwritten logic.

The current branch linter uses `generated: lax`. In golangci-lint `v2.7.2`,
the comment:

```text
Generated by: OpenAPI Generator
```

does not match the configured lax generated-file markers by itself. The exact
lint run reports zero issues, so the current handwritten handler is not known
to be escaping lint through this header.

The ownership and regeneration problem remains valid: a future generator run
can still overwrite the handwritten implementation. That issue is documented
in the source architecture report.

## Commit History Assessment

All current commit subjects satisfy the repository's Conventional Commits
workflow. The problem is commit content and review sequence, not subject
syntax.

Several commits represent temporary development stages that are later replaced
or deleted. For example, detailed patch documents are introduced and later
removed in favor of consolidated documentation.

For an upstream PR, create a clean, bisectable series where every commit:

- builds;
- has its relevant tests;
- does not depend on a later cleanup commit;
- contains one coherent architectural change;
- excludes testbed-only configuration.

Do not squash everything into one large commit. Prefer a small sequence such
as:

1. `feat(smf): add event exposure repository and resolver`
2. `feat(smf): add Nupf event exposure consumer`
3. `feat(smf): implement Nsmf event exposure subscriptions`
4. `test(smf): cover event exposure subscription lifecycle`
5. `docs(smf): document supported event exposure capability`

The exact split depends on whether generated OpenAPI support is merged first.

## Upstream PR Gate

Before opening the free5GC PR:

```text
[ ] Base the branch on current upstream/main
[ ] Resolve the NewUserPlaneInformation test signature change
[ ] Remove or isolate the unsafe multi-AN behavior
[ ] Decide the free5gc/openapi dependency and PR order
[ ] Remove testbed-only configuration from the SMF PR
[ ] Reduce copied specification and historical design material
[ ] Add handler, processor, consumer, and repository tests
[ ] Add concurrency and cancellation tests
[ ] Add missing-UE, UE-release, and UPF-relocation tests or explicit initial-policy tests
[ ] Use the selected OpenAPI `IpAddr` wire representation
[ ] Define supportedFeatures negotiation and UPF success-body handling
[ ] Preserve typed upstream ProblemDetails and retry metadata
[ ] Validate UPF Location origin and subscription path
[ ] Integrate outbound Nupf OAuth and TLS trust configuration
[ ] Declare or implement multi-instance subscription ownership
[ ] Define graceful shutdown and in-flight EES operation policy
[ ] Bound and sanitize upstream response bodies
[ ] Align inbound and outbound EES log categories with SMF conventions
[ ] Decide safe Create retry semantics
[ ] Run gofmt/gofumpt/gci through current golangci-lint
[ ] Run current upstream Go and golangci-lint versions
[ ] Run go build -v ./...
[ ] Run go test -v ./...
[ ] Run focused go test -race for EES packages
[ ] Run the NWDAF testbed E2E flow
[ ] Document testbed results separately from generic unit tests
```

## Recommended Next Step

Treat the prototype branch as the validated source of behavioral
requirements. Build a new upstream integration branch from current
`upstream/main` rather than attempting to polish the existing history in
place.

The first implementation patch on that branch should establish testable
processor, consumer, resolver, and repository interfaces before moving the
current handler logic.
