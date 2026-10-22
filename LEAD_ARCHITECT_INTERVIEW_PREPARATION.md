# Lead / Architect Interview Preparation

This guide uses the repository as a case study. Answers separate **what the code currently does** from **what I would recommend as a Lead/Architect**. In an interview, explain the current behavior honestly, state the trade-offs, and then describe a safe improvement path.

## 1. Architecture and API Design

### 1. Walk me through creating an order, from the HTTP endpoint to persistence and queue processing. Which layer owns each responsibility?

**Answer:** The authenticated `POST /api/Orders` endpoint in `Controllers/OrdersController.cs` reads the user ID from the JWT claims and calls `IOrderService.CreateOrderAsync`. `OrderService` loads that user's cart and products through `IOrderRepository`, checks stock, creates the order and order items, decrements stock, removes cart items, and saves through the Unit of Work. It then calls `IOrderQueueService`; `OrderQueueService` sends the order ID to SQS. The hosted `OrderQueueConsumerService` polls SQS, creates a DI scope for its database work, and changes a pending order to completed. Controllers own HTTP concerns, services own business workflow, repositories own queries, and the queue service/worker own messaging.

**Architect's caveat:** Persistence and queue publication are not atomic. A database save can succeed while SQS publication fails. I would address that using an outbox or another explicit reliability design.

### 2. What do you gain from the controller-service-repository structure? Where might it add unnecessary complexity?

**Answer:** It separates HTTP semantics, business rules, and persistence, which makes responsibilities easier to test and change independently. Interfaces also make it possible to substitute dependencies in unit tests. The trade-off is extra types and indirection; for trivial read-only endpoints, a repository abstraction that only forwards every `DbContext` call may add little value. I would keep the current pattern where it helps express business or query boundaries, rather than introducing abstractions mechanically.

### 3. How would you decide whether a business rule belongs in a controller, service, repository, or EF Core context?

**Answer:** The controller should validate request shape and translate application outcomes to HTTP responses. Business invariants—such as whether an order can be cancelled—belong in the service/domain workflow so they apply regardless of the caller. Repositories should encapsulate persistence-specific queries and updates, not product policy. `DbContext` configuration is appropriate for persistence mappings, constraints, and indexes. I would place a rule at the narrowest layer that guarantees it is consistently enforced.

### 4. Which order routes should be customer-only versus administrator-only? How would you enforce that distinction?

**Answer:** Creating, listing one's own orders, retrieving one's own order, and cancelling an eligible order are customer operations. Listing all orders, filtering all orders, and changing another user's order status are administrative capabilities. The current controller has class-level `[Authorize]`, which requires authentication but does not distinguish roles; the `[Authorize]` role claim is issued, but no role-based policy is evident on these operations. I would define named authorization policies or roles and apply them to administrative routes, then test both allowed and denied roles and verify object ownership for customer routes.

### 5. Which operations should return 201, 204, 400, 404, or 409? Would you change current responses?

**Answer:** Successful order creation should generally return `201 Created` with a `Location` pointing to the new order. A successful cancellation or status update could return `204 No Content` if no response body is needed, or `200 OK` with the updated resource. Malformed inputs are `400 Bad Request`; a genuinely missing order is `404 Not Found`; conflicts such as an invalid state transition, insufficient inventory, or duplicate idempotency key are often `409 Conflict`. The current create endpoint returns `200`, and controller exception handling often turns business failures into `400`; I would introduce consistent problem details and map domain outcomes to deliberate HTTP status codes.

### 6. How would you evolve the API without breaking existing clients?

**Answer:** Prefer additive changes first: optional fields, new routes, and new response fields that existing clients can ignore. For breaking changes, introduce an explicit API version, publish a deprecation window, track client usage, and provide migration documentation. I would avoid silently changing enum meanings or removing fields, and would contract-test important client/API interactions.

### 7. How should a team communicate the difference between a domain capability and an available API feature?

**Answer:** A model or service method only proves that code or a concept exists; it does not prove that clients can use it. I would document separately the public API routes, internal application capabilities, and integrations that are fully configured and operational. For example, auth service methods for profile or password changes exist, but the current `AuthController` does not expose routes for them. Feature claims should be traced through route, registration, implementation, and tests.

## 2. Authentication and Security

