# HFCB Marketplace — Graphical Database Mapping Baseline

**Version:** 5.2 — normalized, extensible baseline  
**Database:** Microsoft SQL Server  
**Application:** Java Spring Boot 3.x  
**Status:** Recommended source of truth for implementation and future Admin FSD revisions

> This replaces the earlier v4 mapping. It preserves normalized business relationships, the many-to-many Property–PA assignment, one accountable PA per inquiry, saved request/response data, Admin maker-checker workflows, auditing, and future extension points.

## 1. Design decision

Use the normalized relational model as the transactional source of truth. Do not merge relationships, messages, approvals, KYC cases, or user activity into JSON arrays.

JSON remains appropriate only for redacted integration payloads, audit snapshots, provider metadata, template variables, and flexible configuration.

### Legend

- `PK`: primary key
- `FK`: foreign key
- `UK`: unique key
- `M`: mandatory / `NOT NULL`
- `O`: optional / nullable
- Mutable tables carry `created_at`, `updated_at`, `created_by`, `updated_by`, and `version`.
- Important transactional tables also use SQL Server temporal fields `SysStartTime` and `SysEndTime`.

---

## 2. Complete graphical system map

```mermaid
flowchart LR
  subgraph IAM[Identity, SSO and RBAC]
    U[users]
    R[roles]
    P[permissions]
    UR[user_roles]
    RP[role_permissions]
    US[user_sessions]
    OTP[otp_challenges]
    U --> UR --> R
    R --> RP --> P
    U --> US
    U --> OTP
    U -. manager_id .-> U
  end

  subgraph CAT[Property catalogue and assignment]
    PC[property_categories]
    PST[property_subtypes]
    PR[properties]
    PPA[property_pa_assignments]
    PM[property_media]
    PAD[property_addresses]
    AM[amenities]
    PAM[property_amenities]
    PC --> PST
    PC --> PR
    PST --> PR
    U --> PR
    PR --> PPA
    U --> PPA
    PR --> PM
    PR --> PAD
    PR --> PAM --> AM
  end

  subgraph VERIFY[Scout and Admin verification]
    SA[scout_assignments]
    PSV[property_scout_verifications]
    WD[workflow_definitions]
    WS[workflow_steps]
    WI[workflow_instances]
    WT[workflow_tasks]
    AD[approval_decisions]
    ESH[entity_status_history]
    PR --> SA
    U --> SA
    SA --> PSV
    WD --> WS
    WD --> WI
    WI --> WT
    WS --> WT
    WT --> AD
    U --> WT
    PR -. workflow entity .-> WI
    PR --> ESH
  end

  subgraph ENG[Customer engagement]
    SP[saved_properties]
    RV[recently_viewed]
    IQ[inquiries]
    IM[inquiry_messages]
    SV[site_visits]
    AA[auction_applications]
    ST[support_tickets]
    TM[ticket_messages]
    U --> SP
    PR --> SP
    U --> RV
    PR --> RV
    U --> IQ
    PR --> IQ
    PPA -. eligible handler .-> IQ
    IQ --> IM
    IQ --> SV
    PR --> AA
    U --> AA
    U --> ST
    ST --> TM
  end

  subgraph TX[Booking, KYC and finance]
    B[bookings]
    BE[booking_extensions]
    WL[property_waitlist]
    T[transactions]
    RF[refunds]
    KC[kyc_cases]
    KA[kyc_attempts]
    KD[kyc_documents]
    OL[offer_letters]
    LA[loan_applications]
    U --> B
    PR --> B
    B --> BE
    PR --> WL
    U --> WL
    B --> T
    T --> RF
    B --> KC
    KC --> KA
    KC --> KD
    B --> OL
    B --> LA
    U --> LA
    PR --> LA
  end

  subgraph INT[Requests, responses and reliability]
    IX[integration_exchanges]
    IREQ[integration_requests]
    IRES[integration_responses]
    CB[callback_events]
    IK[idempotency_keys]
    OB[outbox_events]
    NT[notification_templates]
    ND[notification_deliveries]
    DL[dead_letter_events]
    IX --> IREQ
    IX --> IRES
    IX --> CB
    IK --> IX
    OB --> ND
    NT --> ND
    ND --> DL
  end

  subgraph MASTER[Masters, files and governance]
    FA[file_assets]
    DT[document_types]
    PAY[payment_methods]
    RC[reason_codes]
    CFG[system_configurations]
    AL[audit_logs]
    API[api_request_logs]
    FA --> PM
    FA --> KD
    DT --> KD
    PAY --> T
    RC --> AD
    U --> AL
    U --> API
  end

  IX -. business_entity_type + business_entity_id .-> B
  IX -. business_entity_type + business_entity_id .-> T
  IX -. business_entity_type + business_entity_id .-> KC
```

---

## 3. Identity, SSO and RBAC tables

