# Platform Engineering Take-Home Exercise

## Vendor Signal Normalization & Routing

**Time Estimate:** 3-4 hours  
**Language:** Your choice of Python or TypeScript/Node.js

---

## Overview

Build a small HTTP service that ingests vendor signals from multiple sources (webhooks, status pages), normalizes them into a unified schema, and routes critical events to downstream destinations.

This exercise simulates a real-world platform engineering challenge: building reliable, observable infrastructure that integrates multiple third-party services.

*We don't care if you use AI to build this, but be prepared to answer any and all questions about the codebase, why you did certain things and other questions.*

---

## Requirements

### Core Functionality

Your service must:

1. **Ingest Stripe webhook events** with signature verification
2. **Poll external status pages** (Statuspage.io format) for Spreedly and Braze
3. **Normalize** all signals into a unified internal event schema
4. **Route** critical events to a mock PagerDuty destination
5. **Store** recent events in memory with idempotency
6. **Expose** a query API for recent events

---

## API Specification

### 1. POST `/ingest/stripe`

Accepts Stripe webhook events with signature verification.

**Requirements:**
- Verify webhook signature using Stripe's signing secret
- Return `400` if signature verification fails
- Return `200` with `{ received: true }` on success
- Normalize the event and store it
- Route to PagerDuty if severity is critical

**Environment Variables:**
- `STRIPE_WEBHOOK_SECRET` (required)
- `STRIPE_API_KEY` (optional, not needed for verification)

**Example request:**
```bash
curl -X POST http://localhost:3000/ingest/stripe \
  -H "Content-Type: application/json" \
  -H "Stripe-Signature: t=..." \
  -d @fixtures/stripe/payment_failed.json
```

---

### 2. POST `/ingest/pull-status`

Fetches and processes Statuspage.io summary.json from configured URLs.

**Requirements:**
- Fetch from `SPREEDLY_STATUS_SUMMARY_URL` and `BRAZE_STATUS_SUMMARY_URL`
- If URLs not configured, fall back to local fixtures
- Parse incidents and components
- Normalize and store events
- Route critical incidents

**Response:**
```json
{
  "fetched": {
    "spreedly": 2,
    "braze": 1
  },
  "stored": 3,
  "routed": 1
}
```

**Environment Variables:**
- `SPREEDLY_STATUS_SUMMARY_URL` (optional)
- `BRAZE_STATUS_SUMMARY_URL` (optional)

---

### 3. POST `/destinations/pagerduty`

Mock PagerDuty destination endpoint.

**Requirements:**
- Accept normalized events
- Log receipt with structured logging
- Store in memory for verification
- Return `202 Accepted`

---

### 4. GET `/events`

Query recent normalized events.

**Query Parameters:**
- `limit` (default: 50) - number of events to return

**Response:**
```json
{
  "events": [
    {
      "event_id": "evt_123",
      "source": "stripe",
      "kind": "payment",
      "severity": "critical",
      "service": "stripe",
      "summary": "Payment failed for pi_abc",
      "description": null,
      "started_at": "2024-01-15T10:30:00Z",
      "resolved_at": null,
      "routed": true,
      "delivered_to": ["pagerduty"],
      "raw": { ... }
    }
  ]
}
```

---

### 5. GET `/healthz`

Health check endpoint.

**Response:**
```json
{ "ok": true }
```

---

## Normalized Event Schema

All vendor signals must be normalized to this schema:

```typescript
{
  event_id: string,           // Unique identifier
  source: "stripe" | "spreedly_status" | "braze_status",
  kind: "incident" | "status" | "payment",
  severity: "info" | "warning" | "critical",
  service: string,            // Service name
  summary: string,            // Short description
  description: string | null, // Detailed info
  started_at: string,         // ISO-8601 timestamp
  resolved_at: string | null, // ISO-8601 or null if ongoing
  raw: unknown                // Original event for debugging
}
```

**Validation:**
- Use schema validation (Zod, Pydantic, etc.)
- Enforce types strictly

---

## Normalization Rules

### Stripe Events

Support at least these event types:

