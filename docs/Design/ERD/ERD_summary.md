# PayLink Database: Tables, Columns and Relations
 
This document explains the physical ERD (26 tables, 9 modules). For each table it gives the purpose, the columns, and the story or decision it serves. The last part explains every relation.
 
**Conventions**
- PK = primary key, FK = foreign key, UK = unique.
- All money columns are `decimal(18,2)`, never float or double (NFR-7).
- All timestamps are stored in UTC. The "day" for limits is computed in Africa/Cairo (D-13).
- A balance is never stored. It is the sum of `ledger_entries` (FR-5, US-07).
- Between modules, the code refers to other modules by id only. Whether these are real database FKs is still to be decided (see section 4).
---
 
## 1. Tables by module
 
### 1.1 Accounts and Wallet
 
#### accounts
**Purpose:** Every person who uses the system: customer, agent, support, admin, or the system itself. One role per account (D-20, pending). Serves US-01, US-02.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Account identifier |
| phone | string | UK | Phone number, one account per phone (US-01) |
| name | string | | Display name |
| role | string | | CUSTOMER, AGENT, ADMIN, SUPPORT, SYSTEM |
| level | string | | BASIC or VERIFIED. Decides the daily limit (5,000 or 30,000 EGP) |
| created_at | timestamp | | Registration time (UTC) |
 
#### wallets
**Purpose:** One wallet per account. It holds no balance column. It is the owner that ledger entries belong to. Serves FR-3.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Wallet identifier |
| account_id | uuid | FK, UK | The owning account (exactly one wallet per account) |
 
#### ledger_entries
**Purpose:** The immutable list of every money movement. Balances are computed from it. Rows are only added, never edited or deleted. Serves FR-5, US-07, NFR-1.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Entry identifier |
| wallet_id | uuid | FK | Wallet that is affected |
| payment_id | uuid | FK | Payment that caused it. Null for top-ups |
| type | string | | TOPUP, HOLD, RELEASE, CHARGE, FEE_INCOME, COMMISSION, REFUND |
| amount | decimal | | Amount of the movement |
| created_at | timestamp | | When it happened (UTC) |
 
---
 
### 1.2 Biller Catalog
 
#### billers
**Purpose:** The companies that issue bills (ElectroCo, AquaCo, TopUpNet, NetCo). The admin can add, edit or disable them. Serves FR-6, US-09, US-10.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Biller identifier |
| code | string | UK | Short code used to pick the adapter and configuration |
| name | string | | Display name |
| category | string | | ELECTRICITY, WATER, MOBILE, INTERNET |
| active | boolean | | False hides the biller and rejects new payments |
 
#### biller_services
**Purpose:** The services a biller offers, each with its allowed amount range. Serves FR-7, US-08.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Service identifier |
| biller_id | uuid | FK | The biller that offers it |
| name | string | | Service name |
| min_amount | decimal | | Smallest allowed payment |
| max_amount | decimal | | Largest allowed payment |
| active | boolean | | False disables only this service |
 
#### input_fields
**Purpose:** The details a customer must enter for a service (meter number, phone number). Serves FR-7, US-08.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Field identifier |
| biller_service_id | uuid | FK | Service that requires it |
| name | string | | Technical name (for example meter_number) |
| label | string | | Text shown to the customer |
| validation_pattern | string | | Pattern the input must match |
 
---
 
### 1.3 Fees and Commissions
 
#### fee_rules
**Purpose:** How the fee of one service is calculated. It is the data side of the Strategy pattern: `type` picks the calculation and `params` holds the numbers. Serves FR-16, US-11.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Rule identifier |
| biller_service_id | uuid | FK, UK | Service priced by this rule (one rule per service) |
| type | string | | FIXED, PERCENTAGE, TIERED |
| params | json | | Numbers for the type (for example fixed 2.50, or 1.5% with minimum 1) |
 
