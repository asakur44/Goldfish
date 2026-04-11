---
type: pattern
problem: "Inconsistent error handling across services leads to silent failures and poor debugging experience"
solution: "Structured error responses with error codes, retry with exponential backoff, circuit breaker for external dependencies"
applicability: "Any service making external API calls or handling user-facing requests"
anti_patterns: ["Swallowing exceptions silently", "Infinite retry loops", "Generic 500 errors without context"]
status: accepted
created: 2026-04-11
last_verified: 2026-04-11
triggers: [error, exception, retry, fallback, graceful degradation, error handling]
sources: ["Learned from production incident 2026-02"]
---

# Error Handling Pattern

## Problem
Services handle errors inconsistently — some swallow exceptions, some retry forever, some return generic 500s. Debugging is painful.

## Solution

### 1. Structured Error Responses
All API errors return:
```json
{
  "error": "PAYMENT_PROVIDER_TIMEOUT",
  "message": "Payment provider did not respond within 5s",
  "retry_after": 30,
  "request_id": "req_abc123"
}
```

### 2. Retry Strategy
- Exponential backoff: 1s, 2s, 4s, 8s, max 3 retries
- Only retry on transient errors (5xx, timeouts, network errors)
- Never retry on 4xx (client errors)

### 3. Circuit Breaker
For external dependencies:
- Open after 5 consecutive failures in 60 seconds
- Half-open after 30 seconds (allow one probe request)
- Close on successful probe

## Anti-Patterns
- **Silent exception swallowing**: Always log, always surface to caller
- **Infinite retries**: Cap at 3, then fail with clear error
- **Generic 500**: Include error code and request ID for tracing

<!-- 
TEMPLATE NOTE: Delete this article and create your own patterns.
-->
