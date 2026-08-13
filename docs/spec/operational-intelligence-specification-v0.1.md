# Operational Intelligence Specification

**Version:** 0.1.0-draft  
**Status:** Proof of Concept  
**Canonical profile filename:** `op-intel.yaml`

## 1. Purpose

This specification defines a narrow proof-of-concept protocol for exposing negative-path operational intelligence from an API.

The proof of concept is intended to demonstrate that an API can:

1. load and validate a local operational intelligence profile;
2. collect bounded OpenTelemetry evidence;
3. assess configured exception and HTTP 5xx activity;
4. expose the assessment through a machine-readable API; and
5. distinguish no findings, findings, partial assessment, and inability to assess.

This version is intentionally limited. It is the starting point for a broader Operational Intelligence specification, but broader capabilities are outside the scope of this document.

## 2. Normative Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as normative requirements.

## 3. Scope

### 3.1 In Scope

Version 0.1 includes:

- `GET /intel`
- `POST /intel`
- optional `QUERY /intel` compatibility
- local configuration through `op-intel.yaml`
- configuration validation
- relative and absolute assessment time ranges
- OpenTelemetry as the telemetry model
- a bounded in-process OpenTelemetry collector
- exception assessment
- HTTP 5xx assessment
- JSON responses
- machine-readable errors
- telemetry for all `/intel` interactions
- exclusion of `/intel` traffic from assessed application failures by default

### 3.2 Out of Scope

Version 0.1 does not define:

- remediation
- autonomous action
- issue creation
- ticket creation
- alerting
- notification delivery
- orchestration
- deployment control
- production authentication or authorization
- persistent evidence storage
- cross-service correlation
- distributed topology analysis
- positive-path assessment
- business KPI assessment
- anomaly detection
- provider-specific backends such as Datadog or Azure Monitor
- dashboards other than existing Aspire capabilities
- a general-purpose telemetry query language

An implementation MUST NOT claim conformance to this version based on capabilities that are outside this scope.

## 4. Design Principles

A conforming implementation SHALL follow these principles:

1. **Evidence over assertion.** Findings MUST be supported by collected evidence.
2. **Assessment before action.** This protocol reports operational intelligence and does not remediate.
3. **Vendor-neutral protocol.** The external protocol MUST NOT expose provider-specific telemetry query syntax.
4. **Bounded proof of concept.** The reference collector MUST use bounded memory and bounded retention.
5. **Configuration-driven behavior.** Default assessment behavior MUST be defined by `op-intel.yaml`.
6. **Explicit uncertainty.** Missing, incomplete, or unavailable evidence MUST be represented in the response.
7. **No recursive assessment.** `/intel` activity MUST be excluded from application failure findings by default.

## 5. Conformance

### 5.1 Conforming Implementation

A conforming implementation MUST:

- expose the required protocol endpoints;
- load the canonical profile filename;
- validate the profile before serving assessments;
- implement all mandatory profile sections;
- support the required assessment categories;
- return responses conforming to this specification;
- return machine-readable errors;
- use ISO 8601 date-time values;
- collect evidence without modifying the source telemetry;
- emit telemetry for `/intel` requests; and
- exclude `/intel` requests from negative-path assessment by default.

### 5.2 Conforming Profile

A conforming profile MUST:

- be named exactly `op-intel.yaml`;
- include all mandatory top-level sections;
- pass schema and semantic validation; and
- define a bounded default time range.

### 5.3 Conforming Protocol

A conforming protocol implementation MUST support:

```http
GET /intel
POST /intel
```

An implementation MAY additionally support `QUERY /intel`. `POST /intel` MUST remain supported for the lifetime of version 0.1 even when `QUERY /intel` is available. An implementation MUST NOT require clients to use `QUERY`.

An implementation MAY mount the resource beneath an API-wide prefix, such as:

```http
GET /v1/intel
POST /v1/intel
```

An implementation that supports `QUERY` MAY also expose `QUERY /v1/intel` at the same mounted resource.

The final path segment MUST be `intel`.

## 6. Reference Architecture

```text
API
 ├── GET /intel
 ├── POST /intel
 ├── QUERY /intel (optional compatibility method)
 │
 └── Assessment Coordinator
       ├── Profile Loader and Validator
       │     └── op-intel.yaml
       │
       ├── Operational Evidence Collector
       │     └── Bounded OpenTelemetry Collector
       │
       ├── Negative-Path Analyzer
       │     ├── Exception Analyzer
       │     └── HTTP 5xx Analyzer
       │
       └── Response Builder
```

### 6.1 Assessment Coordinator

The Assessment Coordinator controls the assessment flow.

It MUST:

1. resolve the validated profile;
2. determine the effective time range;
3. construct an internal collection request;
4. invoke the Operational Evidence Collector;
5. pass collected evidence to the configured analyzers;
6. determine the assessment status;
7. construct the protocol response; and
8. record request and response correlation telemetry.

The Assessment Coordinator MUST NOT query provider-specific telemetry systems directly.

### 6.2 Profile Loader and Validator

The Profile Loader and Validator MUST:

- locate `op-intel.yaml`;
- parse YAML;
- validate required sections and fields;
- validate supported values;
- validate time range constraints;
- reject unsupported assessment categories;
- expose a validated in-memory profile to the Assessment Coordinator; and
- prevent assessment execution when the profile is invalid.

### 6.3 Operational Evidence Collector

The Operational Evidence Collector retrieves evidence matching a bounded collection request.

It MUST:

- accept a service identity;
- accept an absolute start and end time;
- accept requested evidence categories;
- return normalized evidence records;
- report evidence availability separately from the evidence array;
- avoid assessing operational meaning;
- avoid producing findings;
- avoid coordinating the complete assessment workflow;
- avoid modifying application state; and
- avoid exposing provider-specific query syntax through `/intel`.

