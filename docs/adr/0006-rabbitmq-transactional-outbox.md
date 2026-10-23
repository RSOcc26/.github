# 6. RabbitMQ with a transactional outbox for events

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

The work-order flow is naturally event-driven: releasing a work order must
start production and reserve material without Workflow knowing about either.
Publishing to a broker after a database commit can lose events (crash between
the two). Publishing before the commit can announce changes that never
happened.

## Decision

- **RabbitMQ** is the message broker: one durable topic exchange
  `companycentral.events`, routing key = event type, one durable queue per
  consuming service and purpose, a dead-letter exchange for messages that keep
  failing.
- **Transactional outbox:** the producer inserts the event into its `outbox`
  table in the same transaction as the state change. A relay in the same
  service publishes pending rows with publisher confirms and marks them sent.
- **Idempotent consumers:** consumers store processed event IDs in an `inbox`
  table, in the same transaction as their own change.
- Events use the common envelope from `docs/architecture.md` and carry a
  `version`. Incompatible payload changes create a new version.
- Kafka and event sourcing are not used in the MVP.

## Consequences

- Delivery is at-least-once. Ordering is guaranteed only per queue, so consumers
  must tolerate duplicates and late events.
- Every service that publishes needs an outbox table and a relay; this is
  shared boilerplate worth extracting into a small internal library later.