#### promotions
**Purpose:** A period during which one biller has a zero fee. Serves US-12.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Promotion identifier |
| biller_id | uuid | FK | The one biller on promotion |
| start_date | date | | First day of the promotion |
| end_date | date | | Last day of the promotion |
 
#### commissions
**Purpose:** The agent's share of the fee (40%), credited once when a payment is Completed. The unique `payment_id` is the database layer of D-9, so a duplicate callback cannot pay a second commission. Serves US-14, FR-17.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Commission identifier |
| payment_id | uuid | FK, UK | Payment that earned it (one commission per payment) |
| agent_id | uuid | FK | Agent account that receives it |
| amount | decimal | | 40% of the payment's fee (0.00 when the fee is 0) |
| credited_at | timestamp | | When it was credited (UTC) |
 
#### config_settings
**Purpose:** Values that the admin can change without redeploying: limits, timeouts, retry counts, commission percentage, settlement safety margin. Serves FR-24, US-15.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| key | string | PK | Setting name |
| value | string | | Current value, parsed in code |
| updated_at | timestamp | | Last change (UTC). Who changed it is in `audit_records` |
 
---
 
### 1.4 Payment Processing
 
#### bill_inquiries
**Purpose:** A saved price check: what the biller says is owed, the fee and the total. The customer sees it before paying. Serves FR-8, US-13. The expiry is pending D-21.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Inquiry identifier |
| biller_service_id | uuid | FK | Service that was asked about |
| account_id | uuid | FK | Account that asked |
| input_values | json | | What the customer entered (for example the meter number) |
| bill_amount | decimal | | Amount the biller reported |
| fee | decimal | | Fee calculated for it |
| total | decimal | | bill_amount + fee |
| expires_at | timestamp | | When the quote stops being valid (pending D-21) |
 
#### payments
**Purpose:** The main record of one payment: who pays, what, how much, and its current state. Serves FR-9, FR-11, US-16, US-18.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Payment identifier |
| reference | uuid | UK | Unique reference sent to the biller and reused on every retry (D-3) |
| customer_id | uuid | FK | Account whose wallet is debited (D-6) |
| agent_id | uuid | FK | Agent who paid on behalf of the customer. Null for app payments |
| biller_service_id | uuid | FK | Service being paid |
| inquiry_id | uuid | FK | Quote it is based on. Null when there is no bill (TopUpNet, NetCo) |
| amount | decimal | | Bill amount. Limits count this value only (D-14) |
| fee | decimal | | Fee snapshot, never recalculated (US-15) |
| total | decimal | | amount + fee, what is held |
| state | string | | INITIATED, FUNDS_HELD, SENT_TO_BILLER, COMPLETED, FAILED, UNKNOWN, REJECTED, REFUNDED |
| created_at | timestamp | | Creation time (UTC). Decides which day it belongs to (D-13) |
 
#### payment_state_changes
**Purpose:** The history of one payment, one row per state move, with a reason. Support reads it to see what happened. Serves FR-20, US-18, US-35. It is a weak entity: it has no meaning without its payment.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Change identifier |
| payment_id | uuid | FK | Payment that changed |
| from_state | string | | State before |
| to_state | string | | State after |
| reason | string | | Why it changed |
| at | timestamp | | When (UTC) |
 
#### idempotency_records
**Purpose:** Remembers that an account already sent a request key, so a repeat returns the first result and never charges twice. Serves FR-10, US-17, D-15, §2.7 case 1.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Record identifier |
| account_id | uuid | FK | Account that sent the key. Unique together with `key` |
| key | string | | The idempotency key. Unique together with `account_id` |
| request_hash | string | | Hash of the request, to reject the same key with a different amount or biller |
| payment_id | uuid | FK, UK | The payment this key created |
| stored_response | json | | The first result, returned on a repeat |
 