### 8. How does the JWT configuration validate tokens? Which settings must be consistent between issuance and validation?

**Answer:** `Program.cs` configures bearer authentication to validate issuer, audience, lifetime, and signing key, with a 30-second clock skew. `AuthService` issues an HMAC-SHA256-signed token using the configured key, issuer, and audience, and currently sets a seven-day expiry. Those issuer, audience, and key values must match between issuance and validation. I would also review token lifetime, signing algorithm/key management, claim mapping, and whether the application needs refresh tokens or key rotation.

### 9. `AuthService` hashes passwords with SHA-256. Why is that unsuitable, and how would you migrate?

**Answer:** SHA-256 is fast and unsalted in this implementation, making offline guessing of stolen hashes much cheaper and allowing identical passwords to have identical hashes. I would use ASP.NET Core Identity's password hasher or a well-reviewed adaptive password hash such as Argon2id, bcrypt, or PBKDF2 with a unique salt and appropriate work factor. For migration, verify an existing hash on login, then rehash the successfully authenticated password with the new scheme and store a versioned hash; force resets only if the old format or threat assessment requires it. I would add tests for correct/incorrect passwords and migration behavior.

### 10. What risks come from a JWT signing key in tracked configuration? How would you rotate keys?

**Answer:** Anyone who obtains the key could potentially mint valid tokens, and removing it from the latest file does not erase its Git history. I would move signing keys to a secret manager or protected deployment configuration and rotate the exposed key if it has been used. For a planned rotation, validation can temporarily accept the current and next keys, tokens can include a key ID, and the old key can be retired after the maximum token lifetime. Access and rotation should be auditable.

### 11. How would you prevent user enumeration in registration, login, and password recovery?

**Answer:** Use generic external messages where account existence is sensitive, keep response shapes and timing reasonably consistent, and rate-limit attempts. Registration may still need to communicate duplicate email to product users, but that decision should be explicit and balanced with abuse risk. Password recovery should always return a generic acknowledgement and send a time-limited, single-use recovery mechanism only if the account exists. Logs should capture outcomes without exposing secrets.

### 12. The password-recovery method uses account details to verify identity. What recovery process would you recommend?

**Answer:** A phone number or other static personal detail is not a strong recovery factor and can often be discovered or socially engineered. I would issue a random, high-entropy, single-use, short-lived reset token through a verified channel, store only a hash of it, and invalidate it after use or expiration. I would rate-limit requests, avoid account-existence disclosure, notify the account owner, and revoke or rotate existing sessions after a successful reset.

### 13. Which endpoints need role-based authorization, and how would you test it?

**Answer:** The all-orders listing/filtering and order-status update endpoints should normally be restricted to an administrative policy; own-order operations should be tied to the authenticated user ID. I would implement a named policy or role requirement and preserve ownership checks in the service. Integration tests should verify anonymous users receive `401`, authenticated customers are forbidden from admin operations with `403`, and administrators are allowed. Unit tests alone cannot fully prove the authorization middleware is wired correctly.

### 14. What should authentication failures return, and what belongs in logs?

**Answer:** Clients should receive generic messages that do not reveal whether an email exists or expose internal exception details. Logs can include a structured outcome, request/correlation ID, and a safely handled account identifier where justified, but never passwords, bearer tokens, signing keys, or recovery tokens. Access to authentication logs should be restricted and retention controlled. The current JWT error event returns exception details for authentication failures, which I would review and make less revealing.

### 15. How would you protect against brute force, credential stuffing, token theft, and replay?

**Answer:** I would add per-account and per-source rate limits, monitoring and progressive delays, breached-password checks, secure TLS, short-lived access tokens, and careful refresh-token rotation/revocation if refresh tokens are introduced. Secrets should not be logged or stored in browser-accessible locations unnecessarily. For high-risk actions, require recent reauthentication or step-up verification. I would also define incident response and alerting for anomalous login patterns.

### 16. What security checks are needed for `/metrics` and Prometheus/Grafana?

**Answer:** Metrics can reveal service topology, route names, resource use, and operational behavior. I would verify whether `/metrics` is accessible without authorization—in this repository it is mapped as an endpoint and should be treated as a separate operational surface—and restrict it by network, ingress policy, or dedicated authentication as appropriate. Prometheus and Grafana should not be exposed publicly by default. The checked-in Compose file uses `latest` images and default `admin` credentials, so it is for local development only until images are pinned, credentials are secret-managed, persistence/network exposure are addressed, and access is controlled.

