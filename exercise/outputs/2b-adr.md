Agent: Claude Code, model Claude Sonnet 5.5 (`claude-sonnet-5-5`). Run `b` (with context). Context files: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`.

# ADR-0002: Publish an "order placed" event through RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Not given (see Open questions)

## Context and problem

Requirement R-12: after an order is placed, Online Boutique must (1) send the order confirmation email through **MailFast**, a new external provider that replaces the mock `emailservice` logic; (2) award **loyalty points** through a new `loyaltyservice` (to be built); (3) record an **"order placed" analytics event** for marketing. Marketing expects more post-order actions next year (SMS, fraud scoring, a recommendations update).

Today `checkoutservice.PlaceOrder` calls `paymentservice`, `shippingservice` and then `emailservice` synchronously over gRPC, in sequence. If the email call fails, checkout only logs a warning and the order still succeeds. Facts from `docs/scenario.md`:

| Topic | Fact |
|---|---|
| Checkout latency | p95 of `PlaceOrder` must stay <= 500 ms (today approximately 300 ms), so the headroom is approximately 200 ms |
| Peak load | 200 orders/s during the 11.11 sale |
| MailFast | p95 1.5 s, p99 8 s, fails approximately 0.5 % of calls, rate limit 50 emails/s |
| Loyalty points | within 1 minute of the order, exactly once |
| Availability | an order must never fail because email, loyalty or analytics is down |
| Team | 4 developers, strong in Go and gRPC, two have used RabbitMQ |
| Operations | everything runs on Kubernetes; open-source, self-hosted components preferred over cloud-only managed services |

Derived from these facts (my arithmetic, not given):

- MailFast's p95 of 1.5 s is three times the whole 500 ms budget, so a synchronous MailFast call cannot fit.
- At 200 orders/s and a 50 emails/s limit, emails can be sent at only a quarter of the order rate. The surplus is 150 emails/s, so a backlog builds during the peak and must drain afterwards.
- A 0.5 % failure rate at 200 calls/s is about one failed email per second, which needs retries.

## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 must stay <= 500 ms at 200 orders/s.
3. **Reliability / correctness:** loyalty points within 1 minute and exactly once.
4. **Compatibility with MailFast limits:** 50 emails/s, 0.5 % failures, slow tail.
5. **Modifiability:** more post-order actions (SMS, fraud scoring, recommendations update) are expected next year.
6. **Cost and operations:** open-source and self-hosted on Kubernetes.
7. **Team fit:** 4 developers, Go and gRPC, two with RabbitMQ experience.

## Considered options

1. **Synchronous gRPC calls from `checkoutservice`** to `emailservice` (MailFast), `loyaltyservice` and an analytics sink, as email is done today.
2. **Event on RabbitMQ (self-hosted).** Checkout publishes one "order placed" event to a fanout/topic exchange. Email, loyalty and analytics each consume from their own queue.
3. **Event on Apache Kafka (self-hosted).** Checkout publishes to a topic. Each action is a consumer group.
4. **Cloud-managed messaging service** (a cloud pub/sub or queue product).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC | Simplest. Fits the team's gRPC skills. No new infrastructure. Immediate results. | MailFast p95 1.5 s breaks the 500 ms p95 budget. 50 emails/s limit cannot serve 200 orders/s. Slow or failing calls threaten "never fail an order". Every new action means changing checkout. |
| 2. RabbitMQ | Checkout only publishes, so latency impact is small and consumer failures stay off the checkout path. Queue per consumer gives independent retries, dead-letter queues and a buffer for the 150/s peak surplus. Adding SMS or fraud scoring means adding a queue and consumer. Open source, runs on Kubernetes, two team members know it. | New stateful component to operate (clustering, monitoring). At-least-once delivery, so consumers must be idempotent. Eventual consistency. Only 2 of 4 developers have used it. |
| 3. Kafka | Durable log, replay of events, high throughput. Good fit for analytics. Open source. | Heavier to operate than the load needs. No team experience mentioned. Per-message retry and rate limiting are less direct than with queues. |
| 4. Cloud-managed | Least operations work. | Conflicts with the preference for open-source, self-hosted components over cloud-only managed services. Lock-in. |

## Decision

Choose **Option 2: publish an "order placed" event to self-hosted RabbitMQ** and implement three consumers: an email consumer (calls MailFast, replacing the mock logic in `emailservice`), `loyaltyservice` and an analytics consumer.

Why it best satisfies the drivers:

- **Availability and performance (1, 2):** after the order succeeds, checkout only publishes one message. Nothing on the checkout path waits for MailFast, loyalty or analytics, so their outages and MailFast's 1.5 s p95 do not touch the 500 ms budget.
- **MailFast limits (4):** the email consumer takes messages from its queue at no more than 50 emails/s. The queue absorbs the peak surplus, and failed sends (about 0.5 %) are retried with back-off and then dead-lettered.
- **Exactly-once points (3):** RabbitMQ delivers at-least-once, so `loyaltyservice` must be idempotent. It records the order ID with a unique constraint and ignores a repeated order ID. It consumes quickly enough to meet the 1-minute target, which is separate from the slower email queue.
- **Modifiability (5):** SMS, fraud scoring and the recommendations update become new queues bound to the same exchange, with no change to checkout.
- **Cost and team fit (6, 7):** RabbitMQ is open source and self-hostable on the existing Kubernetes, and two developers already know it. Kafka was rejected as heavier than needed, and the managed service was rejected on the stated operations preference.

Same shape as ADR-0001 (small interface, swappable behind it): checkout publishes through a small `OrderEvents` interface, so the broker can be changed later without touching business logic.

## Consequences

- **Positive:** `PlaceOrder` p95 stays near today's approximately 300 ms plus one publish (performance). Orders survive email, loyalty and analytics outages (availability). New post-order actions need no checkout change (modifiability). Fits the open-source, self-hosted preference (cost, portability).
- **Negative:** one more stateful component to run and monitor (operations cost). Emails arrive later than the order, and during the 11.11 peak the delay grows with the backlog (user-visible latency of the follow-up action). Consumers need idempotency logic (complexity).
- **Risks and mitigations:**
  - *Email backlog during the sale.* The arrival rate is 200/s against a 50/s limit, so the backlog grows by about 150 messages/s of peak. Mitigation: alert on queue depth and message age, agree an acceptable email delay with the business, and drain after the peak. How long the peak lasts is Not given.
  - *Event lost between order success and publish.* Mitigation: publisher confirms, plus a way to republish if the publish fails (for example a transactional outbox). Whether checkout has a data store for this is Not given.
  - *Duplicate delivery gives double points.* Mitigation: idempotent consumer keyed by order ID with a unique constraint.
  - *Broker outage.* Mitigation: run a clustered RabbitMQ with mirrored/quorum queues. Checkout must treat a publish failure as non-fatal to the order, to keep "never fail".
  - *Only two developers know RabbitMQ.* Mitigation: pair or share knowledge, and use a runbook.

## Validation

- **Load test** `PlaceOrder` at 200 orders/s with the broker on, and check p95 <= 500 ms.
- **Failure tests:** stop `loyaltyservice`, the analytics consumer, MailFast (simulated) and RabbitMQ in turn. Orders must still succeed, and events must be processed after recovery.
- **Rate-limit test:** confirm the email consumer never exceeds 50 emails/s, and measure queue depth and the time to drain after a simulated peak.
- **Duplicate test:** redeliver the same event and check that points are awarded once. Check points appear within 1 minute at peak.
- **Review** after the 11.11 sale, and when the first new post-order action (SMS, fraud scoring) is added, to check it needed no change to checkout.

## Open questions

These were not given, and I did not invent them.

1. How long does the 11.11 peak at 200 orders/s last? This sets the email backlog size and drain time.
2. What email delay is acceptable to the business after an order (minutes, hours)?
3. Which analytics system receives the "order placed" event, and what fields does the event need?
4. Does `checkoutservice` have any data store for an outbox, and what are the rules for losing an event if the publish fails?
5. Where will `loyaltyservice` store points and processed order IDs, and which store is allowed?
6. What are the loyalty rules (points per order), and who owns them?
7. Are there availability targets for the broker itself, and does the team have capacity to operate a clustered RabbitMQ?
8. Is MailFast's rate limit fixed, or can it be raised for the sale?
9. Are SMS, fraud scoring and the recommendations update sure to be built, and would any need a synchronous answer (fraud scoring might need to run before the order completes)?
10. Who are the deciders for this ADR?
