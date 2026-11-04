# API Lenses (backend)

Detect by route or controller definitions, OpenAPI files, server framework config, and the absence of UI code. The "user" here is the API consumer. Cite `file:line` for every hit.

## Error Clarity

- Error responses with no machine-readable code or field-level detail.
- Inconsistent error shapes across endpoints.
- Wrong or overloaded status codes (200 with error body, 500 for bad input).
- Validation that stops at the first error instead of listing all.

## Pagination and Filtering

- List endpoints returning unbounded results.
- Missing sort, filter, or search on collections that grow.
- Inconsistent pagination style between endpoints.

## Idempotency and Safety

- Create or payment-like endpoints with no idempotency key.
- Retried requests that duplicate data.
- Bulk operations with no partial-failure report.

## Documentation and Examples

- No OpenAPI or equivalent spec, or one out of sync with routes.
- Endpoints with no request or response examples.
- No documented auth flow or error catalog.

## Versioning and Compatibility

- No version in path or header; breaking changes hit consumers silently.
- No deprecation signals (headers, docs notes).

## Rate Limits and Quotas

- No rate-limit headers (remaining, reset, retry-after).
- Throttling that returns unexplained failures.

## Observability

- No request id returned or logged for support tracing.
- No health or readiness endpoint.
- Logs without route, status, or latency.

## Naming and Consistency

- Mixed casing, plural vs singular, or verb vs noun routes.
- Same field named differently across resources.
- Timestamps in mixed formats.
