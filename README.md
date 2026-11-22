# BISP Bulk Account Opening – Backend API (Mock Services)

Laravel 13 REST API for **bulk account opening from Excel** with a **maker / checker** workflow across
branch (IB) and Central Operations (COD) teams, plus an admin who can view everything and
**re-assign / re-tag** applications. External systems (NADRA, AML risk engine, Core Banking) are
**mocked** with deterministic rules so every path can be tested.

```
 Excel upload ─► IB MAKER ──(NADRA + Risk mock)──► IB APPROVER ──(documents)──► COD MAKER ──► COD APPROVER ──► Account Opened ──► Regularized
                    ▲                                 │   ▲                         │   ▲              │
                    └──────────── return ─────────────┘   └── revert + discrepancy ─┘   └─── return ───┘
 ADMIN: views all · reassign (same role, other user) · re-tag (move to another role / stage) · transfer workload · users / branches
```

---

## 1. Run it in Laragon

Requirements: Laragon with PHP 8.3+ and MySQL 8 (both included in your Laragon install).

1. Start Laragon → **Start All** (starts MySQL on 3306).
2. Create the database (HeidiSQL or terminal):
   ```powershell
   C:\laragon\bin\mysql\mysql-8.4.3-winx64\bin\mysql.exe -uroot -e "CREATE DATABASE IF NOT EXISTS bisp_backend CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"
   ```
3. From `C:\laragon\www\BISP\BISP_Backend`, migrate and seed the large demo data set:
   ```powershell
   php artisan migrate:fresh --seed
   ```
4. Serve the API:
   ```powershell
   php artisan serve
   ```
   Base URL: `http://localhost:8000/api/v1` (or through Apache: `http://localhost/BISP/BISP_Backend/public/api/v1`).
5. Import `docs/BISP_API.postman_collection.json` into Postman (83 requests, logins save the tokens for you).

`.env` is already set for Laragon (`DB_CONNECTION=mysql`, `root`, no password, DB `bisp_backend`).
Set `SEED_APPLICATIONS` in `.env` to change the seed size (default **6000**).

### Demo logins (password for **all** seeded users: `Password@123`)

| Role         | Email                    | Employee code |
|--------------|--------------------------|---------------|
| Admin        | `admin@bisp.test`        | ADM0001 |
| IB Maker     | `ibmaker@bisp.test`      | IBM0001 (branch LHR001) |
| IB Maker 2   | `ibmaker2@bisp.test`     | IBM0002 (branch LHR001) |
| IB Approver  | `ibapprover@bisp.test`   | IBA0001 (branch LHR001) |
| IB Approver 2| `ibapprover2@bisp.test`  | IBA0002 (branch LHR002) |
| COD Maker    | `codmaker@bisp.test`     | CDM0001 |
| COD Maker 2  | `codmaker2@bisp.test`    | CDM0002 |
| COD Approver | `codapprover@bisp.test`  | CDA0001 |
| COD Approver 2| `codapprover2@bisp.test`| CDA0002 |

Login works with email **or** employee code: `POST /auth/login {"login": "IBM0001", "password": "Password@123"}`.

### Seeded volume (default)

| Table | Rows |
|---|---|
| regions / branches | 15 / 93 (all provinces, GB, AJK) |
| users | 332 (2 IB makers + 1 IB approver per branch, 32 COD makers, 14 COD approvers, 3 admins, ~5% inactive) |
| upload_batches | ~125 (with row error reports) |
| account_applications | 6,000 – spread over **every** workflow status |
| nadra_verifications | ~5,500 |
| risk_assessments | ~4,800 |
| application_documents | ~17,600 |
| discrepancies | ~3,000 (open + resolved revert cycles) |
| application_histories (audit) | ~40,000 |

The demo users have real queues (e.g. `ibmaker@bisp.test` ≈ 265 open items, `codapprover@bisp.test` ≈ 280).

---

## 2. Workflow & statuses

| Status | Queue (role) | Next actions |
|---|---|---|
| `uploaded` | IB Maker | nadra-verify, reject |
| `nadra_failed` | IB Maker | nadra-verify (retry), reject |
| `nadra_verified` | IB Maker | risk-assessment |
| `risk_assessed` | IB Maker | forward → IB Approver |
| `returned_to_ib_maker` | IB Maker | re-run services, forward |
| `pending_ib_approval` | IB Approver | upload documents, forward → COD Maker, return → IB Maker, reject |
| `reverted_to_ib` | IB Approver | resolve discrepancies, upload documents, forward again (goes back to the same COD maker) |
| `pending_cod_review` | COD Maker | **revert with discrepancies** → IB Approver, forward → COD Approver |
| `returned_to_cod_maker` | COD Maker | forward / revert |
| `pending_cod_approval` | COD Approver | open-account, return → COD Maker, reject |
| `account_opened` | COD Approver | regularize |
| `regularized` | – (closed) | – |
| `rejected` | – (closed) | admin can re-tag to re-open |