## 3. EF Core, Persistence, and Transactions

### 17. How do Repository and Unit of Work work here? Are both needed with EF Core?

**Answer:** `GenericRepository<T>` provides basic CRUD operations, `OrderRepository` and `UserRepository` add use-case-specific queries, and `UnitOfWork` coordinates `SaveChangesAsync` through the shared `AppDbContext`. EF Core's `DbContext` already implements repository/unit-of-work-like behavior, so custom abstractions are justified when they clarify application boundaries, isolate complex queries, or improve tests—not just to wrap every method. I would assess duplication and test needs before expanding the abstraction.

### 18. What if database commit succeeds but SQS publication fails during order creation?

**Answer:** The order, stock decrement, and cart clearing may already be committed, while the message is absent, leaving a pending order that is never processed automatically. Retrying the whole request could create duplicate orders unless it is idempotent. I would use a transactional outbox: persist the order and an outbox event in the same database transaction, then have a dispatcher publish and mark the event delivered with retries. This avoids pretending that a SQL transaction and SQS send are one atomic transaction.

### 19. Compare outbox, transactional messaging, retries, and compensation.

**Answer:** An outbox stores the event alongside business data in one local transaction and asynchronously publishes it; it is a strong fit when using a relational database plus SQS. Retries handle transient failures but require idempotent consumers and bounded backoff. A distributed transaction is usually complex and not directly available across SQL Server and SQS in this design. Compensation reverses or offsets a completed action when a later step fails; it is useful for business workflows but does not eliminate the need to record and retry events reliably.

### 20. What concurrency problems can occur during order creation?

**Answer:** Two requests can read the same cart and create duplicate orders, or both can observe sufficient stock and oversell it. A stale read can also produce incorrect totals or inventory after another transaction changes the product. I would use idempotency keys for client retries, database transactions, optimistic concurrency tokens or atomic conditional stock updates, and appropriate isolation/constraints. Concurrency tests should exercise simultaneous requests against a real relational provider, not only mocks.

### 21. How would you prevent overselling?

**Answer:** I would make inventory decrement conditional and atomic, for example an update that subtracts quantity only when `StockQuantity >= requestedQuantity`, then verify the affected row count. A row-version concurrency token is another option that detects stale updates and lets the application retry or reject. For multi-item orders, all stock changes should be in a transaction so a failed line does not leave partial decrements. Constraints and clear conflict responses should reinforce the application logic.

### 22. What indexes and constraints would you consider?

**Answer:** The model already defines a unique index for user email and foreign-key relationships. I would consider indexes supporting order lookup by `(UserId, CreatedAt)` and status/time filtering, plus indexes for common order-item joins, then validate them against actual query plans and workload. I would not add indexes speculatively: every index costs storage and write performance. Constraints should enforce uniqueness and referential integrity; inventory non-negativity may need a database constraint or an atomic update strategy.

### 23. How would you evaluate `Include`/`ThenInclude` for order-query performance and exposure?

**Answer:** Includes are straightforward for returning orders with items and products, but large unbounded collections can increase payload, memory use, and query cost. Multiple collection includes may generate expensive joins or duplicate rows. I would inspect generated SQL and query plans, project directly to DTOs, use `AsNoTracking` for read-only queries, paginate, and consider split queries when appropriate. I would also ensure admin queries do not expose user or product data unnecessarily.

### 24. What migration strategy would you use for a production schema change across multiple API instances?

**Answer:** I would use an expand-and-contract approach: first add backward-compatible schema, deploy code that can work with old and new representations, backfill data safely, and only later remove obsolete schema. Migrations should be reviewed and applied through a controlled deployment process, not unexpectedly at every application startup. Rollback plans, backups, migration duration, locks, and multi-instance compatibility should be considered. I would never apply a migration to production from an unreviewed developer command.

## 4. Order Lifecycle and Inventory

### 25. What should happen to stock when an order is cancelled?

**Answer:** The current order service decrements stock on creation, but stock restoration on cancellation is commented out. The intended rule should be explicit: if stock was reserved, cancellation before fulfillment should release it exactly once; after fulfillment, a return workflow may be needed instead. I would persist state transitions and inventory events transactionally, make release idempotent, prevent cancellation after the relevant cutoff, and test repeated cancellation and concurrent processing.