| Event Type | Severity | Kind | Notes |
|------------|----------|------|-------|
| `payout.failed` | critical | payment | Financial impact |
| `payment_intent.payment_failed` | warning | payment | May need investigation |

**Mapping:**
- `event_id` = Stripe's `event.id`
- `started_at` = Convert `event.created` (unix) to ISO-8601
- `resolved_at` = null (webhooks are point-in-time)
- `summary` = `{event.type}: {object.id}`

---

### Statuspage.io Format

Typical `summary.json` structure:

```json
{
  "incidents": [
    {
      "id": "abc123",
      "name": "API Degradation",
      "status": "investigating",
      "impact": "major",
      "created_at": "2024-01-15T10:00:00Z",
      "updated_at": "2024-01-15T10:15:00Z",
      "resolved_at": null
    }
  ],
  "components": [
    {
      "id": "comp_1",
      "name": "API",
      "status": "operational",
      "updated_at": "2024-01-15T09:00:00Z"
    }
  ]
}
```

**Incident Mapping:**

| Impact | Severity |
|--------|----------|
| critical, major | critical |
| minor | warning |
| none, maintenance | info |

**Component Status Mapping:**

| Status | Create Event? | Severity |
|--------|---------------|----------|
| `operational` | No | - |
| `degraded_performance` | Yes | warning |
| `partial_outage` | Yes | critical |
| `major_outage` | Yes | critical |

---

## Routing Logic

Events should be routed based on severity:

| Severity | Route to PagerDuty? | Condition |
|----------|---------------------|-----------|
| critical | Yes | Always |
| warning | Conditional | Only if `ROUTE_WARNING=true` |
| info | No | Never |

**Delivery:**
- Best-effort: log failures but don't block ingestion
- Mark delivery status in stored event metadata
- Internal call to POST `/destinations/pagerduty`

---

## Idempotency

**Requirements:**
- Track seen `event_id` values in memory
- If duplicate detected:
  - Return `200 { received: true, deduped: true }`
  - Do NOT route/deliver again
  - Do NOT create duplicate in event store

**Implementation suggestion:**
- Use Set or Map: `event_id -> first_seen_timestamp`
- Include in `/events` response metadata if useful

---

## Bonus: AI Hook (Optional)

Add an **optional** endpoint that demonstrates how you would integrate AI/LLM capabilities:

### POST `/ai/summarize`

**Input:**
```json
{
  "text": "Multiple payment failures detected across APAC region..."
}
```

**Output:**
```json
{
  "summary": "Payment failures in APAC",
  "suggested_severity": "critical"
}
```

**Implementation:**
- Use a **deterministic stub** (keyword matching, no real API calls)
- In documentation, describe how you WOULD integrate a real LLM:
  - Prompt engineering
  - PII/secrets redaction
  - Audit logging
  - Fallback behavior
  - Cost controls
  - Latency considerations

---

## Technical Requirements

### Language & Frameworks

**Option 1: TypeScript/Node.js**
- Node 18+
- Fastify or Express
- Zod for validation
- Pino for structured logging
- Jest or Vitest for tests

**Option 2: Python**
- Python 3.10+
- FastAPI or Flask
- Pydantic for validation
- structlog for structured logging
- pytest for tests

### Must Include

- ✅ Structured logging (JSON format)
- ✅ Request correlation IDs
- ✅ Strict type checking (TypeScript strict mode or mypy)
- ✅ Input validation on all endpoints
- ✅ Proper error handling with appropriate status codes
- ✅ At least 3 meaningful unit tests
- ✅ Docker support (Dockerfile + docker-compose.yml)
- ✅ Environment variable configuration
- ✅ `.env.example` file

---

## Fixtures

Provide example fixture files:

### `fixtures/stripe/payout_failed.json`
```json
{
  "id": "evt_1abc",
  "object": "event",
  "type": "payout.failed",
  "created": 1705315200,
  "data": {
    "object": {
      "id": "po_123",
      "amount": 10000,
      "currency": "usd",
      "failure_message": "Insufficient funds"
    }
  }
}
```

