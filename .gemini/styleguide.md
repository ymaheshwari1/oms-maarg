# HotWax OMS Code Review Style Guide

Use this guide when reviewing pull requests for `hotwax/oms`. Prioritize
comments that protect service contracts, REST API consistency, data model
correctness, operational performance, and HotWax OMS domain semantics.

This guide is based on repeated review feedback from Deepak Dixit and Anil K
Patel in this repository. Source IDs in brackets point to the review comments in
the source index at the bottom of this file.

## Review Priorities

- Prefer actionable comments over broad suggestions. Explain the concrete risk,
  expected pattern, and the smallest useful correction.
- Flag correctness, API contract, performance, data placement, and concurrency
  issues before style or formatting issues.
- Do not request cosmetic rewrites unless the current code makes the service
  contract, generated documentation, or operational behavior unclear.

## Moqui Service Contracts

- Service names should follow the existing Moqui verb/noun convention, such as
  `get#SalesOrder`, `cancel#SalesOrderItem`, or `create#ProductUpdateHistory`.
  Flag vague names, unnecessary words like `All`, or nouns that do not describe
  the service result. [A240-NAME](https://github.com/hotwax/oms/pull/240#discussion_r2236143637) [D240-NAME](https://github.com/hotwax/oms/pull/240#discussion_r2253678623) [D220-NOUN](https://github.com/hotwax/oms/pull/220#discussion_r2219314008)
- Do not introduce arbitrary prefixes or suffixes in service names. If an
  existing field such as `productTypeId` can identify the record type, prefer
  that over encoding type information into generated IDs or names. [D220-TYPE](https://github.com/hotwax/oms/pull/220#discussion_r2219341980)
- Do not remove existing in-parameters when they may still be used by callers.
  If a new consolidated parameter such as `keyword` is added, preserve explicit
  filters like `orderId` and `orderName` unless the PR proves they are obsolete.
  [D343-PARAMS](https://github.com/hotwax/oms/pull/343#discussion_r2583622480)
- Define explicit out-parameters for public services. Avoid abstract or loosely
  typed output maps when the service is exposed through REST or Swagger
  documentation. [D240-OUT](https://github.com/hotwax/oms/pull/240#discussion_r2253682259)
- Default values that change business behavior should be caller-provided
  in-parameters. Do not silently default important values such as rejection
  reasons inside the service. [D240-REASON](https://github.com/hotwax/oms/pull/240#discussion_r2284078324)

## REST API Shape

- Keep REST resources aligned with the domain model. A get endpoint for an order
  should live under the order resource and return a consistent order detail
  shape. [A240-ORDER-DETAIL](https://github.com/hotwax/oms/pull/240#discussion_r2236181929) [D240-ONE-GET](https://github.com/hotwax/oms/pull/240#discussion_r2253747644)
- Avoid creating multiple endpoints for the same retrieval or mutation concept.
  Prefer one clear resource path, a master relationship, or an explicit service
  that returns the full detail shape. [D240-ONE-GET](https://github.com/hotwax/oms/pull/240#discussion_r2253747644)
- Review new resource names for REST semantics. Generic names such as `details`
  should be challenged when the same meaning can be represented by the parent
  resource or an existing subresource. [A240-REST-NAME](https://github.com/hotwax/oms/pull/240#discussion_r2236277226)
- If a flow needs a separate API surface, put it in the appropriate REST file.
  For example, BOPIS-specific flows should not be mixed into generic OMS REST
  resources without a clear reason. [D240-BOPIS-REST](https://github.com/hotwax/oms/pull/240#discussion_r2253752233)
- Bulk or multi-item actions should use the existing service multiple pattern
  when appropriate instead of inventing a new endpoint that duplicates item-level
  operations. [D240-MULTI](https://github.com/hotwax/oms/pull/240#discussion_r2284099493)

## Domain Semantics

- Verify that entity fields mean what the code assumes. For example,
  `OrderItemShipGroup.facilityId` is the facility where the order is brokered or
  fulfilled, and it is not always the origin facility.
  [D240-FACILITY-1](https://github.com/hotwax/oms/pull/240#discussion_r2253692858) [D240-FACILITY-2](https://github.com/hotwax/oms/pull/240#discussion_r2284082775)
- Store pickup and BOPIS flows should be modeled consistently. Do not mix order,
  shipment, and picklist models unless the PR explains why that model owns the
  state being queried or changed. [D240-BOPIS-MODEL](https://github.com/hotwax/oms/pull/240#discussion_r2253702944) [D240-SHIPMENT-STATUS](https://github.com/hotwax/oms/pull/240#discussion_r2253696646)
- For BOPIS handoff behavior, do not rely on shipment or picklist records when
  the order model is the source of truth for the app flow. [D240-BOPIS-MODEL](https://github.com/hotwax/oms/pull/240#discussion_r2253702944)
- Check nullable or absent records before assuming related records exist. Empty
  ship groups, missing shipment links, or missing optional associations should be
  handled intentionally. [D240-EMPTY-SHIPGROUP](https://github.com/hotwax/oms/pull/240#discussion_r2253685052) [D240-SHIPGROUP-FIELD](https://github.com/hotwax/oms/pull/240#discussion_r2284070351)
- Challenge business data that appears in the wrong component or seed file.
  Integration-specific seed data, such as Unigate configuration, should live in
  the owning integration/component data area, not generic OMS data files.
  [A244-UNIGATE](https://github.com/hotwax/oms/pull/244#discussion_r2249439625)

## Entity And Data Model Placement

- Add new entities to the existing entity model file that owns the domain unless
  there is a strong reason to create a new entity XML file. [D220-ENTITY-FILE](https://github.com/hotwax/oms/pull/220#discussion_r2374368559)
- Do not create new enumeration or job data if the component does not actually
  manage that enumeration or job lifecycle. [D155-JOB-ENUM](https://github.com/hotwax/oms/pull/155#issuecomment-3116414504)
- When adding integration history tables, think through future ownership and
  integration-table placement. Avoid schema choices that will block later
  migration or make history data hard to query by shop, product, or system
  message. [A220-HISTORY](https://github.com/hotwax/oms/pull/220#discussion_r2185497803)
- Do not persist large raw payloads such as full product JSON unless the PR
  justifies storage size, retention, and query behavior. Prefer hashes, diffs,
  or normalized fields when they satisfy the use case. [D220-LARGE-JSON](https://github.com/hotwax/oms/pull/220#discussion_r2219332415)

## Query And Performance Checks

- Be skeptical of direct database queries in high-volume operational flows.
  Prefer existing SOLR documents, indexes, cached lookups, or view entities when
  they are the established source for the screen or service. [D240-SOLR](https://github.com/hotwax/oms/pull/240#discussion_r2253669960)
- Flag queries against entities that can contain millions of records unless the
  query has selective conditions, appropriate indexes, limits, and avoids
  unnecessary scans. [D240-MILLIONS](https://github.com/hotwax/oms/pull/240#discussion_r2284081324)
- Use `cache="true"` for stable reference data where the surrounding codebase
  already treats that entity as cacheable. [D240-CACHE](https://github.com/hotwax/oms/pull/240#discussion_r2253734823)
- Search behavior should be intentional. Avoid `%keyword%` contains searches on
  large datasets unless the PR justifies the cost. Prefer begins-with or
  ends-with patterns that can use indexes when possible. [D343-LIKE](https://github.com/hotwax/oms/pull/343#discussion_r2583621511)
- Avoid repetitive field-by-field copy logic when iterating over known changed
  keys or a diff map would produce the same result more simply and safely.
  [A220-DIFF](https://github.com/hotwax/oms/pull/220#discussion_r2182741089)

## Concurrency And Transactions

- Review inventory and order detail updates for concurrency risks. If a change
  writes inventory totals, reservations, or `InventoryItemDetail` records, ask
  whether concurrent updates can deadlock or create inconsistent totals.
  [D500-DEADLOCK](https://github.com/hotwax/oms/pull/500#pullrequestreview-4194908612)
- Be careful with `transaction="force-new"`. It should protect a clear boundary,
  not hide partial failures or split one logical update into inconsistent
  commits.
- When a service is called from async jobs, webhooks, or bulk imports, check
  idempotency. Re-running the service should not duplicate records or corrupt
  state.

## Code Cleanliness

- Remove dead or commented-out code instead of leaving disabled blocks in the
  service implementation. [D240-DEAD-CODE](https://github.com/hotwax/oms/pull/240#discussion_r2284080200)
- Prefer existing helpers and framework methods such as `getPlainMap()` when
  they express the intent better than manual mapping. [D240-PLAIN-MAP](https://github.com/hotwax/oms/pull/240#discussion_r2284083621)
- Keep comments focused on why the behavior exists. Do not add comments that
  restate obvious XML actions.
- Do not introduce styling or CSS changes for UI work unless the PR explicitly
  asks for CSS. For Ionic UI work, prefer core Ionic components, avoid `ion-grid`
  and Ionic grid utility layouts, keep screens mobile compatible, and remove
  duplicate information from the UI.

## What To Mention In Review Comments

- Reference the exact contract or domain rule being violated.
- Name the likely production impact: wrong API shape, expensive query, broken
  generated documentation, wrong facility semantics, duplicate data, deadlock
  risk, or misplaced seed data.
- Suggest the expected repository pattern when one exists.
- Avoid approving vague implementations with only a note. If the implementation
  is not aligned with the repository pattern, say what must change before merge.
  [D220-NOT-ALIGNED](https://github.com/hotwax/oms/pull/220#issuecomment-3326538799)

## Source Index

- [A220-DIFF](https://github.com/hotwax/oms/pull/220#discussion_r2182741089): Anil on simplifying diff/key iteration instead of checking each
  attribute separately:
  https://github.com/hotwax/oms/pull/220#discussion_r2182741089
- [A220-HISTORY](https://github.com/hotwax/oms/pull/220#discussion_r2185497803): Anil on thinking ahead about `ProductUpdateHistory` in an
  integration table:
  https://github.com/hotwax/oms/pull/220#discussion_r2185497803
- [D220-NOUN](https://github.com/hotwax/oms/pull/220#discussion_r2219314008): Deepak questioning an unexpected service noun:
  https://github.com/hotwax/oms/pull/220#discussion_r2219314008
- [D220-LARGE-JSON](https://github.com/hotwax/oms/pull/220#discussion_r2219332415): Deepak questioning storage of large product JSON payloads:
  https://github.com/hotwax/oms/pull/220#discussion_r2219332415
- [D220-TYPE](https://github.com/hotwax/oms/pull/220#discussion_r2219341980): Deepak recommending `productTypeId` instead of prefixing:
  https://github.com/hotwax/oms/pull/220#discussion_r2219341980
- [D220-ENTITY-FILE](https://github.com/hotwax/oms/pull/220#discussion_r2374368559): Deepak recommending adding an entity to the existing
  entity file:
  https://github.com/hotwax/oms/pull/220#discussion_r2374368559
- [D220-NOT-ALIGNED](https://github.com/hotwax/oms/pull/220#issuecomment-3326538799): Deepak noting an implementation was not aligned even
  though it was merged:
  https://github.com/hotwax/oms/pull/220#issuecomment-3326538799
- [D155-JOB-ENUM](https://github.com/hotwax/oms/pull/155#issuecomment-3116414504): Deepak closing a PR because the component does not manage
  the job enumeration:
  https://github.com/hotwax/oms/pull/155#issuecomment-3116414504
- [A240-NAME](https://github.com/hotwax/oms/pull/240#discussion_r2236143637): Anil asking to remove `All` from a service name:
  https://github.com/hotwax/oms/pull/240#discussion_r2236143637
- [A240-ORDER-DETAIL](https://github.com/hotwax/oms/pull/240#discussion_r2236181929): Anil asking order detail APIs to follow a common model:
  https://github.com/hotwax/oms/pull/240#discussion_r2236181929
- [A240-REST-NAME](https://github.com/hotwax/oms/pull/240#discussion_r2236277226): Anil questioning the REST resource name `details`:
  https://github.com/hotwax/oms/pull/240#discussion_r2236277226
- [A244-UNIGATE](https://github.com/hotwax/oms/pull/244#discussion_r2249439625): Anil flagging Unigate data in the wrong place:
  https://github.com/hotwax/oms/pull/244#discussion_r2249439625
- [D240-SOLR](https://github.com/hotwax/oms/pull/240#discussion_r2253669960): Deepak asking why SOLR was not used instead of a direct DB query:
  https://github.com/hotwax/oms/pull/240#discussion_r2253669960
- [D240-NAME](https://github.com/hotwax/oms/pull/240#discussion_r2253678623): Deepak recommending a service name like `get#SalesOrder`:
  https://github.com/hotwax/oms/pull/240#discussion_r2253678623
- [D240-OUT](https://github.com/hotwax/oms/pull/240#discussion_r2253682259): Deepak asking for complete out-parameters for Swagger docs:
  https://github.com/hotwax/oms/pull/240#discussion_r2253682259
- [D240-EMPTY-SHIPGROUP](https://github.com/hotwax/oms/pull/240#discussion_r2253685052): Deepak warning about empty ship groups:
  https://github.com/hotwax/oms/pull/240#discussion_r2253685052
- [D240-FACILITY-1](https://github.com/hotwax/oms/pull/240#discussion_r2253692858): Deepak clarifying `OISG.facilityId` semantics:
  https://github.com/hotwax/oms/pull/240#discussion_r2253692858
- [D240-SHIPMENT-STATUS](https://github.com/hotwax/oms/pull/240#discussion_r2253696646): Deepak questioning shipment status in pickup order
  queries:
  https://github.com/hotwax/oms/pull/240#discussion_r2253696646
- [D240-BOPIS-MODEL](https://github.com/hotwax/oms/pull/240#discussion_r2253702944): Deepak warning not to rely on shipment/picklist for BOPIS
  handoff when the order model should drive it:
  https://github.com/hotwax/oms/pull/240#discussion_r2253702944
- [D240-CACHE](https://github.com/hotwax/oms/pull/240#discussion_r2253734823): Deepak asking that a lookup come from cache:
  https://github.com/hotwax/oms/pull/240#discussion_r2253734823
- [D240-ONE-GET](https://github.com/hotwax/oms/pull/240#discussion_r2253747644): Deepak saying order retrieval should not be split across two
  endpoints:
  https://github.com/hotwax/oms/pull/240#discussion_r2253747644
- [D240-BOPIS-REST](https://github.com/hotwax/oms/pull/240#discussion_r2253752233): Deepak recommending a separate BOPIS REST file for a
  distinct BOPIS flow:
  https://github.com/hotwax/oms/pull/240#discussion_r2253752233
- [D240-SHIPGROUP-FIELD](https://github.com/hotwax/oms/pull/240#discussion_r2284070351): Deepak suggesting use of `primaryShipGroupSeqId`:
  https://github.com/hotwax/oms/pull/240#discussion_r2284070351
- [D240-REASON](https://github.com/hotwax/oms/pull/240#discussion_r2284078324): Deepak asking not to default rejection reason internally:
  https://github.com/hotwax/oms/pull/240#discussion_r2284078324
- [D240-DEAD-CODE](https://github.com/hotwax/oms/pull/240#discussion_r2284080200): Deepak asking to remove commented-out code:
  https://github.com/hotwax/oms/pull/240#discussion_r2284080200
- [D240-MILLIONS](https://github.com/hotwax/oms/pull/240#discussion_r2284081324): Deepak warning about unnecessary DB load on large entities:
  https://github.com/hotwax/oms/pull/240#discussion_r2284081324
- [D240-FACILITY-2](https://github.com/hotwax/oms/pull/240#discussion_r2284082775): Deepak clarifying facility semantics are not always origin
  facility:
  https://github.com/hotwax/oms/pull/240#discussion_r2284082775
- [D240-PLAIN-MAP](https://github.com/hotwax/oms/pull/240#discussion_r2284083621): Deepak recommending `getPlainMap()`:
  https://github.com/hotwax/oms/pull/240#discussion_r2284083621
- [D240-MULTI](https://github.com/hotwax/oms/pull/240#discussion_r2284099493): Deepak recommending service multiple or item-level placement for
  cancel behavior:
  https://github.com/hotwax/oms/pull/240#discussion_r2284099493
- [D343-LIKE](https://github.com/hotwax/oms/pull/343#discussion_r2583621511): Deepak asking for begins-with or ends-with search instead of
  contains search:
  https://github.com/hotwax/oms/pull/343#discussion_r2583621511
- [D343-PARAMS](https://github.com/hotwax/oms/pull/343#discussion_r2583622480): Deepak asking not to remove explicit `orderId` and `orderName`
  parameters:
  https://github.com/hotwax/oms/pull/343#discussion_r2583622480
- [D500-DEADLOCK](https://github.com/hotwax/oms/pull/500#pullrequestreview-4194908612): Deepak asking to verify concurrent inventory detail cases for
  deadlock risk:
  https://github.com/hotwax/oms/pull/500#pullrequestreview-4194908612
