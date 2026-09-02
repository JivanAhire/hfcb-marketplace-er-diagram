# HFCB Marketplace — Graphical Database Mapping Baseline

**Version:** 4.0 (extensible baseline)  
**Database:** Microsoft SQL Server  
**Purpose:** Graphical relationship map for the current marketplace scope, saved API request/response data, and anticipated Admin workflows.

> This document is deliberately modular. Future Admin FSD changes can be added to the Administration and Workflow modules without redesigning the customer, property, booking, or payment cores.

## Legend

- **Current core**: required by the supplied marketplace/FSD baseline.
- **Future-ready**: recommended extension point; finalize fields and rules after the relevant Admin FSD is approved.
- `PK` = primary key, `FK` = foreign key, `UK` = unique key.
- Every mutable business table should include `created_at`, `updated_at`, `created_by`, `updated_by`, and `version`.
- Append-only logs/events should normally use `created_at` only and must not be edited.

## 1. Complete system relationship map

```mermaid
flowchart LR
  subgraph IAM[Identity, SSO and RBAC]
    U[users]
    UR[user_roles]
    R[roles]
    RP[role_permissions]
    P[permissions]
    S[user_sessions]
    OTP[otp_challenges]
    U --> UR --> R --> RP --> P
    U --> S
    U --> OTP
    U -. manager_id .-> U
  end

  subgraph CAT[Property catalogue]
    PR[properties]
    PC[property_categories]
    PST[property_subtypes]
    PM[property_media]
    PA[property_addresses]
    AM[amenities]
    PAM[property_amenities]
    PC --> PST
    PC --> PR
    PST --> PR
    PR --> PM
    PR --> PA
    PR --> PAM --> AM
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
    T[transactions]
    RF[refunds]
    KC[kyc_cases]
    KD[kyc_documents]
    OL[offer_letters]
    LA[loan_applications]
    WL[property_waitlist]
    U --> B
    PR --> B
    B --> BE
    B --> T
    T --> RF
    B --> KC
    KC --> KD
    B --> OL
    B --> LA
    U --> LA
    PR --> LA
    U --> WL
    PR --> WL
  end

  subgraph VERIFY[Listing verification and approvals]
    SA[scout_assignments]
    PSV[property_scout_verifications]
    PAR[property_approval_requests]
    WH[workflow_instances]
    WT[workflow_tasks]
    SH[entity_status_history]
    PR --> SA
    U --> SA
    SA --> PSV
    PR --> PAR
    U --> PAR
    WH --> WT
    U --> WT
    PR --> SH
    B --> SH
    IQ --> SH
  end

  subgraph INT[Integration request/response and reliability]
    IX[integration_exchanges]
    REQ[integration_requests]
    RES[integration_responses]
    CB[callback_events]
    IK[idempotency_keys]
    OB[outbox_events]
    NR[notification_deliveries]
    DL[dead_letter_events]
    IX --> REQ
    IX --> RES
    IX --> CB
    IK --> IX
    OB --> NR
    NR --> DL
  end

  subgraph OPS[Audit and configuration]
    AL[audit_logs]
    API[api_request_logs]
    CFG[system_configurations]
    NT[notification_templates]
    DT[document_types]
    PS[payment_methods]
    RC[reason_codes]
    FL[file_assets]
    U --> AL
    U --> API
    DT --> KD
    PS --> T
  end

  U --> PR
  U --> SA
  U --> PSV
  PR --> PSV
  U --> WT
  U --> FL
  IX -. business_entity_type + business_entity_id .-> B
  IX -. business_entity_type + business_entity_id .-> T
  IX -. business_entity_type + business_entity_id .-> KC
  FL -. polymorphic attachment mapping .-> PM
  FL -. polymorphic attachment mapping .-> KD

  classDef core fill:#dbeafe,stroke:#1d4ed8,color:#172554;
  classDef future fill:#fef3c7,stroke:#d97706,color:#451a03,stroke-dasharray:5 5;
  class U,PR,PC,PST,PM,SP,RV,IQ,IM,AA,ST,B,T,KC,KD,LA,PSV,PAR,AL,API core;
  class UR,R,RP,P,S,OTP,PA,AM,PAM,SV,TM,BE,RF,OL,WL,SA,WH,WT,SH,IX,REQ,RES,CB,IK,OB,NR,DL,CFG,NT,DT,PS,RC,FL future;
```