```mermaid
erDiagram
  USERS ||--o{ USERS : manages
  USERS ||--o{ USER_ROLES : has
  ROLES ||--o{ USER_ROLES : assigned
  ROLES ||--o{ ROLE_PERMISSIONS : grants
  PERMISSIONS ||--o{ ROLE_PERMISSIONS : includes
  USERS ||--o{ USER_SESSIONS : opens
  USERS ||--o{ OTP_CHALLENGES : receives

  USERS {
    BIGINT id PK
    INT customer_id UK "M, 9-digit"
    INT reference_id UK "M, 9-digit"
    NVARCHAR first_name "M"
    NVARCHAR middle_name "O"
    NVARCHAR last_name "M"
    NVARCHAR email UK "M"
    NVARCHAR mobile_number UK "M"
    NVARCHAR national_id UK "M"
    NVARCHAR kra_pin "O"
    DATE dob "O"
    NVARCHAR nationality "O"
    NVARCHAR country "O"
    NVARCHAR current_address "O"
    NVARCHAR city "O"
    NVARCHAR postal_address "O"
    NVARCHAR postal_code "O"
    NVARCHAR password_hash "O for SSO"
    BIGINT manager_id FK "O"
    BIT is_active "M"
    NVARCHAR kyc_status "M"
    BIT mfa_enabled "M"
    BIT email_notifications "M"
    BIT sms_notifications "M"
    BIT web_notifications "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  ROLES {
    BIGINT id PK
    NVARCHAR role_code UK "M"
    NVARCHAR role_name "M"
    BIT is_internal "M"
    BIT is_active "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
  }
  PERMISSIONS {
    BIGINT id PK
    NVARCHAR permission_code UK "M"
    NVARCHAR module_code "M"
    NVARCHAR action_code "M"
    NVARCHAR description "O"
    BIT is_active "M"
  }
  USER_ROLES {
    BIGINT user_id PK,FK
    BIGINT role_id PK,FK
    DATETIME2 valid_from "M"
    DATETIME2 valid_to "O"
    BIGINT assigned_by FK "O"
  }
  ROLE_PERMISSIONS {
    BIGINT role_id PK,FK
    BIGINT permission_id PK,FK
    BIGINT granted_by FK "O"
    DATETIME2 granted_at "M"
  }
  USER_SESSIONS {
    BIGINT id PK
    BIGINT user_id FK
    NVARCHAR session_token_hash UK "M"
    DATETIME2 issued_at "M"
    DATETIME2 last_activity_at "M"
    DATETIME2 expires_at "M"
    DATETIME2 revoked_at "O"
    NVARCHAR ip_address "O"
    NVARCHAR user_agent "O"
  }
  OTP_CHALLENGES {
    BIGINT id PK
    BIGINT user_id FK "O"
    NVARCHAR purpose "M"
    NVARCHAR channel "M"
    NVARCHAR destination_masked "M"
    NVARCHAR code_hash "M"
    INT resend_count "M, max 3"
    INT verify_attempts "M, max 3"
    DATETIME2 resend_available_at "M"
    DATETIME2 expires_at "M"
    DATETIME2 verified_at "O"
    NVARCHAR status "M"
  }
```

---

## 4. Property, PA assignment and catalogue tables

```mermaid
erDiagram
  USERS ||--o{ PROPERTIES : owns
  PROPERTY_CATEGORIES ||--o{ PROPERTY_SUBTYPES : contains
  PROPERTY_CATEGORIES ||--o{ PROPERTIES : classifies
  PROPERTY_SUBTYPES ||--o{ PROPERTIES : specializes
  PROPERTIES ||--o{ PROPERTY_PA_ASSIGNMENTS : has
  USERS ||--o{ PROPERTY_PA_ASSIGNMENTS : advisor
  USERS ||--o{ PROPERTY_PA_ASSIGNMENTS : assigner
  PROPERTIES ||--o{ PROPERTY_MEDIA : contains
  FILE_ASSETS ||--o{ PROPERTY_MEDIA : stores
  PROPERTIES ||--o| PROPERTY_ADDRESSES : located_at
  PROPERTIES ||--o{ PROPERTY_AMENITIES : offers
  AMENITIES ||--o{ PROPERTY_AMENITIES : selected

  PROPERTIES {
    BIGINT id PK
    BIGINT owner_id FK "O for bank asset"
    BIGINT assigned_scout_id FK "O"
    BIGINT category_id FK "M"
    BIGINT subtype_id FK "M"
    NVARCHAR name "M"
    NVARCHAR location "M"
    DECIMAL price "M"
    DECIMAL monthly_rent "M"
    NVARCHAR deposit_requirements "O"
    NVARCHAR availability_status "M"
    NVARCHAR completion_status "O"
    DECIMAL land_size "O"
    NVARCHAR land_size_unit "O"
    NVARCHAR unit_plot_size "O"
    INT number_of_units "O"
    INT available_units "O"
    INT bedrooms "O"
    INT bathrooms "O"
    INT parking_spaces "O"
    NVARCHAR furnished_status "O"
    NVARCHAR description "M"
    DATETIME2 auction_date "O"
    DECIMAL open_market_value "O"
    DECIMAL starting_bidding_price "O"
    DECIMAL latitude "O"
    DECIMAL longitude "O"
    BIT is_deleted "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  PROPERTY_PA_ASSIGNMENTS {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT pa_id FK "M"
    BIT is_primary "M"
    NVARCHAR assignment_status "M"
    DATETIME2 valid_from "M"
    DATETIME2 valid_to "O"
    BIGINT assigned_by FK "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  PROPERTY_CATEGORIES {
    BIGINT id PK
    NVARCHAR category_code UK "M"
    NVARCHAR category_name "M"
    INT display_order "M"
    BIT is_active "M"
  }
  PROPERTY_SUBTYPES {
    BIGINT id PK
    BIGINT category_id FK "M"
    NVARCHAR subtype_code UK "M"
    NVARCHAR subtype_name "M"
    BIT is_active "M"
  }
  PROPERTY_MEDIA {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT file_asset_id FK "M"
    NVARCHAR media_type "M"
    INT display_order "M"
    BIT is_primary "M"
    DATETIME2 created_at "M"
  }
  PROPERTY_ADDRESSES {
    BIGINT id PK
    BIGINT property_id FK,UK "M"
    NVARCHAR address_line_1 "M"
    NVARCHAR address_line_2 "O"
    NVARCHAR city "M"
    NVARCHAR county "M"
    NVARCHAR country "M"
    NVARCHAR postal_code "O"
    DECIMAL latitude "O"
    DECIMAL longitude "O"
  }
  AMENITIES {
    BIGINT id PK
    NVARCHAR amenity_code UK "M"
    NVARCHAR amenity_name "M"
    BIT is_active "M"
  }
  PROPERTY_AMENITIES {
    BIGINT property_id PK,FK
    BIGINT amenity_id PK,FK
  }
```

