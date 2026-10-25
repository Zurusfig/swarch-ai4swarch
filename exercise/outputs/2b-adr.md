Agent: Codex; model: GPT-6

# ADR-0002: Use RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not specified; see Open questions.

## Context and problem

R-12 requires confirmation through external MailFast, loyalty points through a new loyaltyservice, and an order placed analytics event. Marketing expects further actions next year (docs/scenario.md, “The new requirement (R-12)”; student requirement).

Today, checkoutservice.PlaceOrder charges the card, ships the order, attempts to empty the cart, constructs OrderResult, and synchronously calls sendOrderConfirmation before returning. Email failure only produces a warning. sendOrderConfirmation uses the emailservice gRPC client (system/online-boutique/src/checkoutservice/main.go, PlaceOrder, sendOrderConfirmation, shipOrder). The mock characterization comes from docs/scenario.md, “R-12”; its implementation is outside the supplied evidence.

| Constraint | Evidence |
|---|---|
| PlaceOrder p95 ≤ 500 ms; current approximately 300 ms; peak 200 orders/s | docs/scenario.md, “Quality goals and constraints” |
| MailFast p95 1.5 s, p99 8 s, approximately 0.5% failures, limit 50 emails/s | docs/scenario.md, “Quality goals and constraints” |
| Loyalty within 1 minute, exactly once; email, loyalty, or analytics outages must never fail an order | docs/scenario.md, “Quality goals and constraints” |
| Four developers, strong Go/gRPC skills; two have RabbitMQ experience; Kubernetes and preference for self-hosted open source | docs/scenario.md, “Quality goals and constraints” |

[Inferred] Waiting for MailFast in PlaceOrder threatens the latency target. At a sustained 200 orders/s, one email per order produces at least 150 queued emails/s against the 50 emails/s limit, before retries. A queue absorbs a bounded burst but cannot solve sustained excess demand (derived from docs/scenario.md constraints).

## Decision drivers

1. **Availability:** post-order dependency outages must not fail orders (docs/scenario.md, quality constraints).
2. **Performance:** keep PlaceOrder p95 ≤ 500 ms at 200 orders/s (same source).
3. **Correctness and timeliness:** one loyalty award per order within 1 minute (same source).
4. **Modifiability:** add future post-order actions without changing checkout for each action (docs/scenario.md, R-12).
5. **Operability and cost:** suit a four-person team and self-hosted Kubernetes deployment (docs/scenario.md, team and operations).

## Considered options

These are proposed designs, not descriptions of existing components.

1. Extend synchronous gRPC orchestration in checkoutservice with a MailFast adapter, loyaltyservice, and analytics calls; tolerate their errors.
2. Persist post-order jobs and let a Go worker orchestrate the three actions, with separate completion and retry state per action.
3. Publish OrderPlaced through RabbitMQ to independent email, loyalty, and analytics consumers, backed by durable event capture.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous calls | Familiar gRPC approach; fewer infrastructure components; immediate results when dependencies respond | MailFast latency conflicts with checkout budget; ignoring failures loses work without durable retries; checkout changes for each new action |
| Durable jobs and worker | Removes remote calls from checkout; durable retries; potentially simpler than operating a broker | Requires a durable job store and scheduler; central orchestration changes for new actions; action isolation and concurrent processing require explicit design |
| RabbitMQ and consumers | Independent retries, backlogs and scaling; new subscribers support future actions; matches stated team experience and hosting preference | Broker plus durable capture and consumer state increase operational cost; duplicate delivery and publication gaps require handling; asynchronous completion needs monitoring |

## Decision

Choose **option 3**, subject to validation below. The proposed isolation and extensibility best match the availability and future-action drivers; RabbitMQ familiarity and the hosting preference support this choice (docs/scenario.md, R-12 and team/operations constraints). The following are proposed changes, not existing behavior:

- Capture a versioned OrderPlaced event durably at the defined successful-order boundary. Include a stable event ID, order ID, occurrence time, user ID and required order data; exclude payment-card data. PlaceOrder currently provides an order UUID, user ID, email and OrderResult, but shows no durable order/event commit or request deduplication (main.go, PlaceOrder). Define the durable order record and transactional outbox together before implementation; do not assume the existing function already supports an outbox transaction.
- Relay committed outbox events to a durable RabbitMQ exchange and three independent durable queues, using persistent messages and publisher confirmations. Retry unconfirmed publication; consumers must tolerate duplicates. Broker unavailability leaves events in the outbox and must not prevent checkout success while durable capture remains available.
- Replace the synchronous confirmation call with an email consumer using a MailFast adapter in place of mock email logic. Enforce a shared 50 emails/s ceiling across replicas, including retries, with bounded concurrency, timeouts, backoff and provider-aware retry handling. MailFast's API protocol and idempotency support are **Not evidenced in the provided files**; confirm them before implementing the adapter (docs/scenario.md identifies the provider and limits only).
- Build loyaltyservice as an independent consumer. In its durable store, atomically record the unique order-award key and update the points balance, then acknowledge delivery. Repeated events must produce no additional award. This provides an exactly-once business effect despite at-least-once delivery; broker delivery alone is insufficient. The storage technology and points rules remain open.
- Have an independent analytics consumer persist the marketing event with an event-ID uniqueness check, then acknowledge. Its destination and schema remain open.
- Retry transient failures independently; quarantine persistent failures for investigation and replay. Alert on outbox age, oldest loyalty event, retry counts and queue age. Prioritize loyalty independently of email backlog; define alert thresholds that leave time to recover before the 1-minute deadline.