### 26. Is it appropriate for an SQS consumer to mark every pending order completed after a delay?

**Answer:** Not if completion means payment captured, packed, shipped, or otherwise fulfilled. The current consumer marks a pending order completed after processing a delayed message, which is a demonstration workflow rather than a full commerce lifecycle. Completion should follow authoritative domain events and payment/fulfillment outcomes. I would model distinct states such as pending payment, confirmed, processing, shipped, completed, cancelled, and failed only as required by product rules.

### 27. How would you prevent invalid order-state transitions?

**Answer:** I would define an explicit transition map or domain method such as `TryTransitionTo`, instead of allowing arbitrary enum assignment throughout services. The method would validate the current and target state and return a meaningful result. Persistence should be updated in a transaction, and concurrency protection should prevent two actors from applying conflicting transitions. Tests should cover every allowed and rejected transition.

### 28. How would you handle client retries after an order request times out?

**Answer:** Support an idempotency key scoped to the user and operation. Persist the key, request fingerprint, and resulting order ID under a uniqueness constraint so a repeated request returns the original result rather than creating another order. Define expiration and behavior when the same key is reused with a different payload. The SQS consumer should also be idempotent because SQS delivery is at least once.

### 29. How would you handle partial failure across multiple line items?

**Answer:** Validate all items, then apply all inventory decrements and order writes in one database transaction; if any condition fails, roll back the whole order. If business operations must occur outside the database, use a saga/outbox and compensating actions, recording progress so retries are safe. The API should clearly distinguish unavailable items from transient infrastructure failures.

### 30. Would you store inventory reservations separately from product stock?

**Answer:** Separate reservations are useful when an order holds stock temporarily while payment or fulfillment is pending. A reservation can have an expiry and status, making release and reconciliation explicit. The trade-off is more state and background cleanup. For a simpler system, atomic decrement plus a reliable cancellation release may suffice, but the choice should match expected payment and fulfillment flows.

## 5. AWS SQS and Background Processing

### 31. Explain the current producer/consumer flow.

**Answer:** `OrderService` saves a pending order, then `OrderQueueService` sends a JSON message containing the order ID with a default delay of 300 seconds. `OrderQueueConsumerService`, registered as a hosted background service, long-polls SQS for up to 10 messages with a 20-second wait, creates a DI scope per message, loads the order, and changes it to completed if still pending. It deletes the message after its processing path. This flow needs reliability improvements before being treated as production-grade.

### 32. What delivery guarantees does SQS provide, and how should consumers behave?

**Answer:** Standard SQS queues provide at-least-once delivery, so duplicates are possible and consumers must be idempotent. A consumer should record or infer whether the event has already been applied, and repeated delivery should not double-decrement stock, send duplicate notifications, or repeat financial actions. FIFO queues can provide ordering/deduplication features within their constraints, but application-level idempotency remains important.

### 33. When should the consumer delete a message?

**Answer:** Only after the required business operation has completed successfully or the message has been deliberately and durably classified as unrecoverable. In the current code, deletion occurs even when deserialization yields null or the order is missing, so those cases are logged and discarded rather than retried or routed to a DLQ. I would distinguish successful processing, retryable failure, and poison message handling; acknowledge only after durable success or explicit dead-letter policy.

### 34. What should happen to malformed or permanently failing messages?

**Answer:** Transient failures should be retried with bounded backoff and a visibility timeout appropriate to processing duration. Repeated failures should move messages to a dead-letter queue through a configured redrive policy, with alarms and a replay/runbook process. Malformed messages should be logged with safe metadata and dead-lettered rather than retried forever or silently deleted. The current worker has a brief five-second receive-error delay but no explicit DLQ policy visible in the repository.

### 35. How would you size visibility timeout and control parallelism?

**Answer:** The visibility timeout should exceed normal high-percentile processing time, with extension/heartbeat if jobs can take longer. Parallelism should be bounded based on database capacity, downstream limits, and queue backlog rather than processing all received messages without control. Batch size, long polling, retry behavior, and shutdown cancellation should be tested under load. Queue age and timeout metrics help tune the settings.

