erDiagram

  accounts {
    uuid id PK
    string phone UK
    string name
    string role "CUSTOMER, AGENT, ADMIN, SUPPORT, SYSTEM"
    string level "BASIC, VERIFIED"
    timestamp created_at "UTC"
  }
  wallets {
    uuid id PK
    uuid account_id FK, UK
  }
  ledger_entries {
    uuid id PK
    uuid wallet_id FK
    uuid payment_id FK "nullable, null for top-ups"
    string type "TOPUP, HOLD, RELEASE, CHARGE, FEE_INCOME, COMMISSION, REFUND"
    decimal amount "decimal(18,2), append-only"
    timestamp created_at "UTC"
  }

  billers {
    uuid id PK
    string code UK
    string name
    string category "ELECTRICITY, WATER, MOBILE, INTERNET"
    boolean active
  }
  biller_services {
    uuid id PK
    uuid biller_id FK
    string name
    decimal min_amount "decimal(18,2)"
    decimal max_amount "decimal(18,2)"
    boolean active
  }
  input_fields {
    uuid id PK
    uuid biller_service_id FK
    string name
    string label
    string validation_pattern
  }

  fee_rules {
    uuid id PK
    uuid biller_service_id FK, UK
    string type "FIXED, PERCENTAGE, TIERED"
    json params
  }
  promotions {
    uuid id PK
    uuid biller_id FK
    date start_date
    date end_date
  }
  commissions {
    uuid id PK
    uuid payment_id FK, UK "one commission per payment (D-9)"
    uuid agent_id FK
    decimal amount "decimal(18,2)"
    timestamp credited_at "UTC"
  }
  config_settings {
    string key PK
    string value
    timestamp updated_at "UTC"
  }

  bill_inquiries {
    uuid id PK
    uuid biller_service_id FK
    uuid account_id FK
    json input_values
    decimal bill_amount "decimal(18,2)"
    decimal fee "decimal(18,2)"
    decimal total "decimal(18,2)"
    timestamp expires_at "pending D-21"
  }
  payments {
    uuid id PK
    uuid reference UK "sent to biller, reused on retries (D-3)"
    uuid customer_id FK
    uuid agent_id FK "nullable"
    uuid biller_service_id FK
    uuid inquiry_id FK "nullable"
    decimal amount "decimal(18,2), bill amount"
    decimal fee "decimal(18,2), snapshot"
    decimal total "decimal(18,2)"
    string state "INITIATED, FUNDS_HELD, SENT_TO_BILLER, COMPLETED, FAILED, UNKNOWN, REJECTED, REFUNDED"
    timestamp created_at "UTC"
  }
  payment_state_changes {
    uuid id PK
    uuid payment_id FK
    string from_state
    string to_state
    string reason
    timestamp at "UTC"
  }
  idempotency_records {
    uuid id PK
    uuid account_id FK "unique with key (D-15)"
    string key "unique with account_id"
    string request_hash
    uuid payment_id FK, UK
    json stored_response
  }
  refund_requests {
    uuid id PK
    uuid payment_id FK
    uuid requested_by FK
    uuid decided_by FK "nullable"
    decimal amount "decimal(18,2)"
    string reason
    string status "PENDING, APPROVED, REJECTED"
    timestamp created_at "UTC"
    timestamp decided_at "UTC, nullable"
  }

  account_blocks {
    uuid id PK
    uuid account_id FK
    uuid blocked_by FK
    uuid lifted_by FK "nullable"
    string type "TEMPORARY_VELOCITY, MANUAL_SUPPORT"
    string reason
    timestamp blocked_at "UTC"
    timestamp until "nullable, null for support blocks"
    timestamp lifted_at "UTC, nullable"
  }

  biller_calls {
    uuid id PK
    uuid payment_id FK "unique with attempt_no"
    uuid biller_id FK
    int attempt_no "unique with payment_id"
    string biller_reference
    string status "CONFIRMED, REFUSED, ACCEPTED_PENDING, TIMED_OUT"
    text raw_request
    text raw_response
    timestamp at "UTC"
  }
  biller_callbacks {
    uuid id PK
    uuid payment_id FK "nullable, unknown reference"
    string reference
    text payload
    string outcome "PROCESSED, DUPLICATE_IGNORED, UNKNOWN_REFERENCE"
    timestamp received_at "UTC"
  }

  notification_templates {
    uuid id PK
    string code "unique with channel"
    string channel "SMS, EMAIL, PUSH"
    string text
  }
  notifications {
    uuid id PK
    uuid template_id FK
    uuid recipient_id FK
    uuid payment_id FK
    string channel "SMS, EMAIL, PUSH"
    string body
    string status "PENDING, DELIVERED, UNDELIVERED"
  }
  notification_attempts {
    uuid id PK
    uuid notification_id FK
    int attempt_no
    boolean success
    timestamp at "UTC"
  }

  settlement_files {
    uuid id PK
    uuid biller_id FK
    date business_date
    timestamp period_from "UTC"
    timestamp period_to "UTC"
    string file_hash UK
    string status
  }
  settlement_entries {
    uuid id PK
    uuid file_id FK
    string reference
    decimal amount "decimal(18,2)"
    string biller_status
  }
  reconciliation_reports {
    uuid id PK
    uuid file_id FK
    date business_date
    timestamp created_at "UTC"
  }
  reconciliation_items {
    uuid id PK
    uuid report_id FK
    uuid payment_id FK "nullable, MISSING_IN_PAYLINK"
    string reference
    string result "MATCHED, MISSING_IN_PAYLINK, MISSING_IN_FILE, AMOUNT_DIFFERENT, STATE_CONFLICT"
    decimal paylink_amount "decimal(18,2), nullable"
    decimal file_amount "decimal(18,2), nullable"
    string paylink_state "nullable"
    boolean reviewed
  }

  audit_records {
    uuid id PK
    uuid actor_id FK
    string action
    string entity_type "polymorphic, no FK"
    uuid entity_id "polymorphic, no FK"
    text old_value
    text new_value
    string reason
    timestamp at "UTC, append-only"
  }

  accounts ||--|| wallets : owns
  wallets ||--o{ ledger_entries : records
  payments |o--o{ ledger_entries : causes

  billers ||--|{ biller_services : offers
  biller_services ||--o{ input_fields : requires
  biller_services ||--|| fee_rules : "priced by"
  billers ||--o{ promotions : "has promotion"
  payments ||--o| commissions : earns
  accounts ||--o{ commissions : "receives as agent"

  accounts ||--o{ bill_inquiries : requests
  biller_services ||--o{ bill_inquiries : quotes
  accounts ||--o{ payments : "makes as customer"
  accounts |o--o{ payments : "acts for as agent"
  biller_services ||--o{ payments : pays
  bill_inquiries |o--o| payments : "quoted as"
  payments ||--o{ payment_state_changes : "goes through"
  accounts ||--o{ idempotency_records : sends
  payments ||--|| idempotency_records : "guarded by"
  payments ||--o{ refund_requests : "may have"
  accounts ||--o{ refund_requests : requests

  accounts ||--o{ account_blocks : "blocked by"

  payments ||--o{ biller_calls : attempts
  billers ||--o{ biller_calls : "received by"
  payments |o--o{ biller_callbacks : "matched by"

  notification_templates ||--o{ notifications : fills
  payments ||--o{ notifications : triggers
  accounts ||--o{ notifications : "sent to"
  notifications ||--o{ notification_attempts : tries

  billers ||--o{ settlement_files : sends
  settlement_files ||--o{ settlement_entries : lists
  settlement_files ||--o{ reconciliation_reports : builds
  reconciliation_reports ||--o{ reconciliation_items : contains
  payments |o--o{ reconciliation_items : "compared in"

  accounts ||--o{ audit_records : performs

  %% ---- one color per module ----
  classDef accountsWallet fill:#dbeafe,stroke:#1d4ed8,color:#000
  classDef billerCatalog fill:#dcfce7,stroke:#15803d,color:#000
  classDef feesCommissions fill:#fef9c3,stroke:#a16207,color:#000
  classDef paymentProcessing fill:#fee2e2,stroke:#b91c1c,color:#000
  classDef fraudChecks fill:#ffedd5,stroke:#c2410c,color:#000
  classDef billerIntegration fill:#ede9fe,stroke:#6d28d9,color:#000
  classDef notificationsMod fill:#cffafe,stroke:#0e7490,color:#000
  classDef reconciliationMod fill:#fce7f3,stroke:#be185d,color:#000
  classDef auditMod fill:#e5e7eb,stroke:#374151,color:#000

  class accounts,wallets,ledger_entries accountsWallet
  class billers,biller_services,input_fields billerCatalog
  class fee_rules,promotions,commissions,config_settings feesCommissions
  class bill_inquiries,payments,payment_state_changes,idempotency_records,refund_requests paymentProcessing
  class account_blocks fraudChecks
  class biller_calls,biller_callbacks billerIntegration
  class notification_templates,notifications,notification_attempts notificationsMod
  class settlement_files,settlement_entries,reconciliation_reports,reconciliation_items reconciliationMod
  class audit_records auditMod