### 6.4 Negative-Path Analyzer

The Negative-Path Analyzer evaluates normalized evidence for the categories enabled in the profile.

Version 0.1 MUST support:

- `exceptions`
- `http_5xx`

The analyzer MUST NOT perform remediation.

### 6.5 Response Builder

The Response Builder MUST produce a JSON object conforming to Section 11.

## 7. Canonical Profile

### 7.1 Filename

The profile filename MUST be:

```text
op-intel.yaml
```

A conforming implementation MUST NOT require a different filename.

An implementation MAY allow an explicit file path to be supplied through application configuration, but the referenced file MUST still be named `op-intel.yaml`.

### 7.2 Mandatory Top-Level Sections

The profile MUST contain these top-level sections:

```yaml
specVersion:
service:
telemetry:
assessment:
query:
output:
```

Unknown top-level sections SHOULD cause validation failure in version 0.1.

### 7.3 Minimal Valid Profile

```yaml
specVersion: "0.1"

service:
  name: "sample-api"

telemetry:
  source: "opentelemetry"
  collector:
    type: "bounded-in-process"
    retention: "PT30M"
    maxRecords: 10000
  excludeRoutes:
    - "/intel"

assessment:
  defaultTimeRange: "PT15M"
  categories:
    exceptions:
      enabled: true
    http_5xx:
      enabled: true

query:
  allowTimeRangeOverride: true
  maximumTimeRange: "PT30M"

output:
  includeEvidence: true
  maximumEvidencePerFinding: 25
```

### 7.4 Excluded Route Matching

Each `telemetry.excludeRoutes` entry MUST be a case-sensitive path pattern. A pattern without `*` is an exact pattern. A pattern containing `*` is a wildcard pattern.

For each request, the implementation MUST select one candidate path for exclusion matching:

1. the normalized OpenTelemetry route template, when available; otherwise
2. the raw request path with its query string and fragment removed.

The candidate path MUST NOT include the scheme, host, port, query string, or fragment. When a normalized route template is available, the implementation MUST use it and MUST NOT also compare the raw request path.

Matching MUST evaluate the complete candidate path. The `*` wildcard MUST match zero or more characters, including `/`. No implicit wildcard is added before or after a configured pattern. Version 0.1 defines no other wildcard metacharacters or escaping rules.

Examples:

| Pattern | Candidate path | Result |
|---|---|---|
| `/intel` | `/intel` | match |
| `/intel` | `/v1/intel` | no match |
| `*/intel` | `/v1/intel` | match |
| `*/intel/*` | `/v1/intel/history/details` | match |
| `*/intel` | `/v1/intel/status` | no match |
| `/orders/{id}` | `/orders/{id}` | match |
| `/orders/{id}` | `/orders/123` | no match when the normalized route template is available |
| `/orders/123` | `/orders/123` | match after the raw request target `/orders/123?verbose=true` is normalized |

A request whose candidate path matches any configured pattern MUST be excluded from negative-path assessment.

If neither a normalized route template nor a raw request path is available, configured path matching cannot be performed. The implementation MUST nevertheless prevent every `/intel` endpoint from creating recursive findings by using an equivalent safeguard, such as an internal request marker or instrumentation-scope exclusion.

## 8. Profile Validation

### 8.1 Validation Timing

The implementation MUST validate `op-intel.yaml` during application startup.

The implementation MUST NOT begin serving successful `/intel` assessments until validation succeeds.

The implementation MAY fail application startup when the profile is invalid.

If the application remains running with an invalid profile, `/intel` MUST return a machine-readable configuration error and MUST NOT perform an assessment.

### 8.2 Required Validation Rules

The validator MUST confirm:

- `specVersion` is present and equals `"0.1"`;
- `service.name` is present and non-empty;
- `telemetry.source` equals `"opentelemetry"`;
- `telemetry.collector.type` equals `"bounded-in-process"`;
- `telemetry.collector.retention` is a valid positive ISO 8601 duration;
- `telemetry.collector.maxRecords` is a positive integer;
- `assessment.defaultTimeRange` is a valid positive ISO 8601 duration;
- at least one supported assessment category is enabled;
- only `exceptions` and `http_5xx` are configured as categories;
- `query.allowTimeRangeOverride` is a boolean;
- `query.maximumTimeRange` is a valid positive ISO 8601 duration;
- `assessment.defaultTimeRange` does not exceed `query.maximumTimeRange`;
- `query.maximumTimeRange` does not exceed collector retention;
- `output.includeEvidence` is a boolean;
- `output.maximumEvidencePerFinding` is a positive integer; and
- at least one `telemetry.excludeRoutes` pattern excludes every configured `/intel` endpoint, unless the implementation explicitly provides an equivalent recursive-assessment safeguard.

### 8.3 Invalid Configuration

Invalid configuration MUST produce a Problem Details response using `application/problem+json`.

Recommended status code:

```http
503 Service Unavailable
```

Example:

```json
{
  "type": "https://example.org/problems/op-intel-invalid-profile",
  "title": "Operational intelligence profile is invalid",
  "status": 503,
  "detail": "The op-intel.yaml profile failed validation.",
  "instance": "/intel",
  "errors": [
    {
      "path": "assessment.defaultTimeRange",
      "code": "time_range_exceeds_maximum",
      "message": "PT45M exceeds query.maximumTimeRange PT30M."
    }
  ]
}
```

The `errors` extension MUST contain one entry for each detected validation error.

## 9. Bounded OpenTelemetry Collector

### 9.1 Purpose

The Bounded OpenTelemetry Collector is the required reference collector for the version 0.1 proof of concept.

