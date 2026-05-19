# FEATURE_DOCUMENTATION.md — Vanigam

> Complete inventory of every feature module, screen, form, action, report and notification flow discovered in the Flutter app at [vanigam/lib/feature/](../lib/feature/) and the Spring Boot endpoints they consume.
>
> Each feature section follows the same template:
> **Purpose · Business workflow · User flow · Backend interaction · Offline behaviour · Sync · Caching · State · APIs · Entities · Notifications · Reports · Validation · Edge cases.**

---

## Module Index

| #   | Module                                              | Primary route(s)                                                                      | Backend controller(s)                    |
| --- | --------------------------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------- |
| 1   | [App Gateway](#1-app-gateway)                       | `AppGatewayRoute` (initial)                                                           | —                                        |
| 2   | [Login](#2-login)                                   | `LoginRoute`, `EmailFormRoute`                                                        | `AuthController`                         |
| 3   | [Registration](#3-registration)                     | `RegistrationRoute`                                                                   | `AuthController`                         |
| 4   | [Organisation](#4-organisation--company)            | `OrganisationSelectionRoute`, `AppContactInfoRoute`                                   | `AuthController`, `DataEntityController` |
| 5   | [Home / Dashboard](#5-home--dashboard)              | `HomeRoute`, `OfflineDashboardRoute`                                                  | sync only                                |
| 6   | [Party](#6-party-customers)                         | `PartyDirectoryRoute`, `PartyFormRoute`, `PartyTransactionRoute`                      | `PartyController`                        |
| 7   | [Product & Inventory](#7-product--inventory)        | `ProductsDirectoryRoute`, `ProductFormRoute`, `ProductListingRoute`, `StockInfoRoute` | `ProductController`, `InvoiceController` |
| 8   | [Invoice](#8-invoice)                               | `InvoiceEntryRoute`, `InvoiceReturnRoute`, `InvoiceTransactionRoute`                  | `InvoiceController`                      |
| 9   | [Voucher](#9-voucher)                               | `VoucherFormRoute`, `VoucherTransactionRoute`                                         | `VoucherController`                      |
| 10  | [Collection](#10-collection)                        | `CollectionRoute`, collection form/details/insight                                    | `CollectionController`                   |
| 11  | [Order](#11-order)                                  | `OrderSummaryRoute`, `OrderDetailsRoute`, `OrderTransactionRoute`                     | `OrderController`                        |
| 12  | [Employee](#12-employee)                            | `EmployeeDirectoryRoute`, `EmployeeFormRoute`                                         | `EmployeeController`                     |
| 13  | [Reports](#13-reports--insights)                    | `ReportRoute`, sales / stock / party / GST                                            | derived data + `data_entity`             |
| 14  | [Transactions](#14-transactions)                    | `InvoiceTransactionRoute`, `VoucherTransactionRoute`, `OrderTransactionRoute`         | mixed                                    |
| 15  | [Sync](#15-sync-engine)                             | invisible                                                                             | every `fetch` endpoint                   |
| 16  | [Notifications](#16-notifications)                  | drawer / banner                                                                       | FCM                                      |
| 17  | [Settings](#17-settings)                            | `SettingsRoute`                                                                       | `AuthController`, `DataEntityController` |
| 18  | [Subscription](#18-subscription)                    | `SubscriptionExpiredRoute`                                                            | feature flags                            |
| 19  | [Export & PDF](#19-export-invoice--pdf)             | invoice share                                                                         | static / local                           |
| 20  | [Errors & Access](#20-errors-access-denied-offline) | `AccessDeniedRoute`, error pages                                                      | —                                        |

---

## 1. App Gateway

- **File:** [lib/feature/app_gateway/](../lib/feature/app_gateway/)
- **Purpose:** First route after Flutter boot — decides where to navigate based on auth + organisation state.
- **User flow:**
  1. Splash → check Firebase auth.
  2. If unauthenticated → `LoginRoute`.
  3. If authenticated but no organisation → `OrganisationSelectionRoute`.
  4. If subscription expired → `SubscriptionExpiredRoute`.
  5. If offline → `OfflineDashboardRoute`.
  6. Otherwise → `HomeRoute`.
- **State:** Reads `AuthRepository`, `AppCacheService`, `internet_connection_checker`.
- **Edge cases:** Token refresh failures, deeplinks captured before nav decision.

---

## 2. Login

- **Files:** [lib/feature/login/](../lib/feature/login/)
- **Purpose:** Google Sign-In on all platforms (mobile + desktop) using `google_sign_in_all_platforms`.
- **User flow:** Tap "Sign in with Google" → OAuth flow → Firebase credential → `GET /api/auth`.
- **Backend:**
  - `GET /api/auth` — `AuthController` — returns `AuthUser` + registered devices; the call also **registers/updates** the current device (`UserDevice` row with `fcmToken`).
- **Offline:** Disabled (login requires network); cached user enables relogin without re-OAuth where possible.
- **Validation:** Email verified flag; `disabled` flag on `AuthUser`.
- **Edge cases:** `maxDevice` reached → server returns 403; revoked tokens force re-login.

### 2.1 Email Form

- **Route:** `EmailFormRoute` — for environments where Google sign-in is unavailable (kept as fallback).

---

## 3. Registration

- **Files:** [lib/feature/register/](../lib/feature/register/)
- **Purpose:** First-time onboarding after Google Sign-In when no `AuthUser` exists server-side.
- **User flow:** Capture name, currency (default ₹), DOB optional → `POST` user creation → org creation flow.
- **Validation:** Currency from supported list, email uniqueness on server.

---

## 4. Organisation & Company

- **Files:** [lib/feature/organisation/](../lib/feature/organisation/) (also reused in settings)
- **Purpose:** Create and switch between organisations and companies. Owners get **1 free org**; additional orgs are subscription‑gated.
- **User flow:**
  1. `OrganisationSelectionRoute` — pick existing or "Create new".
  2. `AppContactInfoRoute` / Settings → Company form (proprietor name, GST no, state code, pincode, mobile, logo).
- **Backend:**
  - `POST /api/auth/organisation` — create org
  - `PUT /api/auth/organisation` — update org
  - `POST /api/auth/fy_account` — create FY
  - `GET /api/auth/organisation/access` — admin access listing
  - `PUT /api/auth/organisation/access` — modify member access
  - `POST /api/data_entity/company` — upsert company (multipart for logo)
  - `DELETE /api/data_entity/organisation` — full org deletion (owner only)
- **Entities involved:** `Organisation`, `Company`, `FyAccount`, `Employee`.
- **Sync:** Selecting an org sets `X-Org-Id` on every subsequent request and triggers a full incremental sync via `AppSyncController`.
- **Feature flags read here:** `estimateInvoiceFeature`, `invoiceCollectionFeature`, `productOfferfeature`, `exportInvoiceFeature`, `inventoryFeature`, `orderFeature`, `smartOrderSuggessionFeature`, `smartStockSuggessionFeature`, `maxCompany`, `maxEmployee`.

---

## 5. Home / Dashboard

- **Files:** [lib/feature/home/](../lib/feature/home/) — `home_pg.dart`, `desktop_home_pg.dart`, `mobile_home_pg.dart`, `dashboard_tab.dart`, FAB + drawer.
- **Purpose:** Operational landing surface; responsive UI splits into desktop / tablet / mobile shells.
- **Sections:**
  - **Today summary** — counts of invoices, collections, orders (computed from Isar locally).
  - **Quick actions FAB** — `flutter_expandable_fab` → new invoice / voucher / order / party / product.
  - **Recent activity list** — last n invoices & vouchers.
  - **Sync banner** — when offline or pending queue > 0.
- **State:** `AppHomeController` (Bloc) + repositories.
- **Offline:** Full functionality.
- **Notifications:** Topic `orgId` triggers silent refresh; drawer displays user notifications.

### 5.1 Offline Dashboard

- **Route:** `OfflineDashboardRoute` — minimal UI when `internet_connection_checker` reports no connectivity; the rest of the app continues to read Isar.

---

## 6. Party (Customers)

- **Files:** [lib/feature/party/](../lib/feature/party/) — `party_directory_pg.dart`, `party_form_pg.dart`, `party_transaction_pg.dart`.
- **Purpose:** Customer / counterparty master with credit-limit, area/zone, GST.
- **Forms:** Business name, contact name, mobile, email, GST, address, state code, pincode, area, tags, types, credit limit, max-due-in-days, image, party groups.
- **Backend:**
  - `POST /api/party/upsert` (`multipart/form-data` for image)
  - `GET /api/party/{id}`
  - `DELETE /api/party/{id}` (APPROVER only)
  - `POST /api/party/fetch` — paginated sync query
- **Entity:** [`Party`](../../vanigam_server/src/main/java/in/subbu/vanigam/entity/master/Party.java).
- **Validation:** GST format (15 chars), mobile (10 digits), email regex; credit limit ≥ 0.
- **Offline:** All CRUD available; image queued for upload.
- **Sync:** `SyncQueryPaginator` (`limit=2000`) with `DataSyncVersion(collection=party)` cursor.
- **Edge cases:** Group reassignment, soft-delete propagation through `DeleteRecord`.

### 6.1 Party Transaction Page

- Per-party ledger view: invoices + collections + vouchers, running balance, days-due indicator.

---

## 7. Product & Inventory

- **Files:** [lib/feature/product/](../lib/feature/product/)
- **Screens:** `products_directory_pg.dart`, `product_form_pg.dart`, `product_listing_pg.dart`, `stock_info_pg.dart`.
- **Form fields:** name, title, description, print label, barcode, brand, category, MRP, price, GST rate, unit, min-stock alert, SKU, HSN, image, tags, inventory toggle, company configs, unit conversions.
- **Backend:**
  - `POST /api/product/upsert` (multipart)
  - `GET /api/product/{id}` (returns 404 placeholder; reads go via fetch)
  - `DELETE /api/product/{id}` (APPROVER)
  - `POST /api/invoice/fetch_stock` — batch stock sync
  - `POST /api/data_entity/recompute_stock` — admin recompute
- **Entities:** [`Product`](../../vanigam_server/src/main/java/in/subbu/vanigam/entity/master/Product.java), [`BatchStock`](../../vanigam_server/src/main/java/in/subbu/vanigam/entity/invoice/BatchStock.java).
- **Reports:** Stock report (low-stock alerts), batch expiry, offer products.
- **Validation:** HSN/SKU uniqueness optional, price > 0, GST rate ∈ {0,5,12,18,28}, expiry future date for batches.
- **Edge cases:** Re-computation when invoice cancelled; multi-company stock segregation via `companyConfigs`.

---

## 8. Invoice

- **Files:** [lib/feature/invoice/](../lib/feature/invoice/) — `invoice_entry_pg.dart` (create), `invoice_return/`, `transaction/`.
- **Purpose:** GST-compliant sales, purchase, return, estimate and export invoices.
- **Form sections:**
  - Header — invoice no (auto via `IdGeneratorService` & `Company.invoiceNoPattern`), date, type, party (autocomplete), GST + state codes, payment mode.
  - Lines — product autocomplete, HSN, batch, expiry, qty, free qty, unit, price (GST inclusive), discount, GST rate, CESS rate, MRP, purchase rate.
  - Footer — discount, GST split (CGST/SGST/IGST inferred from state codes), cash discount, total, remark.
- **Statuses:** DRAFT → SUBMITTED → APPROVED / REJECTED → PRINTED → CANCELLED.
- **Backend:**
  - `POST /api/invoice/upsert`
  - `GET /api/invoice/{id}`
  - `PUT /api/invoice/{id}/status`
  - `POST /api/invoice/fetch` — sync query
  - `DELETE /api/invoice/{id}` (APPROVER)
  - `POST /api/invoice/fetch_stock`
- **Entities:** `Invoice`, `InvoiceEntry`, `BatchStock`.
- **Offline:** Full create/edit, PDF preview, share by file. Sync flushes via `SyncModel`.
- **Notifications:** Status change events broadcast to org topic.
- **Validation:** Buyer/Seller state codes mandatory; CESS only for select HSNs; total = Σ(line.qty × line.price) − discount + GST + cash discount adjustments.
- **Edge cases:** Returns reference `orginalInvoiceNo`; export invoices use `Company.exportInvNoFormat`; estimates skip stock decrement.

### 8.1 Invoice Return

- Dedicated form referencing original invoice; line quantities ≤ original.

---

## 9. Voucher

- **Files:** [lib/feature/voucher/](../lib/feature/voucher/)
- **Purpose:** Manual transactions — payments received, paid, cheque receipts, online transfers.
- **Form fields:** Party, voucher date, amount, source (manual/system), trans type (CREDIT/DEBIT), payment type (CASH/CHEQUE/ONLINE/…), cheque date / number / status, attachments, details, optional `invoiceId` linking, `collectionId` linking.
- **Backend:**
  - `PUT /api/voucher` — create/update
  - `GET /api/voucher` — paginated fetch
  - `DELETE /api/voucher/{id}`
- **Entity:** [`Voucher`](../../vanigam_server/src/main/java/in/subbu/vanigam/entity/collection/Voucher.java).
- **Validation:** Cheque type requires cheque number + date; status defaults to PENDING and transitions to CLEARED on reconciliation.
- **Edge cases:** Voucher cancellation revives invoice outstanding; chq bounce flips `ChequeStatus`.

---

## 10. Collection

- **Files:** [lib/feature/collections/](../lib/feature/collections/) — listing `collection_pg.dart`, plus `form/`, `details/`, `insight/`.
- **Purpose:** Route-level collection sessions covering multiple invoices and parties for a field agent.
- **Lifecycle:** DRAFT → IN_PROGRESS → CLOSED / CANCELLED.
- **User flow:**
  1. Collector opens new session → picks area(s).
  2. Visits each party → records voucher (cash/cheque/online) attached to invoice(s).
  3. Quotation captures ad-hoc fees (e.g., petrol allowance).
  4. End of day → close session → owner reviews.
- **Backend:**
  - `PUT /api/collection` — create
  - `POST /api/collection` — update
  - `PATCH /api/collection` — patch with attached invoice
  - `GET /api/collection/{id}`
  - `GET /api/collection/my_current_collection`
  - `GET /api/collection` — fetch (sync query)
  - `DELETE /api/collection/{id}` (APPROVER)
- **Entity:** [`VCollection`](../../vanigam_server/src/main/java/in/subbu/vanigam/entity/collection/VCollection.java).
- **Insights screen:** Per-collector totals, cash vs cheque split, area-wise heatmap.
- **Offline:** Entire session can be built offline. Pending vouchers replay in dependency order.
- **Notifications:** Owner gets a topic message when a session is `CLOSED` and needs approval.
- **Edge cases:** Only one IN_PROGRESS session per user; closing requires all linked invoices to be SUBMITTED+.

---

## 11. Order

- **Files:** [lib/feature/order/](../lib/feature/order/) — `order_summary_pg.dart`, `order_details/`, `order_transaction_pg.dart`.
- **Purpose:** Capture orders ahead of invoicing (sales rep workflow).
- **Lifecycle:** PENDING → SUBMITTED → CONVERTED_TO_INVOICE / CANCELLED / MODIFIED_INVOICE.
- **Backend:**
  - `POST /api/order/upsert`
  - `DELETE /api/order/delete`
  - `POST /api/order/fetch` — sync query
- **Entities:** `InvoiceOrder`, `InvoiceOrderEntry`.
- **Convert to invoice:** Local action copies entries into a draft `Invoice`, sets `sourceInvoiceId`, and the next sync pushes both.
- **Edge cases:** Modified-after-convert path keeps history (`MODIFIED_INVOICE`).

---

## 12. Employee

- **Files:** [lib/feature/employee/](../lib/feature/employee/) — `employee_directory_pg.dart`, `employee_form_pg.dart`.
- **Purpose:** Owner invites collectors / sales reps; per-resource access matrix.
- **Form fields:** Name, email (unique per org), role display name (`rName`), mobile, address, area, pincode, parties tag (list), products tag (list); access matrix:

| Field              | Levels                            |
| ------------------ | --------------------------------- |
| `partyAccess`      | NONE / VIEWER / EDITOR / APPROVER |
| `productAccess`    | …                                 |
| `collectionAccess` | …                                 |
| `invoiceAccess`    | …                                 |
| `orderAccess`      | …                                 |
| `employeeAccess`   | …                                 |

- **Backend:** `POST /api/employee`, `GET /api/employee/{id}`, `DELETE /api/employee/{id}` (APPROVER).
- **Sync:** New employee triggers FCM to the org so owner devices refresh.
- **Edge cases:** Removing the only APPROVER for invoice is rejected server-side (`IllegalStateException` → 409).

---

## 13. Reports & Insights

- **Files:** [lib/feature/report/](../lib/feature/report/) — `report_pg.dart` + `sales_report/`, `stock/`, party reports.
- **Discovered report screens:**

| Report                | Purpose                                                               |
| --------------------- | --------------------------------------------------------------------- |
| Sales report          | Daily / weekly / monthly totals, GST split, top parties, top products |
| Stock report          | Current stock per batch, low-stock list, expiring batches             |
| Party report          | Outstanding receivables, days-due aging                               |
| GST report            | GSTR-1 friendly summary (sales, returns, exports)                     |
| Collection insight    | Collector performance, area heatmap                                   |
| Order report          | Pending orders, conversion ratio                                      |
| Export invoice report | Export invoices with `Company.exportInvNoFormat`                      |

- **Data source:** All reports are computed from Isar (offline-first). The server only provides incremental rows.
- **PDF export:** Reports can be shared as PDF via [lib/feature/pdf/](../lib/feature/pdf/) renderers.
- **Charts:** `fl_chart` line / bar / pie.

---

## 14. Transactions

- **Files:** [lib/feature/transaction/](../lib/feature/transaction/)
- **Pages:**
  - `invoice_transaction_pg.dart` — filterable invoice ledger.
  - `voucher_transaction_pg.dart` — voucher ledger.
  - `order_transaction_pg.dart` — order ledger.
- **Filters:** Date range, party, company, FY, status, payment type.
- **State:** ChangeNotifier-backed pagination on Isar queries.

---

## 15. Sync Engine

- **Files:** [lib/feature/sync/](../lib/feature/sync/) — `app_sync_controller.dart`, `sync_fetch_func.dart`, `sync_push_func.dart`.
- **Purpose:** Coordinate bidirectional sync.
- **Triggers:**
  - App start (after auth + org selection).
  - FCM silent push (data-only message).
  - Firestore version-doc snapshot listener.
  - Manual pull-to-refresh.
  - Network reconnect.
- **Algorithm:** see [DATABASE_AND_SYNC.md](DATABASE_AND_SYNC.md) §3.
- **UI surfaces:** Banner on home, badge on FAB, snackbars on push failures.

---

## 16. Notifications

- **FCM service:** [lib/core/service/app_messaging_service.dart](../lib/core/service/app_messaging_service.dart) — singleton; subscribes to org topic.
- **Local service:** [lib/core/service/local_messaging_service.dart](../lib/core/service/local_messaging_service.dart) — surface alerts.
- **Handler:** [lib/core/service/handler/notification_handler.dart](../lib/core/service/handler/notification_handler.dart) — deep-link routing.
- **Inferred notification flows:**

| Trigger               | Payload                                       | UI effect          |
| --------------------- | --------------------------------------------- | ------------------ |
| Any entity mutation   | `{ collection, version, orgId }`              | silent → sync pull |
| Collection closed     | `{ type: collection_closed, collectionId }`   | owner banner       |
| Invoice status change | `{ type: invoice_status, invoiceId, status }` | party-aware toast  |
| Cheque bounce         | `{ type: cheque_bounce, voucherId }`          | red banner         |
| Backup completed      | `{ type: backup, when }`                      | settings indicator |
| Subscription expiring | `{ type: subscription }`                      | settings + banner  |

- **Background:** Android isolate registered by `LocalMessagingService.registerBackgroundServices()`.

---

## 17. Settings

- **Files:** [lib/feature/settings/](../lib/feature/settings/) — `settings_pg.dart`, `companies_listing`, `backup`.
- **Sections:**
  - Profile (name, image, currency).
  - Companies list (add/edit via `DataEntityController.upsertCompany`).
  - Backup & restore — Google Drive folder, interval, on-demand backup.
  - Feature flags overview (read-only mirror of `Organisation`).
  - Subscription details.
  - Language & theme.
  - Logout & device deregistration.
- **Edge cases:** Switching org clears Isar query caches but keeps device id.

---

## 18. Subscription

- **Files:** [lib/feature/subscription/](../lib/feature/subscription/) — `presentation/`, `features.service.dart`.
- **Purpose:** Gate features by `Organisation.subscription` and feature flags.
- **Page:** `SubscriptionExpiredRoute` — read-only view + renewal CTA.

---

## 19. Export Invoice & PDF

- **Files:** [lib/feature/export_invoice/](../lib/feature/export_invoice/), [lib/feature/pdf/](../lib/feature/pdf/).
- **Purpose:** Generate PDFs for invoices, vouchers, reports; share via OS share sheet; on desktop save to disk.
- **Templates:** Configurable via `Company.invoiceNoPattern`, `Company.exportInvNoFormat`.

---

## 20. Errors, Access-Denied, Offline

- **Components:** [lib/components/page/access_denied_pg.dart](../lib/components/page/access_denied_pg.dart), [lib/components/error/](../lib/components/error/).
- **Behaviour:**
  - 401 → forced re-login.
  - 403 → `AccessDeniedRoute` with required role.
  - 424 (ORG/FY mismatch) → switch-org prompt.
  - 409 (optimistic lock) → conflict dialog → reload from server.
  - 304 (no change) → silent.
  - 500 / network → retry banner.

---

## Cross-cutting Concerns per Feature

### Caching

- All repositories cache reads in Isar; the network call is mostly used to **synchronise**, not to **read**.
- `app_cache_service.dart` wraps SharedPreferences for non-domain state (last opened company, theme).
- Server-side Caffeine caches `AuthUser` look-ups by `uId` (15 min).

### Validation summary

| Form       | Hard rules                                                                 |
| ---------- | -------------------------------------------------------------------------- |
| Party      | GST 15 chars, mobile 10 digits, credit limit ≥ 0                           |
| Product    | Price > 0, MRP ≥ Price, HSN required for GST > 0                           |
| Invoice    | Buyer + seller state codes; total computed (not user-typed); CESS optional |
| Voucher    | Cheque type → number+date required                                         |
| Collection | At least one voucher to close                                              |
| Order      | At least one entry                                                         |
| Employee   | Unique email per org; at least one APPROVER per resource                   |

### Edge cases by domain

- **Time zones:** `timezone` package + `initializeTimeZones()` invoked before any `getLocation` call (see user memory note). The "company day" boundary affects collection insights.
- **Currency:** Persisted on `AuthUser` and `Organisation`; defaults to ₹.
- **Empty company:** `companyId` is nullable on `Invoice` & `BatchStock` for estimates that pre-date company creation.

---

## Notification ↔ Feature matrix

| Feature          | FCM       | Local | UI surface         |
| ---------------- | --------- | ----- | ------------------ |
| Sync             | ✅ silent | —     | banner             |
| Collection close | ✅        | ✅    | owner drawer       |
| Invoice status   | ✅        | ✅    | toast + drawer     |
| Backup           | —         | ✅    | settings           |
| Update available | —         | ✅    | settings (desktop) |

---

## Report ↔ Entity matrix

| Report             | Reads from                                |
| ------------------ | ----------------------------------------- |
| Sales              | Invoice + InvoiceEntry                    |
| Stock              | BatchStock + Product                      |
| Party              | Invoice + Voucher + VCollection per Party |
| GST                | Invoice + InvoiceEntry (HSN, GST rate)    |
| Collection insight | VCollection + Voucher + Employee          |
| Order conversion   | InvoiceOrder vs Invoice (sourceInvoiceId) |

---

> Any screen reachable from `AppRouter` but not listed above is either an internal sub-route of the screens above (e.g. forms inside listings) or a debug page under [lib/debug/](../lib/debug/) / [lib/dev/](../lib/dev/) excluded from production builds.