Rules enforced by `App\Services\WorkflowService`:
* Only the **assigned user** of the **owning role** can act (403 otherwise; 409 if the application is in another stage).
* Forwarding requires NADRA `verified` + a risk rating; a **sanctions hit** blocks forwarding.
* IB Approver must attach mandatory documents: `cnic_front`, `cnic_back`, `account_opening_form`.
* Re-forwarding a reverted application requires all open discrepancies to be resolved.
* Auto-assignment = least-loaded active user of the next role (IB roles: same branch → same region → anyone).
  Pass `assign_to` to pick a specific user (validated for role / region / active).
* Every action is written to `application_histories` (who, role, from/to status, from/to user, IP, meta).
* A CNIC with an in-flight or completed application cannot be uploaded again (rejected ones can).

**Tracking number**: `BISP-{YYMM}-{BRANCH}-{8-digit id}`, e.g. `BISP-2611-LHR001-00000042`.

---

## 3. Mock services (deterministic)

`GET /mock/rules` returns these rules at runtime.

**NADRA (Verisys mock)** – by CNIC:

| CNIC | Result | Code |
|---|---|---|
| last digit `0` | `not_found` | 104 |
| last digit `9` | `expired` | 105 |
| 7-digit middle block ends in `13` (e.g. `35202-12345`**`13`**`-4`) | `blocked` | 110 |
| last digit `7` | `mismatch` (father/husband name) | 101 |
| anything else | `verified` | 100 |

**Risk engine** – score 0-100, `<35 low`, `<65 medium`, else `high`:
base 10; age <21 or >65 +10; occupation (Money Changer +30, Real Estate +25, Business +15 …);
income >100k +15 / >300k +25; high-risk district (Khyber, Kurram, N. Waziristan, Killa Abdullah, Kech, Gwadar, Zhob, Tharparkar) +15;
NADRA not verified +25 / mismatch +20; no nominee +5; PEP +30; adverse media +10;
**sanctions** (names `Sher Wazir Khan`, `Dost Jan Achakzai`, `Gul Qadir Afridi` or CNIC hash) → always high, blocks forwarding.

**Core Banking** – 16-digit account number and an ISO-13616 valid IBAN `PKkkBISB################`.

---

## 4. API reference (prefix `/api/v1`, Bearer token)

All responses: `{ "success": bool, "message": string, "data": ..., "meta": {pagination} }`.
Lists accept `per_page` (≤200), `page`, `sort`, `direction`.

### Auth & common
| Method | Endpoint | Notes |
|---|---|---|
| POST | `/auth/login` | `{login, password}` → token |
| GET | `/auth/me` · POST `/auth/logout` · POST `/auth/change-password` | |
| GET | `/dashboard` | role-aware counts (queue, status, risk, NADRA; admin: role queues, provinces, 6-month trend) |
| GET | `/lookups`, `/lookups/regions`, `/lookups/branches`, `/lookups/assignees?role=` | enums, upload columns, users to forward to |
| GET | `/applications` | filters: `status` (comma list), `current_role`, `nadra_status`, `risk_level`, `branch_id`, `batch_id`, `assigned_to`, `province`, `gender`, `date_from`, `date_to`, `search` (tracking/CNIC/name/mobile/account), `queue=1`, `has_open_discrepancy=1` |
| GET | `/applications/{id}` | full detail incl. NADRA, risk, documents, discrepancies, `available_actions` |
| GET | `/applications/{id}/timeline` | audit trail |
| GET | `/applications/track/{trackingNo}` | staff lookup by tracking number |
| GET | `/track/{trackingNo}?cnic_last4=1234` | **public** customer tracking (masked) |

### Downloads
| Endpoint | File |
|---|---|
| GET `/batches/template` | empty Excel template (with example row) |
| GET `/batches/sample?rows=100` | template filled with realistic mock beneficiaries |
| GET `/batches/{id}/download` | original uploaded file |
| GET `/batches/{id}/errors` | invalid rows + reasons (xlsx) |
| GET `/batches/{id}/export?format=xlsx\|csv` | batch applications with current status |
| GET `/applications/export?format=xlsx\|csv&...filters` | any filtered list |
| GET `/applications/{id}/pdf` | account opening summary PDF with audit trail |
| GET `/documents/{id}/download` | single document |
| GET `/applications/{id}/documents/zip` | all documents of an application |