## 2. Core identity and access mapping

```mermaid
erDiagram
  USERS ||--o{ USERS : manages
  USERS ||--o{ USER_ROLES : assigned
  ROLES ||--o{ USER_ROLES : contains
  ROLES ||--o{ ROLE_PERMISSIONS : grants
  PERMISSIONS ||--o{ ROLE_PERMISSIONS : includes
  USERS ||--o{ USER_SESSIONS : opens
  USERS ||--o{ OTP_CHALLENGES : receives

  USERS {
    bigint id PK
    int customer_id UK
    int reference_id UK
    string first_name
    string middle_name
    string last_name
    string email UK
    string mobile_number UK
    string national_id UK
    string kra_pin
    date dob
    string nationality
    string country
    string current_address
    string city
    string postal_address
    string postal_code
    string password_hash
    bigint manager_id FK
    boolean is_active
    string kyc_status
    boolean mfa_enabled
    boolean email_notifications
    boolean sms_notifications
    boolean web_notifications
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  ROLES {
    bigint id PK
    string role_code UK
    string role_name
    boolean is_internal
    boolean is_active
    datetime created_at
    datetime updated_at
  }
  PERMISSIONS {
    bigint id PK
    string permission_code UK
    string module_code
    string action_code
    boolean is_active
  }
  USER_ROLES {
    bigint user_id PK,FK
    bigint role_id PK,FK
    datetime valid_from
    datetime valid_to
    bigint assigned_by FK
  }
  ROLE_PERMISSIONS {
    bigint role_id PK,FK
    bigint permission_id PK,FK
    bigint granted_by FK
    datetime granted_at
  }
  USER_SESSIONS {
    bigint id PK
    bigint user_id FK
    string session_token_hash UK
    datetime issued_at
    datetime last_activity_at
    datetime expires_at
    datetime revoked_at
    string ip_address
  }
  OTP_CHALLENGES {
    bigint id PK
    bigint user_id FK
    string purpose
    string channel
    string destination_masked
    string code_hash
    int resend_count
    int verify_attempts
    datetime expires_at
    datetime verified_at
    string status
  }
```

> Future recommendation: avoid keeping a single `users.role` column as the long-term authorization source. Keep it temporarily for compatibility, but use `user_roles`, `roles`, and `role_permissions` for Admin RBAC.

## 3. Property catalogue and verification mapping

```mermaid
erDiagram
  USERS ||--o{ PROPERTIES : owns
  USERS ||--o{ PROPERTIES : assigned_advisor
  USERS ||--o{ PROPERTIES : assigned_scout
  PROPERTY_CATEGORIES ||--o{ PROPERTY_SUBTYPES : contains
  PROPERTY_CATEGORIES ||--o{ PROPERTIES : classifies
  PROPERTY_SUBTYPES ||--o{ PROPERTIES : specializes
  PROPERTIES ||--o{ PROPERTY_MEDIA : has
  PROPERTIES ||--o| PROPERTY_ADDRESSES : located_at
  PROPERTIES ||--o{ PROPERTY_AMENITIES : offers
  AMENITIES ||--o{ PROPERTY_AMENITIES : selected
  PROPERTIES ||--o{ SCOUT_ASSIGNMENTS : assigned
  USERS ||--o{ SCOUT_ASSIGNMENTS : scout
  SCOUT_ASSIGNMENTS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : produces
  USERS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : reviews
  PROPERTIES ||--o{ PROPERTY_APPROVAL_REQUESTS : requires
  USERS ||--o{ PROPERTY_APPROVAL_REQUESTS : requests
  USERS ||--o{ PROPERTY_APPROVAL_REQUESTS : checks

  PROPERTIES {
    bigint id PK
    bigint owner_id FK
    bigint assigned_pa_id FK
    bigint assigned_scout_id FK
    bigint category_id FK
    bigint subtype_id FK
    string name
    string location
    decimal price
    decimal monthly_rent
    string deposit_requirements
    string availability_status
    string completion_status
    decimal land_size
    string land_size_unit
    string unit_plot_size
    int number_of_units
    int available_units
    int bedrooms
    int bathrooms
    int parking_spaces
    string furnished_status
    string description
    datetime auction_date
    decimal open_market_value
    decimal starting_bidding_price
    decimal latitude
    decimal longitude
    boolean is_deleted
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  PROPERTY_CATEGORIES {
    bigint id PK
    string category_code UK
    string category_name
    int display_order
    boolean is_active
  }
  PROPERTY_SUBTYPES {
    bigint id PK
    bigint category_id FK
    string subtype_code UK
    string subtype_name
    boolean is_active
  }
  PROPERTY_MEDIA {
    bigint id PK
    bigint property_id FK
    bigint file_asset_id FK
    string media_type
    string media_url
    int display_order
    boolean is_primary
    datetime created_at
  }
  PROPERTY_ADDRESSES {
    bigint id PK
    bigint property_id FK
    string address_line_1
    string address_line_2
    string city
    string county
    string country
    string postal_code
    decimal latitude
    decimal longitude
  }
  AMENITIES {
    bigint id PK
    string amenity_code UK
    string amenity_name
    boolean is_active
  }
  PROPERTY_AMENITIES {
    bigint property_id PK,FK
    bigint amenity_id PK,FK
  }
  SCOUT_ASSIGNMENTS {
    bigint id PK
    bigint property_id FK
    bigint scout_id FK
    bigint assigned_by FK
    datetime assigned_at
    datetime scheduled_visit_at
    string status
  }
  PROPERTY_SCOUT_VERIFICATIONS {
    bigint id PK
    bigint assignment_id FK
    bigint admin_reviewer_id FK
    bigint checklist_file_id FK
    string comments
    string verification_status
    string admin_comments
    datetime verified_at
    datetime reviewed_at
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  PROPERTY_APPROVAL_REQUESTS {
    bigint id PK
    bigint property_id FK
    bigint requester_id FK
    bigint checker_id FK
    string request_type
    string request_status
    string reason
    string checker_comments
    datetime decided_at
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
```

