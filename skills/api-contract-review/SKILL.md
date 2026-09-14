---
name: api-contract-review
description: Reviews a REST or GraphQL interface change for backward compatibility and design problems before it ships. Detects removed or renamed fields, narrowed types, changed nullability, removed enum values, altered error shapes, and pagination that breaks under concurrent writes, then gives a compatible migration path. Use when adding or changing an endpoint, editing an OpenAPI or GraphQL schema, or deciding whether a change needs a version bump. Do not use for internal function signatures, database schema review, or client-side code.
---

# API Contract Review

An API contract is a promise to code you do not control and cannot deploy. This
skill checks whether a change keeps that promise.

## When to use this skill

- Adding or modifying an endpoint, resolver, or response shape
- Reviewing a diff to an OpenAPI spec or GraphQL schema
- Deciding whether a change requires a version bump or a deprecation window

## When not to use it

- Internal function signatures within one deployable — refactor freely
- Database schema changes — use a migration-focused review instead
- Client code that consumes an API rather than defines one

## Inputs

- The schema or route diff
- Whether the API is public, partner-facing, or internal to one org
- Whether clients are controlled by you (web app you deploy) or not (mobile
  apps, third-party integrations). This changes everything: you cannot force a
  mobile client to upgrade.

## Breaking versus non-breaking

| Change | Breaking? |
|---|---|
| Adding an optional request field | No |
| Adding a response field | No, unless clients validate strictly |
| Making an optional request field required | **Yes** |
| Removing or renaming a response field | **Yes** |
| Narrowing a type (`string` to `enum`) | **Yes** |
| Widening a type (`enum` to `string`) | **Yes** for clients that switch on it |
| Making a non-null field nullable | **Yes** |
| Removing an enum value | **Yes** |
| Adding an enum value | **Yes** for clients with exhaustive matching |
| Changing an error code or shape | **Yes** |
| Changing default sort or page size | **Yes** in practice |

"Adding is safe, removing is breaking" is the rough rule, but the nullability
and enum rows are where teams get caught.

## Procedure

1. **Classify every field-level change** against the table above.
2. **Check error shapes.** Error responses are part of the contract. A client
   branching on `error.code` breaks when you rename the code, even though the
   happy path is untouched.
3. **Check pagination.** Offset pagination skips and duplicates rows when the
   underlying set changes between pages. For anything user-visible and mutable,
   cursor pagination is correct.
4. **Check nullability direction.** Making a field nullable is breaking for
   readers; making it non-null is breaking for writers. Name which side.
5. **Check list responses** are wrapped in an object, not returned as a bare
   array — a bare array cannot gain pagination metadata later without breaking.
6. **Propose the compatible path** for anything breaking: add alongside,
   dual-serve, deprecate with a header and a date, remove in a later release.

## Output contract

```
### Verdict
COMPATIBLE | BREAKING | BREAKING_BUT_ACCEPTABLE

### Breaking changes
- <field or endpoint> — <what breaks, for which clients>
  Migration: <the additive path>

### Design notes
- <non-breaking issues worth fixing now, while the shape is still new>

### Required before merge
- [ ] <version bump, deprecation header, changelog entry, client audit>
```

`BREAKING_BUT_ACCEPTABLE` applies only when every client is known and
deployable by the same team. State that assumption explicitly.

## Worked example

**Diff:**

```diff
 type Order {
   id: ID!
-  status: String!
+  status: OrderStatus!
-  total: Float!
+  totalCents: Int!
 }
```

**Expected output:**

```
### Verdict
BREAKING

### Breaking changes
- `Order.status` — narrowed from String to the OrderStatus enum. Any client
  sending or receiving a value outside the enum now errors. Existing rows with
  legacy statuses will fail serialisation entirely.
  Migration: keep String, document the accepted values, and validate on write.
  Introduce the enum as a parallel field `statusV2` if strict typing matters.
- `Order.total` renamed to `totalCents` and retyped Float to Int. Every client
  reading `total` gets null; every client that upgrades without noticing the
  unit change displays prices 100x wrong.
  Migration: add `totalCents` alongside `total`, mark `total` deprecated with a
  removal date, remove after client telemetry shows zero reads.

### Design notes
- Representing money as Float was the original defect; Int cents is correct.
  The unit change in the same step as the rename is what makes this dangerous —
  a silent 100x error is worse than a null.

### Required before merge
- [ ] Ship `totalCents` additively; do not remove `total` in this release
- [ ] Add @deprecated with a removal date to `total`
- [ ] Confirm no mobile client older than the deprecation window reads `total`
```

## Common mistakes

- Treating field addition as always safe. Clients using strict schema
  validation reject unknown fields.
- Forgetting that adding an enum value breaks exhaustive `switch` in typed
  clients.
- Versioning the whole API for one field change when an additive field would
  have done.
- Deprecating without telemetry, so the removal date arrives with no evidence
  anyone stopped using it.
