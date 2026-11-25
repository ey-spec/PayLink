## Mermaid Code (Domain Class Diagram)
### COPY THE FOLLOWING CODE & PASTE TO : https://mermaid.live/ 

classDiagram
direction LR

%% PayLink domain class diagram (Phase 1)
%% Solid lines = relationships INSIDE a module.
%% Dashed lines = references BETWEEN modules (by id only, never by object).
%% All Decimal fields are exact decimals, never float or double (NFR-7).

namespace AccountsAndWallet {
  class Account {
    +UUID id
    +String phone
    +String name
    +AccountRole role
    +VerificationLevel level
    +DateTime createdAt
  }
  class Wallet {
    +UUID id
    +UUID accountId
    +availableBalance() Decimal
    +heldBalance() Decimal
  }
  class LedgerEntry {
    <<immutable>>
    +UUID id
    +UUID walletId
    +LedgerEntryType type
    +Decimal amount
    +UUID paymentId
    +DateTime createdAt
  }
  class AccountRole {
    <<enumeration>>
    CUSTOMER
    AGENT
    ADMIN
    SUPPORT
    SYSTEM
  }
  class VerificationLevel {
    <<enumeration>>
    BASIC
    VERIFIED
  }
  class LedgerEntryType {
    <<enumeration>>
    TOPUP
    HOLD
    RELEASE
    CHARGE
    FEE_INCOME
    COMMISSION
    REFUND
  }
}

namespace BillerCatalog {
  class Biller {
    +UUID id
    +String code
    +String name
    +BillerCategory category
    +Boolean active
  }
  class BillerService {
    +UUID id
    +String name
    +Decimal minAmount
    +Decimal maxAmount
    +Boolean active
  }
  class InputField {
    +String name
    +String label
    +String validationPattern
  }
  class BillerCategory {
    <<enumeration>>
    ELECTRICITY
    WATER
    MOBILE
    INTERNET
  }
}

namespace FeesAndCommissions {
  class FeeRule {
    +UUID id
    +FeeRuleType type
    +Map params
  }
  class Promotion {
    +UUID id
    +Date startDate
    +Date endDate
  }
  class Commission {
    +UUID id
    +UUID agentId
    +Decimal amount
    +DateTime creditedAt
  }
  class ConfigSetting {
    +String key
    +String value
    +DateTime updatedAt
  }
  class FeeRuleType {
    <<enumeration>>
    FIXED
    PERCENTAGE
    TIERED
  }
}

namespace PaymentProcessing {
  class BillInquiry {
    +UUID id
    +Map inputValues
    +Decimal billAmount
    +Decimal fee
    +Decimal total
  }
  class Payment {
    +UUID id
    +UUID reference
    +UUID customerId
    +UUID agentId
    +Decimal amount
    +Decimal fee
    +Decimal total
    +PaymentState state
    +DateTime createdAt
  }
  class PaymentStateChange {
    +PaymentState fromState
    +PaymentState toState
    +String reason
    +DateTime at
  }
  class IdempotencyRecord {
    +String key
    +String requestHash
  }
  class RefundRequest {
    +UUID id
    +String reason
    +RefundRequestStatus status
    +UUID decidedBy
  }
  class PaymentState {
    <<enumeration>>
    INITIATED
    FUNDS_HELD
    SENT_TO_BILLER
    COMPLETED
    FAILED
    UNKNOWN
    REJECTED
    REFUNDED
  }
  class RefundRequestStatus {
    <<enumeration>>
    PENDING
    APPROVED
    REJECTED
  }
}

namespace FraudChecks {
  class AccountBlock {
    +UUID id
    +UUID accountId
    +BlockType type
    +String reason
    +DateTime blockedAt
    +DateTime until
    +UUID blockedBy
  }
  class BlockType {
    <<enumeration>>
    TEMPORARY_VELOCITY
    MANUAL_SUPPORT
  }
}