## 4. Booking, payment, KYC and offer mapping

```mermaid
erDiagram
  USERS ||--o{ BOOKINGS : buyer
  PROPERTIES ||--o{ BOOKINGS : booked
  BOOKINGS ||--o{ BOOKING_EXTENSIONS : extends
  USERS ||--o{ BOOKING_EXTENSIONS : requests
  BOOKINGS ||--o{ TRANSACTIONS : paid_by
  USERS ||--o{ TRANSACTIONS : pays
  PAYMENT_METHODS ||--o{ TRANSACTIONS : uses
  TRANSACTIONS ||--o{ REFUNDS : may_create
  BOOKINGS ||--o{ KYC_CASES : requires
  USERS ||--o{ KYC_CASES : subject
  PROPERTIES ||--o{ KYC_CASES : concerns
  KYC_CASES ||--o{ KYC_DOCUMENTS : contains
  DOCUMENT_TYPES ||--o{ KYC_DOCUMENTS : classifies
  BOOKINGS ||--o{ OFFER_LETTERS : generates
  BOOKINGS ||--o{ LOAN_APPLICATIONS : finances
  USERS ||--o{ LOAN_APPLICATIONS : applies
  PROPERTIES ||--o{ LOAN_APPLICATIONS : secures
  USERS ||--o{ PROPERTY_WAITLIST : joins
  PROPERTIES ||--o{ PROPERTY_WAITLIST : has

  BOOKINGS {
    bigint id PK
    bigint property_id FK
    bigint buyer_id FK
    string booking_type
    boolean book_for_self
    string full_name
    string national_id_number
    string phone
    string email
    date move_in_date
    string booking_status
    decimal booking_fee
    decimal deposit_charges
    decimal price_locked
    datetime validity_start_time
    datetime validity_expiry_time
    string extension_status
    datetime final_expiry_time
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  BOOKING_EXTENSIONS {
    bigint id PK
    bigint booking_id FK
    bigint requested_by FK
    bigint decided_by FK
    int requested_days
    int approved_days
    string reason
    string status
    datetime requested_at
    datetime decided_at
  }
  TRANSACTIONS {
    bigint id PK
    bigint booking_id FK
    bigint user_id FK
    bigint payment_method_id FK
    decimal amount
    string transaction_reference UK
    string transaction_type
    string status
    datetime payment_date
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  REFUNDS {
    bigint id PK
    bigint transaction_id FK
    string refund_reference UK
    decimal amount
    string reason
    string status
    datetime processed_at
  }
  KYC_CASES {
    bigint id PK
    bigint booking_id FK
    bigint user_id FK
    bigint property_id FK
    string kyc_status
    int identity_sense_attempts
    datetime submission_time
    datetime approval_time
    string rejection_reason
    string additional_doc_request
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  KYC_DOCUMENTS {
    bigint id PK
    bigint kyc_case_id FK
    bigint document_type_id FK
    bigint file_asset_id FK
    string verification_status
    string rejection_reason
    datetime created_at
    datetime updated_at
  }
  OFFER_LETTERS {
    bigint id PK
    bigint booking_id FK
    string crm_document_id
    string version_number
    string status
    bigint generated_file_id FK
    bigint signed_file_id FK
    datetime generated_at
    datetime upload_due_at
    datetime signed_uploaded_at
    datetime verified_at
  }
  LOAN_APPLICATIONS {
    bigint id PK
    bigint booking_id FK
    bigint user_id FK
    bigint property_id FK
    decimal requested_amount
    decimal estimated_emi
    int tenure_years
    string crm_lead_id
    string status
    datetime created_at
    datetime updated_at
  }
  PROPERTY_WAITLIST {
    bigint id PK
    bigint property_id FK
    bigint user_id FK
    bigint booking_id FK
    int queue_position
    string status
    datetime joined_at
    datetime notified_at
  }
  PAYMENT_METHODS {
    bigint id PK
    string method_code UK
    string method_name
    boolean is_online
    boolean is_active
  }
  DOCUMENT_TYPES {
    bigint id PK
    string type_code UK
    string type_name
    string allowed_formats
    decimal max_file_size_mb
    boolean is_active
  }
```