#### refund_requests
**Purpose:** A customer's request to get money back after a payment Completed. Support approves or rejects it. The payment itself stays Completed, so the refund is a separate record (D-18). Serves FR-27, US-37.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Request identifier |
| payment_id | uuid | FK | Completed payment concerned |
| requested_by | uuid | FK | Account that asked |
| decided_by | uuid | FK | Support account that decided. Null while pending |
| amount | decimal | | Amount to return (the bill amount) |
| reason | string | | Customer's reason |
| status | string | | PENDING, APPROVED, REJECTED |
| created_at | timestamp | | When requested (UTC) |
| decided_at | timestamp | | When decided (UTC). Null while pending |
 
---
 
### 1.5 Fraud Checks
 
#### account_blocks
**Purpose:** One row per block on an account. A block is temporary (velocity, 10 minutes) or manual (support, no expiry). An account can pay only if no block is active. Serves FR-19, FR-21, US-23, US-24, D-10.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Block identifier |
| account_id | uuid | FK | Blocked account |
| blocked_by | uuid | FK | Who placed it (the SYSTEM account for velocity) |
| lifted_by | uuid | FK | Who lifted it. Null while active |
| type | string | | TEMPORARY_VELOCITY or MANUAL_SUPPORT |
| reason | string | | Why it was placed |
| blocked_at | timestamp | | Start (UTC) |
| until | timestamp | | Expiry. Null for support blocks |
| lifted_at | timestamp | | When lifted. Null while active |
 
---
 
### 1.6 Biller Integration
 
#### biller_calls
**Purpose:** A log of every attempt to talk to a biller, including retries, with the raw request and answer. It supports Unknown payments and support investigations. Serves FR-13, US-28, US-29.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Call identifier |
| payment_id | uuid | FK | Payment the call belongs to. Unique together with `attempt_no` |
| biller_id | uuid | FK | Biller that was called |
| attempt_no | int | | 1 for the first call, then each retry. Unique together with `payment_id` |
| biller_reference | string | | The biller's own reference for the call |
| status | string | | CONFIRMED, REFUSED, ACCEPTED_PENDING, TIMED_OUT |
| raw_request | text | | What PayLink sent |
| raw_response | text | | What the biller answered |
| at | timestamp | | When (UTC) |
 
#### biller_callbacks
**Purpose:** A log of the biller's late answers (for example TopUpNet's "success" minutes later), with what the system did with each one. A duplicate is logged and ignored. Serves FR-14, US-27, §2.7 case 6.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Callback identifier |
| payment_id | uuid | FK | Matched payment. Null when the reference is unknown |
| reference | string | | Reference in the callback |
| payload | text | | Raw body received |
| outcome | string | | PROCESSED, DUPLICATE_IGNORED, UNKNOWN_REFERENCE |
| received_at | timestamp | | When received (UTC) |
 
---
 
### 1.7 Notifications
 
#### notification_templates
**Purpose:** The message texts, one per outcome and channel, so no text is hard-coded in the payment code. Serves FR-22, US-31.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Template identifier |
| code | string | | Outcome name (for example PAYMENT_COMPLETED). Unique together with `channel` |
| channel | string | | SMS, EMAIL, PUSH. Unique together with `code` |
| text | string | | Message text with placeholders |
 
#### notifications
**Purpose:** One message to one person about one payment, with its delivery status. Serves FR-22, US-31.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Notification identifier |
| template_id | uuid | FK | Template it was filled from |
| recipient_id | uuid | FK | Account that receives it |
| payment_id | uuid | FK | Payment it is about |
| channel | string | | SMS, EMAIL, PUSH |
| body | string | | Final text after filling the template |
| status | string | | PENDING, DELIVERED, UNDELIVERED |
 
#### notification_attempts
**Purpose:** Each try to send a notification. After 3 failed retries the notification becomes UNDELIVERED. Serves FR-23, US-32, §2.7 case 5. It is a weak entity.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Attempt identifier |
| notification_id | uuid | FK | Notification being sent |
| attempt_no | int | | Try number |
| success | boolean | | Whether the provider accepted it |
| at | timestamp | | When (UTC) |
 
---
 
### 1.8 Reconciliation
 
