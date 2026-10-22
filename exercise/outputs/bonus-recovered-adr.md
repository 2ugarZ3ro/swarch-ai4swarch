Agent: Claude Code, model Claude Sonnet 5.5 (`claude-sonnet-5-5`).

Decision reconstructed (chosen by the agent, since the student left it open): **use gRPC with a shared Protocol Buffers contract for service-to-service communication.**
Evidence allowed (chosen by the agent): `system/online-boutique/protos/demo.proto`, `system/online-boutique/kubernetes-manifests/frontend.yaml`, `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`, plus `docs/adr/template.md`.

# ADR-NNNN: Use gRPC with a shared Protocol Buffers contract between services

- **Status:** Reconstructed
- **Date:** Not evidenced in the provided files
- **Deciders:** Not evidenced in the provided files

## Context and problem

Online Boutique is split into separate services (cart, catalogue, checkout, payment, shipping, currency, email, recommendation, ads) that call each other at runtime. `frontend` calls seven of them and `checkoutservice` calls six. The system needs one agreed way for these services to call each other.

## Decision drivers

Not stated in the evidence. The following are **[Inferred]**: low per-call overhead, a strict machine-checked interface, and freedom to implement each service in its own language.

## Considered options

The evidence shows only the chosen option. These alternatives are **[Inferred]**:

1. gRPC with Protocol Buffers (chosen)
2. REST/JSON over HTTP
3. Asynchronous messaging through a broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| gRPC + protobuf | One typed contract file for all services. Generated client and server code. Compact binary messages | Needs generated code and tooling. Browsers cannot call it directly, which is why `frontend` is a separate HTTP layer. Callers wait for replies |
| REST/JSON | Simple, easy to debug, browsers can call it | No single enforced contract. Larger messages. Contract drift between services |
| Broker messaging | Callers do not wait. Failures are isolated | New infrastructure. Harder to follow a request. Not suited to "reply now" calls such as a price quote |

## Decision

All service-to-service calls use **gRPC**, with every service interface and message defined once in `protos/demo.proto` (package `hipstershop`). Only the `frontend` serves HTTP to users. Services find each other by Kubernetes Service name and port, given to the caller as environment variables such as `PRODUCT_CATALOG_SERVICE_ADDR`.

## Consequences

- **Positive:** a single shared contract keeps callers and callees consistent (modifiability); Kubernetes health checks use gRPC directly (operability).
- **Positive (Inferred):** compact messages and generated code help performance and let teams choose languages.
- **Negative:** calls are synchronous request/response, so `checkoutservice` waits on six dependencies in one flow (availability, performance). The proto contract is shared by all services, so changing it can affect many callers (modifiability).
- **Negative:** sensitive data travels in messages. `ChargeRequest` carries `credit_card_number` and `credit_card_cvv` (security). Whether the channel is encrypted or authenticated is Not evidenced.
- **Risks and mitigations:** retries, timeouts and circuit breaking are Not evidenced in the provided files, so a slow dependency may slow checkout. Verifying this would need the service source.

## Validation

Not evidenced. A real ADR would say how the decision is checked, for example latency measurements of `PlaceOrder`.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | Business capabilities are split into separate services | Evidenced | `demo.proto` (`service CartService`, `ProductCatalogService`, `CheckoutService`, `PaymentService`, ...) |
| 2 | `frontend` depends on seven services | Evidenced | `frontend.yaml` env vars `PRODUCT_CATALOG_SERVICE_ADDR`, `CURRENCY_SERVICE_ADDR`, `CART_SERVICE_ADDR`, `RECOMMENDATION_SERVICE_ADDR`, `SHIPPING_SERVICE_ADDR`, `CHECKOUT_SERVICE_ADDR`, `AD_SERVICE_ADDR` |
| 3 | `checkoutservice` depends on six services | Evidenced | `checkoutservice.yaml` env vars `PRODUCT_CATALOG_SERVICE_ADDR`, `SHIPPING_SERVICE_ADDR`, `PAYMENT_SERVICE_ADDR`, `EMAIL_SERVICE_ADDR`, `CURRENCY_SERVICE_ADDR`, `CART_SERVICE_ADDR` |
| 4 | Interfaces are defined once in a shared protobuf contract | Evidenced | `demo.proto` (`syntax = "proto3"`, `package hipstershop`) |
| 5 | Service-to-service calls use gRPC | Evidenced | `demo.proto` (`rpc` definitions); `checkoutservice.yaml` (`grpc:` probes, Service port name `grpc`) |
| 6 | The frontend is the only HTTP entry point | Evidenced for HTTP at `frontend`; "only" is **Inferred** | `frontend.yaml` (`httpGet /_healthz`, Service `frontend-external` of `type: LoadBalancer`) |
| 7 | Services are located by Kubernetes Service name and port | Evidenced | `checkoutservice.yaml` (`productcatalogservice:3550`); `Service` objects with matching names |
| 8 | Generated code is used for clients and servers | Inferred | `demo.proto` has `option go_package`, which shows Go output only; other languages are not evidenced |
| 9 | Services are written in different languages | Inferred | Not shown in the evidence (only image names appear) |
| 10 | Chosen for performance (compact, low overhead) | Inferred | No rationale is written in the evidence |
| 11 | Chosen for a strict, typed interface | Inferred | Consistent with `demo.proto`, but no rationale is written |
| 12 | REST and broker messaging were considered and rejected | Inferred | Alternatives are not in the evidence |
| 13 | Browsers cannot call gRPC directly, so the frontend exposes HTTP | Assumption | General industry knowledge about gRPC and browsers |
| 14 | REST has weaker contract enforcement than protobuf | Assumption | General industry knowledge |
| 15 | Synchronous calls couple checkout's availability to its dependencies | Inferred | The calls and addresses are evidenced (claim 3), but no timeout or retry settings are shown, and the effect is not demonstrated |
| 16 | Card number and CVV are carried in messages | Evidenced | `demo.proto` (`CreditCardInfo.credit_card_number`, `credit_card_cvv`) |
| 17 | The channel is or is not encrypted | Not evidenced in the provided files | No TLS settings appear in the manifests |
| 18 | Changing the shared proto can affect many callers | Assumption | General knowledge; the single shared file is evidenced (claim 4) |
