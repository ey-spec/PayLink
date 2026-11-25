graph paylink_erd {
  graph [fontname="Helvetica", splines=true, nodesep=0.4, ranksep=0.8];
  node  [fontname="Helvetica", fontsize=10];
  edge  [fontsize=9];

  // ---------- Entities ----------
  node [shape=box];
  accounts; wallets; ledger_entries;
  billers; biller_services; input_fields;
  fee_rules; promotions; commissions; config_settings;
  bill_inquiries; payments; idempotency_records; refund_requests;
  account_blocks;
  biller_calls; biller_callbacks;
  notification_templates; notifications;
  settlement_files; reconciliation_reports; reconciliation_items;
  audit_records;

  // Weak entities (double border)
  node [shape=box, peripheries=2];
  payment_state_changes; notification_attempts; settlement_entries;

  // ---------- Key attributes ----------
  node [shape=ellipse, peripheries=1];
  k_accounts [label=<<u>id</u>>];            accounts -- k_accounts;
  k_wallets [label=<<u>id</u>>];             wallets -- k_wallets;
  k_ledger [label=<<u>id</u>>];              ledger_entries -- k_ledger;
  k_billers [label=<<u>id</u>>];             billers -- k_billers;
  k_services [label=<<u>id</u>>];            biller_services -- k_services;
  k_fields [label=<<u>id</u>>];              input_fields -- k_fields;
  k_rules [label=<<u>id</u>>];               fee_rules -- k_rules;
  k_promos [label=<<u>id</u>>];              promotions -- k_promos;
  k_comm [label=<<u>id</u>>];                commissions -- k_comm;
  k_config [label=<<u>key</u>>];             config_settings -- k_config;
  k_inq [label=<<u>id</u>>];                 bill_inquiries -- k_inq;
  k_pay [label=<<u>id</u>>];                 payments -- k_pay;
  k_psc [label=<<u>id</u>>];                 payment_state_changes -- k_psc;
  k_idem [label=<<u>id</u>>];                idempotency_records -- k_idem;
  k_refund [label=<<u>id</u>>];              refund_requests -- k_refund;
  k_block [label=<<u>id</u>>];               account_blocks -- k_block;
  k_call [label=<<u>id</u>>];                biller_calls -- k_call;
  k_cb [label=<<u>id</u>>];                  biller_callbacks -- k_cb;
  k_tpl [label=<<u>id</u>>];                 notification_templates -- k_tpl;
  k_notif [label=<<u>id</u>>];               notifications -- k_notif;
  k_att [label=<<u>id</u>>];                 notification_attempts -- k_att;
  k_sf [label=<<u>id</u>>];                  settlement_files -- k_sf;
  k_se [label=<<u>id</u>>];                  settlement_entries -- k_se;
  k_rep [label=<<u>id</u>>];                 reconciliation_reports -- k_rep;
  k_item [label=<<u>id</u>>];                reconciliation_items -- k_item;
  k_audit [label=<<u>id</u>>];               audit_records -- k_audit;

  // ---------- Relationships ----------
  node [shape=diamond, peripheries=1];

  // Accounts and Wallet
  r1 [label="Owns"];
  accounts -- r1 [label="1"];  r1 -- wallets [label="1"];
  r2 [label="Records"];
  wallets -- r2 [label="1"];   r2 -- ledger_entries [label="M"];
  r3 [label="Causes"];
  payments -- r3 [label="1"];  r3 -- ledger_entries [label="M"];

  // Biller Catalog and Fees
  r4 [label="Offers"];
  billers -- r4 [label="1"];   r4 -- biller_services [label="M"];
  r5 [label="Requires"];
  biller_services -- r5 [label="1"];  r5 -- input_fields [label="M"];
  r6 [label="Priced by"];
  biller_services -- r6 [label="1"];  r6 -- fee_rules [label="1"];
  r7 [label="Has promotion"];
  billers -- r7 [label="1"];   r7 -- promotions [label="M"];
  r8 [label="Earns"];
  payments -- r8 [label="1"];  r8 -- commissions [label="0..1"];
  r9 [label="Receives (agent)"];
  accounts -- r9 [label="1"];  r9 -- commissions [label="M"];

  // Payment Processing
  r10 [label="Requests"];
  accounts -- r10 [label="1"]; r10 -- bill_inquiries [label="M"];
  r11 [label="Quotes"];
  biller_services -- r11 [label="1"]; r11 -- bill_inquiries [label="M"];
  r12 [label="Makes (customer)"];
  accounts -- r12 [label="1"]; r12 -- payments [label="M"];
  r13 [label="Acts for (agent)"];
  accounts -- r13 [label="1"]; r13 -- payments [label="M"];
  r14 [label="Pays"];
  biller_services -- r14 [label="1"]; r14 -- payments [label="M"];
  r15 [label="Quoted as"];
  bill_inquiries -- r15 [label="1"]; r15 -- payments [label="0..1"];
  r16 [label="Sends"];
  accounts -- r16 [label="1"]; r16 -- idempotency_records [label="M"];
  r17 [label="Guarded by"];
  payments -- r17 [label="1"]; r17 -- idempotency_records [label="1"];
  r18 [label="May have"];
  payments -- r18 [label="1"]; r18 -- refund_requests [label="M"];
  r19 [label="Goes through", peripheries=2];
  payments -- r19 [label="1"]; r19 -- payment_state_changes [label="M"];

  // Fraud Checks
  r20 [label="Blocked by"];
  accounts -- r20 [label="1"]; r20 -- account_blocks [label="M"];

  // Biller Integration
  r21 [label="Attempts"];
  payments -- r21 [label="1"]; r21 -- biller_calls [label="M"];
  r22 [label="Received by"];
  billers -- r22 [label="1"];  r22 -- biller_calls [label="M"];
  r23 [label="Matched by"];
  payments -- r23 [label="1"]; r23 -- biller_callbacks [label="M"];

  // Notifications
  r24 [label="Fills"];
  notification_templates -- r24 [label="1"]; r24 -- notifications [label="M"];
  r25 [label="Triggers"];
  payments -- r25 [label="1"]; r25 -- notifications [label="M"];
  r26 [label="Sent to"];
  accounts -- r26 [label="1"]; r26 -- notifications [label="M"];
  r27 [label="Tries", peripheries=2];
  notifications -- r27 [label="1"]; r27 -- notification_attempts [label="M"];

  // Reconciliation
  r28 [label="Sends file"];
  billers -- r28 [label="1"];  r28 -- settlement_files [label="M"];
  r29 [label="Lists", peripheries=2];
  settlement_files -- r29 [label="1"]; r29 -- settlement_entries [label="M"];
  r30 [label="Builds"];
  settlement_files -- r30 [label="1"]; r30 -- reconciliation_reports [label="M"];
  r31 [label="Contains"];
  reconciliation_reports -- r31 [label="1"]; r31 -- reconciliation_items [label="M"];
  r32 [label="Compared in"];
  payments -- r32 [label="1"]; r32 -- reconciliation_items [label="M"];

  // Audit
  r33 [label="Performs"];
  accounts -- r33 [label="1"]; r33 -- audit_records [label="M"];
}