**Assignment constraints**

- One property can have many active PAs and one PA can manage many properties.
- Only one active assignment per `(property_id, pa_id)`.
- Only one active `is_primary = 1` assignment per property.
- `inquiries.assigned_pa_id` remains one PA per inquiry for SLA ownership.

---

## 5. Scout verification and Admin workflow tables

```mermaid
erDiagram
  PROPERTIES ||--o{ SCOUT_ASSIGNMENTS : requires
  USERS ||--o{ SCOUT_ASSIGNMENTS : scout
  USERS ||--o{ SCOUT_ASSIGNMENTS : assigner
  SCOUT_ASSIGNMENTS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : produces
  USERS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : reviewer
  FILE_ASSETS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : checklist
  WORKFLOW_DEFINITIONS ||--o{ WORKFLOW_STEPS : defines
  WORKFLOW_DEFINITIONS ||--o{ WORKFLOW_INSTANCES : instantiates
  WORKFLOW_INSTANCES ||--o{ WORKFLOW_TASKS : creates
  WORKFLOW_STEPS ||--o{ WORKFLOW_TASKS : controls
  USERS ||--o{ WORKFLOW_TASKS : assigned
  ROLES ||--o{ WORKFLOW_TASKS : assigned_role
  WORKFLOW_TASKS ||--o{ APPROVAL_DECISIONS : records
  USERS ||--o{ APPROVAL_DECISIONS : decides
  REASON_CODES ||--o{ APPROVAL_DECISIONS : explains
  USERS ||--o{ ENTITY_STATUS_HISTORY : changes

  SCOUT_ASSIGNMENTS {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT scout_id FK "M"
    BIGINT assigned_by FK "M"
    DATETIME2 assigned_at "M"
    DATETIME2 scheduled_visit_at "O"
    NVARCHAR status "M"
    DATETIME2 completed_at "O"
  }
  PROPERTY_SCOUT_VERIFICATIONS {
    BIGINT id PK
    BIGINT assignment_id FK "M"
    BIGINT admin_reviewer_id FK "O"
    BIGINT checklist_file_id FK "M"
    NVARCHAR comments "M"
    NVARCHAR verification_status "M"
    NVARCHAR admin_comments "O"
    DATETIME2 verified_at "O"
    DATETIME2 reviewed_at "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  WORKFLOW_DEFINITIONS {
    BIGINT id PK
    NVARCHAR workflow_code UK "M"
    NVARCHAR entity_type "M"
    INT definition_version "M"
    BIT is_active "M"
  }
  WORKFLOW_STEPS {
    BIGINT id PK
    BIGINT workflow_definition_id FK "M"
    NVARCHAR step_code "M"
    INT sequence_number "M"
    NVARCHAR required_permission "M"
    BIT requires_checker "M"
    NVARCHAR conditions_json "O"
  }
  WORKFLOW_INSTANCES {
    BIGINT id PK
    BIGINT workflow_definition_id FK "M"
    NVARCHAR entity_type "M"
    BIGINT entity_id "M"
    NVARCHAR status "M"
    BIGINT started_by FK "M"
    DATETIME2 started_at "M"
    DATETIME2 completed_at "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    INT version "M"
  }
  WORKFLOW_TASKS {
    BIGINT id PK
    BIGINT workflow_instance_id FK "M"
    BIGINT workflow_step_id FK "M"
    BIGINT assigned_user_id FK "O"
    BIGINT assigned_role_id FK "O"
    NVARCHAR status "M"
    DATETIME2 due_at "O"
    DATETIME2 actioned_at "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    INT version "M"
  }
  APPROVAL_DECISIONS {
    BIGINT id PK
    BIGINT workflow_task_id FK "M"
    BIGINT decided_by FK "M"
    BIGINT reason_code_id FK "O"
    NVARCHAR decision "M"
    NVARCHAR comments "O"
    DATETIME2 decided_at "M"
  }
  ENTITY_STATUS_HISTORY {
    BIGINT id PK
    NVARCHAR entity_type "M"
    BIGINT entity_id "M"
    NVARCHAR old_status "O"
    NVARCHAR new_status "M"
    BIGINT changed_by FK "O"
    NVARCHAR reason "O"
    DATETIME2 changed_at "M"
  }
```