## 5. Inquiry, auction, support and engagement mapping

```mermaid
erDiagram
  USERS ||--o{ SAVED_PROPERTIES : saves
  PROPERTIES ||--o{ SAVED_PROPERTIES : saved
  USERS ||--o{ RECENTLY_VIEWED : views
  PROPERTIES ||--o{ RECENTLY_VIEWED : viewed
  USERS ||--o{ INQUIRIES : raises
  USERS ||--o{ INQUIRIES : handles
  PROPERTIES ||--o{ INQUIRIES : receives
  INQUIRIES ||--o{ INQUIRY_MESSAGES : includes
  USERS ||--o{ INQUIRY_MESSAGES : sends
  INQUIRIES ||--o{ SITE_VISITS : schedules
  USERS ||--o{ SITE_VISITS : attends
  PROPERTIES ||--o{ AUCTION_APPLICATIONS : receives
  USERS ||--o{ AUCTION_APPLICATIONS : submits
  USERS ||--o{ SUPPORT_TICKETS : raises
  USERS ||--o{ SUPPORT_TICKETS : handles
  SUPPORT_TICKETS ||--o{ TICKET_MESSAGES : includes

  SAVED_PROPERTIES {
    bigint user_id PK,FK
    bigint property_id PK,FK
    datetime saved_at
  }
  RECENTLY_VIEWED {
    bigint id PK
    bigint user_id FK
    bigint property_id FK
    datetime viewed_at
  }
  INQUIRIES {
    bigint id PK
    bigint property_id FK
    bigint user_id FK
    bigint assigned_pa_id FK
    string full_name
    string email
    string mobile_number
    string inquiry_type
    string message
    string preferred_contact_method
    string preferred_contact_time
    boolean communication_consent
    boolean site_visit_requested
    string inquiry_status
    datetime actioned_at
    datetime closed_at
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  INQUIRY_MESSAGES {
    bigint id PK
    bigint inquiry_id FK
    bigint sender_id FK
    string message_text
    datetime sent_at
  }
  SITE_VISITS {
    bigint id PK
    bigint inquiry_id FK
    bigint attendee_user_id FK
    datetime scheduled_at
    string status
    string outcome
    datetime completed_at
  }
  AUCTION_APPLICATIONS {
    bigint id PK
    bigint property_id FK
    bigint user_id FK
    string national_id
    string full_name
    string email
    string mobile_number
    string crm_lead_id
    string status
    datetime created_at
    datetime updated_at
  }
  SUPPORT_TICKETS {
    bigint id PK
    bigint user_id FK
    bigint assigned_agent_id FK
    string crm_service_request_id UK
    string issue_category
    string issue_details
    bigint screenshot_file_id FK
    string ticket_status
    int feedback_rating
    datetime created_at
    datetime updated_at
    bigint created_by
    bigint updated_by
    int version
  }
  TICKET_MESSAGES {
    bigint id PK
    bigint ticket_id FK
    bigint sender_id FK
    string message_text
    boolean is_internal_note
    datetime sent_at
  }
```

