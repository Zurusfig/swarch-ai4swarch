# ADR-0002: Use RabbitMQ for Post-Order Actions

**Status:** Proposed
**Date:** 2026-10-25
**Author:** Ittichet Thongsang
**Deciders:** To be confirmed

## Context and problem

Requirement R-12 adds three actions after checkout:

* Send confirmation emails through MailFast.
* Award loyalty points.
* Record analytics events.

More actions may be added in the future.
Currently, checkout sends emails synchronously. Adding more service calls could slow checkout or introduce failures.
*Sources: `docs/scenario.md` (R-12); `system/online-boutique/src/checkoutservice/main.go` (PlaceOrder, sendOrderConfirmation).*
**System constraints:**

* Checkout p95 ≤ 500 ms (currently \~300 ms).
* Peak traffic: 200 orders/s.
* MailFast: p95 1.5 s, p99 8 s, \~0.5% failures, limit 50 requests/s.
* Loyalty: exactly once per order, within 1 minute.
* Post-order service outages must not fail orders.
* Team: 4 developers, 2 with RabbitMQ experience.

*Source: `docs/scenario.md` (Quality goals and constraints).*

## Decision drivers

* **Performance:** Keep checkout fast.
* **Availability:** Service failures must not affect orders.
* **Correctness:** Prevent duplicate loyalty awards.
* **Modifiability:** Easily add future actions.
* **Operability:** Keep maintenance manageable.

## Considered options

1. **Synchronous gRPC:** Checkout directly calls each service.
2. **Background worker:** Save jobs and process them later.
3. **RabbitMQ:** Send events to separate service queues.

## Pros and cons of the options

| Option | Pros | Cons |
| ------ | ---- | ---- |
| Synchronous gRPC | Simple, familiar | Slows checkout; failed actions can be lost |
| Background worker | Fast checkout; supports retries | Requires job storage and worker management |
| RabbitMQ | Independent processing; easy to extend; team has experience | More infrastructure; must handle duplicate messages |

## Decision

**Choose RabbitMQ (proposed).**
RabbitMQ separates checkout from email, loyalty, and analytics. Separate queues let each action handle its own failures, retries, and workload. A job-based worker could also provide this isolation, but would require additional design.
RabbitMQ also makes future actions easier to add through new consumers. Since two team members already know RabbitMQ, the additional infrastructure is more manageable.
**Implementation:**

1. **Durable events:** Propose a transactional outbox to reliably store `OrderPlaced` events. This depends on first defining durable order storage and handling failures after payment or shipping.
2. **RabbitMQ:** Use durable queues, persistent messages, and publisher confirmations for reliable delivery. Retry failed publications. Consumers acknowledge messages only after successful processing.
3. **Email worker:** Send through MailFast. Limit all workers combined to **50 requests/s, including retries**.
4. **Loyalty worker:** Store a unique award record and update points in one database transaction. This prevents duplicate awards when implemented correctly.
5. **Analytics worker:** Record each event once using its unique event ID.
6. **Recovery:** Retry temporary failures, monitor delays, and investigate repeated failures.

**Before approval, resolve:**

* How to safely record orders and events, including crashes after payment or shipping.
* What email delay is acceptable during peak traffic.
* What loyalty outage duration the one-minute deadline must cover.

## Consequences

**Positive:**

* **Performance:** Checkout avoids waiting for external services.
* **Availability:** Post-order failures do not directly affect orders.
* **Modifiability:** New consumers can be added without changing checkout for every action.
* **Correctness:** Atomic award records and points updates prevent duplicate awards when implemented correctly.

**Negative:**

* **Operability:** RabbitMQ and workers need monitoring and maintenance.
* **Timeliness:** Emails may be delayed during high traffic.
* **Reliability:** Extra safeguards are needed to prevent lost or duplicate work.

**Key risks:**

| Risk | Mitigation |
| ---- | ---------- |
| Crash after payment but before saving event | Define durable storage and order reconciliation |
| RabbitMQ unavailable | Keep events in the proposed outbox and retry |
| Duplicate loyalty messages | Unique award record and atomic transaction |
| MailFast overloaded | Shared rate limit and backlog monitoring |
| Loyalty outage | Define supported outage duration and recovery plan |
| Duplicate email after timeout | Use MailFast idempotency if supported |

## Validation

1. **Load test:** Checkout p95 ≤ 500 ms at 200 orders/s.
2. **Service failures:** Email, loyalty, and analytics outages must not fail checkout.
3. **Duplicate events:** Verify loyalty points are awarded only once.
4. **Crash recovery:** Verify successfully placed orders do not lose events.
5. **RabbitMQ outage:** Verify stored events are delivered after recovery.
6. **MailFast:** Verify total requests, including retries, stay ≤ 50/s.
7. **Loyalty timing:** Verify awards complete within 60 seconds under agreed outage conditions.
8. **Recovery:** Test retries, failed-message handling, and analytics deduplication.

## Open questions

* Who approves and maintains the system?
* Where will orders and events be stored? How are crashes after payment handled?
* What loyalty outage duration must the one-minute deadline cover?
* What email delay is acceptable? Can MailFast capacity increase?
* Does MailFast support duplicate-request protection?
* What are the loyalty rules and analytics storage requirements?
* What infrastructure and recovery budget is available?