---

## 6. Engagement, inquiry and support tables

```mermaid
erDiagram
  USERS ||--o{ SAVED_PROPERTIES : saves
  PROPERTIES ||--o{ SAVED_PROPERTIES : saved
  USERS ||--o{ RECENTLY_VIEWED : views
  PROPERTIES ||--o{ RECENTLY_VIEWED : viewed
  USERS ||--o{ INQUIRIES : submits
  USERS ||--o{ INQUIRIES : handles
  PROPERTIES ||--o{ INQUIRIES : receives
  INQUIRIES ||--o{ INQUIRY_MESSAGES : contains
  USERS ||--o{ INQUIRY_MESSAGES : sends
  INQUIRIES ||--o{ SITE_VISITS : schedules
  USERS ||--o{ SITE_VISITS : assigned
  USERS ||--o{ SUPPORT_TICKETS : raises
  USERS ||--o{ SUPPORT_TICKETS : handles
  SUPPORT_TICKETS ||--o{ TICKET_MESSAGES : contains
  USERS ||--o{ TICKET_MESSAGES : sends

  SAVED_PROPERTIES {
    BIGINT user_id PK,FK
    BIGINT property_id PK,FK
    DATETIME2 saved_at "M"
  }
  RECENTLY_VIEWED {
    BIGINT id PK
    BIGINT user_id FK "M"
    BIGINT property_id FK "M"
    DATETIME2 viewed_at "M"
  }
  INQUIRIES {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT user_id FK "O guest"
    BIGINT assigned_pa_id FK "O"
    NVARCHAR full_name "M"
    NVARCHAR email "M"
    NVARCHAR mobile_number "M"
    NVARCHAR inquiry_type "M"
    NVARCHAR message "M"
    NVARCHAR preferred_contact_method "M"
    NVARCHAR preferred_contact_time "O"
    BIT communication_consent "M"
    BIT site_visit_requested "M"
    DATETIME2 requested_visit_at "O"
    NVARCHAR inquiry_status "M"
    DATETIME2 first_response_at "O"
    DATETIME2 actioned_at "O"
    DATETIME2 closed_at "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  INQUIRY_MESSAGES {
    BIGINT id PK
    BIGINT inquiry_id FK "M"
    BIGINT sender_id FK "O guest"
    NVARCHAR message_text "M"
    DATETIME2 sent_at "M"
  }
  SITE_VISITS {
    BIGINT id PK
    BIGINT inquiry_id FK "M"
    BIGINT assigned_user_id FK "O"
    DATETIME2 scheduled_at "M"
    NVARCHAR status "M"
    NVARCHAR outcome "O"
    DATETIME2 completed_at "O"
  }
  SUPPORT_TICKETS {
    BIGINT id PK
    BIGINT user_id FK "M"
    BIGINT assigned_agent_id FK "O"
    NVARCHAR crm_service_request_id UK "M"
    NVARCHAR issue_category "M"
    NVARCHAR issue_details "M"
    BIGINT screenshot_file_id FK "O"
    NVARCHAR ticket_status "M"
    INT feedback_rating "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  TICKET_MESSAGES {
    BIGINT id PK
    BIGINT ticket_id FK "M"
    BIGINT sender_id FK "O"
    NVARCHAR message_text "M"
    BIT is_internal_note "M"
    DATETIME2 sent_at "M"
  }
```

---

## 7. Booking, payment, KYC, loan and auction tables

