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

| ID | Question or ambiguity (source) | Decision | Reasoning and alternatives rejected | Status |
|---|---|---|---|---|
| D-1 | How does real money reach the biller? (§2.2 steps 7 and 9, §2.3) | Real transfer is out of scope. The ledger records the charge, bill amount, fee and commission. Billers are compared only by settlement files. | Modelling bank transfers would add a whole payments-infrastructure project the brief does not ask for. | Decided, confirm with TA |
| D-2 | Is a fee or commission kept when a payment fails? (§2.2 step 7, §2.6) | No fee and no commission on failure. The full hold (bill amount and fee) is released to the customer. | Fee and commission are earned only when the service is delivered, so nothing has to be clawed back later. | Team-agreed, confirm with TA |
| D-3 | How is the unique payment reference made? (§2.2 step 6) | One UUID per payment, created before the biller call, stored, and reused on every retry. | A stable reference lets the biller recognise a retry and lets reconciliation match records. | Proposed |
| D-4 | What does the agent commission do to PayLink's income? (§2.6) | The agent gets 40% of the fee and PayLink keeps 60% (1.00 and 1.50 EGP on a 2.50 fee). A promotion fee of 0 gives a commission of 0. | The commission is derived from the fee, so no special case is needed for promotions. | Proposed, confirm with TA |
| D-5 | Hard cases need tests, but some of their FRs are only Should. (§2.7, §3.3) | Stories for notification retry and reconciliation (US-32, 33, 34) are treated as Must. | §2.7 requires one test per hard case, and Phase 2 grading says all 8 hard cases pass. | Proposed |
| D-6 | From whose wallet does an agent pay? (FR-17, US-21) | An agent payment debits the customer's wallet. The agent's wallet only receives commission and never funds a payment. Customer consent (OTP or PIN) is out of scope, and the risk is noted for the report. | Alternative rejected: paying from the agent's own float wallet would need a settlement process the brief does not define. Using the customer's wallet keeps one debited wallet per payment and a simple ledger. | Proposed by Karim, confirm with TA |
| D-7 | How does an agent act for a customer? (FR-17, FR-25) | The agent logs in with their own agent account and never uses the customer's credentials. Only accounts with the agent role may pay on behalf of a customer. The payment records the agent_id. | Shared credentials would break auditability (who did it?) and least privilege. | Proposed by Karim |
| D-8 | Can an agent register a customer at the terminal? (FR-1) | No. Registration is self-service only. | Agent registration would need identity checks and consent rules the brief does not give, and FR-1 only says that a customer registers. | Proposed by Karim, confirm with TA |
| D-9 | When is the commission credited, and how are duplicates stopped? (§2.7 case 6, FR-14, FR-17) | The commission is credited once, only when a payment reaches Completed, including a late Completed from a retry, callback or settlement file. Unknown, Failed and Rejected earn nothing. Duplicates are blocked at two layers: the state machine (final states refuse transitions) and a database unique constraint of one commission entry per payment. | Alternative rejected: using the idempotency key. It protects the customer-to-PayLink request, not a biller's callback. Two layers give defence in depth, because the database still protects us if the state-machine code has a bug. | Proposed by Karim |
| D-10 | How do the velocity block and the support block interact? (FR-19, FR-21, US-23, US-24) | An account has independent blocks, each with a type (velocity or support) and an optional expiry. Velocity blocks expire after 10 minutes. Support blocks have no expiry. An account can pay only if no block is active. Support can lift any block, and unblocking clears all active blocks. Every lift is audited with who, when and why. | Alternative rejected: a single blocked flag, because the 10-minute expiry could silently unblock an account that support blocked on purpose. Support can lift both types because US-24 does not restrict it, and the audit log covers the abuse risk. | Proposed by Karim, confirm with TA |
| D-11 | What does "not listed in the settlement file" mean? (§2.5, §2.7 case 7, FR-26) | A settlement file covers a defined period. A payment in Unknown is resolved as not paid (Failed, then Refunded) only if the file's period covers it, with a configurable safety margin around the cutoff. Payments inside the margin stay Unknown and are decided by the next file. If any file lists the payment as paid, it becomes Completed. A flagged difference never changes a payment or wallet automatically. | Alternative rejected: "not listed means Failed". A payment sent near midnight may appear in the next day's file, and refunding it would pay the customer back for money the biller already received. §2.5 says Unknown is only refunded when the system is sure. | Proposed by Karim, confirm with TA |
| D-12 | Which payments count toward the daily limit? (§2.6, FR-18, US-22) | FundsHeld, SentToBiller, Unknown and Completed count. Rejected, Failed and Refunded do not, so a freed amount can be used again. The limit check and the hold happen in the same database transaction. | Alternative rejected: counting only Completed. Payments in flight or Unknown would not be counted, so concurrent payments could pass the limit. Failures are not counted, because a biller's refusal should not cost the customer part of their limit. | Proposed by Karim, confirm with TA |
| D-13 | What does "per day" mean? (§2.2 step 1, §2.6) | The limit resets at 00:00 Africa/Cairo. A payment belongs to the day it was created, not the day it completes. Timestamps are stored in UTC, and the day boundary is computed in the Africa/Cairo time zone. | Alternatives rejected: a rolling 24 hours (hard to explain to customers) and a UTC day (resets at an odd hour for Egyptian customers). Storing UTC avoids 23 or 25 hour days when the offset changes. | Proposed by Karim, confirm with TA |
| D-14 | Does the limit include the service fee? (§2.6, FR-18, US-22) | No. The daily limit and the single-payment range (5 to 10,000 EGP) apply to the bill amount only. The wallet balance check uses the total (bill plus fee), because that is what is held. | Alternative rejected: counting the total, because a customer's real limit would then depend on each biller's fee rule. US-22's example (4,800 paid plus 200 allowed) already assumes the bill amount. | Proposed by Karim, confirm with TA |
| D-15 | Who owns an idempotency key? (FR-10, §2.7 case 1, US-17) | A key is unique per account (account and key). Two accounts can send the same key without colliding, and no account can receive another account's result. For an agent payment, the key belongs to the agent's account. Enforced by a database unique constraint. A reused key with a different amount or biller is rejected. | Alternative rejected: a global key, because one customer could receive another customer's payment result, which is a data leak. | Proposed by Karim, confirm with TA |
| D-16 | Which account does the velocity rule count? (§2.6, FR-19, US-23) | The rule counts payments per paying wallet, whether the customer or an agent makes them. The velocity block goes on the customer's account, so a busy agent is never blocked for serving many customers. Spreading payments across many customers to avoid the rule is an agent-abuse case and is out of scope. | Alternative rejected: counting by the requesting account, which would block a busy agent for 10 minutes and stop its customers being served. | Proposed by Karim, confirm with TA |
| D-17 | Do top-ups count toward the daily limit? (§2.6, FR-4, FR-18) | No. The limit applies only to money going out to billers. A wallet can be topped up by any amount. Top-up limits are out of scope because the brief gives no number. | The brief says "daily payment limit". Inventing a top-up limit would add a requirement the brief does not state. | Proposed by Karim, confirm with TA |
| D-18 | What happens when a Completed payment is refunded? (FR-27, §2.5, US-37) | US-37 stays in the pack as a Could. A refund after the biller confirmed is a business loss for PayLink, because the brief defines no biller reversal. If support approves it, the customer receives the bill amount in their wallet, and PayLink keeps the fee and any commission already paid. The payment stays Completed (a final state), and the refund is a separate record (Pending, Approved or Rejected) with its own ledger and audit entries. The audit record states that PayLink absorbed the cost. | Support approval is the business control for the loss. Alternatives rejected: reversing the commission (could push an agent's wallet negative, breaking NFR-1) and dropping the story (the brief lists it as a Could). | Proposed by Karim, confirm with TA |
| D-19 | Does a "not found" answer to a re-query fail an Unknown payment? (§2.5, FR-13, US-29) | No. Only an explicit refusal moves Unknown to Failed. "Not found" is treated as no answer, so the payment stays Unknown with the hold kept until a later answer or reconciliation (D-11) decides it. | §2.5 lists only two ways out of Unknown to Failed: the biller refuses, or reconciliation finds no payment. "Not found" can mean the biller has not finished yet, so refunding on it risks a double payment. | Proposed by Karim, confirm with TA |

### Open questions (no decision yet)

| ID | Question | Status |
|---|---|---|
| Q-1 | How does an agent settle the cash collected from customers with PayLink? | Open, probably out of scope like D-1, confirm with TA |
| Q-2 | How long can a payment stay Unknown if the settlement file never arrives, and who reviews it? | Open, confirm with TA |
| Q-3 | What is the daily upload cutoff for the NetCo batch? (NetCo is optional) | Open |




# Business Analysis Gaps To-Do

| # | Task | Details | Owner | Done |
|---|---|---|---|---|
| 1 | Decide login and roles (D-20) | Simple token login, one role per account (customer, agent, support, admin). OTP, MFA and password reset are out of scope. Confirm with TA. | | [ ] |
| 2 | Write US-39 "Log in and be identified with a role" | Must priority. Given/when/then, including a wrong password and a role that is not allowed. | | [ ] |
| 3 | Decide the inquiry rule (D-21) | (a) An inquiry expires after a configurable time, e.g. 10 minutes. (b) No partial payment: pay the full inquired bill. (c) For TopUpNet and NetCo there is no bill, so the customer enters the amount (5 to 10,000 EGP). | | [ ] |
| 4 | Update US-13 and US-16 for the inquiry rule | Add the expiry and the "customer enters the amount" cases. | | [ ] |
| 5 | Update the story-title list (section 3) and priority table (section 5) | Add US-39 and re-count the totals (should be 39 stories). | | [ ] |
| 6 | Add the hard-case traceability table | Cases 1 to 8 mapped to US-17, 28, 26, 20, 32, 27, 34, 30. | | [ ] |
| 7 | Add 2 sentences to the problem summary | Agents pay from the customer's wallet; a refund after Completed is a loss PayLink absorbs. | | [ ] |
| 8 | Send the "confirm with TA" list to the TA | Every decision marked "confirm with TA", plus Q-1 to Q-3. | | [ ] |
| 9 | Export `requirements.md` to PDF | The pack needs both the PDF and the `.md`. | | [ ] |