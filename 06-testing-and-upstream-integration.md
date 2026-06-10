# Testing and Upstream Integration

## Current Test Results

The following read-only test commands were run with Go build cache redirected
to `/tmp`:

```text
SMF:
  go test ./internal/context ./internal/sbi
  PASS

NWDAF:
  go test ./internal/sbi/consumer ./internal/sbi/processor
  PASS

go-upf:
  go test ./internal/ees
  package compiles, but reports [no test files]
```

No complete testbed E2E run was performed during this review.

## High: No Direct SMF EES Tests

There are no focused tests for:

- Nsmf Create validation;
- Nsmf Location;
- resolver error mapping;
- missing UE / no active session behavior;
- Nsmf-to-Nupf request mapping;
- exact `IpAddr` JSON representation;
- `supportedFeatures` negotiation;
- UPF Create response handling;
- typed UPF ProblemDetails and retry metadata;
- state rollback;
- Delete cascade;
- UPF `Location` origin validation;
- UE release cleanup policy;
- UPF relocation policy;
- replicated SMF routing behavior;
- graceful shutdown and in-flight operation behavior;
- relative and absolute UPF Location handling;
- concurrent Delete;
- restart/reconciliation behavior.

`internal/sbi` currently compiles without package tests.

## High: Multi-AN Test Covers Only the Testbed-Friendly Case

The regression test verifies two disconnected AN-UPF pairs where each UPF uses
a different S-NSSAI.

It does not test the dangerous case where both disconnected UPFs support the
same DNN and S-NSSAI.

Required additional case:

```text
gNB1 -> UPF1 -- internet / 010203
gNB2 -> UPF2 -- internet / 010203
```

The expected selection must depend on the UE's actual ingress AN, not global
candidate order.

## Medium: go-upf EES Lost Unit-Test Coverage

The latest go-upf `internal/ees` package has no test files. Important contract
behavior is therefore not protected:

- `repPeriod` parsing;
- Create validation;
- absolute Location generation;
- Delete path handling;
- notification payload fields;
- correlation ID preservation.

## Medium: Cross-Repository Contract Test Is Missing

A lightweight integration test should start:

- a fake NWDAF callback;
- an SMF Event Exposure server with an injected resolver;
- a fake or real go-upf EES server.

It should assert the exact wire payload and full lifecycle:

1. NWDAF-style Nsmf Create;
2. exact Nupf Create body;
3. returned Location;
4. UPF Notify correlation;
5. Nsmf Delete;
6. exact Nupf Delete URI.

This would have detected both the duplicated Nsmf Location and the reporting
period field mismatch.

## Medium: SMF Branch Is Far Behind upstream/main

At review time:

- local SMF: `8668372`
- local upstream/main: `796a2de`
- divergence: 107 commits behind, 14 commits ahead

Upstream changes include fixes around:

- OAuth callbacks;
- UP node initialization;
- duplicate PFCP release;
- URR handling;
- user-plane configuration;
- general error propagation.

### Recommended Direction

Before major EES expansion:

1. create a dedicated integration branch;
2. rebase or port the EES commits onto current upstream;
3. keep EES behavior changes separate from topology and URR fixes;
4. run upstream SMF tests after each logical port;
5. run the cross-repository EES contract test;
6. run the full testbed E2E last.

## Medium: Submodule Reproducibility

The outer `5G_Infrastructure` repository currently records older NWDAF and
go-upf submodule commits than the manually pulled working trees.

This is not an error during development, but a successful test should record:

- outer testbed commit;
- NWDAF commit;
- go-upf commit;
- SMF commit;
- deployed configuration hashes.

Without these values, an E2E result cannot be reproduced reliably.

## Suggested Test Priority

1. Exact Nsmf and Nupf Location tests.
2. `repPeriod` end-to-end assertion.
3. Multi-AN same-slice disconnected topology test.
4. Multiple PDU sessions for one SUPI.
5. Missing UE and no-UE-IP resolver errors.
6. UE release and UPF relocation lifecycle policy.
7. UPF Create ambiguous retry behavior.
8. UPF Delete failure and orphan reconciliation.
9. `IpAddr`, feature negotiation, and typed UPF response contract tests.
10. Cross-origin `Location` rejection and relative Location resolution.
11. OAuth/TLS-enabled NWDAF-SMF-UPF flow.
12. Multi-instance routing and graceful-shutdown lifecycle tests.