[Inferred] An arbitrarily long loyaltyservice outage cannot coexist with an unconditional 1-minute award guarantee. This proposal preserves orders and retries work, but does not claim to satisfy that deadline during every outage. Resolve the outage envelope before acceptance (docs/scenario.md, loyalty and availability constraints).

## Consequences

- **Positive:** post-order remote latency and consumer failures leave the checkout request path; durable buffering supports recovery (quality attributes: performance, availability).
- **Positive:** separate subscriptions allow future actions without adding synchronous checkout dependencies, provided the event contract contains their needed data (quality attribute: modifiability).
- **Positive:** atomic loyalty deduplication protects against duplicate points after redelivery (quality attribute: correctness).
- **Negative:** broker, outbox relay and durable consumer state require deployment, backup, recovery and monitoring work for the four-person team (quality attributes: operability, cost; team size: docs/scenario.md).
- **Negative:** confirmation and analytics become eventually completed; MailFast backlog grows during peaks above its limit (quality attributes: timeliness, user experience; limits: docs/scenario.md).
- **Risks and mitigations:** a crash between payment/shipping and durable order/event capture can omit an event. Payment and shipping precede the response in PlaceOrder, with no durable completion record shown (main.go, PlaceOrder). Define reconciliation and stable request identity across retries; test every boundary before acceptance (quality attributes: reliability, correctness).
- **Risks and mitigations:** durable capture failure cannot be hidden without risking lost post-order work. Define recovery behavior separately from consumer outages; a transactional outbox alone cannot atomically commit the existing remote payment/shipping effects (main.go, PlaceOrder). Treat this as an unresolved acceptance risk (quality attributes: availability, durability).
- **Risks and mitigations:** ambiguous MailFast timeouts can cause duplicate email. Use provider idempotency if supported; otherwise establish an explicit duplicate-versus-loss policy. Add dead-letter replay controls and protect retained email/order data with access restrictions and an agreed retention policy (quality attributes: reliability, privacy).

## Validation

All checks below are proposed acceptance tests, not completed results.

- Load-test 200 orders/s with the supplied MailFast latency, failure and rate-limit profile; require PlaceOrder p95 ≤ 500 ms and no checkout failures caused by unavailable email, loyalty or analytics (docs/scenario.md, constraints).
- Demonstrate one loyalty award per order within 60 seconds in the agreed operating envelope. Inject duplicate events, concurrent consumers and crashes before/after the loyalty transaction and acknowledgement; verify balances and award records.
- Kill checkout and relay processes around durable capture, publication and confirmation. Restart RabbitMQ and consumers; reconcile every successfully completed order against all three action records and verify no lost events.
- Hold the broker unavailable while durable capture stays healthy; verify successful checkout and eventual relay. Separately inject durable-store failures and validate the agreed recovery policy.
- Verify the combined MailFast request rate, including retries, never exceeds 50/s. Measure backlog growth and drain time over the agreed sale duration; verify loyalty latency remains independent of email backlog.
- Exercise quarantine, alerting and replay; verify analytics deduplication and consumer compatibility with event schema changes.

## Open questions

- Who decides and owns the broker, consumers and production incident response?
- What defines an order as placed, and where will durable order state and the outbox live? What reconciliation covers crashes after payment or shipping, and what stable request key prevents retries creating multiple logical orders? These mechanisms are **Not evidenced in the provided files** (main.go, PlaceOrder).
- What behavior is required when durable capture itself fails after remote payment/shipping succeeds?
- What outage duration must the 1-minute loyalty guarantee cover, and what recovery capacity and alert thresholds are required? An unbounded outage cannot meet this deadline.
- What is the sale peak duration, acceptable email delay and available MailFast quota increase? At sustained 200 orders/s, the stated 50 emails/s limit cannot keep pace.
- What MailFast API, authentication, timeout, idempotency and retry semantics apply?
- How are points calculated, which orders qualify, and how are cancellation/refund adjustments handled? Which transactional store will hold loyalty balances and award keys?
- Where must marketing events be recorded, with which schema, retention and delivery deadline?
- What broker/store availability topology, retention, disk capacity, recovery targets and operational budget are approved?
