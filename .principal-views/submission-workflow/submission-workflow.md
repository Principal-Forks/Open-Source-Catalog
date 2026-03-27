# NASA Open Source Catalog - Submission Workflow (OpenTelemetry)

## Overview

This canvas models the NASA Open Source Catalog submission workflow using OpenTelemetry (OTEL) conventions to demonstrate how the system would emit telemetry if it were instrumented. While the current implementation is a static data repository without runtime code, this conceptual model provides:

- **Documentation** of the event-driven submission lifecycle
- **Preparation** for future instrumentation efforts
- **Visibility** into the logical workflow for stakeholders
- **Framework** for performance monitoring if the system evolves

## Problem Statement

Understanding and monitoring the submission-to-publication pipeline is critical for:
- Identifying bottlenecks in validation or publication
- Tracking submission success rates
- Debugging failed submissions
- Measuring end-to-end latency
- Ensuring SLA compliance for federal reporting

## Telemetry Model

### OTEL Resource

**Service:** `nasa-open-source-catalog`
- **Namespace:** `nasa.ocio`
- **Environment:** `production`
- **Version:** `1.0.0`

This resource represents the catalog service and provides context for all spans and events.

### Key Events

Events represent point-in-time occurrences with no duration.

#### 1. `catalog.submission.received`
**When:** A new project submission arrives through any channel
**Attributes:**
- `submission.source`: `github_pr | web_form | robot_branch`
- `submission.project_name`: Name of the submitted project
- `submission.nasa_center`: NASA center submitting the project

**Use Case:** Track submission volume by source, identify peak submission times

#### 2. `catalog.validation.started`
**When:** Travis CI begins validation of the submission
**Attributes:**
- `validator`: `travis_ci`
- `commit.sha`: Git commit hash being validated

**Use Case:** Correlate validation start with submission received to measure queue time

#### 3. `catalog.validation.completed`
**When:** Schema validation finishes
**Attributes:**
- `validation.result`: `success | failure`
- `validation.errors`: Array of validation errors if failed
- `validation.duration_ms`: Time taken for validation

**Use Case:** Track validation success rate, identify common validation errors

#### 4. `catalog.submission.merged`
**When:** Submission is merged to the master branch
**Attributes:**
- `git.branch`: `master`
- `git.commit_sha`: Final commit hash
- `merge.author`: Username who approved/merged

**Use Case:** Track merge velocity, identify frequent contributors

#### 5. `catalog.publication.completed`
**When:** Data is successfully published to both portals
**Attributes:**
- `publication.success`: Boolean success flag
- `publication.portals`: Array of portals (code.nasa.gov, code.gov)
- `publication.timestamp`: ISO 8601 timestamp

**Use Case:** Monitor publication uptime, track publication latency

#### 6. `catalog.history.recorded`
**When:** Version snapshot is written to code-history.json
**Attributes:**
- `history.file`: `code-history.json`
- `history.uuid`: UUID of the snapshot
- `history.snapshot_size`: Size of the snapshot in bytes

**Use Case:** Monitor history file growth, ensure snapshots are captured

### Key Spans

Spans represent operations with duration and can have parent-child relationships.

#### 1. `catalog.submission.process` (Parent Span)
**Duration:** From submission received to merge completed
**Kind:** `internal`
**Attributes:**
- `catalog.operation`: `process_submission`
- `git.branch`: `master`
- `submission.id`: UUID for tracking

**Child Spans:**
- `catalog.schema.validate`
- `catalog.data.persist`

**Use Case:** Measure end-to-end submission processing time, identify slowest stages

#### 2. `catalog.schema.validate` (Child Span)
**Duration:** Schema validation against code-nasa-schema.json
**Kind:** `internal`
**Attributes:**
- `schema.version`: `draft-04`
- `schema.file`: `code-nasa-schema.json`
- `validation.fields_checked`: Number of fields validated

**Use Case:** Identify schema validation performance, detect slow validations

#### 3. `catalog.data.persist` (Child Span)
**Duration:** Writing to catalog.json, code.json, and code-history.json
**Kind:** `internal`
**Attributes:**
- `db.system`: `git`
- `db.operation`: `write`
- `files.written`: `3`

**Use Case:** Monitor Git write performance, detect filesystem issues

#### 4. `catalog.publication.execute` (Parent Span)
**Duration:** From publication start to completion
**Kind:** `client`
**Attributes:**
- `publication.targets`: `2`
- `pipeline.stage`: `production`

**Child Spans:**
- `catalog.publish.code_nasa_gov`
- `catalog.publish.code_gov`

**Use Case:** Track publication duration, identify portal-specific issues

#### 5. `catalog.publish.code_nasa_gov` (Child Span)
**Duration:** HTTP request to code.nasa.gov
**Kind:** `client`
**Attributes:**
- `http.url`: `https://code.nasa.gov`
- `http.method`: `POST`
- `http.status_code`: `200`

**Use Case:** Monitor code.nasa.gov API performance and availability

#### 6. `catalog.publish.code_gov` (Child Span)
**Duration:** HTTP request to code.gov
**Kind:** `client`
**Attributes:**
- `http.url`: `https://code.gov`
- `http.method`: `POST`
- `http.status_code`: `200`

