# Order Placement Idempotency Plan

## Goal

Make retries of `POST /api/Orders` safe: repeated requests using the same idempotency key for the same authenticated user must return the same order result rather than create another order, decrement stock again, or enqueue duplicate work.

This document is a plan only. No application code or database schema has been changed.

## Current Behavior and Gap

- `OrdersController.CreateOrder` is authenticated, accepts no request body or idempotency key, and returns a generic success message after calling the service.
- `OrderService.CreateOrderAsync` reads the user's cart, decrements product stock, creates the order, removes cart items, and commits through `IUnitOfWork`.
- SQS publication occurs after the database save. If publication fails, the order may already exist while the request fails.
- No idempotency key, request-deduplication record, unique order request constraint, or transactional outbox was found.
- The SQS consumer checks that an order is still pending before completing it. This is a useful state guard but is not a general idempotency mechanism and is not sufficient to protect concurrent duplicate processing.

## Recommended Behavior

1. Require an `Idempotency-Key` request header on order placement. The client generates a high-entropy key for one logical placement attempt and reuses it for retries of that attempt.
2. Scope a key to the authenticated user and operation. One user's key must never deduplicate another user's request.
3. The first accepted request creates exactly one order and stores a durable record associating `(UserId, Operation, IdempotencyKey)` with that order.
4. A retry with the same key returns the original order/result and does not reread or recreate from the now-cleared cart, decrement stock again, or create another queue event.
5. Reusing a key for a different logical placement must not silently create or return a different order. Return a documented conflict response. With the current bodyless endpoint, the key identifies the entire placement attempt; clients must generate a new key for a new attempt. If a checkout request body is introduced later, persist a canonical request fingerprint and compare it on retries.
6. Concurrent requests with the same key must result in one order. The database, not an in-memory lock, is the authority for uniqueness.

## Proposed Persistence Design

Add an `OrderIdempotencyRecord` (or equivalently named) table with fields such as:

- `Id`
- `UserId`
- `Operation` (for example, `CreateOrder`)
- `IdempotencyKey` (bounded length; never log the raw key)
- `RequestFingerprint` (nullable for the existing bodyless endpoint; populated if a request body is added)
- `OrderId`
- `State` (for example, `Completed`; an externally visible `Processing` state is optional)
- `ResponseStatusCode` and, if the API needs exact replay, a stored response representation
- `CreatedAt` and `ExpiresAt`

Add a unique database index on `(UserId, Operation, IdempotencyKey)`. Add a foreign key from the record to the resulting order when compatible with the chosen lifecycle. Decide explicitly whether key records expire, how long they remain, and how cleanup avoids allowing a retry to create a second order after a record is removed.

An alternative is to put the unique key directly on `Order`. That is simpler if an idempotency record has no independent lifecycle or response state. A separate record is more flexible for concurrent in-progress requests, response replay, expiration, and future checkout payload validation.

## Request Flow

1. Validate the header: reject missing, blank, malformed, or overlong values with `400 Bad Request`. Document a key format and length limit.
2. Derive the authenticated `UserId` from the validated JWT principal. Never trust a user ID supplied by the client for the deduplication scope.
3. Look up the idempotency record by user, operation, and key **before reading the cart**:
   - `Completed`: return the original order/result.
   - `Processing`: only applicable if a reservation was deliberately committed separately; return the documented in-progress response in that design.
   - Existing record with a different request fingerprint, when request-body support exists: return `409 Conflict`.
4. For a first request, perform the idempotency reservation, order creation, inventory changes, cart clearing, and outbox-event creation in one database transaction.
5. Commit once all durable records are consistent. The unique index resolves races when requests with the same key arrive simultaneously.
6. Return a stable response, preferably `201 Created` with the created order location/body. Replays return the documented original result (or a clearly defined replay response).
7. If a duplicate-key insert loses a race, roll back that attempt, reload the winning idempotency record, and return its result. Do not continue order creation after losing the reservation.

The transaction must not contain a remote SQS call. SQS is not part of the SQL transaction and holding database locks while waiting for the network would not make the operations atomic.

With the recommended single database transaction, another request generally cannot observe an uncommitted `Processing` row: it will wait or hit the unique constraint, then reload the winner's committed result. If the API must return an in-progress response while the first attempt is still running, the reservation must be committed separately and protected by a lease/timeout plus recovery for abandoned reservations. That adds operational complexity and should be a deliberate choice rather than an assumed behavior.

## Reliable SQS Publication

Use a transactional outbox to close the current gap between saving an order and publishing its SQS message:

