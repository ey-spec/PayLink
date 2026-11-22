# PayLink Requirements

## 1. Problem Summary

PayLink is an Egyptian company, similar to Fawry or Vodafone Cash, that lets people pay
utility bills (electricity, water, mobile top-up, internet) from a digital wallet. Customers
pay through a mobile app or through agents, who are small shops with a PayLink terminal.
Billers are the utility companies that issue the bills and receive the money.

PayLink earns a service fee on each payment and pays agents a commission. It also holds
customers' money in wallets, so it must be exact: a customer must never be charged twice,
charged for a payment the biller never received, or left with a negative balance.

We are building the backend that does this. A payment is checked, its money is held in the
wallet, and it is sent to the biller. The money is only charged once the biller confirms.
The hard part is failure: billers can be slow, can refuse, can confirm late, or can send
the same callback twice. The system must handle each case safely and keep a record of
every money movement. Each night it compares its records with the billers' settlement
files and reports any difference.

The system must also be easy to extend. Adding a new biller, or changing one biller's API,
must only touch that biller's integration and configuration, not the rest of the system.

## 2. Stakeholders

| Stakeholder | Role | Main interest |
|---|---|---|
| Customer | Pays bills from a wallet via the app | Fast, correct payments; clear status and receipts; money is never lost or double-charged |
| Agent | Pays on behalf of customers at a PayLink terminal | Reliable payments; commission credited when the payment completes |
| Biller (ElectroCo, AquaCo, TopUpNet, NetCo) | Issues bills and receives payments | Being paid correctly and quickly; matching records at settlement |
| Admin | Manages billers, services, fees, and limits | Change rules and billers without redeploying; view daily totals |
| Support employee | Investigates problems | Search a payment by reference and see its full history; block or unblock accounts; review reconciliation differences |
| PayLink management | Owns the business | Fee and commission income; trust; handling the busiest day of the month |
| Regulators | Oversee companies that hold customer money | Audit records of money movements; no unexplained charges |
| Notification providers (SMS, email, push) | External services that deliver messages | Receive messages in the expected format; they can fail and need retries |
| Development team (us) | Builds and maintains PayLink | Clear requirements; a modular system that is testable and deployable |
| Instructor and TAs | Grade the project | Requirements met, AI use shown honestly, decisions explained |


## 3. User Stories (titles only)

### E1. Accounts and Wallet
- US-01 Register as a customer with a basic level (FR-1)
- US-02 Upgrade a customer to verified (FR-2)
- US-03 View available and held balance (FR-3)
- US-04 View transaction history (FR-3)
- US-05 Top up the wallet by simulated card (FR-4)
- US-06 Top up the wallet by agent cash (FR-4)
- US-07 Record a ledger entry for every balance change (FR-5)

### E2. Biller Catalog
- US-08 See the list of billers and their required input fields (FR-7)
- US-09 Admin adds and edits a biller and its services (FR-6)
- US-10 Admin disables a biller or service (FR-6)

### E3. Fees and Commissions
- US-11 Calculate the fee by biller rule: fixed, percentage, tiered (FR-16)
- US-12 Apply a promotion period with a zero fee (FR-16)
- US-13 Run a bill inquiry and see amount, fee and total (FR-8)
- US-14 Credit the agent commission when a payment completes (FR-17)
- US-15 Admin edits fee and limit values without redeploying (FR-24)

### E4. Payment Processing
- US-16 Pay a bill from the wallet (FR-9)
- US-17 Return the original result for a repeated idempotency key (FR-10, §2.7 case 1)
- US-18 Allow only the payment lifecycle transitions (FR-11)
- US-19 Hold funds before the biller call and release them on failure (FR-12)
- US-20 Reject one of two simultaneous payments when the balance covers only one (§2.7 case 4)
- US-21 Agent pays on behalf of a customer (FR-17)

### E5. Fraud Checks
- US-22 Reject payments above the single or daily limit with a clear reason (FR-18)
- US-23 Block an account after more than 5 payments in 60 seconds (FR-19)
- US-24 Support blocks and unblocks an account (FR-21)

### E6. Biller Integration
- US-25 Pay an ElectroCo bill (REST/JSON) (FR-15)
- US-26 Pay an AquaCo bill (XML) and handle its refusal (FR-15, §2.7 case 3)
- US-27 Accept the TopUpNet asynchronous callback and ignore duplicates (FR-14, §2.7 case 6)
- US-28 Mark a payment Unknown when the biller times out (FR-13, §2.7 case 2)
- US-29 Re-query an Unknown payment after 1, 5 and 15 minutes (FR-13)
- US-30 Add a new biller by adding only an integration and configuration (§2.7 case 8)

