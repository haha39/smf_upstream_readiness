# Transport Security and Endpoint Trust

## Scope

This document separates three concerns that are easy to conflate:

- inbound authorization for consumers calling Nsmf Event Exposure;
- outbound authorization for SMF calling Nupf Event Exposure;
- trust validation for the UPF endpoint and returned resource URI.

## Current Security Boundary

The EES route is registered through the normal SMF SBI router. When OAuth is
enabled, inbound Nsmf requests pass through free5GC's existing authorization
middleware.

The unimplemented part is the outbound Nupf path. It uses a plain HTTP client
without free5GC token-context integration or an explicit UPF trust policy.

Relevant code:

- `../smf/internal/sbi/server.go`
- `../smf/internal/sbi/nupf_eventexposure_client.go`

## High: UPF `Location` Is Used Without Origin Validation

After UPF Create, SMF stores the returned `Location`. Nsmf Delete later uses an
absolute `Location` verbatim.

A faulty or malicious peer could therefore direct SMF to issue DELETE against
an unrelated host. Even in a trusted testbed, an incorrect proxy or UPF
response can cause cleanup to target the wrong service.

### Recommended Direction

Treat `Location` as untrusted protocol input:

1. parse it as a URI;
2. reject unsupported schemes and user information;
3. resolve relative values against the selected UPF API root;
4. require an absolute value to match the expected scheme, host, and port;
5. validate that the path belongs to the Nupf subscription collection.

Store the validated canonical URI rather than the raw header value.

## High: Outbound Nupf OAuth Is Not Implemented

Existing SMF consumers normally call `GetTokenCtx` for the target service and
NF type before invoking a generated client. The current Nupf helper sends no
access token.

This prevents the cascade from working when the UPF enforces service-based
authorization, even though inbound Nsmf authorization is enabled correctly.

### Recommended Direction

Obtain a token context for the Nupf Event Exposure service and UPF target, then
pass it through the generated or equivalent consumer transport. Add tests for:

- OAuth disabled;
- valid token acquisition;
- token acquisition failure;
- UPF rejection of a missing or invalid token.

## Medium: Outbound TLS and mTLS Policy Is Undefined

The client can call an HTTPS URL using Go defaults, but there is no
EES-specific policy for:

- trusted UPF certificate authorities;
- certificate name validation when static IP endpoints are used;
- client certificate authentication;
- transport reuse and connection limits;
- minimum TLS configuration expected by deployment.

The SMF SBI server's inbound HTTP/2 and TLS configuration does not solve these
outbound trust requirements.

### Recommended Direction

Use a reusable consumer transport owned by the application. Its TLS settings
should come from generic free5GC SBI configuration where possible, with
per-UPF trust configuration only when required by deployment.

Do not add insecure certificate verification bypasses for testbed convenience.
Keep laboratory certificates and endpoint values in the deployment repository.

## Medium: Security Behavior Is Not Covered by EES Tests

The existing testbed uses HTTP without OAuth, so it proves functional
interoperability but not production SBI security.

Required coverage includes:

- inbound Nsmf authorization with OAuth enabled and disabled;
- outbound Nupf token propagation;
- trusted and untrusted TLS certificates;
- rejected cross-origin `Location` values;
- relative `Location` resolution;
- absence of tokens, credentials, and notification URIs in logs.

Security tests should use local fake servers and test certificates rather than
depending on the full testbed.