**Use Case:** Monitor code.gov API performance and federal compliance reporting

## Workflow Patterns

### Happy Path Flow

1. **Submission Received** → Event emitted with source metadata
2. **Processing Begins** → Parent span starts
3. **Validation Started** → Event emitted, child span begins
4. **Schema Validated** → Child span completes, event emitted with result
5. **Data Persisted** → Child span writes to Git
6. **Submission Merged** → Event emitted with commit SHA
7. **Publication Begins** → Parent span starts
8. **Portals Published** → Two child spans execute in parallel
9. **Publication Completed** → Event emitted confirming success
10. **History Recorded** → Event emitted with snapshot metadata

**Expected Duration:** 2-5 minutes end-to-end

### Error Scenarios

#### Schema Validation Failure

**Flow:**
1. Submission received
2. Processing span starts
3. Validation started
4. **Schema validation fails** (span status: error)
5. `catalog.validation.completed` event with `validation.result: failure`
6. Processing span ends with error status

**Telemetry Attributes:**
- `error`: `true`
- `error.type`: `SchemaValidationError`
- `error.message`: Specific validation failure
- `validation.errors`: Array of field-level errors

**Recovery:** Contributor fixes issues and resubmits

#### Publication Failure

**Flow:**
1. Normal processing completes successfully
2. Publication span starts
3. **One portal publish fails** (e.g., code.gov timeout)
4. Child span has error status
5. `catalog.publication.completed` event with `publication.success: false`

**Telemetry Attributes:**
- `error`: `true`
- `error.type`: `PublicationError`
- `http.status_code`: `504` (Gateway Timeout)
- `failed.portal`: `code.gov`

**Recovery:** Retry mechanism or manual intervention

## Observability Benefits

### Performance Monitoring

**Metrics to Track:**
- **P50/P95/P99 submission processing duration** - Identify outliers
- **Validation success rate** - Track data quality
- **Publication latency by portal** - Detect API degradation
- **Submissions per hour** - Understand load patterns

### Debugging

**Use Cases:**
1. **Slow Submission:** Examine child spans to find bottleneck (validation vs. persistence vs. publication)
2. **Failed Publication:** Check HTTP status codes and retry patterns
3. **Missing History Snapshots:** Verify `catalog.history.recorded` events are emitted

### Alerting

**Potential Alerts:**
- Validation failure rate > 20%
- Publication to code.gov failing for > 5 minutes
- End-to-end submission time > 10 minutes
- No submissions received in 24 hours (unexpected)

## Implementation Considerations

### If This Were Instrumented

**Services to Instrument:**

1. **code-submission-app.code.nasa.gov** (Web Form)
   - Emit `catalog.submission.received` on form submit
   - Track form validation duration

2. **GitHub Actions/Travis CI** (Validation)
   - Emit `catalog.validation.started` and `catalog.validation.completed`
   - Create `catalog.schema.validate` span

3. **Publication Service** (Hypothetical)
   - Create `catalog.publication.execute` span
   - Track HTTP requests to portals as child spans

4. **Git Operations** (Merge/Write)
   - Emit `catalog.submission.merged` on PR merge
   - Create `catalog.data.persist` span for file writes

### OTEL Collector Configuration

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:
    timeout: 10s
  attributes:
    actions:
      - key: service.namespace
        value: nasa.ocio
        action: insert

exporters:
  jaeger:
    endpoint: jaeger-collector:14250
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

### Sampling Strategy

**Recommended:**
- **Head-based sampling:** 100% for errors, 10% for successful submissions
- **Tail-based sampling:** Sample slow submissions (> 5 minutes) at 100%

This ensures full visibility into failures while managing telemetry volume.

## Future Enhancements

Potential additions to the telemetry model:

1. **User Journey Tracking:** Connect submissions to specific NASA centers or users
2. **Dependency Tracking:** Monitor GitHub API, Git operations, external APIs
3. **Business Metrics:** Track NASA-wide open source portfolio growth
4. **Compliance Metrics:** Ensure M-16-21 policy compliance is measurable
5. **Security Events:** Track authentication, authorization, and access patterns

## Comparison to Current State

| Aspect | Current (No Instrumentation) | With OTEL |
|--------|------------------------------|-----------|
| **Debugging** | Manual log inspection | Distributed tracing with context |
| **Performance** | No visibility | P50/P95/P99 latency metrics |
| **Failures** | Git history only | Real-time alerts and error tracking |
| **Bottlenecks** | Unknown | Span analysis shows slowest operations |
| **Trends** | Manual analysis | Automated dashboards and trends |
| **Alerting** | None | Proactive alerts on SLA violations |

## Conclusion

While this catalog is currently a static data repository, modeling it as an event-driven system with OpenTelemetry conventions provides:
- **Clear documentation** of the logical workflow
- **Preparation** for future instrumentation if services are built
- **Stakeholder visibility** into the submission lifecycle
- **Best practices** for observability in distributed systems

This conceptual model demonstrates that even data-centric systems can benefit from structured telemetry thinking.