### 36. Which dependencies should be scoped in the background worker?

**Answer:** A hosted service is effectively singleton, so it should not directly retain scoped services such as `AppDbContext`. The current worker correctly creates a DI scope for message processing and resolves the context from it. I would keep that pattern, pass the stopping token through I/O, dispose scopes promptly, and avoid resolving services from the root provider outside a scope.

### 37. How would you monitor queue health?

**Answer:** Monitor visible and in-flight message counts, oldest-message age, enqueue/dequeue rates, processing duration, retry count, DLQ depth, and failures. Add structured logs containing message ID, order ID, outcome, and trace/correlation context while avoiding sensitive payload logging. The current OpenTelemetry setup has general ASP.NET Core, HttpClient, and runtime metrics, but no explicit business or SQS metrics; I would add bounded-cardinality instruments and alert thresholds.

### 38. How would you test the queue flow without live AWS?

**Answer:** Unit-test producer request construction and consumer outcomes with a mocked `IAmazonSQS` and test database/repository abstractions. Add integration tests against LocalStack or an isolated test queue when practical to validate AWS-compatible behavior such as visibility, deletion, and redrive configuration. Keep tests deterministic by controlling time, message IDs, and failure injection, and never point automated tests at production queues.

## 6. Observability: Logs, Metrics, and Traces

### 39. What does the OpenTelemetry configuration measure and how is it exposed?

**Answer:** `Program.cs` configures ASP.NET Core instrumentation for incoming requests, HttpClient instrumentation for outgoing client requests, and runtime instrumentation for .NET runtime measurements. The Prometheus exporter is mapped at `/metrics`; Prometheus is configured to scrape on a 15-second interval, and the Compose file starts Prometheus and Grafana. The API also has correlation middleware that places correlation and active `Activity` identifiers in log scopes.

### 40. How do metrics, logs, and traces differ?

**Answer:** Metrics are aggregated numeric time series useful for rates, latency, saturation, and alerts. Logs are discrete, contextual events useful for understanding a particular failure or business outcome. Traces connect spans across components to show request causality and time spent. They complement one another: a metric can alert on elevated latency, a trace can locate the slow dependency, and logs can explain the specific event.

### 41. What does correlation/W3C context in logging provide, and what is missing?

**Answer:** It gives log entries identifiers that can help correlate a request, its active W3C `Activity`, and the ASP.NET Core connection. The named `CorrelatedClient` adds `X-Correlation-ID`, and HttpClientFactory diagnostics can propagate W3C context when that client is used. However, the repository does not configure an OpenTelemetry tracing pipeline or trace exporter/backend, and it has no outbound HTTP integrations currently using the named client. Logging IDs are not equivalent to exporting and querying distributed traces.

### 42. What would you add for trace export and context propagation over HTTP and SQS?

**Answer:** Configure `.WithTracing(...)` with ASP.NET Core and HttpClient instrumentation, add an OTLP exporter, and deploy/configure a collector and trace backend with sampling and resource attributes. Use W3C `traceparent` propagation for HTTP. For SQS, inject trace context into message attributes when publishing and extract it when consuming, then create producer/consumer spans with correct parent or link semantics. Verify end-to-end in an integration environment and protect sensitive attributes.

### 43. Which business metrics would you add, and how do you avoid high cardinality?

**Answer:** Counters could track order creation, cancellation, and processing outcomes; histograms could track order-processing duration and queue wait time; gauges can reflect current backlog if sourced appropriately. Labels should be bounded, such as outcome or payment method from a finite set, not order ID, user ID, email, or arbitrary exception text. High-cardinality identifiers belong in traces/logs, not metric dimensions.

### 44. How should logs correlate requests, orders, traces, and messages safely?

**Answer:** Use structured fields such as trace ID, span ID, correlation ID, order ID, SQS message ID, operation, and outcome. Avoid logging passwords, tokens, signing keys, payment credentials, or unnecessary personal information. Establish retention, access controls, redaction, and audit practices. Correlation identifiers should be validated or safely encoded before being returned or logged to prevent log injection and header abuse.

### 45. What dashboards and alerts would you create?

