# PORTFOLIO_SUMMARY.md — Vanigam

> A recruiter-friendly, interview-ready brief on the engineering complexity, architectural decisions and scale considerations behind **Vanigam** — a multi-tenant, offline-first, cross-platform field-sales & GST-billing operational platform.

---

## 1. Elevator Pitch

> **Vanigam** is a production-grade enterprise platform that digitises the daily field workflow of distributors, wholesalers, retailers and small lenders — from GST-compliant invoicing and route-based payment collection to multi-company accounting — across **Android, iOS, macOS and Windows**, with a **single Flutter codebase** and an **offline-first, sync-everywhere** core backed by **Spring Boot 3.5** and **Oracle Autonomous Database** with **automatic list partitioning per tenant**.

---

## 2. Why this project is advanced

1. **Real offline-first, not "cached reads".** The Flutter client treats Isar as the source of truth; the network is a sync side-effect. UI never blocks on connectivity.
2. **Idempotent sync engine with version cursors.** A custom `SyncQueryPaginator` performs incremental pulls with exponential-backoff retries and `seenIds` deduplication; a local `SyncModel` queue replays pending operations in dependency order using composite-id idempotency.
3. **Server-broadcast cache invalidation.** Mutations bump `DataSyncVersion(orgId, collection)` and emit a **silent FCM push** to a per-organisation topic, plus a Firestore version document. Devices wake, pull deltas, and reconcile via Hibernate `@Version` optimistic locking.
4. **Multi-tenant Oracle partitioning.** Every transactional table is **list-partitioned by `(organisation_id, fy_account_id)` with `AUTOMATIC` partition creation** — buggy queries can't cross tenants and onboarding a new tenant or financial year requires zero DBA work.
5. **Contract-first generated client.** Springdoc emits OpenAPI 3; a custom Dart CLI patches `swagger_parser` output (e.g. `File` → `MultipartFile`) and drives `build_runner` to produce ~80 typed DTOs with `dart_mappable` JSON mappers.
6. **Per-resource RBAC at four levels** (NONE / VIEWER / EDITOR / APPROVER) propagated from `Employee` rows into a ThreadLocal `RequestContext` enforced by `AuthInterceptor`.
7. **Vault-driven production secrets.** OCI Vault OCIDs replace plain-text DB passwords; `start-application.sh` fetches them at boot.
8. **Desktop auto-updater** that consults a Google-Drive-hosted `versions.json` weekly and installs Windows builds via `WindowsUpdateInstallService`.

---

## 3. Engineering Complexity Matrix

| Dimension             | Evidence                                                                                                                                                 |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Codebase scope        | 20+ feature modules, 50+ typed `auto_route` routes, 15 JPA entities, 9 controllers, 15 services, 11 repositories                                         |
| Platform reach        | Android, iOS, macOS, Windows from one Dart codebase; responsive engine + platform plugins (camera, secure storage, biometrics, window manager)           |
| Sync engine           | Bidirectional, paginated, retried, deduplicated, idempotent, conflict-aware                                                                              |
| Data model            | Multi-tenant, multi-FY, multi-company, multi-currency, multi-batch inventory                                                                             |
| Security              | Firebase Auth + ThreadLocal tenant context + per-resource RBAC + Oracle partition isolation + OCI Vault + AES field encryption                           |
| Background processing | FCM background isolate, scheduled backups, periodic update checks, partition initialiser, desktop installer                                              |
| Reporting             | Offline reports + chart rendering + PDF export across sales, stock, GST, party, collection insight                                                       |
| DevEx                 | Melos scripts, generated Dio client, generated Isar collections, generated `auto_route` router, generated `flutter_gen` assets, generated `intl` strings |
| Ops                   | OCI Compute deploy via `deploy.sh`, Netdata + Tailon, Actuator, graceful 30 s shutdown                                                                   |

---

## 4. Architecture Sophistication

```mermaid
flowchart LR
    UI[Flutter UI<br/>auto_route 50+ routes]
    REPO[Repositories<br/>ChangeNotifier]
    ISAR[(Isar<br/>offline-first)]
    QUEUE[(SyncModel<br/>idempotent queue)]
    DIO[Generated Dio<br/>contract-first]
    API[Spring Boot 3.5]
    INT[AuthInterceptor<br/>RequestContext]
    SVC[Services<br/>@Transactional]
    CAF[(Caffeine L1)]
    ORA[(Oracle ADW<br/>list-partitioned)]
    FCM[FCM topic = orgId]
    FS[(Firestore)]
    VAULT[OCI Vault]

    UI --> REPO --> ISAR
    REPO --> QUEUE --> DIO
    DIO --> API --> INT --> SVC --> CAF
    SVC --> ORA
    SVC --> FCM
    SVC --> FS
    API --> VAULT
    FCM -.-> UI
    FS -.-> UI
```

