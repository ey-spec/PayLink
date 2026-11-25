sequenceDiagram
autonumber
actor C as Customer or Agent
participant API as Payment API
participant PS as PaymentService
participant IDM as IdempotencyRecord
participant FC as FraudChecks
participant W as Wallet and Ledger
participant AD as BillerAdapter
participant B as Biller (ElectroCo)
participant N as Notifications
participant AU as Audit

C->>API: POST /payments (inquiryId, idempotencyKey)
API->>PS: pay(request)
PS->>IDM: find(key)

alt key already processed
    IDM-->>PS: stored result
    PS-->>API: original result
    API-->>C: same response, no new hold or charge
else new key
    PS->>IDM: save(key, requestHash)
    PS->>PS: load BillInquiry (amount, fee, total)
    PS->>PS: create Payment INITIATED with reference UUID
    PS->>FC: check(account, amount, total)
    Note over FC: chain of rules: blocked, single limit, daily limit, velocity

    alt fraud check fails
        FC-->>PS: rejected (reason)
        PS->>PS: state INITIATED to REJECTED
        PS->>AU: record(state change, reason)
        PS-->>API: Rejected (reason)
        API-->>C: clear rejection reason
    else checks pass
        FC-->>PS: ok
        PS->>W: hold(walletId, total, paymentId)
        Note over W: atomic check and update, balance never goes negative

        alt insufficient funds
            W-->>PS: insufficient funds
            PS->>PS: state INITIATED to REJECTED
            PS-->>API: Rejected (insufficient funds)
            API-->>C: clear rejection reason
        else hold succeeded
            W->>W: write LedgerEntry HOLD
            W->>AU: record(HOLD, who, why)
            W-->>PS: held
            PS->>PS: state INITIATED to FUNDS_HELD
            PS->>PS: state FUNDS_HELD to SENT_TO_BILLER
            PS->>AD: pay(reference, amount, account fields)
            AD->>B: translated request (biller format, signed if needed)

            alt biller confirms
                B-->>AD: paid
                AD-->>PS: CONFIRMED
                PS->>PS: state SENT_TO_BILLER to COMPLETED
                PS->>W: charge(paymentId)
                W->>W: write CHARGE and FEE_INCOME entries
                opt payment made by an agent
                    PS->>W: credit agent commission (40% of fee)
                    W->>W: write COMMISSION entry
                end
                W->>AU: record(CHARGE, FEE_INCOME, COMMISSION)
                PS-)N: event PaymentCompleted
                N-)C: SMS and push receipt
            else biller refuses
                B-->>AD: refused (code)
                AD-->>PS: REFUSED (mapped reason)
                PS->>PS: state SENT_TO_BILLER to FAILED
                Note over PS,W: continues in Diagram 2 (release and refund)
            else no answer within 10 seconds
                AD-->>PS: TIMED_OUT
                PS->>PS: state SENT_TO_BILLER to UNKNOWN
                Note over W: hold stays, never refunded automatically
                PS-)N: event PaymentUnknown
                N-)C: your payment is being processed
                Note over PS,AD: re-query after 1, 5, 15 min with the same reference
            end

            PS-->>API: current state and reference
            API-->>C: response
        end
    end
end