**Answer:** For the API: request rate, latency percentiles, status-code/error rate, and resource saturation. For SQS: queue depth, oldest-message age, processing latency, failures, and DLQ depth. For the database: connection pressure, query latency, timeouts, and pool exhaustion. Alerts should use actionable thresholds, ownership, and runbooks, and should avoid paging on transient noise. Grafana currently has no provisioned data source or dashboards in this repository.

## 7. CORS, Configuration, and Deployment

### 46. What does the current CORS policy allow, and how would you make it environment-specific?

**Answer:** The policy allows origins `http://localhost:3000` and `https://your-frontend-app.com`, any request header, and any HTTP method; it does not call `AllowCredentials`. The HTTPS origin is an example placeholder and should be replaced with a trusted deployed origin. I would bind allowed origins from environment-specific configuration, validate the configured values at startup, and keep development origins out of production policy.

### 47. Why is CORS not authentication or authorization?

**Answer:** CORS is a browser-enforced policy that controls whether JavaScript from an origin may read a cross-origin response. It does not stop direct HTTP clients, scripts, or attackers from sending requests, and it does not establish user identity or permissions. Authentication and authorization must be independently enforced by the API.

### 48. What would you check if Prometheus cannot scrape the API?

**Answer:** The checked-in `prometheus.yml` targets `host.docker.internal:7204` without specifying HTTPS; Prometheus therefore normally attempts HTTP, while the launch profile uses HTTPS on 7204 and HTTP on 5190. I would align target port and scheme with the profile, verify the API is listening on a host-accessible interface, ensure Docker can resolve `host.docker.internal`, and inspect Prometheus target health and logs. For HTTPS, configure certificate trust correctly rather than disabling TLS verification casually.

### 49. What should change in the local Prometheus/Grafana Compose setup before production?

**Answer:** Pin image versions instead of using `latest`, replace default Grafana credentials with managed secrets, persist required data, restrict network/host port exposure, and use TLS/authentication at the appropriate boundary. Provision the Prometheus data source and dashboards as code, and monitor the monitoring stack itself. The checked-in Compose file only starts Prometheus and Grafana; it does not start the API or provision dashboards/data sources.

### 50. How would you separate environment configuration and secrets?

**Answer:** Keep nonsecret defaults in configuration and inject environment-specific values via deployment configuration. Use User Secrets locally and a managed secret provider in deployed environments for JWT keys, database credentials, and other secrets. Validate required settings at startup, use least privilege, rotate secrets, and ensure deployment logs do not print them. Avoid checking machine-specific settings or credentials into Git.

## 8. Testing and Quality Strategy

### 51. What should be unit-tested and what needs integration tests?

**Answer:** Unit-test controller response mapping, service business rules, order transitions, producer request construction, consumer idempotency/failure paths, and correlation middleware/handler behavior. Integration-test routing, authentication and authorization middleware, EF mappings/transactions, migrations, and actual endpoint behavior with a relational database. SQS integration can use LocalStack or a dedicated test queue. The current suite includes controller, service, SQS, and middleware tests, but test existence alone does not prove production integrations work.

### 52. How would you test order-creation failure cases?

**Answer:** Unit tests should cover empty cart, invalid product, insufficient stock, correct totals, stock changes, cart removal, persistence failure, and queue failure. Integration tests should verify database transaction behavior and actual constraints. Queue failure after database commit deserves a test that demonstrates the current pending-order gap and later validates an outbox implementation. Concurrency and idempotency require simultaneous requests and a database capable of realistic transaction semantics.

### 53. How would you test JWT authorization and user isolation?

**Answer:** Create integration tests with a test authentication scheme or real test-issued JWTs, then call the HTTP pipeline. Verify anonymous access is rejected, customer tokens can access only their own orders, another user's order is not disclosed, and admin routes are forbidden to customers but allowed to authorized administrators. Include claim edge cases and invalid/expired tokens. Controller unit tests with a manually constructed principal are useful but do not validate JWT configuration or middleware ordering.

### 54. What are the benefits and limits of Moq-based tests?

**Answer:** Mocks make tests fast and isolate decision logic from the database and AWS, allowing precise failure-path simulation. They can also encode implementation details, miss EF query/mapping issues, and falsely pass when dependency wiring or middleware is wrong. I would balance them with integration tests against a relational provider, HTTP pipeline tests, and a small number of AWS-compatible environment tests.

### 55. What should CI run?