It exists to make recent OpenTelemetry evidence queryable by the `/intel` assessment flow without introducing a persistent telemetry backend.

It is not an authoritative telemetry store.

### 9.2 Collection Behavior

The collector MUST:

- observe completed spans;
- observe recorded exception events;
- preserve the application's normal OpenTelemetry export pipeline;
- retain a normalized in-memory projection;
- enforce configured retention;
- enforce configured maximum record count;
- evict expired records;
- evict records when the maximum record count is exceeded;
- support concurrent writes and reads safely;
- return only records inside the requested time range;
- return only records matching requested categories; and
- exclude `/intel` traffic from assessed evidence by default.

The collector MUST NOT:

- block normal telemetry export;
- mutate exported telemetry;
- persist evidence to disk;
- retain evidence indefinitely;
- determine findings;
- determine assessment status; or
- perform remediation.

### 9.2.1 Exception Events and Span Error Status

An OpenTelemetry exception event and a span status of `ERROR` are independent signals. A conforming implementation MUST determine whether to record each signal separately.

When an exception associated with an active span causes the instrumented operation to fail, the implementation MUST, before the span ends:

- record the exception as an OpenTelemetry exception event on the active span; and
- set the active span status to `ERROR`.

This requirement includes an exception caught by application or framework error handling when the resulting HTTP response has a status code between `500` and `599`, inclusive, and an exception that escapes the instrumented operation.

When an exception is caught and the instrumented operation successfully recovers, the implementation MUST NOT set the span status to `ERROR` solely because the exception occurred. The implementation MAY record the recovered exception event when permitted by application telemetry policy.

When an operation fails without an exception, the implementation MUST set the span status to `ERROR` and MUST NOT fabricate an exception event.

Recording an exception or error status MUST NOT expose additional error details to the client.

### 9.2.2 Retention and Capacity Eviction

The collector MUST evaluate retention using each normalized evidence record's `timestamp`.

For an evaluation instant `now`, the retention cutoff MUST be:

```text
now - telemetry.collector.retention
```

A record whose `timestamp` is earlier than the retention cutoff MUST be treated as expired and MUST NOT be returned by a collection request. A record whose `timestamp` equals the retention cutoff is not expired.

Before enforcing `telemetry.collector.maxRecords`, the collector MUST remove all expired records. If the remaining record count exceeds `telemetry.collector.maxRecords`, the collector MUST evict records in oldest-first order until the configured bound is satisfied.

Oldest-first order MUST be determined by ascending evidence `timestamp`. When two or more records have the same `timestamp`, the collector MUST use their insertion order as the tie-breaker and evict the record inserted earliest first.

The collector MUST apply this ordering independently of the collector's internal storage iteration order.

### 9.2.3 Consistent Snapshot Reads

Each collection request MUST evaluate a logically consistent snapshot of the collector at one read instant.

The snapshot MUST include every eligible record committed before that read instant and MUST exclude every record committed after it. Records written concurrently with the read MUST appear either wholly in that snapshot or wholly in a subsequent snapshot; a collection result MUST NOT contain a partially written record or otherwise reflect a partially applied write.

Retention evaluation, requested time-range filtering, category filtering, and result construction for one collection request MUST operate against the same logical snapshot.

These requirements apply to both single-threaded and multi-threaded runtimes. They define observable behavior and MUST NOT be interpreted as requiring a particular locking, synchronization, or data-structure implementation.

### 9.3 Internal Collection Request

The Assessment Coordinator MUST provide an internal request containing:

```text
serviceName
startTime
endTime
categories
```

All date-time values MUST be ISO 8601 instants in UTC when serialized or logged.

Conceptual model:

```json
{
  "serviceName": "sample-api",
  "startTime": "2026-07-30T00:00:00Z",
  "endTime": "2026-07-30T00:15:00Z",
  "categories": [
    "exceptions",
    "http_5xx"
  ]
}
```

This model is internal and is not part of the public `/intel` contract.

### 9.4 Collection Result

The collector MUST return:

```text
availability
evidence[]
warnings[]
errors[]
```

`availability` MUST be one of:

```text
available
partial
unavailable
```

An empty evidence array MUST NOT be used to represent telemetry unavailability.

Conceptual model:

```json
{
  "availability": "available",
  "evidence": [],
  "warnings": [],
  "errors": []
}
```

### 9.5 Normalized Evidence Record

Each evidence record MUST contain:

```text
timestamp
serviceName
traceId
spanId
operationName
spanKind
category
```

Each record MAY contain:

```text
parentSpanId
httpMethod
httpRoute
httpStatusCode
spanStatus
exceptionType
exceptionMessage
```

`category` MUST be one of:

```text
exception
http_5xx
```

Each normalized evidence record MUST represent exactly one observed occurrence. A normalized evidence record MUST NOT aggregate multiple occurrences.

The normalized record MUST NOT contain by default:

- request bodies;
- response bodies;
- authorization headers;
- cookies;
- secrets;
- access tokens;
- arbitrary headers; or
- stack traces.

## 10. Assessment Semantics

### 10.1 Effective Time Range

For `GET /intel`, the effective time range MUST be:

```text
generatedAt - assessment.defaultTimeRange
through
generatedAt
```

For `POST /intel` and supported `QUERY /intel`, the effective time range MUST be resolved from the request.

The implementation MUST NOT automatically widen the time range when no evidence is found.

### 10.2 Exception Assessment

When `assessment.categories.exceptions.enabled` is `true`, the analyzer MUST identify normalized evidence records categorized as `exception`.

Exception evidence SHOULD be grouped by:

1. `exceptionType`;
2. `operationName`; and
3. `httpRoute`, when present.