### E7. Notifications
- US-31 Send SMS and push on completion and on failure (FR-22)
- US-32 Retry a failed notification 3 times, then mark it undelivered (FR-23, §2.7 case 5)

### E8. Reconciliation, Support and Audit
- US-33 Produce a daily reconciliation report from a settlement file (FR-26)
- US-34 Flag a payment that the settlement file lists but PayLink has as Failed (§2.7 case 7)
- US-35 Support searches a payment by reference and sees its history (FR-20)
- US-36 Write an audit record for every money movement and admin action (FR-25)
- US-37 Request a refund of a completed payment and support approves it (FR-27)
- US-38 Admin views daily totals (FR-28)

## 4. Non-Functional Requirements (measurable)

| ID | Quality | Requirement | How we measure it | Target | Priority |
|---|---|---|---|---|---|
| NFR-1 | Correctness | No customer is charged twice and no balance goes negative | Automated duplicate-request and concurrent-payment tests (§2.7 cases 1 and 4) | 0 double charges, 0 negative balances | Must |
| NFR-2 | Performance | The system handles a sustained payment load against the mock billers | Load test of 100 payments per second for 1 minute, reporting throughput and failed requests | 100 payments/s for 60 s | Should |
| NFR-3 | Resilience | A biller outage does not stop payments to other billers | Test with one mock biller switched off while the others receive payments | 100% of payments to the other billers still processed | Must |
| NFR-4 | Modifiability | Adding a biller touches only the Biller Integration module and configuration | `git diff` of the pull request that adds a biller (§2.7 case 8) | 0 files changed outside that module and configuration | Must |
| NFR-5 | Auditability | Every money movement has an audit record with who, when and why | Query comparing ledger entries with audit records | 100% of ledger entries have a matching audit record | Must |
| NFR-6 | Testability | Domain logic is well covered by automated tests | Line coverage reported by the CI pipeline | At least 70% on domain logic | Must |
| NFR-7 | Precision | All money values use exact decimal types | Code search and lint rule for floating-point types in money code, plus rounding tests (§2.6) | 0 float or double types for money | Must |

## 5. Priorities (Must / Should / Could)

Priorities of FR-1 to FR-28 are copied from the brief. A story takes the priority of the FR it
serves. Must items cannot be removed (§3.2).

| Priority | User stories |
|---|---|
| **Must** | US-01, 03, 04, 05, 06, 07, 08, 09, 10, 11, 12, 13, 16, 17, 18, 19, 20, 22, 25, 26, 27, 28, 29, 30, 31, 35, 36 |
| **Should** | US-02, 14, 15, 21, 23, 24 |
| **Could** | US-37, 38 |
| **Hard-case stories promoted to Must (see D-5)** | US-32, 33, 34 |

**Decision D-5 (hard cases outrank FR priority):** US-32 (notification retry), US-33 and US-34
(reconciliation) serve FRs marked Should (FR-23, FR-26), but §2.7 cases 5 and 7 require an
automated test that passes in Phase 2. We therefore treat these three stories as Must.
- Reason: §2.7 requires one test per hard case, and the Phase 2 grading says "all 8 hard cases pass".
- Status: proposed, team to approve.

## 6. Questions, Ambiguities and Decisions

| ID | Question or ambiguity (source) | Decision | Status |
|---|---|---|---|
| D-1 | How does real money reach the biller? (§2.2 steps 7 and 9, §2.3) | Real transfer is out of scope. The ledger records the charge, bill amount, fee and commission. Billers are compared only by settlement files. | Decided, confirm with TA |
| D-2 | Is a fee kept when a payment fails? (§2.2 step 7, §2.6) | No fee on failure. The full hold is released to the customer. | Proposed, confirm with TA |
| D-3 | How is the unique payment reference made? (§2.2 step 6) | One UUID per payment, created before the biller call, stored, and reused on every retry. | Proposed |
| D-4 | What does the agent commission do to PayLink's income? (§2.6) | The agent gets 40% of the fee and PayLink keeps 60% (1.00 and 1.50 EGP on a 2.50 fee). | Proposed, confirm with TA |
| D-5 | Hard cases need tests, but some of their FRs are only Should. (§2.7, §3.3) | Stories for notification retry and reconciliation (US-32, 33, 34) are treated as Must. | Proposed |

