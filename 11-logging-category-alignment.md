# Logging Category Alignment

## Medium: Outbound Nupf Logs Use the Inbound SBI Category

The current EES handler logs both inbound Nsmf handling and outbound UPF
subscription results with `logger.SBILog`.

Relevant code:

- `../smf/internal/sbi/api_eventexposure.go`
- `../smf/internal/sbi/nupf_eventexposure_client.go`

Using `SBILog` for Nsmf request handling is reasonable. The less aligned part
is logging SMF-to-UPF Create/Delete results under the same category. In
existing SMF code, outbound calls to other NFs generally use the consumer side
of the SBI stack and are logged through `ConsumerLog` or the relevant consumer
layer.

The Nupf client itself does not log, which is good because it avoids duplicate
logs. The issue is where the final outbound operation result is categorized.

### Recommended Direction

For the clean PR branch:

1. keep inbound Nsmf request/response logs under `SBILog`;
2. log outbound Nupf Create/Delete outcomes from the consumer or processor
   boundary under `ConsumerLog` or a clearly named EES consumer logger;
3. log each failure once at the boundary that maps it to ProblemDetails;
4. avoid logging full notification URIs, authorization data, or unbounded UPF
   response bodies.

This is a PR hygiene issue rather than a current testbed blocker.