1. Add an outbox row for the order-processing event inside the same database transaction as the order, stock changes, cart removal, and idempotency record.
2. Add a hosted dispatcher that reads undelivered outbox rows, publishes them to SQS, and marks them delivered only after a successful send.
3. Make dispatch retries safe. An event may be delivered more than once if the process crashes after SQS accepts it but before the database marks it delivered.
4. Make the consumer idempotent using durable event/order processing state and atomic state transitions. A conditional database update or concurrency token should prevent two workers from both applying the same transition.
5. Configure a queue redrive policy and dead-letter queue for poison messages; do not silently delete messages that failed deserialization or required processing.

If an outbox is deliberately deferred, document that order creation can succeed while queue publication fails and provide a reconciliation/retry process. This is a weaker reliability guarantee and should not be described as end-to-end exactly-once processing.

## API Contract Decisions

- Header: `Idempotency-Key`.
- Scope: authenticated user plus operation.
- Retention: choose a period aligned with client retry behavior and order lifecycle; retain records long enough that delayed retries cannot create duplicate orders.
- Replay response: choose between returning the exact original response or returning the current order resource. Keep the result stable and document it.
- In-progress response: define status code, response body, and client retry guidance.
- Key conflict: return `409 Conflict` if a key is reused with a different request fingerprint.
- Creation response: prefer `201 Created` with a resource location and order representation.
- New checkout attempts: use a new key. Reusing a key intentionally means “retry/replay this same attempt.”
- Logging: log outcome and a safe correlation value; do not log raw idempotency keys, tokens, or sensitive checkout payloads.

## Implementation Phases

### Phase 1: Contract and persistence

- Agree on header format, key length, retention, replay response, in-progress response, and whether a request fingerprint is needed.
- Add the persistence entity, EF mapping, unique index, migration, and repository/unit-of-work operations.
- Add concurrency-safe reservation and replay behavior.

### Phase 2: Atomic order creation

- Update the controller contract and `IOrderService` to accept the idempotency key.
- Check the key before reading the cart.
- Persist the idempotency result with the order and inventory/cart updates in one database transaction.
- Return a stable order response for both first requests and retries.
- Add a client update to generate, persist for retry, and reuse the same key for the same logical attempt.

### Phase 3: Outbox and consumer safety

- Persist the order-processing event in the transaction.
- Add an outbox dispatcher and SQS publishing retry behavior.
- Make the consumer transition atomic/idempotent and add DLQ/redrive configuration.
- Propagate trace context and correlation identifiers through queue message attributes if end-to-end tracing is required.

### Phase 4: Rollout and operations

- Deploy the additive schema before application code that depends on it.
- Monitor duplicate-key conflicts, replay counts, in-progress records, outbox age/failures, queue age, and DLQ depth.
- Define record cleanup, outbox recovery, message replay, and order reconciliation runbooks.
- Keep the old behavior only behind a deliberate compatibility strategy; do not allow unkeyed and keyed requests to bypass deduplication inconsistently.

## Test Plan

### Unit tests

- Missing, blank, and invalid idempotency keys are rejected.
- A first request creates an order and stores one successful idempotency result.
- A replay returns the same order and does not load/consume the cart again or decrement stock again.
- A different user can use the same key without colliding.
- A different payload with the same key returns conflict if request-body/fingerprint support is introduced.
- Existing `Processing` records produce the documented in-progress result.
- SQS publication failure does not lose the durable order event when outbox is implemented.
- Duplicate consumer deliveries do not apply the business transition more than once.

### Database/integration tests

- The unique index prevents two concurrent requests for the same user/operation/key from creating two orders.
- Competing requests produce one order, one stock decrement, one cart consumption, and a consistent replay response.
- Rollback leaves no partial order, stock update, cart deletion, idempotency record, or outbox row.
- A different key creates a separate order when the cart contains a new placement.
- Verify relational transaction behavior with SQL Server or a compatible relational test environment; EF Core's in-memory provider is not sufficient for validating unique constraints and relational transactions.

## Acceptance Criteria

- Repeating the same placement request with the same key returns the same logical order.
- Concurrent same-key requests create exactly one order and apply inventory/cart changes once.
- A new logical placement with a new key is not deduplicated against a prior placement.
- Database uniqueness is enforced by a migration and not just application-level checks.
- Order persistence and event publication have a documented reliability guarantee; with the outbox, the event is durably retried until published or explicitly dead-lettered.
- Tests cover replay, concurrency, failures, cross-user key scope, and consumer duplicate delivery.
- API documentation describes key generation, reuse, retention, and client retry behavior.

## Risks and Decisions to Resolve Before Implementation

- Does the API need to preserve the exact original response, or is returning the current order representation sufficient?
- Should clients be required to send an idempotency key immediately, or should a short transition period support legacy clients?
- What retry window and record retention period do clients need?
- Is checkout going to gain a request body (shipping/payment details)? If so, define canonicalization and fingerprint behavior before implementation.
- Should the outbox be included in the same delivery or split into a separate prerequisite? The recommended production-safe path includes it because the current order-to-SQS boundary is not atomic.