#### settlement_files
**Purpose:** The biller's nightly file as received, with the period it covers. The hash stops the same file from being processed twice. Serves FR-26, US-33, D-11.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | File identifier |
| biller_id | uuid | FK | Biller that sent it |
| business_date | date | | Day the file is for |
| period_from | timestamp | | Start of the covered period (UTC) |
| period_to | timestamp | | End of the covered period (UTC) |
| file_hash | string | UK | Fingerprint of the file content |
| status | string | | Processing status |
 
#### settlement_entries
**Purpose:** The lines inside a settlement file, one per payment the biller says it received. Weak entity.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Entry identifier |
| file_id | uuid | FK | File it belongs to |
| reference | string | | Payment reference in the file |
| amount | decimal | | Amount the biller recorded |
| biller_status | string | | Status according to the biller |
 
#### reconciliation_reports
**Purpose:** The result of one comparison run for one file. Serves US-33.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Report identifier |
| file_id | uuid | FK | File that was compared |
| business_date | date | | Day of the report |
| created_at | timestamp | | When the report was made (UTC) |
 
#### reconciliation_items
**Purpose:** One line per payment in a report: matched, missing, or different, with both amounts and states. A flagged difference never changes a payment or wallet automatically (D-11). Serves US-33, US-34, §2.7 case 7.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Item identifier |
| report_id | uuid | FK | Report it belongs to |
| payment_id | uuid | FK | Matching payment. Null when PayLink has none |
| reference | string | | Payment reference |
| result | string | | MATCHED, MISSING_IN_PAYLINK, MISSING_IN_FILE, AMOUNT_DIFFERENT, STATE_CONFLICT |
| paylink_amount | decimal | | Amount in PayLink. Null if missing |
| file_amount | decimal | | Amount in the file. Null if missing |
| paylink_state | string | | PayLink's payment state. Null if missing |
| reviewed | boolean | | Whether support reviewed it |
 
---
 
### 1.9 Audit
 
#### audit_records
**Purpose:** The permanent log of who did what, when and why, for every money movement and admin action. It cannot be edited or deleted. Serves FR-25, US-36, NFR-5.
 
| Column | Type | Key | Meaning |
|---|---|---|---|
| id | uuid | PK | Record identifier |
| actor_id | uuid | FK | Account that acted |
| action | string | | What was done |
| entity_type | string | | Kind of thing changed (ledger entry, biller, fee rule...). No FK |
| entity_id | uuid | | Id of the thing changed. No FK |
| old_value | text | | Value before (admin actions) |
| new_value | text | | Value after (admin actions) |
| reason | string | | Why |
| at | timestamp | | When (UTC) |
 
`entity_type` + `entity_id` is a polymorphic link, so it has no real FK. This is deliberate: one table covers ledger entries and many kinds of admin actions.
 
---
 
## 2. Relations explained
 
Format: parent **relation** child (cardinality). Cardinality reads from the parent side.
 
### Accounts and Wallet
- **accounts Owns wallets (1 to 1):** every account has exactly one wallet, and every wallet belongs to one account. Agents have a wallet too, which receives their commission.
- **wallets Records ledger_entries (1 to many):** a wallet's balance is the sum of its entries.
- **payments Causes ledger_entries (0..1 to many):** one payment creates several entries (hold, charge, fee, commission). Top-ups have no payment, so `payment_id` is nullable.
### Biller Catalog
- **billers Offers biller_services (1 to 1..many):** a biller has at least one service.
- **biller_services Requires input_fields (1 to many):** each service lists the details to enter.
### Fees and Commissions
- **biller_services Priced by fee_rules (1 to 1):** exactly one fee rule per service.
- **billers Has promotion promotions (1 to many):** one biller can have promotions over time.
- **payments Earns commissions (1 to 0..1):** a payment earns at most one commission, enforced by the unique `payment_id`.
- **accounts Receives as agent commissions (1 to many):** an agent account can receive many commissions.
### Payment Processing
- **accounts Requests bill_inquiries (1 to many):** an account can run many price checks.
- **biller_services Quotes bill_inquiries (1 to many):** a service can be quoted many times.
- **accounts Makes as customer payments (1 to many):** the customer whose wallet is debited.
- **accounts Acts for as agent payments (0..1 to many):** an agent can make many payments. A payment may have no agent.
- **biller_services Pays payments (1 to many):** each payment targets one service.
- **bill_inquiries Quoted as payments (0..1 to 0..1):** a payment may be based on one inquiry. TopUpNet and NetCo have no inquiry.
- **payments Goes through payment_state_changes (1 to many):** the history of the payment.
- **accounts Sends idempotency_records (1 to many):** keys belong to the sending account (D-15).
- **payments Guarded by idempotency_records (1 to 1):** one key record per payment.
- **payments May have refund_requests (1 to many):** a payment can have several requests, but only one approved (US-37).
- **accounts Requests refund_requests (1 to many):** the customer who asked.
 