```mermaid
erDiagram
  USERS ||--o{ BOOKINGS : buyer
  PROPERTIES ||--o{ BOOKINGS : booked
  BOOKINGS ||--o{ BOOKING_EXTENSIONS : extends
  USERS ||--o{ BOOKING_EXTENSIONS : decides
  PROPERTIES ||--o{ PROPERTY_WAITLIST : has
  USERS ||--o{ PROPERTY_WAITLIST : joins
  BOOKINGS ||--o{ TRANSACTIONS : funds
  USERS ||--o{ TRANSACTIONS : pays
  PAYMENT_METHODS ||--o{ TRANSACTIONS : uses
  TRANSACTIONS ||--o{ REFUNDS : creates
  BOOKINGS ||--o| KYC_CASES : requires
  USERS ||--o{ KYC_CASES : subject
  PROPERTIES ||--o{ KYC_CASES : concerns
  KYC_CASES ||--o{ KYC_ATTEMPTS : checks
  KYC_CASES ||--o{ KYC_DOCUMENTS : contains
  DOCUMENT_TYPES ||--o{ KYC_DOCUMENTS : classifies
  FILE_ASSETS ||--o{ KYC_DOCUMENTS : stores
  BOOKINGS ||--o{ OFFER_LETTERS : generates
  FILE_ASSETS ||--o{ OFFER_LETTERS : stores
  BOOKINGS ||--o{ LOAN_APPLICATIONS : finances
  USERS ||--o{ LOAN_APPLICATIONS : applies
  PROPERTIES ||--o{ LOAN_APPLICATIONS : secures
  PROPERTIES ||--o{ AUCTION_APPLICATIONS : receives
  USERS ||--o{ AUCTION_APPLICATIONS : submits

  BOOKINGS {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT buyer_id FK "M"
    NVARCHAR booking_type "M"
    BIT book_for_self "M"
    NVARCHAR full_name "M snapshot"
    NVARCHAR national_id_number "M snapshot"
    NVARCHAR phone "M snapshot"
    NVARCHAR email "M snapshot"
    DATE move_in_date "O"
    NVARCHAR booking_status "M"
    DECIMAL booking_fee "M, KES 5000"
    DECIMAL deposit_charges "M"
    DECIMAL price_locked "M"
    DATETIME2 validity_start_time "M"
    DATETIME2 validity_expiry_time "M"
    NVARCHAR extension_status "M"
    DATETIME2 final_expiry_time "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  BOOKING_EXTENSIONS {
    BIGINT id PK
    BIGINT booking_id FK "M"
    BIGINT requested_by FK "M"
    BIGINT decided_by FK "O"
    INT requested_days "M, 1 or 2"
    INT approved_days "O"
    NVARCHAR reason "M"
    NVARCHAR status "M"
    DATETIME2 requested_at "M"
    DATETIME2 decided_at "O"
  }
  PROPERTY_WAITLIST {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT user_id FK "M"
    BIGINT booking_id FK "O"
    INT queue_position "M"
    NVARCHAR status "M"
    DATETIME2 joined_at "M"
    DATETIME2 notified_at "O"
  }
  TRANSACTIONS {
    BIGINT id PK
    BIGINT booking_id FK "M"
    BIGINT user_id FK "M"
    BIGINT payment_method_id FK "M"
    DECIMAL amount "M"
    NVARCHAR transaction_reference UK "M"
    NVARCHAR transaction_type "M"
    NVARCHAR status "M"
    DATETIME2 payment_date "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  REFUNDS {
    BIGINT id PK
    BIGINT transaction_id FK "M"
    NVARCHAR refund_reference UK "M"
    DECIMAL amount "M"
    NVARCHAR reason "M"
    NVARCHAR status "M"
    DATETIME2 processed_at "O"
  }
  KYC_CASES {
    BIGINT id PK
    BIGINT booking_id FK,UK "M"
    BIGINT user_id FK "M"
    BIGINT property_id FK "M"
    NVARCHAR kyc_status "M"
    INT identity_sense_attempts "M max 3"
    DATETIME2 submission_time "O"
    DATETIME2 approval_time "O"
    NVARCHAR rejection_reason "O"
    NVARCHAR additional_doc_request "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  KYC_ATTEMPTS {
    BIGINT id PK
    BIGINT kyc_case_id FK "M"
    INT attempt_number "M max 3"
    BIGINT integration_exchange_id FK "O"
    NVARCHAR result_status "M"
    NVARCHAR failure_reason "O"
    DATETIME2 attempted_at "M"
  }
  KYC_DOCUMENTS {
    BIGINT id PK
    BIGINT kyc_case_id FK "M"
    BIGINT document_type_id FK "M"
    BIGINT file_asset_id FK "M"
    NVARCHAR verification_status "M"
    NVARCHAR rejection_reason "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
  }
  OFFER_LETTERS {
    BIGINT id PK
    BIGINT booking_id FK "M"
    NVARCHAR crm_document_id "M"
    NVARCHAR version_number "M"
    NVARCHAR status "M"
    BIGINT generated_file_id FK "M"
    BIGINT signed_file_id FK "O"
    DATETIME2 generated_at "M"
    DATETIME2 upload_due_at "M"
    DATETIME2 signed_uploaded_at "O"
    DATETIME2 verified_at "O"
  }
  LOAN_APPLICATIONS {
    BIGINT id PK
    BIGINT booking_id FK "M"
    BIGINT user_id FK "M"
    BIGINT property_id FK "M"
    DECIMAL requested_amount "M"
    DECIMAL estimated_emi "M"
    INT tenure_years "M"
    NVARCHAR crm_lead_id "O"
    NVARCHAR status "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    INT version "M"
  }
  AUCTION_APPLICATIONS {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT user_id FK "O guest"
    NVARCHAR national_id "M"
    NVARCHAR full_name "M"
    NVARCHAR email "M"
    NVARCHAR mobile_number "M"
    NVARCHAR crm_lead_id "O"
    NVARCHAR status "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    INT version "M"
  }
```

