sequenceDiagram
autonumber
participant B as Biller (AquaCo)
participant AD as BillerAdapter
participant PS as PaymentService
participant SM as PaymentState
participant W as Wallet and Ledger
participant N as Notifications
participant AU as Audit
actor C as Customer

alt trigger: biller refuses live
    B-->>AD: refused with error code
    AD->>AD: map code to PayLink reason (unknown code gives generic reason, raw code logged)
    AD-->>PS: REFUSED (reason)
else trigger: Unknown payment resolved later
    B-->>AD: retry or settlement says not paid
    AD-->>PS: REFUSED or not found
end

PS->>SM: canTransition(current, FAILED)
alt current state is SENT_TO_BILLER or UNKNOWN
    SM-->>PS: allowed
    PS->>PS: write PaymentStateChange (to FAILED, reason)
    PS->>AU: record(state change, reason)

    PS->>W: release(walletId, total, paymentId)
    W->>W: write LedgerEntry RELEASE (held to available)
    Note over W: full total returned, no fee kept (D-2), no commission credited
    W->>AU: record(RELEASE, who, why)
    W-->>PS: released

    PS->>SM: canTransition(FAILED, REFUNDED)
    SM-->>PS: allowed
    PS->>PS: write PaymentStateChange (to REFUNDED)
    PS->>AU: record(state change)

    PS-)N: event PaymentFailed (reason)
    N-)C: SMS and push with clear reason
else current state is final or not eligible
    SM-->>PS: refused
    PS->>PS: log refused transition
    Note over PS,W: no ledger entry, no change to wallet
end