### `fixtures/stripe/payment_failed.json`
```json
{
  "id": "evt_2def",
  "object": "event",
  "type": "payment_intent.payment_failed",
  "created": 1705315800,
  "data": {
    "object": {
      "id": "pi_456",
      "amount": 5000,
      "currency": "usd",
      "last_payment_error": {
        "message": "Card declined"
      }
    }
  }
}
```

### `fixtures/statuspage/spreedly_summary.json`
```json
{
  "page": {
    "id": "spreedly",
    "name": "Spreedly Status"
  },
  "incidents": [
    {
      "id": "inc_spreedly_1",
      "name": "Payment Gateway Latency",
      "status": "investigating",
      "impact": "major",
      "created_at": "2024-01-15T10:00:00Z",
      "updated_at": "2024-01-15T10:30:00Z",
      "resolved_at": null
    }
  ],
  "components": [
    {
      "id": "comp_api",
      "name": "API",
      "status": "operational",
      "updated_at": "2024-01-15T09:00:00Z"
    }
  ]
}
```

### `fixtures/statuspage/braze_summary.json`
```json
{
  "page": {
    "id": "braze",
    "name": "Braze Status"
  },
  "incidents": [],
  "components": [
    {
      "id": "comp_dashboard",
      "name": "Dashboard",
      "status": "degraded_performance",
      "updated_at": "2024-01-15T11:00:00Z"
    }
  ]
}
```

---

## Documentation Requirements

### README.md

Your submission must include:

1. **Setup Instructions**
   - Dependencies installation
   - Environment configuration
   - How to run locally
   - How to run with Docker

2. **Usage Examples**
   - curl commands for each endpoint
   - How to trigger ingestion with fixtures
   - How to verify routing behavior

3. **Architecture Overview**
   - High-level system diagram (ASCII or description)
   - Key design decisions
   - Data flow explanation

4. **Testing**
   - How to run tests
   - What is tested
   - Test coverage approach

5. **Security Considerations**
   - Stripe signature verification (why raw body matters)
   - Secret management
   - Input validation

6. **Production Readiness Discussion**
   - What would you add for production?
   - Tradeoffs made for this exercise
   - Scalability considerations
   - Observability improvements
   - Persistence strategy
   - Queue/retry mechanisms
   - Rate limiting

---

## Evaluation Criteria

We'll evaluate based on:

### Core Functionality (40%)
- ✅ All endpoints work as specified
- ✅ Stripe signature verification works
- ✅ Normalization is correct
- ✅ Routing logic is accurate
- ✅ Idempotency is implemented

### Code Quality (30%)
- ✅ Clean, readable code
- ✅ Proper error handling
- ✅ Type safety
- ✅ Structured logging
- ✅ Input validation

### Testing (15%)
- ✅ Meaningful unit tests
- ✅ Test coverage of critical paths
- ✅ Tests are runnable

### Documentation (10%)
- ✅ Clear setup instructions
- ✅ Architecture explanation
- ✅ Production considerations

### DevOps (5%)
- ✅ Docker works correctly
- ✅ Environment configuration
- ✅ Easy to run locally

---

## Submission

1. Push your code to a **private** GitHub repository
2. Invite the reviewers (we'll provide usernames)
3. Include:
   - All source code
   - Tests
   - Fixtures
   - Dockerfile + docker-compose.yml
   - README.md
   - .env.example

**Do NOT include:**
- `node_modules/` or virtual environments
- Real secrets or API keys
- Build artifacts

---

## Time Expectations

This exercise is designed to take **3-4 hours**. We understand you may not have time to implement everything perfectly. 

**Prioritize in this order:**
1. Core endpoints working (stripe ingestion, status polling, events query)
2. Proper normalization and routing
3. Tests for critical logic
4. Documentation
5. Bonus features (AI hook)

**If short on time:**
- Mock external HTTP calls in tests
- Document what you would improve given more time
- Focus on demonstrating your thought process

---

## Questions?

If anything is unclear, please email us or open an issue in the repository we shared with you. We'll respond within 24 hours.

Good luck! 🚀