Notable architectural choices:

- **Clean architecture** in Flutter — presentation depends on state; state depends on repositories; data adapters (Isar, Dio) are isolated.
- **Repository + sync mixin pattern** centralises offline-first behaviour; new features inherit retries, pagination, and deletion propagation by composition.
- **ThreadLocal `RequestContext`** removes user/org plumbing from every service method without sacrificing safety (cleared on `afterCompletion`).
- **`@DbForeignKey` annotation + `ForeignKeyRunner`** validates cross-entity references at app boot — catches mis-mapped associations before traffic.
- **`@PartitionKey` + `AUTOMATIC LIST` partitions** combine ORM-friendly modelling with database-level tenant isolation.
- **Silent FCM cache invalidation** keeps payloads tiny and battery-friendly; the actual data still flows over REST so syncs are resumable.

---

## 5. Scalability Highlights

| Layer         | How it scales                                                                                                                                                |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| API           | Stateless; horizontal scale on OCI Compute behind a load balancer. Caffeine cache is per-node (acceptable with 15 min TTL).                                  |
| Database      | Oracle Autonomous DB; list-partitioning enables partition pruning and per-tenant maintenance.                                                                |
| Sync          | Page-based incremental pulls; per-device cursors mean a million-row org doesn't impact a hundred-row org.                                                    |
| Notifications | FCM topic fan-out delegates the fan-out cost to Google's infra.                                                                                              |
| Reports       | Computed client-side off Isar — server is shielded from analytical workloads.                                                                                |
| Future        | Redis L2, Kafka between mutations and broadcasts, read replicas for reports, Oracle AI DB for vector-based "smart order" suggestions (flag already present). |

---

## 6. Production-Engineering Story

- **Operational readiness:** structured logs, Actuator, Netdata + Tailon, scripted deploy with SSH automation, graceful shutdown.
- **Field deployment:** auto-updates for Windows; per-org Google-Drive backups; per-collector RBAC; FCM-token refresh handled with topic re-subscription.
- **Auditability:** soft-delete audit table mirrored on the client; `@Version` on every mutable entity; immutable `updatedAt` cursors.
- **Resilience:** exponential-backoff retries, 50-page sync cap, last-event-wins coordinator, conflict reconcile path.
- **Security:** Firebase ID token validation, tenant header cross-check, per-resource RBAC, partition isolation, OCI-Vault-driven secrets, AES key for sensitive payloads, secure storage on device.

---

## 7. Real-World Operational Workflows

- **Field collector** runs an entire route disconnected — start session → record cash/cheque vouchers per party → close session → server-side approval workflow.
- **Owner** reviews and approves SUBMITTED invoices and CLOSED collections; per-resource APPROVER role required.
- **Sales rep** drafts orders that convert into invoices (`PENDING → SUBMITTED → CONVERTED_TO_INVOICE`).
- **Accountant** runs offline GST/sales/stock reports computed from Isar; exports PDF for filing.
- **Admin** ops triggers `recompute_stock` if drift is detected after invoice cancellations.

---

## 8. Resume Bullet Points

- Designed and shipped **Vanigam**, a multi-tenant offline-first field-sales platform on **Android / iOS / macOS / Windows** with a **single Flutter codebase** spanning 20+ feature modules and 50+ typed routes.
- Built an **idempotent bidirectional sync engine** with version cursors, paginated incremental pulls, exponential-backoff retries, deduplication and last-event-wins coordination — running fully against Isar offline.
- Implemented **Springdoc → swagger_parser → patched Dio** generation pipeline that keeps 80+ DTOs in lock-step with backend contracts on every CI run.
- Modelled **multi-tenant Oracle Autonomous DB** with `AUTOMATIC LIST` partitioning by `(organisation_id, fy_account_id)` and a custom `@DbForeignKey` startup validator.
- Engineered **silent FCM cache invalidation** keyed by organisation topic + Firestore version documents, decoupling broadcast from REST payload size.
- Enforced **per-resource RBAC** with four access levels propagated through a ThreadLocal `RequestContext` and the `AuthInterceptor`.
- Productionised **OCI deployment** with vault-driven secrets retrieval, Netdata/Tailon observability, and SSH-automated zero-touch releases.
- Built a **desktop auto-update** pipeline via Google-Drive-hosted manifests and a custom `WindowsUpdateInstallService`.

---

## 9. Interview Talking Points