## 6. Saved request and response mapping

Use separate exchange, request, response, and callback tables. This is safer and more extensible than adding every integration payload directly to `api_request_logs`.

```mermaid
erDiagram
  USERS ||--o{ INTEGRATION_EXCHANGES : initiates
  INTEGRATION_EXCHANGES ||--|| INTEGRATION_REQUESTS : stores_request
  INTEGRATION_EXCHANGES ||--o{ INTEGRATION_RESPONSES : stores_attempts
  INTEGRATION_EXCHANGES ||--o{ CALLBACK_EVENTS : receives
  IDEMPOTENCY_KEYS ||--o| INTEGRATION_EXCHANGES : protects
  OUTBOX_EVENTS ||--o{ NOTIFICATION_DELIVERIES : dispatches
  NOTIFICATION_TEMPLATES ||--o{ NOTIFICATION_DELIVERIES : formats
  NOTIFICATION_DELIVERIES ||--o{ DEAD_LETTER_EVENTS : exhausts_retries

  INTEGRATION_EXCHANGES {
    bigint id PK
    string trace_id UK
    string correlation_id
    string integration_code
    string operation_code
    string direction
    string business_entity_type
    bigint business_entity_id
    bigint user_id FK
    string status
    int attempt_count
    datetime started_at
    datetime completed_at
    datetime created_at
  }
  INTEGRATION_REQUESTS {
    bigint id PK
    bigint exchange_id FK,UK
    string http_method
    string endpoint
    string headers_json
    string payload_json
    string payload_hash
    string content_type
    string source_ip
    datetime sent_at
  }
  INTEGRATION_RESPONSES {
    bigint id PK
    bigint exchange_id FK
    int attempt_number
    int http_status
    string headers_json
    string payload_json
    string error_code
    string error_message
    bigint duration_ms
    datetime received_at
  }
  CALLBACK_EVENTS {
    bigint id PK
    bigint exchange_id FK
    string provider_event_id UK
    string callback_type
    string headers_json
    string payload_json
    string payload_hash
    string processing_status
    int processing_attempts
    datetime received_at
    datetime processed_at
  }
  IDEMPOTENCY_KEYS {
    bigint id PK
    string idempotency_key UK
    string operation_code
    string request_hash
    string resource_type
    bigint resource_id
    string status
    datetime expires_at
    datetime created_at
  }
  OUTBOX_EVENTS {
    bigint id PK
    string aggregate_type
    bigint aggregate_id
    string event_type
    string payload_json
    string status
    int retry_count
    datetime available_at
    datetime published_at
    datetime created_at
  }
  NOTIFICATION_TEMPLATES {
    bigint id PK
    string template_code UK
    string channel
    string subject_template
    string body_template
    int version
    boolean is_active
  }
  NOTIFICATION_DELIVERIES {
    bigint id PK
    bigint outbox_event_id FK
    bigint template_id FK
    bigint recipient_user_id FK
    string channel
    string destination
    string provider_reference
    string status
    int attempt_count
    string last_error
    datetime next_retry_at
    datetime sent_at
    datetime created_at
  }
  DEAD_LETTER_EVENTS {
    bigint id PK
    bigint notification_delivery_id FK
    string source_type
    bigint source_id
    string payload_json
    string failure_reason
    datetime failed_at
    datetime resolved_at
  }
```

### Request/response storage rules

1. `api_request_logs` is for inbound Marketplace REST diagnostics.
2. `integration_exchanges` is the parent record for outbound/inbound external-system calls such as CRM, IdentitySense, M-Pesa, Core Banking, Boma Yangu, Email, and SMS.
3. Store one request in `integration_requests` and all retries/responses in `integration_responses`.
4. Store asynchronous provider callbacks in `callback_events`; use `provider_event_id` and `payload_hash` to reject duplicates.
5. Mask or encrypt National IDs, tokens, passwords, OTPs, card details, bank statements, and authorization headers. Do not save secrets as plain JSON.
6. Add retention and purge policies. Request/response logs should not be retained forever merely because they are useful for debugging.

## 7. Future Admin workflow extension map

These are extension points, not final Admin schema decisions.

