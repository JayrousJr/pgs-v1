# Bugando PGS: system flows (as-is) and rewrite target

This document maps every runtime flow in the current Payment Gateway System (PGS) and proposes the
target architecture for the rewrite. All file references are relative to `ci/app/` unless stated.

- Live base URL (from `Controllers/Malipo.php` test calls): `https://bmc.lipaswitch.co.tz/`
- Install-time base URL: `http://<host>/bmc/public/`
- eHMS (hospital system) host: `http://10.15.0.21:30005/ehms/payment/*`

## Contents

1. [Actors and system context](#1-actors-and-system-context)
2. [Request pipeline (every request)](#2-request-pipeline-every-request)
3. [Remote kill switch](#3-remote-kill-switch)
4. [Bill creation: eHMS to PGS](#4-bill-creation-ehms-to-pgs)
5. [Control number generation](#5-control-number-generation)
6. [Reference lookup (shared by bill intake and every bank)](#6-reference-lookup)
7. [Bank / mobile-money validation flows](#7-bank--mobile-money-validation-flows)
8. [Payment posting and settlement](#8-payment-posting-and-settlement)
9. [Payment sync back to eHMS (cron)](#9-payment-sync-back-to-ehms-cron)
10. [Reconciliation and other jobs](#10-reconciliation-and-other-jobs)
11. [Bill lifecycle state](#11-bill-lifecycle-state)
12. [Endpoint inventory](#12-endpoint-inventory)
13. [Defects by flow](#13-defects-by-flow)
14. [Target architecture for the rewrite](#14-target-architecture-for-the-rewrite)

---

## 1. Actors and system context

```mermaid
flowchart LR
    EHMS["eHMS<br/>Hospital system<br/>10.15.0.20 / .21"]
    STAFF["Hospital staff<br/>browser"]
    subgraph PGS["PGS (bmc.lipaswitch.co.tz)"]
        CI["CodeIgniter 4 app<br/>ci/app"]
        JOOM["Joomla install bmc1<br/>sessions + users"]
        DB[("MySQL bmc_1")]
    end
    AIRTEL["Airtel Money<br/>XML"]
    CRDB["CRDB Bank<br/>JSON"]
    NMB["NMB via mpayafrica<br/>JSON"]
    MKCB["Mkombozi Bank<br/>AES-GCM JSON"]
    VENDOR["lipaswitch.co.tz/bmc.php<br/>vendor licence server"]
    CRON["cron / scheduler"]

    EHMS -- "POST Api/postpatbills (new bill)" --> CI
    CI -- "POST BillPayment (payment confirm)" --> EHMS
    STAFF -- "login, manual bills, reports" --> CI
    AIRTEL -- "aipros1..5" --> CI
    CRDB -- "crdbverify / crdbpost" --> CI
    NMB -- "nmbbillrequest / nmbprocesscallback" --> CI
    MKCB -- "mkcbfetchcontrol / mkcbprocesscontrol" --> CI
    CI -- "nmbrecon pull" --> NMB
    CRON -- "Api/postbillpayment, dosynchrepeat, domastersynch, Test/lsp" --> CI
    CI -- "Test/lsp pull flags" --> VENDOR
    CI --- JOOM
    CI --- DB
    JOOM --- DB
```

## 2. Request pipeline (every request)

Every request (staff UI, eHMS, banks, crons) goes through the same boot path.

```mermaid
flowchart TD
    A["HTTP request"] --> B["public/.htaccess<br/>rewrite to index.php/$1"]
    B --> C["public/index.php<br/>error_reporting(0)<br/>boot CodeIgniter from ../ci/app"]
    C --> D["public/joom.php<br/>boot Joomla from hardcoded<br/>C:/server/htdocs/bmc1"]
    D --> E["Config/Routes.php<br/>setAutoRoute(true)<br/>/Controller/method/args"]
    E --> F{"Config/Filters.php<br/>controller in filter list?"}
    F -- "Login, /" --> F1["CheckLogin"]
    F -- "Dashboard, Billing, Patients,<br/>Transactions, Refund" --> F2["CheckPublic (must be logged in)"]
    F -- "Api, Malipo, Test, Verify, Old copies" --> G["no filter, public"]
    F1 --> H
    F2 --> H
    G --> H["BaseController::initController"]
    H --> H1["new Engine model<br/>Joomla session user"]
    H1 --> H2["Helpers/global_cache.php<br/>kill switch check A"]
    H2 -- "flag set" --> X["die('') blank 200"]
    H2 -- "ok" --> H3["getGlobalCookies + getStatusCookies<br/>kill switch check B"]
    H3 -- "flag matches method" --> X
    H3 -- "ok" --> I["Controller method runs"]
    I --> J["Most API methods end with die(json)<br/>so HTTP status is always 200"]
```

## 3. Remote kill switch

A vendor-controlled switch decides whether PGS serves requests. It is checked on every request,
including eHMS bill intake and bank payment callbacks.

```mermaid
sequenceDiagram
    autonumber
    participant Cron as Caller (cron or anyone)
    participant PGS as PGS Test::lsp
    participant V as lipaswitch.co.tz/bmc.php
    participant DB as bmc_users id=967 (GID)
    participant Any as Any later request

    Cron->>PGS: GET /Test/lsp (no auth, no filter)
    PGS->>V: POST {"siku": now}
    V-->>PGS: JSON e.g. {act_key, act_type, otpKey}
    PGS->>DB: Engine::doGlobalUpdate(json) writes columns on user 967
    Note over PGS,DB: Any JSON keys returned are written to bmc_users as-is
    Any->>PGS: request
    PGS->>DB: read user 967 act_key / act_type / otpKey
    alt act_type=1 and act_key date passed, or otpKey=1
        PGS-->>Any: die("") everything blocked (except /lsp)
    else act_type 2..6 and date passed
        PGS-->>Any: die() only for login / CRDB / NMB / MKCB / eHMS endpoints
    else
        PGS-->>Any: normal processing
    end
```

| `act_type` | Blocked methods (`Helpers/global_helper.php:874`) |
|---|---|
| 1 | everything |
| 2 | `process` (staff login) |
| 3 | `crdbverify`, `crdbpost` |
| 4 | `nmbbillrequest`, `nmbprocesscallback` |
| 5 | `mkcbfetchcontrol`, `mkcbprocesscontrol` |
| 6 | `postpatbills`, `postbillpayment` (eHMS integration) |

## 4. Bill creation: eHMS to PGS

Entry point: `POST https://bmc.lipaswitch.co.tz/api/postpatbills` → `Controllers/Api.php:56`.

### 4.1 Sequence

```mermaid
sequenceDiagram
    autonumber
    participant H as eHMS
    participant A as Api::postpatbills
    participant G as global_helper
    participant E as Engine model
    participant DB as MySQL

    H->>A: POST SEND_BILL_DATA {RequestKey, RequestHash, Bill{...,BillItems[]}}
    A->>G: checkIfAllowed() IP 10.15.0.20/.21 or Joomla user 967
    A->>G: json_validator()
    A->>G: validateApiBillData() required fields
    A->>G: verifyApiKeyHashPair() sha256(md5(key + shared secret))
    A->>E: getBillDetailsForMalipoValidation(BillNumber)  duplicate check
    E->>DB: open bill with bill_num or control_num = BillNumber?
    Note over E,DB: Fallback looks up patients and may INSERT a dummy bill (section 6)
    A->>E: fetchPatientRecordNew(PatientId)
    alt patient not found
        A->>E: savePatientDetails() INSERT bmc_patient_profile + bmc_patient_wallet
    end
    loop each BillItem
        A->>E: fetchLocalItemId(ItemId, desc, cost)
        alt product code unknown
            E->>DB: INSERT bmc_bill_product + bmc_bill_product_cost
        end
        A->>A: total = cost*qty, minus disc%, plus tax%
    end
    A->>E: saveBillDetails(dataMap, items)
    E->>G: getBillNumber() generate control number (section 5)
    E->>DB: BEGIN, INSERT bmc_bills, INSERT bmc_bills_items, COMMIT
    E->>DB: wallet balance > 0?
    opt wallet has credit
        E->>E: updatePayBillUsingWallet() close bill, queue bmc_bill_cleared
    end
    A->>E: getBillUsingId(bill_id)
    A->>G: genApiBillPostResponse(200, Success)
    A->>DB: saveApiLogs() bmc_gway_api_logs (request + response)
    A-->>H: SEND_BILL_RESPONSE {ControlNumber: E000..., BillNumber, StatusCode, StatusMsg}
```

### 4.2 Decision flow

```mermaid
flowchart TD
    S(["POST /api/postpatbills"]) --> K{"Kill switch passes?"}
    K -- no --> KX["blank response"]
    K -- yes --> A1{"Caller IP 10.15.0.20/.21<br/>or user 967?"}
    A1 -- no --> R1["422 Access Denied"]
    A1 -- yes --> A2{"Valid JSON?"}
    A2 -- no --> R2["422 Invalid Json"]
    A2 -- yes --> A3{"Required fields present?"}
    A3 -- no --> R3["422 Missing data in request"]
    A3 -- yes --> A4{"Key/hash pair valid?"}
    A4 -- no --> R4["422 Invalid Hash Key Pair"]
    A4 -- yes --> A5{"Open bill already exists<br/>for BillNumber?"}
    A5 -- yes --> R5["422 Duplicate Entry"]
    A5 -- no --> A6{"Patient exists by PatientId?"}
    A6 -- no --> A7["Create patient + wallet"]
    A7 -- fails --> R6["422 Failed to get patient"]
    A7 -- ok --> A8
    A6 -- yes --> A8["For each item: resolve or create product,<br/>compute line total"]
    A8 --> A9{"Any item qty=0 or cost=0?"}
    A9 -- yes --> R7["422 Failed to save bill"]
    A9 -- no --> A10["saveBillDetails<br/>generate control number E..."]
    A10 -- "insert fails" --> R7
    A10 -- ok --> A11{"Patient wallet balance > 0?"}
    A11 -- yes --> A12["Pay bill from wallet,<br/>queue for eHMS sync"]
    A11 -- no --> A13
    A12 --> A13["200 Success<br/>ControlNumber = bill_num<br/>BillNumber = control_num"]
    R1 & R2 & R3 & R4 & R5 & R6 & R7 & A13 --> L["Log to bmc_gway_api_logs<br/>die(json), HTTP 200"]
```

### 4.3 Field mapping

| eHMS request field | PGS storage | Used? |
|---|---|---|
| `Bill.BillNumber` | `bmc_bills.control_num` (naming is inverted) | yes |
| `Bill.PatientId` | `bmc_patient_profile.pat_num` | yes |
| `Bill.PatientName` | split on first space into `sur_name` / `other_names` | only for new patients |
| `Bill.PatientPhone` | `bmc_patient_profile.phone` | only for new patients |
| `Bill.BillTotal` | not stored, not checked | no |
| `Bill.AmountPaid` | ignored | no |
| `Bill.BillCreatedDate` | `bmc_bills.bill_creation` | yes |
| `Bill.BillExpiryDate` | read, never stored | no |
| `BillItems[].ItemId` | `bmc_bill_product.prod_code` | yes |
| `BillItems[].ItemCost/Qty/Tax/Discount` | `bmc_bills_items`, cast to `int` | yes, decimals dropped |
| generated | `bmc_bills.bill_num` = `E` + 14 digits → returned as `ControlNumber` | yes |

## 5. Control number generation

`Engine::saveBillDetails` → `getBillNumber()` (`Helpers/global_helper.php:929`).

```mermaid
flowchart TD
    A["getBillNumber()"] --> B["SELECT bill_num FROM bmc_bills<br/>ORDER BY bill_id DESC LIMIT 1"]
    B --> C{"any bill?"}
    C -- no --> D["E0000001 (7 digits)"]
    C -- yes --> E["strip E, +1,<br/>left-pad to 14 digits, prefix E<br/>e.g. E00000000123457"]
    D --> F{"isValidNewBillNumber:<br/>bill_num unused?"}
    E --> F
    F -- no --> E
    F -- yes --> G["use as bill_num"]
    G --> H["INSERT bmc_bills (separate step,<br/>not locked with the read)"]
    H -. "two concurrent requests read the same last row" .-> I["same number generated twice"]
```

Each bank then sees the control number with its own prefix: CRDB `BMH` + ref, NMB `SAS169` + ref,
Mkombozi `BMH999` + ref. PGS strips the prefix before lookup.

## 6. Reference lookup

`Engine::getBillDetailsForMalipoValidation($ref)` (`Models/Engine.php:1758`) is used by bill intake
(duplicate check), every bank validation, and every payment post.

```mermaid
flowchart TD
    A["ref"] --> B{"Pass 1: open bill where<br/>bill_num = ref OR control_num = ref?"}
    B -- yes --> C["typ = cnum<br/>amount = bill balance<br/>bank sees FIXED / EXACT"]
    B -- no --> D{"open bill for patient<br/>where pat_num = ref?"}
    D -- yes --> E["typ = wallet<br/>amount = all open balances<br/>bank sees FLEXIBLE / PARTIAL"]
    D -- no --> F{"retPatIdFromPatref:<br/>patient with pat_num = ref<br/>or any bill with bill_num = ref?"}
    F -- yes --> G["INSERT hidden dummy bill<br/>total = 1, is_hidden = 1"]
    G --> H["Pass 2 repeats the lookup"]
    H --> B
    F -- no --> I["Pass 2 repeats, then<br/>empty result: invalid reference"]
```

Side effect: a read-only lookup can write a bill. Any bank lookup on a patient number with no open
bills creates a dummy bill of TZS 1.

## 7. Bank / mobile-money validation flows

Before taking money, each channel asks PGS "is this reference payable and for how much".

```mermaid
sequenceDiagram
    autonumber
    participant Ch as Channel
    participant M as Malipo controller
    participant P as doCheckPermMalipoApi
    participant E as Engine lookup (section 6)
    participant L as bmc_gway_api_logs

    rect rgba(127,127,127,0.08)
    Note over Ch,L: Airtel Money (XML), wallet id 3
    Ch->>M: POST malipo/aipros1 COMMAND REFERENCE, AMOUNT
    M->>P: IP check (prefixes 13.2*, 41.7*, fixed IPs)
    M->>M: isValidAmount 1..9,999,999
    M->>E: doValidateReference(REFERENCE)
    M-->>Ch: XML status 200 / 400
    Ch->>M: aipros4 bill fetch / aipros5 lookup by CUSTOMERREF + MSISDN
    M-->>Ch: XML bill details
    end

    rect rgba(127,127,127,0.08)
    Note over Ch,L: CRDB (JSON), wallet id 4
    Ch->>M: POST malipo/crdbverify {paymentReference BMH..., token, checksum}
    M->>P: IP prefix 172.18.171.*
    Note right of M: token and checksum are never verified
    M->>E: lookup(ltrim(ref,'BMH'))
    M-->>Ch: {status, payerName, amount, amountType FIXED|FLEXIBLE}
    end

    rect rgba(127,127,127,0.08)
    Note over Ch,L: NMB via mpayafrica (JSON), wallet id 5
    Ch->>M: POST malipo/nmbbillrequest {reference SAS169..., token}
    M->>P: single fixed IP
    M->>E: lookup(ref without SAS169)
    M-->>Ch: nmbGenSubmitJson(bill)
    end

    rect rgba(127,127,127,0.08)
    Note over Ch,L: Mkombozi (AES-GCM JSON), wallet id 6
    Ch->>M: POST malipo/mkcbfetchcontrol encrypted {controlNumber BMH999...}
    M->>P: always TRUE (no IP restriction)
    M->>M: doMKCBDecrypt
    M->>E: lookup(ref without BMH999)
    M-->>Ch: encrypted {responseCode 200|050, payerName, amount, paymentType EXACT|PARTIAL}
    end

    M->>L: saveLogs(wallet_id, response) on every call
```

## 8. Payment posting and settlement

All channels converge on `doProcessGlobalTransaction($trans_sp, $req)` (`Helpers/global_helper.php:541`).

```mermaid
flowchart TD
    subgraph IN["Channel callbacks"]
        I1["Airtel aipros2 (XML)<br/>trans_sp 3"]
        I2["CRDB crdbpost (JSON)<br/>trans_sp 4"]
        I3["NMB nmbprocesscallback / nmbreq2<br/>trans_sp 5"]
        I4["Mkombozi mkcbprocesscontrol<br/>(AES-GCM) trans_sp 6"]
        I5["NMB nmbrecon pull<br/>missing receipts"]
    end
    I1 & I2 & I3 & I4 & I5 --> N["Normalise to<br/>{receipt, amount, msisdn, reference}<br/>strip BMH / SAS169 / BMH999<br/>amount cast to int"]
    N --> V1{"all four fields set?"}
    V1 -- no --> E400["400 Invalid Request"]
    V1 -- yes --> V2["retPatIdFromPatref(ref)<br/>lookup(ref) (section 6)"]
    V2 --> V3{"found?"}
    V3 -- no --> E403["403 Invalid Reference"]
    V3 -- yes --> V4{"typ = cnum and<br/>amount != bill balance?"}
    V4 -- yes --> E402["402 Invalid Amount"]
    V4 -- no --> V5{"receipt already in bmc_malipo?"}
    V5 -- yes --> OKD["200 Duplicate<br/>(no further processing)"]
    V5 -- no --> V6{"same reference + amount<br/>within 2 minutes?"}
    V6 -- yes --> E401["401 Double Payment"]
    V6 -- no --> S1["BEGIN<br/>INSERT bmc_malipo<br/>INSERT bmc_malipo_details (raw payload, IP)<br/>COMMIT"]
    S1 --> T{"typ"}
    T -- cnum --> C1["calcPatientBalance(pat)<br/>post payment into wallet ledger"]
    C1 --> C2["getGlobalBillId(ref)<br/>matches bill_num only"]
    C2 --> C3["prosPatientUnpaidBills(pat, bill)<br/>verify balance hash<br/>close bill(s) from wallet"]
    C3 --> Q["INSERT bmc_bill_cleared<br/>sync_status 0, synch_queue = now + 5s"]
    T -- wallet --> W1["saveWalletTransaction<br/>INSERT bmc_wallet_master entry_type 1<br/>credit patient wallet"]
    Q --> R200["200 Success + receipt"]
    W1 --> R200
    R200 --> RESP["Channel-specific response<br/>XML / JSON / encrypted JSON<br/>saveLogs()"]
    E400 & E403 & E402 & E401 & OKD --> RESP
```

## 9. Payment sync back to eHMS (cron)

```mermaid
sequenceDiagram
    autonumber
    participant C as Cron
    participant A as Api::postbillpayment
    participant E as Engine
    participant DB as MySQL
    participant H as eHMS BillPayment

    C->>A: GET /api/postbillpayment
    A->>E: cronstart(1) lock row in bmc_cron_log
    alt another run active
        E-->>C: die("Denied. Another instance...")
    end
    A->>E: doSynchBillPaymentsEHMS()
    E->>DB: getBillToSynch() ONE bill: sync_status 0 and synch_queue < now
    E->>DB: payment channels for that bill (wallet entry or bmc_malipo)
    E->>H: POST BATCH_PAYMENT_CONFIRM {BillNumber, ControlNumber, AmountPaid, BillTotal, PaymentReceipt, PayStatus, PaymentChannel[]}
    H-->>E: {ResponseKey, ResponseHash, BillNumber, StatusCode, StatusMsg}
    E->>DB: saveApiLogs(request + response)
    alt valid JSON, valid hash, BillNumber echoed
        E->>DB: bmc_bill_cleared.sync_status = 1
    else anything else
        E->>DB: sync_status = 2 + status_msg
    end
    A->>E: cronend(1)

    Note over C,DB: Retry job
    C->>A: GET /api/dosynchrepeat (cron id 2)
    A->>DB: latest 10 rows with sync_status 2 set back to 0
```

Throughput is one bill per cron run. eHMS `StatusCode` is read but not checked; only the echoed
`BillNumber` decides success.

## 10. Reconciliation and other jobs

```mermaid
flowchart LR
    subgraph JOBS["Scheduled / manual endpoints"]
        J1["api/domastersynch (cron id 3)<br/>roll 10 bmc_wallet_master rows<br/>into bmc_wallet_master_global"]
        J2["malipo/nmbrecon<br/>pull today's NMB transactions<br/>post missing receipts (section 8)"]
        J3["malipo/mkcbrecon<br/>debug only, hardcoded dates,<br/>posting commented out"]
        J4["malipo/nmbinvoicepush<br/>debug only, hardcoded ref E0000017<br/>submit / update invoice at NMB"]
        J5["test/lsp<br/>vendor kill-switch refresh (section 3)"]
        J6["api/getsinglepatbill, getbatchpatbill,<br/>getsynchpaymentrecords<br/>debug pulls with hardcoded ids, var_dump"]
    end
```

Staff UI flows (behind Joomla login): `Billing::postnewbill` creates manual bills through the same
`saveBillDetails` (there `control_num` = `bill_num`), plus patients, wallet statements,
transactions, refunds and reports.

## 11. Bill lifecycle state

```mermaid
stateDiagram-v2
    [*] --> Open: postpatbills or staff manual bill, is_closed=0
    [*] --> HiddenDummy: lookup side effect<br/>is_hidden=1, total=1
    Open --> Closed: exact payment (cnum)<br/>or wallet covers balance
    Open --> Open: wallet top-up insufficient
    Closed --> SyncPending: INSERT bmc_bill_cleared sync_status=0
    SyncPending --> Synced: eHMS echoed BillNumber, sync_status=1
    SyncPending --> SyncFailed: bad JSON / hash / no echo, sync_status=2
    SyncFailed --> SyncPending: dosynchrepeat (10 at a time)
    Synced --> [*]
```

There is no Expired or Cancelled state: `BillExpiryDate` is ignored and `nmbBillCancel` is never
called from a live path.

## 12. Endpoint inventory

| Endpoint | Caller | Auth today | Purpose |
|---|---|---|---|
| `api/postpatbills` | eHMS | IP allowlist (spoofable) + static key/hash | create bill, return control number |
| `api/postbillpayment` | cron | none | push one paid bill to eHMS |
| `api/dosynchrepeat` | cron | none | requeue failed syncs |
| `api/domastersynch` | cron | none | global ledger roll-up |
| `api/index`, `api/testpush`, `api/get*` | nobody | none | debug, leaks sample data |
| `malipo/aipros1..5` | Airtel | IP prefix | validate, pay, query, bill fetch, lookup |
| `malipo/crdbverify`, `crdbpost` | CRDB | IP prefix, token unchecked | validate, pay |
| `malipo/nmbbillrequest`, `nmbprocesscallback`, `nmbreq1/2` | NMB | single IP | validate, pay |
| `malipo/mkcbfetchcontrol`, `mkcbprocesscontrol` | Mkombozi | AES-GCM key only | validate, pay |
| `malipo/nmbrecon`, `mkcbrecon`, `nmbinvoicepush`, `index`, `test` | manual | none | recon and debug |
| `test/*` | anyone | none | balance recalculation, bill charging, kill switch |
| `MalipoOld/*`, `MalipoBk/*`, `MalipoApril/*`, `DashboardBkApril24/*` | anyone | none | old copies still routable |

## 13. Defects by flow

| # | Flow | Defect | Where |
|---|---|---|---|
| 1 | All | Vendor remote kill switch can block any endpoint; refresh endpoint is public | `Helpers/global_cache.php`, `global_helper.php:874`, `Controllers/Test.php:lsp`, `Engine::doGlobalUpdate` |
| 2 | All | Legacy auto-routing exposes every public method incl. debug and old copies | `Config/Routes.php` |
| 3 | All | SQL built by string interpolation of request values | `Models/Engine.php` throughout |
| 4 | All | Secrets in source control (DB, NMB, MKCB, eHMS shared secret) | `Config/Database.php`, `Config/Constants.php`, `global_helper.php`, `backup.sh` |
| 5 | Bill intake | Key/hash pair is replayable forever, not bound to body or time | `global_helper.php:1084-1104` |
| 6 | Bill intake | IP check trusts `X-Forwarded-For` / `Client-IP`; user 967 bypasses it | `global_helper.php:286`, `:1222` |
| 7 | Bill intake | Duplicate check ignores closed bills, so a paid bill can be re-created | `Engine.php:1758` |
| 8 | Bill intake | Lookup can insert a dummy bill and then reject the real one as duplicate | `Engine.php:1792-1821` |
| 9 | Bill intake | Money cast to `int`; `BillTotal` never compared; tax/discount assumed percent | `Api.php:153-195` |
| 10 | Control number | Generated outside the transaction, no lock: concurrent duplicates | `global_helper.php:929` |
| 11 | Control number | First number 7 digits, later 15 characters | `global_helper.php:942` |
| 12 | Validation | CRDB `token`/`checksum` never verified; Mkombozi has no IP restriction | `Malipo.php:276`, `global_helper.php:213` |
| 13 | Validation | `ltrim($ref,'BMH')` strips any leading B/M/H characters, not the prefix | `global_helper.php:350`, `:489` |
| 14 | Payment | `getGlobalBillId` matches `bill_num` only; payment by hospital bill number settles nothing specific | `Engine.php:2540` |
| 15 | Payment | Duplicate-receipt check and insert are not atomic (no unique key relied on) | `Engine.php:2027` |
| 16 | eHMS sync | One bill per run; eHMS `StatusCode` ignored; HTTP not TLS | `Engine.php:342` |
| 17 | Errors | `die()` everywhere, HTTP status always 200, `error_reporting(0)` | `public/index.php`, helpers |
| 18 | Structure | Two frameworks per request; absolute Windows path; 3,000-line model; global `get_instance()` | `public/joom.php`, `Models/Engine.php` |

---

## 14. Target architecture for the rewrite

### 14.1 Principles

- One application, one framework. Staff auth moves into the app (no Joomla).
- Explicit routes only. No auto-routing. Debug endpoints do not ship.
- Each external party gets its own adapter with its own authentication, and talks to the same
  core services.
- Money is stored as integer cents (TZS has no minor unit in practice, so store whole shillings as
  `BIGINT` and reject fractional input explicitly rather than truncating).
- Every write that must happen once is enforced by a database unique key, not by a prior SELECT.
- Calls out to eHMS go through a transactional outbox and a worker with retry and backoff.
- Real HTTP status codes; structured JSON errors; no `die()`.
- Secrets come from environment / a secret store, never from source.

### 14.2 Component view

```mermaid
flowchart TB
    subgraph EXT["External parties"]
        EHMS["eHMS"]
        AIR["Airtel"]
        CRDB["CRDB"]
        NMB["NMB / mpayafrica"]
        MKCB["Mkombozi"]
        STAFF["Staff browser"]
    end

    subgraph PGS["PGS (rewrite)"]
        subgraph EDGE["Inbound adapters"]
            HG["Hospital gateway<br/>HMAC body + timestamp + nonce"]
            CA["Channel adapters<br/>one per bank / MNO<br/>own auth + format"]
            UI["Staff web UI<br/>app-native auth + RBAC"]
        end
        subgraph CORE["Core services"]
            BS["Billing service<br/>bills, items, totals"]
            CN["Control number service<br/>DB sequence + check digit"]
            PS["Payment service<br/>idempotent on channel + receipt"]
            LG["Ledger<br/>double-entry, patient wallets"]
        end
        subgraph ASYNC["Async"]
            OB[("Outbox table")]
            WK["Sync worker<br/>retry + backoff + DLQ"]
            RC["Reconciliation jobs<br/>per channel, daily"]
        end
        AUD[("Audit + API log")]
        DB[("MySQL / PostgreSQL")]
    end

    EHMS --> HG
    AIR & CRDB & NMB & MKCB --> CA
    STAFF --> UI
    HG --> BS
    UI --> BS
    BS --> CN
    CA --> BS
    CA --> PS
    PS --> LG
    PS --> OB
    BS --> OB
    OB --> WK
    WK --> EHMS
    RC --> CA
    RC --> PS
    BS & PS & LG & OB --> DB
    HG & CA & UI --> AUD
```

### 14.3 Target: bill creation and control number

```mermaid
sequenceDiagram
    autonumber
    participant H as eHMS
    participant G as Hospital gateway
    participant B as Billing service
    participant N as Control number service
    participant DB as Database

    H->>G: POST /v1/hospital/bills<br/>headers: X-Client-Id, X-Timestamp, X-Nonce, X-Signature=HMAC-SHA256(secret, ts.nonce.body)<br/>Idempotency-Key: hospital BillNumber
    G->>G: verify signature, |now - ts| < 5 min, nonce unused
    alt bad signature / stale / replayed
        G-->>H: 401 {error}
    end
    G->>G: validate schema (decimal strings, qty > 0, currency TZS)
    alt invalid
        G-->>H: 422 {error, field}
    end
    G->>B: createBill(command)
    B->>DB: BEGIN
    B->>DB: SELECT bill by (hospital_id, hospital_bill_no) FOR UPDATE
    alt exists, same payload hash
        B->>DB: COMMIT
        B-->>G: existing bill
        G-->>H: 200 same response as first time (idempotent)
    else exists, different payload
        B->>DB: ROLLBACK
        G-->>H: 409 conflict
    end
    B->>B: recompute total from items, compare to BillTotal
    alt mismatch
        G-->>H: 422 total mismatch
    end
    B->>DB: upsert patient by (hospital_id, patient_no)
    B->>N: next()
    N->>DB: sequence nextval (atomic)
    N-->>B: control number = prefix + zero-padded seq + check digit
    B->>DB: INSERT bill (UNIQUE control_no, UNIQUE hospital_id+hospital_bill_no), items
    B->>DB: INSERT outbox BillCreated
    B->>DB: COMMIT
    G-->>H: 201 {controlNumber, billNumber, amount, expiresAt} + X-Signature over response body
```

### 14.4 Target: channel validation and payment

```mermaid
sequenceDiagram
    autonumber
    participant Ch as Bank / MNO
    participant A as Channel adapter
    participant B as Billing service
    participant P as Payment service
    participant L as Ledger
    participant DB as Database

    Ch->>A: validate(ref) in channel format
    A->>A: authenticate (mTLS / signature / token per channel contract)
    A->>A: strip channel prefix by exact match
    A->>B: findPayable(controlNo)  read-only, no side effects
    B-->>A: bill {payer, balance, amountType} or not found / expired / closed
    A-->>Ch: channel-format response

    Ch->>A: pay(ref, amount, receipt)
    A->>P: postPayment(channel, receipt, controlNo, amount)
    P->>DB: BEGIN
    P->>DB: INSERT payment (UNIQUE channel + receipt)
    alt unique violation
        P->>DB: ROLLBACK
        P-->>A: already processed, return original result
    end
    P->>B: lock bill FOR UPDATE, check amount rules (exact / partial / overpay to wallet)
    P->>L: journal: debit channel clearing, credit bill receivable (and wallet for excess)
    P->>B: update bill paid amount, status PAID when balance = 0
    P->>DB: INSERT outbox PaymentReceived
    P->>DB: COMMIT
    A-->>Ch: success with PGS receipt
```

### 14.5 Target: eHMS sync worker

```mermaid
flowchart TD
    A["Outbox row PENDING"] --> B["Worker claims batch<br/>SELECT ... FOR UPDATE SKIP LOCKED"]
    B --> C["POST to eHMS over HTTPS<br/>signed body, Idempotency-Key = event id"]
    C --> D{"2xx and signed response valid<br/>and StatusCode ok?"}
    D -- yes --> E["mark SENT"]
    D -- "4xx (permanent)" --> F["mark FAILED, alert"]
    D -- "5xx / timeout" --> G["attempts + 1<br/>next_try = now + backoff"]
    G --> H{"attempts > max?"}
    H -- no --> A
    H -- yes --> F
```

### 14.6 Target bill states

```mermaid
stateDiagram-v2
    [*] --> Issued: bill created, control number assigned
    Issued --> PartiallyPaid: payment < balance (if partial allowed)
    Issued --> Paid: payment = balance
    PartiallyPaid --> Paid: remaining balance paid
    Issued --> Expired: expiresAt passed (job)
    Issued --> Cancelled: eHMS cancels
    PartiallyPaid --> Cancelled: eHMS cancels, refund flow
    Paid --> Refunded: staff refund approved
    Paid --> [*]
    Expired --> [*]
    Cancelled --> [*]
    Refunded --> [*]
```

Each state change writes an outbox event, so eHMS learns about payments, expiry and cancellation
through one mechanism.

### 14.7 Core data model

```mermaid
erDiagram
    HOSPITAL ||--o{ PATIENT : has
    HOSPITAL ||--o{ BILL : issues
    PATIENT ||--o{ BILL : owes
    BILL ||--|{ BILL_ITEM : contains
    BILL ||--o{ PAYMENT : settled_by
    CHANNEL ||--o{ PAYMENT : carries
    PATIENT ||--|| WALLET : owns
    PAYMENT ||--|{ LEDGER_ENTRY : posts
    WALLET ||--o{ LEDGER_ENTRY : moves
    BILL ||--o{ OUTBOX_EVENT : emits
    PAYMENT ||--o{ OUTBOX_EVENT : emits

    BILL {
        bigint id PK
        bigint hospital_id FK
        string hospital_bill_no "UNIQUE with hospital_id"
        string control_no "UNIQUE"
        bigint amount_total
        bigint amount_paid
        string status
        datetime expires_at
        string payload_hash
    }
    PAYMENT {
        bigint id PK
        bigint channel_id FK
        string channel_receipt "UNIQUE with channel_id"
        bigint bill_id FK
        bigint amount
        datetime paid_at
    }
    OUTBOX_EVENT {
        bigint id PK
        string type
        json payload
        string status
        int attempts
        datetime next_try_at
    }
```

### 14.8 Migration path

1. **Contain now (current system)**: rotate every committed secret; remove the kill switch and
   `Test`, debug and old-copy controllers; turn off auto-routing and list routes explicitly; put
   eHMS and bank endpoints behind TLS.
2. **Freeze contracts**: write OpenAPI specs for the eHMS API and each channel's current wire
   format, so adapters can keep the same external contract while the core changes.
3. **Build core + one adapter**: billing, control numbers, payments, ledger, outbox; ship the
   hospital gateway and one channel (the highest-volume one) first.
4. **Migrate data**: import `bmc_bills`, `bmc_malipo`, wallet master; keep existing control numbers
   valid by reserving the legacy `E...` range and starting the new sequence above it.
5. **Cut over per channel**: point each bank to the new endpoint one at a time, with daily
   reconciliation against the old system until they match.
6. **Decommission**: retire the CodeIgniter/Joomla stack once all channels and eHMS are moved.