Each non-empty group MUST produce one finding.

### 10.3 HTTP 5xx Assessment

When `assessment.categories.http_5xx.enabled` is `true`, the analyzer MUST identify normalized evidence records where `httpStatusCode` is between `500` and `599`, inclusive.

HTTP 5xx evidence SHOULD be grouped by:

1. `httpMethod`;
2. `httpRoute`; and
3. `httpStatusCode`.

Each non-empty group MUST produce one finding.

### 10.4 No Findings

When:

- evidence availability is `available`; and
- no enabled analyzer produces a finding;

the assessment status MUST be `no_findings`.

`no_findings` means no configured negative-path evidence was observed during the assessed time range. It MUST NOT be represented as a general declaration that the service is healthy.

### 10.5 Findings

When:

- evidence availability is `available`; and
- one or more enabled analyzers produce findings;

the assessment status MUST be `findings`.

### 10.6 Partial Assessment

The assessment status MUST be `partial` when:

- the collector reports partial availability;
- one enabled category can be assessed but another cannot;
- evidence was truncated in a way that may affect the assessment; or
- collection warnings materially limit confidence.

A partial assessment MAY contain findings.

### 10.7 Unable to Assess

The assessment status MUST be `unable_to_assess` when:

- telemetry is unavailable;
- the collector cannot satisfy the collection request;
- no enabled category can be assessed; or
- an internal collection failure prevents analysis.

An unable-to-assess response MUST NOT claim no findings.

## 11. Protocol

### 11.1 GET /intel

`GET /intel` executes the default profile-defined assessment.

Request body:

```text
None
```

Successful response content type:

```http
application/json
```

Example:

```http
GET /intel
```

### 11.2 POST /intel

`POST /intel` executes an assessment with a caller-supplied time range.

Version 0.1 supports only a time range override.

A request MUST contain exactly one of:

- `duration`; or
- `startTime` and `endTime`.

Relative request:

```json
{
  "timeRange": {
    "duration": "PT5M"
  }
}
```

Absolute request:

```json
{
  "timeRange": {
    "startTime": "2026-07-30T00:00:00Z",
    "endTime": "2026-07-30T00:05:00Z"
  }
}
```

The requested range MUST NOT exceed `query.maximumTimeRange`.

When `query.allowTimeRangeOverride` is `false`, `POST /intel` and supported `QUERY /intel` MUST return the same machine-readable error.

### 11.3 Optional QUERY Compatibility

An implementation MAY support `QUERY /intel` when its complete request path supports the method. A supported `QUERY /intel` request MUST use the same request body and MUST have the same validation, scope resolution, response, error, rate-limiting, caching, telemetry, and security semantics as `POST /intel`.

Supporting `QUERY /intel` MUST NOT create a separate protocol contract and MUST NOT remove or restrict `POST /intel` support.

QUERY adoption remains limited across OpenAPI tooling, generated clients, frameworks, proxies, gateways, servers, and middleware. Clients SHOULD use `POST /intel` unless they have verified end-to-end QUERY support. Clients MAY use `QUERY /intel` when every relevant component in the request path is known to support it. Clients SHOULD NOT select QUERY through trial-and-error requests that may be rejected or transformed before reaching the application.

Version 0.1 defines no capability-discovery endpoint.

When `QUERY /intel` is unsupported and the request reaches the application, the implementation MUST NOT execute an assessment and MUST return:

```http
405 Method Not Allowed
Allow: GET, POST
Content-Type: application/problem+json
```

The response body MUST conform to RFC 9457 Problem Details. Its `type` MUST be `urn:op-intel:problem:query-method-not-supported`, and it MUST include:

```json
{
  "code": "query_method_not_supported"
}
```

Hosting infrastructure MAY reject QUERY before the request reaches the application. The preceding response requirements apply when the application receives the unsupported method.

### 11.4 Unsupported Scoped-Assessment Fields

Unknown request fields MUST cause request validation failure.

Version 0.1 MUST NOT accept:

- arbitrary telemetry filters;
- provider query expressions;
- route filters;
- exception type filters;
- service overrides;
- category overrides; or
- output-shape overrides.

These may be considered in a future version.

## 12. Response Contract

### 12.1 Top-Level Response

A successful assessment response MUST be a JSON object containing:

```text
specVersion
assessmentId
service
generatedAt
scope
telemetry
status
summary
findings
```

`assessmentId` MUST be a UUID version 4 represented as a lowercase canonical hyphenated string in the form `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`, where `y` is one of `8`, `9`, `a`, or `b`.

Consumers of successful assessment responses MUST ignore unknown fields that they do not understand. Unknown response fields are reserved for forward-compatible extension and MUST NOT cause response processing to fail.

### 12.2 Status Values

`status` MUST be one of:

```text
no_findings
findings
partial
unable_to_assess
```

The `summary` object MUST contain:

```text
uniqueFindingCount
exceptionCount
http5xxCount
```

`uniqueFindingCount` MUST equal the number of objects in the top-level `findings` array. It counts distinct grouped findings in the response and MUST NOT be interpreted as the sum of the occurrence counts represented by those findings.

`exceptionCount` MUST equal the sum of `count` across findings whose `category` is `exceptions`.

`http5xxCount` MUST equal the sum of `count` across findings whose `category` is `http_5xx`.

Because one operational event MAY contribute to findings in more than one category, consumers MUST NOT sum `exceptionCount` and `http5xxCount` to infer a count of unique operational events.

The `summary.message` field MUST be present when `status` is `no_findings`. It MAY be present for any other status.

### 12.3 Example: No Findings

