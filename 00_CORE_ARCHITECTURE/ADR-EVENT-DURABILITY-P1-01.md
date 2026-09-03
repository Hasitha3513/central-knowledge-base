# ADR: Minimal Database Outbox for Durable Internal Events

- **Status:** Accepted
- **Date:** 2026-09-03
- **Decision:** P1-01 Event Contract Durability and Envelope Hardening

## Context

Source inspection found 32 event classes or persisted event records, but only ten have real consumers. Nine consumed events perform local, idempotent after-commit work. Delivery's five-type US-69 customer-notification family crosses into Notification and losing that handoff after a successful Delivery commit would lose an accepted customer-notification fact.

## Decision

Use one shared technical PostgreSQL outbox for only events implementing the canonical durable envelope. Flyway V60 creates `integration_outbox_event`; the shared boundary is its sole owner. Delivery publishes through `DurableEventPublisher` within the originating transaction. A Tenant-aware bounded worker claims rows after commit and invokes explicit version-aware handlers outside that transaction.

The guarantee is at-least-once, not exactly-once. `(tenant_id,event_id,consumer_name)` prevents duplicate producer persistence, while Notification's existing Tenant-scoped execution key deduplicates consumer side effects. Claims are bounded to 50 per Tenant, have a five-minute lease, and stop after five attempts. Unsupported versions are parked as `UNSUPPORTED`; exhausted transient failures become `FAILED`. No global ordering is promised.

## Why PostgreSQL and Not a Broker

The application is a Spring Modulith modular monolith and has no approved external broker requirement. PostgreSQL provides atomic business-write/outbox-write semantics with the existing operational platform and avoids speculative Kafka/RabbitMQ infrastructure. A future external integration may consume the stable envelope behind a new approved adapter without changing business-module ownership.

## Security, Privacy, and Retention

Tenant identity comes from trusted aggregate/execution context and is reconstructed explicitly for background work. Payloads are deterministic, allow-listed, and capped at 32 KiB; raw addresses/contact data, credentials, provider secrets, POD evidence, Rider private/medical data, and all US-70 tokens, hashes, magic links, and access codes are prohibited. Published rows are retained at least 30 days and failed/unsupported rows at least 90 days; purge requires a separately approved Tenant-qualified maintenance action.

## Consequences

- US-69 semantics are unchanged while its module handoff becomes crash-recoverable.
- Local ETA invalidation and `OperationalNotificationEvent` remain local after-commit.
- The 22 unconsumed events are not rewritten merely for consistency.
- Consumer failures cannot roll back committed source business state.
- Operations must monitor retry and terminal rows; replay remains possible and consumers must remain idempotent.