### Fraud Checks
- **accounts Blocked by account_blocks (1 to many):** an account can have several blocks at once (velocity and support, D-10).
### Biller Integration
- **payments Attempts biller_calls (1 to many):** the first call plus every retry.
- **billers Received by biller_calls (1 to many):** which biller was called.
- **payments Matched by biller_callbacks (0..1 to many):** callbacks are matched by reference. Unknown references have no payment.
### Notifications
- **notification_templates Fills notifications (1 to many):** many messages use one template.
- **payments Triggers notifications (1 to many):** completion and failure each create messages.
- **accounts Sent to notifications (1 to many):** the recipient.
- **notifications Tries notification_attempts (1 to many):** up to the first try plus 3 retries.
### Reconciliation
- **billers Sends settlement_files (1 to many):** one file per biller per period.
- **settlement_files Lists settlement_entries (1 to many):** the lines of the file.
- **settlement_files Builds reconciliation_reports (1 to many):** reports built from the file.
- **reconciliation_reports Contains reconciliation_items (1 to many):** one item per compared payment.
- **payments Compared in reconciliation_items (0..1 to many):** a missing-in-PayLink item has no payment.
### Audit
- **accounts Performs audit_records (1 to many):** the actor of each record.
---
 
## 3. Constraints that protect the hard cases
 
| Constraint | Table | Protects | Serves |
|---|---|---|---|
| UK reference | payments | Stable reference for retries and reconciliation | D-3 |
| UK (account_id, key) | idempotency_records | No double charge on repeated requests | D-15, §2.7 case 1 |
| UK payment_id | commissions | One commission per payment, even with duplicate callbacks | D-9, §2.7 case 6 |
| UK (payment_id, attempt_no) | biller_calls | No duplicate attempt records | US-29 |
| UK (code, channel) | notification_templates | One template per outcome and channel | US-31 |
| UK file_hash | settlement_files | A file is never processed twice | US-33 |
| UK account_id | wallets | One wallet per account | US-01 |
| Append-only | ledger_entries, audit_records | Nothing is edited or deleted | US-07, US-36 |
 
---
 
## 4. Open points and risks
 
- **Pending decisions:** login credentials on `accounts` (D-20) and `bill_inquiries.expires_at` (D-21) are not final.
- **PayLink system wallet:** fee income and refunds after Completed need a PayLink wallet. Only the SYSTEM role hints at it today. This needs a decision.
- **Sign rules for the ledger:** the document does not yet say how HOLD, RELEASE and CHARGE change available versus held balance. Write this down before coding the balance calculation.
- **Cross-module foreign keys:** real database FKs between modules weaken the modular monolith and make the later event-driven split harder (NFR-4). Decide between real FKs and stored ids only.
- **Index needed:** the daily-limit query needs an index on `payments (customer_id, created_at)` to support 100 payments per second (NFR-2).
- **Composite unique keys and indexes** cannot be drawn in Mermaid. They must be written in the migration scripts.
- **Unknown retry schedule:** the 1, 5 and 15 minute re-queries have no table yet. They may be derived from `biller_calls` or need their own table.