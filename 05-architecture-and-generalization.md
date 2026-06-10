# Architecture and Generalization

## Medium: Nupf Client Does Not Follow Existing free5GC Consumer Design

The Nupf implementation is a set of package-level helper functions in the SBI
server package.

Relevant code:

- `../smf/internal/sbi/nupf_eventexposure_client.go`

It directly:

- reads global configuration;
- creates a new `http.Client` per operation;
- concatenates URL strings;
- returns handler-oriented ProblemDetails;
- has no interface for testing;
- has no OAuth token integration;
- has no NRF/service discovery integration.

Other free5GC consumers generally use:

- a consumer service object;
- reusable generated OpenAPI clients;
- `GetTokenCtx`;
- NF service profile `ApiPrefix`;
- typed request and error models.

### Recommended Direction

Move Nupf communication behind a consumer interface owned by the SMF consumer
layer. Inject it into the Event Exposure processor or handler.

The transport layer should return typed upstream results. HTTP ProblemDetails
mapping should remain at the Nsmf boundary.

## Medium: Handler Contains Too Much Business Logic

`HTTPCreateIndividualSubcription` currently performs:

- JSON parsing;
- validation;
- session resolution;
- defaulting;
- Nsmf-to-Nupf mapping;
- HTTP client invocation;
- state persistence;
- response mapping;
- logging.

This makes focused testing difficult and couples protocol handling to session
and UPF lifecycle policy.

### Recommended Direction

Separate:

1. Nsmf HTTP handler;
2. contract validator;
3. subscription application service;
4. session/UPF resolver;
5. Nupf consumer;
6. subscription repository.

## Medium: Local Protocol Types Duplicate OpenAPI Models

Custom structs are used because the current `free5gc/openapi v1.2.3` generated
Nsmf models do not contain the reviewed `UPF_EVENT` extension fields, and the
module does not provide a generated Nupf Event Exposure client.

This is understandable, but local types should be isolated in a dedicated
protocol package and generated from the authoritative specification where
possible. Keeping them inside the handler/client files makes schema drift more
likely.

The `repPeriod` versus `reportPeriod` mismatch demonstrates this drift risk.

## Medium: Static Endpoint Configuration Is the Only Nupf Discovery Method

`nupfEeApiRoot` is attached directly to each configured UPF topology node.

This is useful for static deployments, but a general implementation should
allow a resolver strategy such as:

1. explicit per-UPF configuration;
2. discovered service endpoint;
3. endpoint learned from UPF association metadata.

## Medium: Fixed Test NF ID in Example Configuration

The example config sets:

```yaml
nupfEeNfId: "550e8400-e29b-41d4-a716-446655440000"
```

The implementation already falls back to the actual SMF NF instance ID when
this setting is absent. A general example should prefer that fallback and use
the fixed value only in testbed-specific configuration.

Relevant code:

- `../smf/smfcfg.yaml`
- `../smf/internal/sbi/nupf_eventexposure_client.go`

## Medium: Service Registration Metadata Is Not Service-Specific

The SMF NF profile constructs one shared service-version structure using the
PDU Session URI and assigns it to every registered SMF service.

Relevant code:

- `../smf/internal/context/nf_profile.go`

This means Nsmf Event Exposure discovery metadata can advertise PDU Session
version details. The EES route works with statically configured NWDAF endpoints,
but discovery-oriented consumers may receive misleading metadata.

## Low: Configuration File Mixes Product Example and Testbed Topology

`../smf/smfcfg.yaml` contains:

- fixed laboratory addresses;
- two specific slices;
- two test UPFs;
- a fixed EES NF ID;
- project-specific URR values.

For public use, separate:

- a minimal generic example;
- a dedicated testbed configuration;
- documentation for optional EES fields.

## Reasonable General-Purpose Choices

The following design decisions are not considered problematic customization:

- direct UPF-to-NWDAF notification;
- `notifId` to `notifyCorrelationId` mapping;
- `bundledEventNotifyUri` to `eventNotifyUri` mapping;
- per-UPF EES endpoints as one supported resolver source;
- storing linkage only after successful UPF Create;
- mutex protection for local subscription state;
- use of TS 29.571 ProblemDetails at the Nsmf boundary;
- enabling volume threshold only when its configured value is greater than zero.