---

## 8. Saved request, response, callback and notification tables

```mermaid
erDiagram
  USERS ||--o{ API_REQUEST_LOGS : invokes
  USERS ||--o{ INTEGRATION_EXCHANGES : initiates
  IDEMPOTENCY_KEYS ||--o| INTEGRATION_EXCHANGES : protects
  INTEGRATION_EXCHANGES ||--|| INTEGRATION_REQUESTS : request
  INTEGRATION_EXCHANGES ||--o{ INTEGRATION_RESPONSES : responses
  INTEGRATION_EXCHANGES ||--o{ CALLBACK_EVENTS : callbacks
  OUTBOX_EVENTS ||--o{ NOTIFICATION_DELIVERIES : dispatches
  NOTIFICATION_TEMPLATES ||--o{ NOTIFICATION_DELIVERIES : formats
  USERS ||--o{ NOTIFICATION_DELIVERIES : receives
  NOTIFICATION_DELIVERIES ||--o{ DEAD_LETTER_EVENTS : fails_to

  API_REQUEST_LOGS {
    BIGINT id PK
    BIGINT user_id FK "O"
    NVARCHAR trace_id "M"
    NVARCHAR request_uri "M"
    NVARCHAR http_method "M"
    NVARCHAR request_headers_redacted "O"
    NVARCHAR request_body_redacted "O"
    NVARCHAR response_headers_redacted "O"
    NVARCHAR response_body_redacted "O"
    INT status_code "M"
    BIGINT duration_ms "M"
    NVARCHAR client_ip "O"
    DATETIME2 created_at "M"
  }
  INTEGRATION_EXCHANGES {
    BIGINT id PK
    NVARCHAR trace_id UK "M"
    NVARCHAR correlation_id "O"
    NVARCHAR integration_code "M"
    NVARCHAR operation_code "M"
    NVARCHAR direction "M"
    NVARCHAR business_entity_type "O"
    BIGINT business_entity_id "O"
    BIGINT user_id FK "O"
    NVARCHAR status "M"
    INT attempt_count "M"
    DATETIME2 started_at "M"
    DATETIME2 completed_at "O"
    DATETIME2 created_at "M"
  }
  INTEGRATION_REQUESTS {
    BIGINT id PK
    BIGINT exchange_id FK,UK "M"
    NVARCHAR http_method "M"
    NVARCHAR endpoint "M"
    NVARCHAR headers_json_redacted "O"
    NVARCHAR payload_json_redacted "O"
    NVARCHAR payload_hash "M"
    NVARCHAR content_type "O"
    DATETIME2 sent_at "M"
  }
  INTEGRATION_RESPONSES {
    BIGINT id PK
    BIGINT exchange_id FK "M"
    INT attempt_number "M"
    INT http_status "O"
    NVARCHAR headers_json_redacted "O"
    NVARCHAR payload_json_redacted "O"
    NVARCHAR payload_hash "O"
    NVARCHAR error_code "O"
    NVARCHAR error_message "O"
    BIGINT duration_ms "M"
    DATETIME2 received_at "M"
  }
  CALLBACK_EVENTS {
    BIGINT id PK
    BIGINT exchange_id FK "O"
    NVARCHAR provider_event_id UK "M"
    NVARCHAR callback_type "M"
    NVARCHAR headers_json_redacted "O"
    NVARCHAR payload_json_redacted "M"
    NVARCHAR payload_hash "M"
    NVARCHAR processing_status "M"
    INT processing_attempts "M"
    DATETIME2 received_at "M"
    DATETIME2 processed_at "O"
  }
  IDEMPOTENCY_KEYS {
    BIGINT id PK
    NVARCHAR idempotency_key UK "M"
    NVARCHAR operation_code "M"
    NVARCHAR request_hash "M"
    NVARCHAR resource_type "O"
    BIGINT resource_id "O"
    NVARCHAR status "M"
    DATETIME2 expires_at "M"
    DATETIME2 created_at "M"
  }
  OUTBOX_EVENTS {
    BIGINT id PK
    NVARCHAR aggregate_type "M"
    BIGINT aggregate_id "M"
    NVARCHAR event_type "M"
    NVARCHAR payload_json "M"
    NVARCHAR status "M"
    INT retry_count "M"
    DATETIME2 available_at "M"
    DATETIME2 published_at "O"
    DATETIME2 created_at "M"
  }
  NOTIFICATION_TEMPLATES {
    BIGINT id PK
    NVARCHAR template_code UK "M"
    NVARCHAR channel "M"
    NVARCHAR subject_template "O"
    NVARCHAR body_template "M"
    INT template_version "M"
    BIT is_active "M"
  }
  NOTIFICATION_DELIVERIES {
    BIGINT id PK
    BIGINT outbox_event_id FK "M"
    BIGINT template_id FK "M"
    BIGINT recipient_user_id FK "O"
    NVARCHAR channel "M"
    NVARCHAR destination_masked "M"
    NVARCHAR provider_reference "O"
    NVARCHAR status "M"
    INT attempt_count "M"
    NVARCHAR last_error "O"
    DATETIME2 next_retry_at "O"
    DATETIME2 sent_at "O"
    DATETIME2 created_at "M"
  }
  DEAD_LETTER_EVENTS {
    BIGINT id PK
    BIGINT notification_delivery_id FK "O"
    NVARCHAR source_type "M"
    BIGINT source_id "M"
    NVARCHAR payload_json_redacted "O"
    NVARCHAR failure_reason "M"
    DATETIME2 failed_at "M"
    DATETIME2 resolved_at "O"
  }
```

