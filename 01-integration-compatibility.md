# Integration Compatibility

## Current Result

The latest local NWDAF and go-upf revisions remain compatible with the main
SMF EES flow:

1. NWDAF creates an Nsmf Event Exposure subscription.
2. SMF resolves the UE session and creates a Nupf Event Exposure subscription.
3. UPF sends notifications directly to NWDAF.
4. NWDAF deletes the Nsmf subscription.
5. SMF attempts to delete the linked UPF subscription.

The latest pulls did not change the core NWDAF-to-SMF or SMF-to-UPF API code.

## High: Reporting Period Field Mismatch

### Evidence

SMF sends the TS 29.564 field:

```json
{
  "eventReportingMode": {
    "trigger": "PERIODIC",
    "repPeriod": 5
  }
}
```

Relevant code:

- `../smf/internal/sbi/nupf_eventexposure_client.go`
- 3GPP TS 29.564 V19.5.0 OpenAPI definition reviewed from a local,
  untracked reference copy

The current go-upf request model reads:

```go
ReportPeriod int `json:"reportPeriod,omitempty"`
```

Relevant code:

- `../5G_Infrastructure/go-upf-ess/go-upf/internal/ees/api.go`

### Impact

- UPF silently ignores the standard `repPeriod`.
- UPF falls back to its configured aggregator period.
- Create still returns `201`, so the defect is hidden from SMF and NWDAF.
- A period requested by NWDAF is not guaranteed to take effect.
- The current testbed appears correct because NWDAF, SMF URR, and UPF EES are
  all configured around five-second intervals.

### Recommended Direction

Fix go-upf to consume `repPeriod` according to its OpenAPI specification.
If backward compatibility is required, go-upf may temporarily accept both
field names while emitting and documenting only `repPeriod`.

SMF should not replace the standard field with `reportPeriod` merely to match
the current testbed implementation.

## Resolved by Current go-upf: Absolute UPF Location

The current go-upf returns an absolute `Location`:

```text
http://<upf-host>:8088/nupf-ee/v1/ee-subscriptions/<id>
```

This allows the current SMF Delete client to use the URI verbatim. Earlier
go-upf behavior returned a relative URI, which exposed a path duplication bug
in SMF.

The SMF relative-URI branch remains unsafe and should still be corrected for
interoperability with other compliant UPFs.

Relevant code:

- `../smf/internal/sbi/nupf_eventexposure_client.go`
- `../5G_Infrastructure/go-upf-ess/go-upf/internal/ees/api.go`

## Compatible Notification Contract

The current UPF notification contains:

- `correlationId`
- `notificationItems`
- `eventType`
- `ueIpv4Addr`
- `timeStamp`
- `startTime`
- `userDataUsageMeasurements`
- volume and throughput measurements

NWDAF consumes the same shape and routes data using the correlation ID.

Relevant code:

- `../5G_Infrastructure/go-upf-ess/go-upf/internal/ees/notifier.go`
- `../5G_Infrastructure/NWDAF/NWDAF/internal/sbi/processor/upf_notify.go`

## Authentication Limitation

The tested environment uses HTTP without OAuth for NWDAF-to-SMF and
SMF-to-UPF communication.

SMF applies its inbound OAuth middleware when OAuth is enabled, but the current
NWDAF SMF consumer does not add a bearer token. The SMF Nupf client also does
not acquire a token for UPF requests.

This does not block the current testbed, but it prevents direct use in a
deployment where SBI OAuth is enabled.