```json
{
  "specVersion": "0.1",
  "assessmentId": "7d9f4b3e-6c21-4a8d-9f52-1e7b3c6a0d45",
  "service": {
    "name": "sample-api"
  },
  "generatedAt": "2026-07-30T00:15:00Z",
  "scope": {
    "startTime": "2026-07-30T00:00:00Z",
    "endTime": "2026-07-30T00:15:00Z",
    "categories": [
      "exceptions",
      "http_5xx"
    ]
  },
  "telemetry": {
    "source": "opentelemetry",
    "collector": "bounded-in-process",
    "availability": "available",
    "warnings": []
  },
  "status": "no_findings",
  "summary": {
    "uniqueFindingCount": 0,
    "exceptionCount": 0,
    "http5xxCount": 0,
    "message": "No configured negative-path evidence was observed during the assessed time range."
  },
  "findings": []
}
```

### 12.4 Example: Findings

```json
{
  "specVersion": "0.1",
  "assessmentId": "2a6e8c1d-5b47-4f90-a3d2-8c1e6b7f4a59",
  "service": {
    "name": "sample-api"
  },
  "generatedAt": "2026-07-30T00:15:00Z",
  "scope": {
    "startTime": "2026-07-30T00:00:00Z",
    "endTime": "2026-07-30T00:15:00Z",
    "categories": [
      "exceptions",
      "http_5xx"
    ]
  },
  "telemetry": {
    "source": "opentelemetry",
    "collector": "bounded-in-process",
    "availability": "available",
    "warnings": []
  },
  "status": "findings",
  "summary": {
    "uniqueFindingCount": 2,
    "exceptionCount": 3,
    "http5xxCount": 2,
    "message": "Configured negative-path evidence was observed during the assessed time range."
  },
  "findings": [
    {
      "id": "finding-001",
      "category": "exceptions",
      "severity": "error",
      "title": "Unhandled InvalidOperationException observed",
      "description": "Three InvalidOperationException events were observed for POST /orders.",
      "firstObservedAt": "2026-07-30T00:03:10Z",
      "lastObservedAt": "2026-07-30T00:12:44Z",
      "count": 3,
      "dimensions": {
        "exceptionType": "System.InvalidOperationException",
        "operationName": "POST /orders",
        "httpRoute": "/orders"
      },
      "evidence": [
        {
          "timestamp": "2026-07-30T00:03:10Z",
          "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
          "spanId": "00f067aa0ba902b7",
          "operationName": "POST /orders",
          "category": "exception",
          "exceptionType": "System.InvalidOperationException",
          "exceptionMessage": "Order state was invalid."
        },
        {
          "timestamp": "2026-07-30T00:08:21Z",
          "traceId": "7a3c1e9b5d2f4a608c7e1b3d9f5a2c40",
          "spanId": "1b2c3d4e5f607182",
          "operationName": "POST /orders",
          "category": "exception",
          "exceptionType": "System.InvalidOperationException",
          "exceptionMessage": "Order state was invalid."
        },
        {
          "timestamp": "2026-07-30T00:12:44Z",
          "traceId": "9d2f6a1c4b8e30f57c1a6d9e2b4f7083",
          "spanId": "2c3d4e5f60718293",
          "operationName": "POST /orders",
          "category": "exception",
          "exceptionType": "System.InvalidOperationException",
          "exceptionMessage": "Order state was invalid."
        }
      ]
    },
    {
      "id": "finding-002",
      "category": "http_5xx",
      "severity": "error",
      "title": "HTTP 500 responses observed",
      "description": "Two HTTP 500 responses were observed for POST /orders.",
      "firstObservedAt": "2026-07-30T00:03:10Z",
      "lastObservedAt": "2026-07-30T00:12:44Z",
      "count": 2,
      "dimensions": {
        "httpMethod": "POST",
        "httpRoute": "/orders",
        "httpStatusCode": 500
      },
      "evidence": [
        {
          "timestamp": "2026-07-30T00:03:10Z",
          "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
          "spanId": "00f067aa0ba902b7",
          "operationName": "POST /orders",
          "category": "http_5xx",
          "httpMethod": "POST",
          "httpRoute": "/orders",
          "httpStatusCode": 500
        },
        {
          "timestamp": "2026-07-30T00:12:44Z",
          "traceId": "9d2f6a1c4b8e30f57c1a6d9e2b4f7083",
          "spanId": "2c3d4e5f60718293",
          "operationName": "POST /orders",
          "category": "http_5xx",
          "httpMethod": "POST",
          "httpRoute": "/orders",
          "httpStatusCode": 500
        }
      ]
    }
  ]
}
```

### 12.5 Example: Truncated Evidence

The following finding assumes `output.includeEvidence` is `true` and `output.maximumEvidencePerFinding` is `1`:

```json
{
  "id": "finding-001",
  "category": "exceptions",
  "severity": "error",
  "title": "Unhandled InvalidOperationException observed",
  "description": "Three InvalidOperationException events were observed for POST /orders.",
  "firstObservedAt": "2026-07-30T00:03:10Z",
  "lastObservedAt": "2026-07-30T00:12:44Z",
  "count": 3,
  "dimensions": {
    "exceptionType": "System.InvalidOperationException",
    "operationName": "POST /orders",
    "httpRoute": "/orders"
  },
  "evidence": [
    {
      "timestamp": "2026-07-30T00:03:10Z",
      "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
      "spanId": "00f067aa0ba902b7",
      "operationName": "POST /orders",
      "category": "exception",
      "exceptionType": "System.InvalidOperationException",
      "exceptionMessage": "Order state was invalid."
    }
  ],
  "evidenceTruncated": true
}
```

### 12.6 Finding Requirements

Each finding MUST contain:

```text
id
category
severity
title
description
firstObservedAt
lastObservedAt
count
dimensions
```