namespace BillerIntegration {
  class BillerCall {
    +UUID id
    +Integer attemptNo
    +String billerReference
    +BillerCallStatus status
    +String rawRequest
    +String rawResponse
    +DateTime at
  }
  class BillerCallback {
    +UUID id
    +String reference
    +String payload
    +CallbackOutcome outcome
    +DateTime receivedAt
  }
  class BillerCallStatus {
    <<enumeration>>
    CONFIRMED
    REFUSED
    ACCEPTED_PENDING
    TIMED_OUT
  }
  class CallbackOutcome {
    <<enumeration>>
    PROCESSED
    DUPLICATE_IGNORED
    UNKNOWN_REFERENCE
  }
}

namespace Notifications {
  class Notification {
    +UUID id
    +UUID recipientId
    +Channel channel
    +String body
    +NotificationStatus status
  }
  class NotificationTemplate {
    +String code
    +Channel channel
    +String text
  }
  class NotificationAttempt {
    +Integer attemptNo
    +Boolean success
    +DateTime at
  }
  class Channel {
    <<enumeration>>
    SMS
    EMAIL
    PUSH
  }
  class NotificationStatus {
    <<enumeration>>
    PENDING
    DELIVERED
    UNDELIVERED
  }
}

namespace Reconciliation {
  class SettlementFile {
    +UUID id
    +Date businessDate
    +String status
  }
  class SettlementEntry {
    +String reference
    +Decimal amount
    +String billerStatus
  }
  class ReconciliationReport {
    +UUID id
    +Date businessDate
    +DateTime createdAt
  }
  class ReconciliationItem {
    +String reference
    +DifferenceType result
    +Decimal paylinkAmount
    +Decimal fileAmount
    +PaymentState paylinkState
    +Boolean reviewed
  }
  class DifferenceType {
    <<enumeration>>
    MATCHED
    MISSING_IN_PAYLINK
    MISSING_IN_FILE
    AMOUNT_DIFFERENT
    STATE_CONFLICT
  }
}

namespace Audit {
  class AuditRecord {
    <<immutable>>
    +UUID id
    +UUID actorId
    +String action
    +String entityType
    +UUID entityId
    +String oldValue
    +String newValue
    +String reason
    +DateTime at
  }
}

%% ---- inside a module ----
Account "1" -- "1" Wallet : owns
Wallet "1" *-- "0..*" LedgerEntry : records
Account --> AccountRole
Account --> VerificationLevel
LedgerEntry --> LedgerEntryType

Biller "1" *-- "1..*" BillerService : offers
BillerService "1" *-- "0..*" InputField : requires
Biller --> BillerCategory

BillerService "1" -- "1" FeeRule : priced by
Biller "1" -- "0..*" Promotion : may have
FeeRule --> FeeRuleType

Payment "1" *-- "0..*" PaymentStateChange : history
Payment "1" -- "1" IdempotencyRecord : guarded by
Payment "1" -- "0..1" BillInquiry : based on
Payment "1" -- "0..*" RefundRequest : may have
Payment --> PaymentState
RefundRequest --> RefundRequestStatus

AccountBlock --> BlockType

Payment "1" -- "0..*" BillerCall : attempts
BillerCall --> BillerCallStatus
BillerCallback --> CallbackOutcome

Notification "0..*" -- "1" NotificationTemplate : filled from
Notification "1" *-- "0..*" NotificationAttempt : attempts
Notification --> Channel
Notification --> NotificationStatus

SettlementFile "1" *-- "0..*" SettlementEntry : lists
ReconciliationReport "1" *-- "0..*" ReconciliationItem : contains
ReconciliationReport --> SettlementFile : built from
ReconciliationItem --> DifferenceType

%% ---- between modules (ids only) ----
Payment ..> Account : customer and agent
Payment ..> BillerService : pays
Payment ..> LedgerEntry : holds and charges
Payment ..> Commission : may earn
Commission ..> Wallet : credits agent
Notification ..> Payment : about
BillerCall ..> Biller : sent to
BillerCallback ..> Payment : matched by reference
AccountBlock ..> Account : blocks
ReconciliationItem ..> Payment : compares
SettlementFile ..> Biller : from
AuditRecord ..> LedgerEntry : justifies