### Integration usage

- `api_request_logs`: inbound Marketplace HTTP diagnostics.
- `integration_exchanges`: parent correlation for CRM, Core Banking, IdentitySense, M-Pesa, Boma Yangu, email, and SMS.
- `integration_requests`: one outbound request per exchange.
- `integration_responses`: all response/retry attempts.
- `callback_events`: asynchronous provider notifications with duplicate protection.
- `idempotency_keys`: client and provider operation deduplication.
- `outbox_events`: reliable post-commit dispatch.
- `notification_deliveries`: recipient/channel retries.
- `dead_letter_events`: exhausted failures requiring support action.

---

## 9. Master, file and audit tables

```mermaid
erDiagram
  DOCUMENT_TYPES ||--o{ KYC_DOCUMENTS : classifies
  PAYMENT_METHODS ||--o{ TRANSACTIONS : supports
  REASON_CODES ||--o{ APPROVAL_DECISIONS : explains
  USERS ||--o{ FILE_ASSETS : uploads
  USERS ||--o{ AUDIT_LOGS : acts

  DOCUMENT_TYPES {
    BIGINT id PK
    NVARCHAR type_code UK "M"
    NVARCHAR type_name "M"
    NVARCHAR allowed_formats "M"
    DECIMAL max_file_size_mb "M"
    BIT is_active "M"
  }
  PAYMENT_METHODS {
    BIGINT id PK
    NVARCHAR method_code UK "M"
    NVARCHAR method_name "M"
    BIT is_online "M"
    BIT is_active "M"
  }
  REASON_CODES {
    BIGINT id PK
    NVARCHAR reason_code UK "M"
    NVARCHAR reason_type "M"
    NVARCHAR description "M"
    BIT is_active "M"
  }
  SYSTEM_CONFIGURATIONS {
    BIGINT id PK
    NVARCHAR config_key UK "M"
    NVARCHAR config_value "M"
    NVARCHAR config_type "M"
    NVARCHAR description "O"
    BIT is_sensitive "M"
    BIT is_active "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  FILE_ASSETS {
    BIGINT id PK
    BIGINT uploaded_by FK "O"
    NVARCHAR storage_provider "M"
    NVARCHAR storage_key UK "M"
    NVARCHAR original_file_name "M"
    NVARCHAR content_type "M"
    BIGINT file_size_bytes "M"
    NVARCHAR checksum_sha256 "M"
    NVARCHAR malware_scan_status "M"
    NVARCHAR encryption_key_ref "O"
    NVARCHAR retention_class "M"
    DATETIME2 created_at "M"
  }
  AUDIT_LOGS {
    BIGINT id PK
    BIGINT user_id FK "O"
    NVARCHAR action_type "M"
    NVARCHAR entity_type "O"
    BIGINT entity_id "O"
    NVARCHAR before_state_json_redacted "O"
    NVARCHAR after_state_json_redacted "O"
    NVARCHAR ip_address "O"
    NVARCHAR trace_id "O"
    DATETIME2 created_at "M"
  }
```

---

## 10. Core workflow graphics

### Property listing and approval

```mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> SUBMITTED: Seller submits
  SUBMITTED --> DUPLICATE_CHECK
  DUPLICATE_CHECK --> REJECTED: CRM duplicate found
  DUPLICATE_CHECK --> UNDER_REVIEW: Scout assigned
  UNDER_REVIEW --> APPROVAL_PENDING: Scout verification submitted
  APPROVAL_PENDING --> PUBLISHED: Checker approves
  APPROVAL_PENDING --> REJECTED: Checker rejects
  REJECTED --> SUBMITTED: Seller edits and resubmits
  PUBLISHED --> RESERVED: Successful booking fee
  RESERVED --> SOLD: Completion
  PUBLISHED --> UNPUBLISH_PENDING: Maker request
  UNPUBLISH_PENDING --> UNPUBLISHED: Checker approves
```

### Booking and payment

