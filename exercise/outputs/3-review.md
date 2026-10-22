Agent: Claude Code, model Claude Sonnet 5.5 (`claude-sonnet-5-5`). Reviewed ADR: `exercise/outputs/2b-adr.md` (chosen by the agent, since no `adr` path was given). Files read: `prompts/3-review-adr.md`, `exercise/outputs/2b-adr.md`, `docs/scenario.md`.

# ATAM-style review of ADR-0002 (RabbitMQ for post-order actions)

## 1. Quality-attribute scenarios

| # | Source · Stimulus · Environment | Response | Measure |
|---|---|---|---|
| S1 Performance | Customer · places order · 11.11 sale, 200 orders/s | `PlaceOrder` completes, including the publish | p95 <= 500 ms |
| S2 Availability | MailFast, `loyaltyservice` or analytics down (or RabbitMQ down) · orders keep arriving · normal and peak load | Orders still succeed; post-order work resumes on recovery | 0 orders failed because of these; no event lost |
| S3 Correctness | Order placed · consumer crashes and the event is redelivered · peak load | Points awarded once; customer gets one email | points within 1 min, exactly once; 1 email per order |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Email, loyalty and analytics are off the checkout path. | MailFast p95 is 1.5 s against a 500 ms budget (approximately 200 ms headroom over today's 300 ms). The synchronous option could never meet S1. |
| Non-risk | Loyalty is idempotent on order ID with a unique constraint. | At-least-once delivery plus a dedupe key meets "exactly once" (S3) without a distributed transaction. |
| Non-risk | Self-hosted RabbitMQ over Kafka or a managed service. | Fits the open-source, self-hosted preference and two developers' experience; Kafka's replay is not needed by any stated requirement. |
| Sensitivity point | Length of the sale peak (not known). | Backlog grows about 150 emails/s (200 orders/s against the 50/s limit). A 10-minute peak is about 90,000 queued emails, so the confirmation delay depends mainly on this number. |
| Sensitivity point | MailFast's 50 emails/s limit. | Retries of the roughly 0.5 % failures use the same 50/s budget, so effective throughput is slightly lower than the ADR says. |
| Trade-off point | Async events trade performance and availability against timeliness and consistency. | Checkout is fast and robust, but a confirmation email may arrive long after the order during the sale. The ADR defers the acceptable delay to the business. |
| Risk | Publish happens after the order succeeds, with no committed mechanism if the publish fails. The outbox is only "for example". | A crash or broker outage at that moment silently loses the event: no points, no email, which breaks S2 and S3. |
| Risk | Idempotency is designed for points only. | A redelivered event can send the customer a second email, and double-count the analytics event (S3). |
| Risk | Loyalty "within 1 minute" is not tied to broker or consumer outages. | During an outage or backlog, points can exceed 1 minute with no stated recovery target. |
| Risk | Only 2 of 4 developers know RabbitMQ, and it becomes a new stateful production component right before the 11.11 sale. | Operational mistakes (clustering, disk alarms) would hit the sale itself. |

## 3. Missing or weak

- **Options not considered:** (a) a hybrid, asynchronous email only, with loyalty and analytics as direct gRPC calls guarded by timeouts and circuit breakers; that uses the team's gRPC strength and avoids a broker, but would couple checkout to each new action; (b) a Go-friendly lighter broker (e.g. NATS JetStream) or a database-backed queue; (c) an explicit transactional outbox as its own option; (d) MailFast batching, if it has a batch API.
- **Likely wrong:** "mirrored/quorum queues". Classic mirrored queues are deprecated, so the ADR should say quorum queues only.
- **Unsupported claim:** "p95 stays near 300 ms plus one publish". No publish cost is measured, and it is not said whether checkout waits for the broker's publisher confirm, which is itself a latency cost.
- **Not stated:** the acceptable email delay at peak, how long the backlog takes to drain, where in `PlaceOrder` the publish happens (after shipping?), and what checkout does when the broker is down.
- **Fraud scoring:** the ADR itself admits it may need a synchronous answer, which an async event cannot give. That undermines the "all future actions are just new consumers" claim.
- **Consequences understate:** poison messages, dead-letter handling and queue monitoring are named only briefly.

## 4. Verdict

**Accept with changes.** The core decision (async event, RabbitMQ) is sound for this scenario. Three most important changes:

1. **Make publishing reliable**: commit to an outbox or equivalent, and say what happens when the broker is down, so no order loses its event (S2).
2. **Extend idempotency to every consumer** (email dedupe, analytics dedupe) and set a recovery target for the 1-minute loyalty goal.
3. **Quantify the peak**: get the peak duration and the accepted email delay, then size the backlog, and consider the hybrid option for loyalty and analytics. Also correct "mirrored queues" to quorum queues.
