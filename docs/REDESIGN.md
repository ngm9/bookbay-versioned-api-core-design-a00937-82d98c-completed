# Bookbay API core redesign

The full design (diagram + sections) is on the Utkrusht whiteboard. This note captures the reasoning.

## Target module structure
```
app/
  api/legacy/    compatibility adapters: frozen legacy URLs, status codes and shapes
  api/v2/        thin routers per resource (users, listings, orders, cancellations, refunds)
  contracts/     Pydantic v2 DTOs, Page[T], Problem (single source for the frozen OpenAPI)
  services/      use cases (OrderService, ListingService, RefundService); depend on ports only
  domain/        entities, OrderStatus state machine, DomainError hierarchy (no FastAPI/ORM)
  policies/      buyer / seller / admin authorization rules
  ports/         Protocols: repositories, UnitOfWork, IdempotencyStore, Outbox, Clock
  adapters/persistence/  SQLAlchemy repositories + SqlUnitOfWork (one tx per command)
  errors/        DomainError -> application/problem+json with a stable code
  notifications/ outbox relay (after commit)
tests/          in-memory fakes for every port; contract + service tests need no DB
```
Dependency rule: api -> contracts/services -> domain/ports <- adapters. Nothing inside services imports FastAPI or SQLAlchemy.

## Contract decisions
- **Versioning:** URI major version `/v2` for partners. Legacy paths keep working unchanged for the six-month window, with `Deprecation` and `Sunset` headers. `/v2` evolves additively; a breaking change means `/v3`.
- **Collections:** one convention everywhere: `?limit=` (default 50, max 100), opaque `cursor`, `sort=-created_at`, explicit filters. Response `{data: [...], page: {next_cursor, limit}}`; `next_cursor: null` means end of list.
- **Errors:** RFC 9457 problem details `{type, title, status, code, detail, instance, errors[]}`. `code` is stable and machine-readable; 500s never leak internals.
- **Checkout as a resource:** `POST /v2/orders` with required `Idempotency-Key`, body `{cart_id, cart_version}` -> `201` + `Location`. Cancellations and refunds are sub-resources (`POST /v2/orders/{id}/cancellations`, `/refunds`), each idempotent; illegal transitions -> `409 ORDER_INVALID_TRANSITION`.

## No second order on retry
1. `idempotency_keys` row (PK `principal_id, endpoint, key`) is inserted **in the same transaction** as the order, together with the request hash and the final response.
2. Retry with the same key -> replay the stored status + body (`Idempotent-Replayed: true`).
3. Concurrent request with the same key -> PK conflict -> `409 IDEMPOTENCY_IN_PROGRESS` + `Retry-After`.
4. Same key, different body -> `422 IDEMPOTENCY_KEY_REUSED`.
5. Backstop for legacy clients that send no key: `UNIQUE(buyer_id, cart_id, cart_version)` on `orders`; the legacy adapter turns the conflict into a replay of the existing order.
6. Crash mid-transaction rolls back the order and the key together, so a retry is safe. Keys are kept at least 7 days.

## Trade-offs
- Postgres-backed idempotency instead of Redis: atomic with the order, no new platform (budget constraint), survives restarts; costs one extra row per write (~15 writes/s at peak).
- Compatibility adapters instead of a second codebase: one service layer, so fixes such as idempotency reach the mobile app too.
- Cursor over offset pagination: stable under inserts and index-backed for the 250 ms p95 read target; cannot jump to page N.
- Transactional outbox instead of BackgroundTasks: notifications survive crashes and stay off the 700 ms p95 order path.
- Ports/Protocols add a little indirection but make validation, serialization, services and error mapping testable without a database or network.

## Migration plan (strangler)
1. Introduce contracts, errors and services; move one route at a time behind them.
2. Re-point each legacy route to the new service through its adapter (contract tests pin the legacy shape).
3. Ship `/v2` and freeze the OpenAPI document (stable `operation_id`s) for the partner client.
4. Monitor legacy traffic; remove `api/legacy/` at sunset (6 months).