`severity` MUST equal `error` in version 0.1.

The `severity` field is reserved for forward-compatible severity values in a future specification version. Version 0.1 producers MUST emit `error`, and version 0.1 consumers MUST NOT infer finer-grained severity from this field.

`category` MUST be one of:

```text
exceptions
http_5xx
```

The `id` field MUST be unique within the containing assessment response. It identifies only that finding object in that response and MUST NOT be treated as stable across assessments. Consumers correlating findings across assessments SHOULD use the finding's `category` and `dimensions` rather than `id`.

The `count` field MUST equal the total number of occurrences represented by that specific grouped finding. It is not a count of finding objects.

When `output.includeEvidence` is `true`, each finding MUST contain an `evidence` array. When `output.includeEvidence` is `false`, each finding MUST omit both `evidence` and `evidenceTruncated`. Evidence suppression is not evidence truncation, and `count` MUST remain present.

When `output.includeEvidence` is `true`, evidence returned per finding MUST NOT exceed `output.maximumEvidencePerFinding`.

When `count` is less than or equal to `output.maximumEvidencePerFinding`, the `evidence` array MUST contain exactly `count` records and `evidenceTruncated` MUST be omitted.

When `count` is greater than `output.maximumEvidencePerFinding`, the `evidence` array MUST contain exactly `output.maximumEvidencePerFinding` records and the finding MUST contain:

```json
{
  "evidenceTruncated": true
}
```

The `evidenceTruncated` field MUST NOT be present with a value of `false`.

Version 0.1 producers MUST NOT emit `totalEvidenceCount`. The required finding-level `count` already reports the total occurrences represented by the finding regardless of whether evidence is complete, truncated, or suppressed.

## 13. Error Handling

Protocol and configuration errors MUST use RFC 9457 Problem Details with content type:

```http
application/problem+json
```

### 13.1 Malformed Scoped-Assessment Request

Recommended status:

```http
400 Bad Request
```

### 13.2 Unsupported Scope Override

Recommended status:

```http
400 Bad Request
```

### 13.3 Time Range Exceeds Maximum

Recommended status:

```http
400 Bad Request
```

### 13.4 Invalid Profile

Recommended status:

```http
503 Service Unavailable
```

### 13.5 Internal Assessment Failure

Recommended status:

```http
500 Internal Server Error
```

An internal protocol failure MUST be distinguished from an `unable_to_assess` result.

Use `unable_to_assess` when the protocol executed successfully but operational evidence could not be obtained.

Use a Problem Details error when the `/intel` protocol itself failed.

### 13.6 Unsupported QUERY Method

An unsupported `QUERY /intel` request that reaches the application MUST use the HTTP `405` and Problem Details response defined in Section 11.3.

### 13.7 Rate Limit Exceeded

When an implementation rejects an `/intel` request because of rate limiting, it MUST return:

```http
429 Too Many Requests
Content-Type: application/problem+json
Retry-After: <delay-seconds-or-http-date>
```

The response body MUST conform to RFC 9457 Problem Details. Its `type` MUST be `urn:op-intel:problem:rate-limit-exceeded`, and it MUST include the extension member:

```json
{
  "code": "rate_limit_exceeded"
}
```

The `Retry-After` value MUST identify when the client may make a subsequent request. This specification does not prescribe a retry duration.

Example:

```json
{
  "type": "urn:op-intel:problem:rate-limit-exceeded",
  "title": "Assessment request rate limit exceeded",
  "status": 429,
  "detail": "Retry the assessment request after the interval indicated by the Retry-After header.",
  "instance": "/intel",
  "code": "rate_limit_exceeded"
}
```

### 13.8 Rate-Limiting and Cache Guidance

The numeric rate limit, enforcement scope, and coordination model are implementation-specific. Implementations SHOULD select limits based on assessment cost and deployment capacity rather than adopting a value from this specification.

Implementations SHOULD:

- prevent unbounded queued assessment work;
- consider whether enforcement is per service instance or shared across instances;
- account separately for the cost of executing an assessment and serving a cached response;
- behave deterministically for requests evaluated against the same limiter state; and
- continue emitting telemetry for rate-limited `/intel` requests.

An implementation MAY cache successful assessment responses to reduce repeated assessment work. Caching MUST NOT replace rate limiting. Cache duration and invalidation policy are implementation-specific.

A cached response MUST preserve the original `assessmentId` and `generatedAt`. An implementation MUST NOT present a cached response as a newly executed assessment. A cached response MUST satisfy the request's effective assessment scope and MUST NOT be reused for a different scope.

Telemetry for rate-limited or cached `/intel` requests MUST remain excluded from negative-path assessment by the same route exclusion or equivalent recursive-assessment safeguard used for other `/intel` traffic.

## 14. Telemetry and Logging

All `/intel` interactions MUST emit telemetry.

The implementation MUST record:

- request start time;
- request end time;
- HTTP method;
- route;
- assessment ID, when assigned;
- profile filename;
- profile version;
- effective time range;
- enabled categories;
- collector availability;
- result status;
- finding count; and
- protocol error code, when applicable.

The implementation SHOULD correlate `/intel` request telemetry using trace and span identifiers.

The implementation MUST NOT log secrets or restricted evidence fields.

Telemetry emitted by `/intel` MUST remain visible in Aspire or the configured OpenTelemetry destination, even though it is excluded from negative-path assessment by default.

## 15. Security

Version 0.1 permits unauthenticated local use for proof-of-concept development.

A production implementation MUST protect `/intel`.

Production security requirements are outside this version.

The POC MUST:

- avoid exposing request or response bodies from business operations;
- avoid exposing authorization headers;
- avoid exposing cookies;
- avoid exposing secrets;
- avoid exposing access tokens; and
- limit returned exception messages to values already permitted by application telemetry policy.

