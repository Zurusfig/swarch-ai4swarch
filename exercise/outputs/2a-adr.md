Agent: Codex; model: GPT-6

# ADR-0002: Process post-order actions through durable events

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not specified; see Open questions.

## Context and problem

The student's supplied requirement requires Online Boutique, after an order is placed, to:

1. Send confirmation email through the new external provider MailFast, replacing the mock emailservice logic.
2. Award loyalty points through a new loyaltyservice to be built.
3. Record an "order placed" analytics event for marketing.

Source for these facts: the requirement in the student's message, items 1–3. No system context files were provided. Existing order placement behavior, persistence, dependencies, protocols, and failure handling: Not evidenced in the provided files.

The decision concerns how to trigger and reliably complete these three actions. Everything below describing implementation is a proposal, not a recovered fact about Online Boutique.

## Decision drivers

Proposed priorities, to be confirmed by the deciders:

- **Reliability:** Retain unfinished work and recover from transient failures.
- **Correctness:** Avoid duplicate loyalty awards and duplicate recorded events during retries.
- **Availability and performance:** Allow order completion without waiting for all three downstream actions.
- **Modifiability:** Isolate MailFast integration and allow each action to evolve independently.
- **Operability and cost:** Balance recovery controls against additional infrastructure and support work.

The requirement does not define delivery deadlines, consistency guarantees, or acceptable operating cost.

## Considered options

1. Call the MailFast email adapter, loyaltyservice, and analytics recorder directly from the order placement flow, retaining durable action status for retries.
2. Publish a durable order-placed event and process it through independent email, loyalty, and analytics consumers.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Direct calls with durable retry status | Fewer messaging components; immediate outcomes are visible to the caller; a central flow is straightforward to trace. | Downstream calls increase response latency and couple availability; partial success still requires durable recovery and deduplication; adding actions changes the central flow. |
| 2. Durable event with independent consumers | Consumers recover independently; downstream outages need not delay order completion; new consumers can be added without extending the order request path. | Requires durable messaging, monitoring, replay procedures, and duplicate handling; completion is eventually consistent; reliable publication must be coordinated with order placement. |

These comparisons are **[Inferred]** engineering trade-offs of the proposed options, not observations of the current implementation.

## Decision

Propose option 2, subject to confirming that eventual completion satisfies the requirement. Its independent recovery paths support the three required actions without making their response times part of the order request path.

- Define the successful order-placement boundary before implementation. Durably record the order-placed event with that transition. Use a transactional outbox if the order store supports a suitable transaction; otherwise resolve a reliable publication mechanism before accepting this design. Do not assume an existing order database or transaction capability.
- Deliver the event through durable messaging with independent subscriptions for email, loyalty, and analytics. Acknowledge each delivery only after its action succeeds or its durable recovery state is recorded. Broker product and protocol remain undecided.
- Replace the mock emailservice behavior with a MailFast adapter. Keep provider credentials and API-specific mapping within that adapter. MailFast protocol, authentication, and API guarantees remain unverified.
- Build loyaltyservice to calculate and durably award points using rules to be supplied. Make each award idempotent for its order and action, with the duplicate check and award committed atomically.
- Have an analytics consumer durably record the marketing event in a destination to be agreed. Deduplicate using a stable event identifier.
- Define a versioned event contract containing a stable event identifier, order identifier, and only the data each action needs. Resolve customer identity and privacy requirements before fixing its payload.
- Retry transient failures with bounded backoff; retain exhausted or permanent failures for alerting and controlled replay. Track completion separately for each action. Assume duplicate deliveries are possible; do not claim end-to-end exactly-once execution.
- For MailFast, use a provider idempotency key or outcome reconciliation if supported. An ambiguous timeout can otherwise cause duplicate email on retry; this limitation requires an explicit delivery-policy decision.

## Consequences

- **Positive:** Independent recovery can keep order placement available during downstream outages, provided durable publication remains available (quality attribute: availability; **[Inferred]**).
- **Positive:** Consumer isolation localizes provider changes and future post-order actions (quality attribute: modifiability; **[Inferred]**).
- **Negative:** Email, points, and analytics may become visible at different times after order placement (quality attribute: consistency and user experience).
- **Negative:** Durable messaging, consumer state, and recovery procedures add operational work and infrastructure expense (quality attribute: operability and cost).
- **Risks and mitigations:** A failure between order placement and event publication can lose required work. Validate atomic recording or another proven recovery mechanism before rollout (quality attribute: reliability).
- **Risks and mitigations:** Duplicate deliveries can repeat effects. Use atomic deduplication for loyalty and analytics, and verify MailFast idempotency or reconciliation capabilities (quality attribute: correctness).
- **Risks and mitigations:** Backlogs or permanent failures can leave actions unfinished. Monitor action completion, oldest pending age, retries, and failed deliveries; assign ownership and replay procedures (quality attribute: operability and reliability).
- **Risks and mitigations:** Events may expose customer information. Minimize payloads and agree access, retention, and deletion controls (quality attribute: security and privacy).

## Validation

Before acceptance:

- Review the order-placement boundary and demonstrate that crashes cannot leave a successfully placed order without recoverable post-order work.
- Demonstrate all three actions for a placed order: MailFast confirmation, one correct loyalty award, and one recorded marketing event.
- Inject consumer crashes, duplicate deliveries, provider timeouts, and messaging interruptions. Verify eventual recovery and absence of duplicate loyalty awards or analytics records.
- Test an ambiguous MailFast response and document whether duplicate email can be prevented or reconciled.
- Verify that failure of one consumer does not prevent the other two actions, and that failed work can be inspected and replayed safely.
- Measure order response latency and per-action completion delay under representative load. Set acceptance thresholds with stakeholders before testing; no numerical targets were supplied.
- Review event payload access and retention, monitoring, alert ownership, and the operational cost estimate.

## Open questions

- Who are the deciders and operational owners?
- What exactly establishes that an order is placed, and what persistence or transaction facilities exist?
- Is eventual completion acceptable for all three actions? What are their deadlines, outage tolerances, and failure notification requirements?
- Which MailFast API, authentication, quotas, delivery-status, idempotency, and reconciliation capabilities are available?
- What customer identity, points formula, eligibility, and cancellation/refund rules should loyaltyservice use?
- Where should marketing analytics be recorded, with what schema, retention, and deduplication rules?
- What traffic volume, latency targets, hosting constraints, budget, and team capabilities govern the messaging choice?
- What event data may be shared with each consumer and MailFast, and what privacy requirements apply?