- **"Walk me through the sync engine."** — Talk about `SyncModel` idempotency (composite id `entity-id-op`), the single `_runner` Future for last-event-wins, `SyncQueryPaginator` with limit=2000 / retries=3 / 50-page cap, `DataSyncVersion` cursors, and the silent-FCM wake-up combined with a Firestore version doc as a redundant signal channel.
- **"How do you isolate tenants?"** — Three lines of defence: `AuthInterceptor` rejects header mismatches with 424; services filter by `RequestContext.getOrgId()`; Oracle list partitioning prunes at the storage layer.
- **"Why Isar instead of SQLite?"** — Multi-platform native engine, faster transactional writes, schema mirrors server entities directly, supports indices for offline reports.
- **"How do you handle conflicts?"** — Hibernate `@Version` produces `409`; the client refetches authoritative state and shows a conflict dialog only when user-visible fields differ.
- **"What if Firestore is slow?"** — FCM is the primary trigger; Firestore is a secondary listener. Both are eventually consistent — the **next user action triggers another pull anyway**.
- **"How do you regenerate the API client?"** — `melos run generate:swagger` runs the custom CLI that wipes generated, runs `swagger_parser`, patches `File` → `MultipartFile`, runs `build_runner`, formats.
- **"Why ThreadLocal for context?"** — Avoids parameter pollution across the service tree; cleared on `afterCompletion` to prevent cross-request leakage. Risk surface limited because services are synchronous.
- **"How is RBAC enforced?"** — `ActionAccess` fields on `Employee` per resource (party, product, collection, invoice, order, employee); checked in services via static accessors on `RequestContext` (e.g. `RequestContext.invoiceAccess()`). UI mirrors with `access_denied_pg.dart`.
- **"How would you scale to 10× tenants?"** — Add Redis L2, fan-out broadcasters via Kafka, read replicas for reports, switch reports to materialised views, and lean further on Oracle AI DB for vector workloads (flag already in `Organisation`).

---

## 10. System Design Discussion Hooks

- **CAP positioning:** AP-first — accept temporary inconsistency between devices in favour of always-on UX in the field; eventually consistent via `DataSyncVersion`.
- **Idempotency:** `SyncModel` composite key + server upsert semantics make retries safe; deletes are soft (audit trail in `DELETE_RECORD`).
- **Backpressure:** 50-page sync cap + `seenIds` dedupe prevent runaway loops; per-page limit=2000 keeps payloads bounded.
- **Failure isolation:** Per-tenant partitions + stateless API + Caffeine-only cache localise blast radius.
- **Observability:** Caffeine stats, Actuator metrics, Netdata system metrics, Tailon log streaming — all available without redeploys.
- **Future evolution:** CRDT line-merging for parallel invoice edits, real-time SSE channel for owner dashboards, vector search via Oracle AI DB for the "smart order suggestion" feature flag.

---

## 11. Numbers Worth Quoting

| Metric                  | Value                                       |
| ----------------------- | ------------------------------------------- |
| Supported platforms     | 4 (Android, iOS, macOS, Windows)            |
| Flutter feature modules | 20+                                         |
| Typed routes            | 50+                                         |
| JPA entities            | 15                                          |
| Spring controllers      | 9                                           |
| Spring services         | 15                                          |
| Isar collections        | 12+                                         |
| Generated DTOs          | ~80                                         |
| Caffeine L1 size        | 1 200 entries / 15-min TTL                  |
| Sync page size          | 2 000 rows                                  |
| Sync retries            | 3 × exponential (500 ms → 1.5 s)            |
| Sync hard cap           | 50 pages per wave (100 000 rows)            |
| Partition strategy      | List-AUTOMATIC by `(org_id, fy_account_id)` |

---

## 12. Closing Statement

Vanigam demonstrates the ability to:

- Translate a **fuzzy operational problem** (small lenders, distributors, route collectors) into a **clean domain model** with multi-tenant boundaries.
- Design a **distributed sync protocol** that respects mobile constraints (battery, intermittent connectivity, conflicts).
- Ship a **single codebase across four platforms** without sacrificing native UX through a responsive engine and platform plugins.
- Bake **production engineering** into the fabric — partitioning, vault-driven secrets, observability, graceful shutdowns, audit trails.
- Maintain **velocity at scale** via generated clients, melos-orchestrated scripts, and a feature-first folder structure that new engineers can navigate in a day.

> This is the kind of system one usually finds behind paid SaaS: **multi-tenant, offline-resilient, real-time, audit-grade**. It is built end-to-end by an engineer who can own the entire stack from Oracle DDL to Windows installer.