## 16. Required POC Behavior

A version 0.1 POC is complete only when all of the following are demonstrated:

1. The application runs locally under Aspire.
2. OpenTelemetry traces are visible in Aspire.
3. The application loads `op-intel.yaml`.
4. A valid profile permits `/intel` assessments.
5. An invalid profile produces a machine-readable configuration error.
6. `GET /intel` uses the profile default time range.
7. `POST /intel` accepts a valid relative time range.
8. `POST /intel` accepts a valid absolute time range.
9. `POST /intel` rejects ranges exceeding the configured maximum.
10. A generated exception that causes its instrumented operation to fail is recorded on its active span, the span status is set to `ERROR`, and the exception is collected and returned as a finding.
11. A generated HTTP 5xx response is collected and returned as a finding.
12. No matching negative-path evidence returns `no_findings`.
13. Telemetry unavailability returns `unable_to_assess`.
14. Partial evidence availability returns `partial`.
15. `/intel` requests emit telemetry.
16. `/intel` telemetry does not create recursive findings.
17. Collector retention is bounded.
18. Collector record count is bounded.
19. Normal OpenTelemetry export continues while collection is active.
20. No remediation or external action occurs.

## 17. Acceptance Criteria

### AC-001 — Canonical Profile

Given the application starts, when operational intelligence configuration is loaded, then the implementation MUST load a file named exactly `op-intel.yaml`.

### AC-002 — Valid Configuration

Given a conforming profile, when the application starts, then the profile MUST validate successfully and `/intel` MUST be available.

### AC-003 — Invalid Configuration

Given a non-conforming profile, when `/intel` is requested, then the implementation MUST return `application/problem+json` and MUST NOT execute an assessment.

### AC-004 — Default Assessment

Given a valid profile, when `GET /intel` is requested, then the implementation MUST assess the configured default time range and categories.

### AC-005 — Scoped Assessment

Given a valid override within the configured maximum, when `POST /intel` is requested, then the implementation MUST assess exactly the requested time range.

### AC-006 — Unsupported Scope

Given a request containing unsupported fields, when `POST /intel` or supported `QUERY /intel` is requested, then the implementation MUST reject the request.

### AC-007 — Exception Finding

Given an exception associated with an active OpenTelemetry span is caught by application or framework error handling and produces an HTTP 5xx response, when the span ends, then the implementation MUST have recorded an exception event and MUST have set the span status to `ERROR`.

Given an exception escapes an instrumented operation, when the span ends, then the implementation MUST have recorded an exception event and MUST have set the span status to `ERROR`.

Given an exception is caught and the instrumented operation successfully recovers, when the span ends, then the implementation MUST NOT have set the span status to `ERROR` solely because the exception occurred.

Given an instrumented operation fails without an exception, when the span ends, then the implementation MUST have set the span status to `ERROR` and MUST NOT have fabricated an exception event.

Given that recorded exception event occurs within the assessed time range, when `/intel` executes, then the response MUST contain an exception finding supported by normalized evidence.

### AC-008 — HTTP 5xx Finding

Given an HTTP 5xx response occurs within the assessed time range, when `/intel` executes, then the response MUST contain an HTTP 5xx finding supported by normalized evidence.

### AC-009 — No Findings

Given telemetry is available and no configured negative-path evidence occurs, when `/intel` executes, then the status MUST be `no_findings`.

### AC-010 — Telemetry Unavailable

Given telemetry cannot be collected, when `/intel` executes successfully, then the status MUST be `unable_to_assess` and MUST NOT be `no_findings`.

### AC-011 — Bounded Collection

Given a record timestamp is earlier than the retention cutoff, when retention is evaluated, then the collector MUST remove the record before enforcing the maximum record count and MUST NOT return it in a collection result.

Given a record timestamp equals the retention cutoff, when retention is evaluated, then the collector MUST NOT remove the record as expired.

Given the non-expired record count exceeds `telemetry.collector.maxRecords`, when the bound is enforced, then the collector MUST evict records by ascending evidence timestamp until the configured bound is satisfied.

Given multiple records share the oldest evidence timestamp, when capacity eviction is required, then the collector MUST evict those records in insertion order, earliest inserted first.

### AC-012 — Consistent Collector Snapshot

Given records are written while a collection request executes, when the collector constructs the result, then each concurrent record MUST appear either wholly in that result or wholly in a subsequent result and MUST NOT appear partially.

Given a collection request evaluates retention, time-range filtering, category filtering, and result construction, then all operations MUST use the same logical snapshot.

### AC-013 — Existing Export Preserved

Given normal OpenTelemetry export is configured, when the bounded collector is enabled, then telemetry MUST continue to appear in Aspire.

### AC-014 — Recursive Findings Prevented

Given `/intel` emits its own telemetry, when later assessments execute, then `/intel` telemetry MUST NOT create exception or HTTP 5xx findings by default.

### AC-015 — No Remediation

Given findings are produced, when the assessment completes, then the implementation MUST NOT modify application state, create work items, send notifications, or perform remediation.

### AC-016 — Response Summary Semantics

Given a successful assessment response, then `uniqueFindingCount` MUST equal the number of objects in `findings`, `exceptionCount` MUST equal the sum of `count` across exception findings, and `http5xxCount` MUST equal the sum of `count` across HTTP 5xx findings.

Given the response status is `no_findings`, then `summary.message` MUST be present.

### AC-017 — Finding Identity and Forward Compatibility

Given a successful assessment response contains multiple findings, then every finding `id` MUST be unique within that response.

Given equivalent findings occur in separate assessments, then consumers MUST NOT rely on finding `id` remaining stable across those responses and SHOULD correlate them using `category` and `dimensions`.