**Answer:** Restore dependencies, build the solution with warnings reviewed, run unit and integration tests, validate formatting/analyzers, scan dependencies and secrets, and validate container/Compose configuration. Pull requests should also check EF migration consistency and API contract changes. Deployment stages should run smoke tests, health checks, migration safety checks, and telemetry verification before production traffic is shifted.

### 56. How do you measure meaningful test coverage?

**Answer:** Focus on critical behavior and failure boundaries, not only line percentage. Use coverage to identify untested code, but assess whether tests assert business outcomes, authorization boundaries, transactional behavior, retries, idempotency, and observability. Mutation testing or fault injection can help determine whether assertions detect regressions. Track escaped defects and test reliability as well.

## 9. Leadership and Architecture Decisions

### 57. What would you address first as Lead/Architect, and in what order?

**Answer:** I would first assess exposure and correctness: move/rotate the tracked JWT key, replace SHA-256 password hashing, and enforce role-based access to admin operations. Next I would address order reliability—idempotency, transaction boundaries, outbox, inventory concurrency, and SQS poison-message handling—because these affect money and stock. Then I would improve API coverage and contracts, operational dashboards/alerts, trace export, and production deployment hardening. I would sequence work with product risk, measurable acceptance criteria, and backward-compatible releases.

### 58. How do you distinguish “implemented in a service,” “exposed as an endpoint,” and “production-ready”?

**Answer:** A service method is internal capability; an endpoint additionally needs routing, request/response contract, authentication/authorization, validation, and documentation. Production-ready means the behavior also has reliable persistence and retries, security controls, observability, operational ownership, tested deployment, and failure/recovery procedures. For example, an auth service method without a controller route is not an available client API, and a configured metrics exporter alone is not a complete observability platform.

### 59. What review standards would you set for migrations, authentication, queues, and observability?

**Answer:** Require explicit risk analysis, backward compatibility, tests, operational impact, and rollback/recovery notes. Migrations need SQL/schema review and data-volume/locking consideration. Authentication changes need threat-model review and secret-handling checks. Queue changes need idempotency, retry/DLQ semantics, and failure tests. Observability changes need cardinality/privacy review, sampling or cost expectations, and proof that metrics/traces are actually exported and queryable.

### 60. What would you capture in architecture decision records?

**Answer:** Record decisions with context, options considered, chosen approach, consequences, security/reliability trade-offs, and conditions for revisiting. Useful ADRs here could cover the outbox and SQS delivery model, password hashing migration, token/key rotation, order-state lifecycle, admin authorization, observability/export destinations, and API versioning. Keep ADRs short, linked from relevant code/docs, and supersede rather than silently rewriting historic decisions.

### 61. How would you roadmap missing product/cart/payment/email/recommendation features?

**Answer:** Start with product requirements and domain boundaries, then deliver product/category read and write APIs, cart operations, inventory reservation, and idempotent order placement as a coherent commerce flow. Add payment through a provider adapter with signed webhook verification, idempotent event processing, and explicit payment/order state transitions. Add email through an outbox-backed notification service with retries and templates. Recommendations should follow only after defining consent, data quality, measurable outcomes, and privacy policy. Deliver incrementally behind feature flags or compatible API versions with tests and operational metrics.

### 62. Describe a production incident involving delayed or duplicate orders. What do you inspect first?

**Answer:** I would establish customer impact and stop further harm first, then inspect API errors/latency, database health, SQS queue depth and oldest-message age, in-flight messages, DLQ, and recent deployments/config changes. Correlate order IDs, message IDs, timestamps, and trace/log context to determine whether the order was never enqueued, repeatedly delivered, stuck in processing, or incorrectly marked complete. Apply a safe mitigation—such as pausing a consumer or disabling a faulty path—without deleting evidence. After recovery, reconcile orders and inventory, communicate status, and complete a blameless review with corrective actions for idempotency, alerts, runbooks, and testing.

## Interview Practice

- Answer with the **current implementation first**, then clearly label recommendations as future improvements.
- Use one concrete example from the repository per answer; avoid claiming features that are only models, unexposed service methods, or placeholders.
- For architecture questions, cover correctness, security, reliability, operational impact, alternatives, and how you would validate the decision.
- For leadership questions, explain sequencing, ownership, stakeholder communication, measurable outcomes, and how you would reduce delivery risk.