### IB Maker
| POST | Endpoint |
|---|---|
| upload Excel (multipart `file`, `remarks`) | `/batches` |
| run NADRA / Risk / forward for a whole batch | `/batches/{id}/nadra-verify`, `/batches/{id}/risk-assessment`, `/batches/{id}/forward` |
| single application | `/applications/{id}/nadra-verify`, `/risk-assessment`, `/forward {assign_to?, remarks?}`, `/reject {remarks}` |
| bulk by ids | `/applications/bulk/{nadra-verify\|risk-assessment\|forward}` `{ids:[..]}` |
| list own batches | GET `/batches`, GET `/batches/{id}` |

### IB Approver
| POST | Endpoint |
|---|---|
| upload documents (multipart `documents[i][type]`, `documents[i][file]`, pdf/jpg/png ≤5MB) | `/applications/{id}/documents` |
| delete own document | DELETE `/documents/{id}` |
| forward to COD Maker | `/applications/{id}/forward` |
| return to IB Maker | `/applications/{id}/return {remarks}` |
| resolve discrepancy | `/discrepancies/{id}/resolve {remarks}` |
| reject | `/applications/{id}/reject {remarks}` |

### COD Maker
| POST | Endpoint |
|---|---|
| revert with discrepancies | `/applications/{id}/revert {discrepancies:[{category, description}], remarks?}` |
| forward to COD Approver | `/applications/{id}/forward` |

Discrepancy categories: `document_missing, document_unclear, data_mismatch, signature_mismatch, nadra_mismatch, kyc_incomplete, high_risk, other`.

### COD Approver
| POST | Endpoint |
|---|---|
| open account (mock CBS) | `/applications/{id}/open-account` |
| regularize | `/applications/{id}/regularize` |
| return to COD Maker | `/applications/{id}/return {remarks}` |
| reject | `/applications/{id}/reject {remarks}` |
| bulk | `/applications/bulk/{open-account\|regularize}` |

### Admin (`/admin/...`)
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/admin/applications/{id}/reassign {user_id, remarks?}` | move to another user **of the same role** |
| POST | `/admin/applications/{id}/retag {role, user_id?, status?, remarks}` | move to **another role/stage** (e.g. IB Approver → COD Maker) |
| POST | `/admin/applications/bulk-reassign {ids, user_id}` | |
| POST | `/admin/users/{id}/transfer-workload {to_user_id}` | move all open work (leave / resignation) |
| GET | `/admin/workload?role=` | open / high-priority / oldest item per user |
| GET | `/admin/audit-logs?action=&performed_by=&application_id=&date_from=` | |
| GET | `/admin/reports/summary` | KPIs, turnaround, by-branch |
| CRUD | `/admin/users` (+ `toggle-status`, `reset-password`) | |
| CRUD | `/admin/regions`, `/admin/branches` | organisation structure |

### Mock services (direct)
`GET /mock/rules`, `POST /mock/nadra/verify`, `POST /mock/risk/assess`, `POST /mock/cbs/iban`.

---

## 5. Excel upload format

First sheet, first row = headings (case-insensitive). Required columns marked *.

`BISP Beneficiary ID, CNIC*, Full Name*, Father Husband Name*, Mother Maiden Name, Date Of Birth*, Gender*, Marital Status, Mobile Number*, Email, Address*, City*, District, Province*, Occupation, Source Of Income, Monthly Income, Account Type, Purpose Of Account, Nominee Name, Nominee CNIC, Nominee Relation`

Normalisation: CNIC with or without dashes; mobile `03xx…`, `92…` or with a dropped leading zero;
dates `YYYY-MM-DD`, `DD/MM/YYYY`, `DD-MM-YYYY` or Excel serial dates; `M/F` gender. Max 5,000 rows per file.
Ready-made files: `docs/samples/BISP_Sample_200_rows_seed11.xlsx`, `docs/samples/BISP_Sample_1000_rows_seed12.xlsx`, or
`php artisan bisp:sample-file 500`.

---

## 6. Tests

```powershell
php artisan test
```
33 tests / 257 assertions (SQLite in-memory): auth, Excel upload (CSV + XLSX, validation, duplicates, downloads),
complete end-to-end workflow including revert/resolve and return cycles, role and assignment guards,
sanctions block, bulk actions, admin reassign / re-tag / workload transfer / user & org management, mock service rules and IBAN validity.

## 7. Code map

```
app/Enums                     Role, ApplicationStatus (owner role per status), DocumentType, DiscrepancyCategory
app/Services/WorkflowService  all state transitions + audit trail
app/Services/AssignmentService least-loaded / branch-aware assignment
app/Services/BulkUploadService Excel parsing, normalisation, validation, tracking numbers
app/Services/Mock/*           NADRA, Risk engine, Core Banking mocks
app/Http/Controllers/Api/V1   Auth, Dashboard, Lookup, Batch, Application, Document, Discrepancy, Tracking, MockService, Admin/*
app/Exports, app/Imports      Excel template / exports / error reports
database/seeders              Organization, User, Application (large, consistent demo data)
docs/                         Postman collection, sample upload files
```