Given a successful assessment response contains an unknown field, then a conforming consumer MUST ignore that field and continue processing all fields it understands.

### AC-018 — Evidence Output

Given `output.includeEvidence` is `true` and a finding `count` does not exceed `output.maximumEvidencePerFinding`, then the finding MUST contain exactly `count` evidence records and MUST omit `evidenceTruncated`.

Given `output.includeEvidence` is `true` and a finding `count` exceeds `output.maximumEvidencePerFinding`, then the finding MUST contain exactly `output.maximumEvidencePerFinding` evidence records and MUST contain `evidenceTruncated` with the value `true`.

Given `output.includeEvidence` is `false`, then every finding MUST omit `evidence` and `evidenceTruncated` while retaining `count`.

Given no normalized evidence records match an enabled category, then the analyzer MUST NOT produce a finding for that category.

### AC-019 — Route Exclusion

Given a normalized route template is available, when exclusion patterns are evaluated, then the implementation MUST match only the complete normalized route template and MUST NOT match the raw request path.

Given no normalized route template is available but a raw request path is available, when exclusion patterns are evaluated, then the implementation MUST remove the query string and fragment before matching the complete raw path.

Given an exclusion pattern contains `*`, when it is evaluated, then each `*` MUST match zero or more characters including `/`.

Given no route template or raw path is available for an `/intel` request, then an equivalent safeguard MUST prevent that request's telemetry from creating recursive findings.

### AC-020 — Rate-Limited Assessment

Given an `/intel` request is rejected by rate limiting, then the implementation MUST return HTTP `429`, RFC 9457 Problem Details with code `rate_limit_exceeded`, and a valid `Retry-After` header.

Given a rate-limited `/intel` request emits telemetry, when later assessments execute, then that telemetry MUST NOT create exception or HTTP 5xx findings.

Given a cached assessment response is returned, then its `assessmentId`, `generatedAt`, and effective assessment scope MUST describe the original assessment and MUST NOT imply that a new assessment executed.

### AC-021 — Scoped Assessment Transport

Given a valid relative or absolute time-range override, when `POST /intel` is requested, then the implementation MUST apply the request contract defined in Section 11.2.

Given `query.allowTimeRangeOverride` is `false`, when `POST /intel` or supported `QUERY /intel` is requested, then both methods MUST return the same machine-readable error and MUST NOT execute an assessment.

Given an implementation supports `QUERY /intel`, when equivalent POST and QUERY requests are made, then both methods MUST apply identical request, validation, scope, response, error, rate-limiting, caching, telemetry, and security semantics.

Given `QUERY /intel` is unsupported and reaches the application, then the implementation MUST NOT execute an assessment and MUST return HTTP `405`, `Allow: GET, POST`, RFC 9457 Problem Details, and code `query_method_not_supported`.

## 18. Suggested Implementation Boundaries

The following names are non-normative but recommended:

```csharp
IOperationalProfileLoader
IOperationalProfileValidator
IOperationalEvidenceCollector
IOperationalAnalyzer
IOperationalAssessmentCoordinator
IOperationalResponseBuilder
```

Recommended concrete POC components:

```csharp
YamlOperationalProfileLoader
OperationalProfileValidator
BoundedOpenTelemetryCollector
ExceptionAnalyzer
Http5xxAnalyzer
OperationalAssessmentCoordinator
OperationalResponseBuilder
```

The implementation MAY combine components internally, but behavior MUST remain consistent with the responsibility boundaries in this specification.

## 19. Suggested Repository Artifacts

The initial repository SHOULD contain:

```text
/specification/operational-intelligence-specification.md
/specification/op-intel.schema.json
/examples/op-intel.yaml
/examples/responses/no-findings.json
/examples/responses/findings.json
/examples/responses/unable-to-assess.json
/src/
/tests/
README.md
LICENSE
NOTICE
```

The README SHOULD document:

- prerequisites;
- how to run the Aspire AppHost;
- how to generate a test exception;
- how to generate a test HTTP 500 response;
- how to call `GET /intel`;
- how to call `POST /intel`;
- whether `QUERY /intel` is supported and, when it is, how to call it;
- how to validate `op-intel.yaml`;
- how to observe telemetry in Aspire; and
- known POC limitations.

## 20. Future Work

The following are explicitly deferred:

- positive-path intelligence;
- configurable operational questions;
- additional analyzers;
- persistent collectors;
- Datadog collectors;
- Azure Monitor collectors;
- cross-service evidence;
- correlation across deployments and dependencies;
- policy-controlled remediation;
- autonomous self-healing;
- notification and ticket integrations; and
- production security requirements.

Future versions MUST preserve the separation between collection, assessment, and action.

---

## Appendix A — POC Request Examples

### Default assessment

```bash
curl -i http://localhost:5000/intel
```

### Relative POST assessment

```bash
curl -i \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "timeRange": {
      "duration": "PT5M"
    }
  }' \
  http://localhost:5000/intel
```

### Absolute POST assessment

```bash
curl -i \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "timeRange": {
      "startTime": "2026-07-30T00:00:00Z",
      "endTime": "2026-07-30T00:05:00Z"
    }
  }' \
  http://localhost:5000/intel
```

### Optional QUERY compatibility

When the implementation and complete request path support QUERY, the same request body MAY be sent with `-X QUERY`. POST remains required throughout version 0.1.

## Appendix B — POC Non-Goals

The POC MUST NOT expand to support a feature solely because the architecture could support it.

The POC remains complete when it proves:

```text
validated configuration
        +
bounded OTel collection
        +
negative-path assessment
        +
machine-readable /intel response
```

Everything else is deferred.