```mermaid
stateDiagram-v2
  [*] --> BOOKING_SUBMITTED
  BOOKING_SUBMITTED --> PAYMENT_SUCCESSFUL: KES 5000 callback
  PAYMENT_SUCCESSFUL --> RESERVED: Lock property and start one 7-day timer
  RESERVED --> EXTENSION_REQUESTED: Day 4 or 5
  EXTENSION_REQUESTED --> RESERVED: Pending or approved 1-2 days
  EXTENSION_REQUESTED --> CANCELLED: Rejected and deadline passed
  RESERVED --> BOOKED: First deposit confirmed
  RESERVED --> CANCELLED: Final deadline expired
  CANCELLED --> WAITLIST_NOTIFIED
  BOOKED --> OFFER_LETTER_PENDING
  OFFER_LETTER_PENDING --> COMPLETED: Correct signed version uploaded
  OFFER_LETTER_PENDING --> CRM_DECISION_REQUIRED: 14 working days expired
  CRM_DECISION_REQUIRED --> CANCELLED: CRM explicitly releases
```

### KYC

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> VERIFIED: Reusable approved KYC exists
  PENDING --> AUTOMATED_CHECK
  AUTOMATED_CHECK --> VERIFIED: IdentitySense succeeds
  AUTOMATED_CHECK --> AUTOMATED_CHECK: Failure and attempt less than 3
  AUTOMATED_CHECK --> PENDING_MANUAL_REVIEW: Third failure
  PENDING_MANUAL_REVIEW --> VERIFIED: Admin approval
  PENDING_MANUAL_REVIEW --> REJECTED: Admin rejection
```

### Admin maker-checker

```mermaid
flowchart LR
  A[Seller or Content Manager submits] --> WI[workflow_instance]
  WI --> M[Maker task]
  M --> C[Operations Checker task]
  C -->|Approve| AP[approval_decision APPROVED]
  C -->|Reject| RJ[approval_decision REJECTED]
  AP --> SH[entity_status_history]
  RJ --> SH
  SH --> OB[outbox_event]
  OB --> ND[notification_delivery]
  ND -->|Retries exhausted| DL[dead_letter_event]
```

---

## 11. Mandatory audit and temporal standard

Apply to mutable business, master, assignment, and workflow tables:

| Field | MSSQL type | Rule |
|---|---|---|
| `created_at` | `DATETIME2` | `NOT NULL`, UTC default |
| `updated_at` | `DATETIME2` | `NOT NULL`, updated on every change |
| `created_by` | `BIGINT` | User FK; nullable only for migration/system action |
| `updated_by` | `BIGINT` | User FK; nullable only for system action |
| `version` | `INT` | `NOT NULL`, optimistic lock |

Use system versioning for `users`, `properties`, `property_pa_assignments`, `bookings`, `transactions`, `kyc_cases`, `workflow_instances`, and `workflow_tasks`.

Append-only logs should not be modified except for explicit processing fields. Store business transition reasons in `entity_status_history`; temporal history alone does not identify the business reason adequately.

---

## 12. Simplified-model comparison

The eight-table simplified design must not be the production transactional model.

| Simplified approach | Recommended relational structure |
|---|---|
| `users.saved_property_ids` JSON | `saved_properties` |
| `users.recently_viewed_ids` JSON | `recently_viewed` |
| `properties.assigned_pa_ids` JSON | `property_pa_assignments` |
| Scout columns in `properties` | `scout_assignments`, `property_scout_verifications` |
| Approval columns in `properties` | `workflow_instances`, `workflow_tasks`, `approval_decisions` |
| Mortgage JSON in `bookings` | `loan_applications` |
| Auction as booking subtype | `auction_applications` |
| User-level KYC attempt count only | `kyc_cases`, `kyc_attempts` |
| Inquiry chat JSON | `inquiry_messages` |
| Support as inquiry subtype | `support_tickets`, `ticket_messages` |
| One `system_logs` table | Purpose-specific audit, API, integration, callback, and retry tables |

The simplified representation may be used as a reporting view, cache, search document, or API response projection.

---

## 13. Security and retention

1. Never persist passwords, OTP values, access tokens, refresh tokens, session tokens, authorization headers, CVV, or card data in plaintext.
2. Redact or encrypt National ID, KRA PIN, contact details, bank accounts, document URLs, and integration payload fields.
3. Store binaries in encrypted object storage and retain only metadata/storage keys in `file_assets`.
4. Apply separate retention policies to API logs, integration payloads, callbacks, audit events, notifications, and uploaded documents.
5. Partition high-volume tables by date and archive/purge them according to compliance rules.
6. Restrict request/response data using dedicated support and audit permissions.
7. Use hashes for duplicate detection and forensic integrity.

---

## 14. Future update strategy

When a new Admin FSD arrives:

1. Map each actor to `roles` and `permissions`.
2. Add or version `workflow_definitions` and `workflow_steps` rather than adding approval columns to domain tables.
3. Reuse `workflow_instances`, `workflow_tasks`, and `approval_decisions`.
4. Add reason values to `reason_codes`.
5. Record every business status transition in `entity_status_history`.
6. Publish notifications/integrations through `outbox_events`.
7. Add a migration and increment this baseline version.