```mermaid
erDiagram
  WORKFLOW_DEFINITIONS ||--o{ WORKFLOW_STEPS : defines
  WORKFLOW_DEFINITIONS ||--o{ WORKFLOW_INSTANCES : instantiates
  WORKFLOW_INSTANCES ||--o{ WORKFLOW_TASKS : creates
  WORKFLOW_STEPS ||--o{ WORKFLOW_TASKS : controls
  USERS ||--o{ WORKFLOW_TASKS : assigned
  USERS ||--o{ APPROVAL_DECISIONS : decides
  WORKFLOW_TASKS ||--o{ APPROVAL_DECISIONS : records
  REASON_CODES ||--o{ APPROVAL_DECISIONS : explains
  USERS ||--o{ STAFF_ATTENDANCE : records
  USERS ||--o{ STAFF_ATTENDANCE : staff_member
  USERS ||--o{ ADVISOR_ASSIGNMENTS : advisor
  USERS ||--o{ ADVISOR_ASSIGNMENTS : manager
  ENTITY_STATUS_HISTORY ||--o{ WORKFLOW_INSTANCES : traces

  WORKFLOW_DEFINITIONS {
    bigint id PK
    string workflow_code UK
    string entity_type
    int version
    boolean is_active
  }
  WORKFLOW_STEPS {
    bigint id PK
    bigint workflow_definition_id FK
    string step_code
    int sequence_number
    string required_permission
    boolean requires_checker
  }
  WORKFLOW_INSTANCES {
    bigint id PK
    bigint workflow_definition_id FK
    string entity_type
    bigint entity_id
    string status
    datetime started_at
    datetime completed_at
  }
  WORKFLOW_TASKS {
    bigint id PK
    bigint workflow_instance_id FK
    bigint workflow_step_id FK
    bigint assigned_user_id FK
    bigint assigned_role_id FK
    string status
    datetime due_at
    datetime actioned_at
  }
  APPROVAL_DECISIONS {
    bigint id PK
    bigint workflow_task_id FK
    bigint decided_by FK
    bigint reason_code_id FK
    string decision
    string comments
    datetime decided_at
  }
  ENTITY_STATUS_HISTORY {
    bigint id PK
    string entity_type
    bigint entity_id
    string old_status
    string new_status
    bigint changed_by FK
    string reason
    datetime changed_at
  }
  REASON_CODES {
    bigint id PK
    string reason_code UK
    string reason_type
    string description
    boolean is_active
  }
  STAFF_ATTENDANCE {
    bigint id PK
    bigint staff_user_id FK
    date attendance_date
    string attendance_status
    bigint recorded_by FK
  }
  ADVISOR_ASSIGNMENTS {
    bigint id PK
    bigint advisor_user_id FK
    bigint manager_user_id FK
    string advisor_type
    datetime valid_from
    datetime valid_to
    boolean is_active
  }
```

## 8. Mandatory auditing standard

Apply this to every mutable business/master/workflow table:

| Column | MSSQL type | Null | Rule |
|---|---|---:|---|
| `created_at` | `DATETIME2` | No | Default `SYSUTCDATETIME()` |
| `updated_at` | `DATETIME2` | No | Set by application on every update |
| `created_by` | `BIGINT` | Yes | FK to `users.id`; null only for system/migration records |
| `updated_by` | `BIGINT` | Yes | FK to `users.id`; null only for system jobs |
| `version` | `INT` | No | Optimistic lock, default `1` |
| `is_active` | `BIT` | Conditional | Use for configurable master records |

For temporal transactional tables, additionally use `SysStartTime`, `SysEndTime`, and `SYSTEM_VERSIONING`. For append-only logs, do not add `updated_at` unless records genuinely transition through processing states.

## 9. Recommended boundary decisions

- Keep file binary content outside MSSQL; store metadata in `file_assets` and encrypted object-storage URLs.
- Replace free-text role, category, subtype, document type, payment method, and reason values with foreign keys to master tables.
- Keep status history in `entity_status_history`; temporal history alone does not capture business reason or acting user clearly.
- Use `workflow_*` tables for future maker-checker/admin approvals instead of adding more approval columns to every domain table.
- Use the transactional outbox pattern so notification failures never roll back bookings, payments, KYC, approvals, or property state.
- Partition or archive high-volume `api_request_logs`, `integration_responses`, `audit_logs`, and `recently_viewed` records by date.
- Treat this as schema baseline **v4.0**. Future Admin FSD updates should increment the version and include a migration/change log.
