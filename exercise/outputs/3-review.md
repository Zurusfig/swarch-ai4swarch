Agent: Codex; model: GPT-6

## Quality-attribute scenarios

| Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| Customers | Place 200 orders/s | 11.11 peak; post-order services unavailable | Complete checkout and retain post-order work | PlaceOrder p95 ≤ 500 ms; zero orders fail because email, loyalty or analytics is down. Include persistence overhead in the load test. [S:24–28; A:59–60] |
| Loyalty delivery path | Redelivers an order event after a crash | Peak load and an explicitly bounded outage | Commit one award and points update per order | Exactly one award within 60 s of the order, including outage and queue time. Supported outage duration remains undefined. [S:27; A:62,70,106] |
| Customers and MailFast | Generate 200 confirmation emails/s; provider calls fail or time out | Peak; MailFast p95 1.5 s, p99 8 s, ~0.5% failures | Rate-limit workers, retain pending emails and drain backlog | ≤ 50 requests/s including retries; measure oldest-email age and drain time against an agreed deadline, currently absent. [S:25–26; A:61,69] |

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Reliable publication depends on order storage and payment/shipping recovery that remain proposals. [A:59,68,91–92] | A crash between an external side effect and the durable order/event record can leave completed work without a recoverable event. Reliability is conditional on resolving this boundary. |
| Non-risk | Unique award records and points updates in one transaction address duplicate loyalty delivery. [A:62,79,93] | Supports correctness, provided uniqueness is keyed by order and survives republication with a different event ID. |
| Sensitivity point | [Inferred] One email per order at 200 orders/s, with at most 50 sends/s, adds at least 150 pending emails/s: 9,000 per peak minute, before retries. [S:10,25–26; A:61] | Peak duration, storage capacity and post-peak arrivals determine email delay and drain time; queues cannot remove this capacity deficit. |
| Sensitivity point | Loyalty's 60-second budget includes outage, publication delay, queue wait and processing. [S:27; A:70,106] | [Inferred] An outage consuming the entire minute makes the deadline impossible; the supported outage bound and recovery throughput must be quantified together. |
| Trade-off point | Separate RabbitMQ queues isolate action workloads and support future consumers, but add broker/worker maintenance. [A:55–56,78,83] | Modifiability and availability improve at an operability/cost expense for four developers. Two know RabbitMQ; operational ownership remains unresolved. [S:29; A:111,117] |

## Missing or weak

- **Options:** The persisted background-worker option already offers retries and isolation [A:49,55]. Compare storage, recovery, fan-out and operating costs with RabbitMQ under equivalent guarantees. A staged worker-first rollout is not evaluated [A:38–56].
- **Failure semantics:** Define order acceptance, the atomic order/outbox boundary, payment/shipping reconciliation, and publication/acknowledgement crash behavior [A:59–60,68]. Existing durable order storage is **Not evidenced in the provided files**.
- **Recovery:** Specify retry bounds/backoff, poison-message quarantine/replay, retention/capacity limits and alerts tied to loyalty age and email backlog [A:64,85,107]. Concrete mechanisms are **Not evidenced in the provided files**.
- **Claims and consequences:** Current email failure already permits order success [S:18]; the proposed benefit is broader isolation and latency protection. Analytics “once” needs an atomic deduplication/write boundary [A:63]. MailFast idempotency remains unverified [A:96,115]. Include Kubernetes and the self-hosted/open-source preference in the option comparison [S:30].

## Verdict

**Accept with changes** — accept the direction, with final approval conditional on resolving the ADR's own prerequisites [A:66–70]. The three most important changes are:

1. Define durable order/outbox storage and payment/shipping reconciliation; test crashes across persistence and publication boundaries.
2. Agree an email delay target and peak duration; size backlog storage, concurrency and drain capacity while enforcing the shared 50/s limit including retries.
3. Define loyalty outage/recovery bounds and order-keyed atomic deduplication; verify end-to-end completion within 60 s at peak load.

References: **A** = `docs/adr/0002-post-order-actions.md`; **S** = `docs/scenario.md`. Numbers refer to file lines. These are the only review evidence files.
