# HFCB Marketplace — Admin Workflow ER Mapping

**Document version:** 1.0  
**Database baseline:** Normalized ER v5.2 / Admin Module Part 3 v24.0  
**Platform:** Java Spring Boot 3.x + Microsoft SQL Server  
**Scope:** Admin RBAC, projects, property approvals, PA management, scout review, booking extensions, loan configuration, payment reconciliation, audit, integration, and notifications.

> This document is the Admin-specific companion to `HFCB_GRAPHICAL_ER_MAPPING.md`. It uses the normalized model and does not store business relationships or approval history in JSON arrays.

## 1. Source comparison and naming decisions

The following Admin capabilities were identified from the supplied normalized master blueprint and the existing graphical baseline:

| Admin capability | Canonical table(s) |
|---|---|
| Admin users and access control | `users`, `roles`, `permissions`, `user_roles`, `role_permissions` |
| Developer project management | `projects`, `properties` |
| Property–PA allocation | `property_pa_assignments` |
| PA absence/availability | `pa_absences` |
| Property Scout allocation | `scout_assignments` |
| Scout due-diligence review | `property_scout_verifications` |
| Generic maker-checker processing | `workflow_definitions`, `workflow_steps`, `workflow_instances`, `workflow_tasks`, `approval_decisions` |
| Business status history | `entity_status_history`, `reason_codes` |
| Loan calculator administration | `loan_calculator_configurations`, `property_loan_config_assignments` |
| Booking-extension decisions | `booking_extensions`, `bookings` |
| Payment reconciliation | `payment_reconciliation_queries`, `transactions`, `bookings` |
| Secure uploads | `file_assets` |
| Audit and HTTP diagnostics | `audit_logs`, `api_request_logs` |
| Reliable integrations and notifications | `integration_exchanges`, `outbox_events`, `notification_deliveries`, `dead_letter_events` |

### Canonical naming rules

- Use `approval_decisions.decision`, not mixed names such as `approval_status` and `request_status` in different modules.
- Use `assigned_user_id` and `assigned_role_id` in `workflow_tasks`.
- Use `decided_by` and `decided_at` for approval decisions.
- Use `status` for a table's current processing state.
- Use `entity_type` + `entity_id` only in generic workflow/history/integration tables. Domain relationships continue to use real foreign keys.
- Keep `property_approval_requests` only as a temporary legacy/migration table. New Admin approvals use the generic workflow tables.

---

## 2. Complete Admin module graphical map

```mermaid
flowchart TD
  subgraph ACCESS[Admin Identity and RBAC]
    U[users]
    R[roles]
    P[permissions]
    UR[user_roles]
    RP[role_permissions]
    U --> UR --> R
    R --> RP --> P
  end

  subgraph CATALOGUE[Project and Property Administration]
    PJ[projects]
    PR[properties]
    PPA[property_pa_assignments]
    LCC[loan_calculator_configurations]
    PLCA[property_loan_config_assignments]
    PJ --> PR
    PR --> PPA
    U --> PPA
    LCC --> PLCA
    PR --> PLCA
  end

  subgraph PA_ADMIN[PA and Scout Administration]
    PAA[pa_absences]
    SA[scout_assignments]
    PSV[property_scout_verifications]
    FA[file_assets]
    U --> PAA
    PR --> SA
    U --> SA
    SA --> PSV
    FA --> PSV
  end

  subgraph WORKFLOW[Maker-Checker Workflow]
    WD[workflow_definitions]
    WS[workflow_steps]
    WI[workflow_instances]
    WT[workflow_tasks]
    AD[approval_decisions]
    RC[reason_codes]
    ESH[entity_status_history]
    WD --> WS
    WD --> WI
    WI --> WT
    WS --> WT
    U --> WT
    R --> WT
    WT --> AD
    U --> AD
    RC --> AD
    WI --> ESH
  end

  subgraph OPERATIONS[Booking and Payment Operations]
    B[bookings]
    BE[booking_extensions]
    T[transactions]
    PRQ[payment_reconciliation_queries]
    B --> BE
    U --> BE
    B --> PRQ
    T --> PRQ
    U --> PRQ
  end

  subgraph RELIABILITY[Audit, Integration and Notification]
    AL[audit_logs]
    IX[integration_exchanges]
    OB[outbox_events]
    ND[notification_deliveries]
    DL[dead_letter_events]
    U --> AL
    WI -. business entity .-> IX
    PRQ -. CRM sync .-> IX
    AD --> OB
    OB --> ND
    ND --> DL
  end

  PR -. entity_type PROPERTY .-> WI
  PJ -. entity_type PROJECT .-> WI
  BE -. entity_type BOOKING_EXTENSION .-> WI
  PRQ -. entity_type PAYMENT_RECONCILIATION .-> WI
```

---

## 3. Admin RBAC ER diagram

```mermaid
erDiagram
  USERS ||--o{ USER_ROLES : has
  ROLES ||--o{ USER_ROLES : assigned
  ROLES ||--o{ ROLE_PERMISSIONS : grants
  PERMISSIONS ||--o{ ROLE_PERMISSIONS : includes
  USERS ||--o{ USER_ROLES : assigned_by
  USERS ||--o{ ROLE_PERMISSIONS : granted_by

  USERS {
    BIGINT id PK
    INT customer_id UK "M"
    INT reference_id UK "M"
    NVARCHAR email UK "M"
    BIGINT manager_id FK "O"
    BIT is_active "M"
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
```

### Initial Admin role catalogue

- `SYSTEM_ADMIN`
- `ADMIN_CONTENT_MANAGER`
- `ADMIN_OPERATIONS_MAKER`
- `ADMIN_OPERATIONS_CHECKER`
- `LEGAL_OFFICER`
- `CUSTOMER_SERVICE_AGENT`
- `PA_MANAGER_INTERNAL`
- `PA_MANAGER_EXTERNAL`
- `INTERNAL_PA`
- `EXTERNAL_PA`
- `PROPERTY_SCOUT`

Roles remain configurable records; they should not be hard-coded as a single `users.role` value.

---

## 4. Projects, properties, PA assignments and loan configuration

```mermaid
erDiagram
  PROJECTS ||--o{ PROPERTIES : groups
  USERS ||--o{ PROPERTIES : owns
  PROPERTIES ||--o{ PROPERTY_PA_ASSIGNMENTS : has
  USERS ||--o{ PROPERTY_PA_ASSIGNMENTS : advisor
  USERS ||--o{ PROPERTY_PA_ASSIGNMENTS : assigned_by
  LOAN_CALCULATOR_CONFIGURATIONS ||--o{ PROPERTY_LOAN_CONFIG_ASSIGNMENTS : applies
  PROPERTIES ||--o{ PROPERTY_LOAN_CONFIG_ASSIGNMENTS : receives

  PROJECTS {
    BIGINT id PK
    NVARCHAR project_name "M"
    NVARCHAR description "O"
    NVARCHAR status "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  PROPERTIES {
    BIGINT id PK
    BIGINT owner_id FK "O"
    BIGINT project_id FK "O"
    NVARCHAR name "M"
    NVARCHAR location "M"
    DECIMAL price "M"
    NVARCHAR availability_status "M"
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
    INT version "M"
  }
  LOAN_CALCULATOR_CONFIGURATIONS {
    BIGINT id PK
    NVARCHAR config_name "M"
    DECIMAL interest_rate "M"
    INT tenure_years "M"
    DECIMAL max_loan_amount "M"
    NVARCHAR scope "M"
    DATE effective_from "M"
    DATE effective_to "O"
    BIT is_active "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
  PROPERTY_LOAN_CONFIG_ASSIGNMENTS {
    BIGINT id PK
    BIGINT property_id FK "M"
    BIGINT loan_config_id FK "M"
    DATETIME2 valid_from "M"
    DATETIME2 valid_to "O"
    BIT is_active "M"
    BIGINT assigned_by FK "O"
  }
```

### Why `property_loan_config_assignments` is used

The supplied master blueprint places `loan_config_id` directly on `properties`, but also supports `ALL` and `SELECTED_PROPERTIES` scopes with future configuration changes. A versioned assignment table avoids mass-updating every property and preserves which configuration was effective at a given time.

For a global configuration, the application resolves the active `scope = ALL` record unless a property-specific active assignment overrides it.

---

## 5. PA absence and Scout verification ER diagram

```mermaid
erDiagram
  USERS ||--o{ PA_ABSENCES : advisor
  USERS ||--o{ PA_ABSENCES : recorded_by
  PROPERTIES ||--o{ SCOUT_ASSIGNMENTS : requires
  USERS ||--o{ SCOUT_ASSIGNMENTS : scout
  USERS ||--o{ SCOUT_ASSIGNMENTS : assigned_by
  SCOUT_ASSIGNMENTS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : produces
  USERS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : reviewed_by
  FILE_ASSETS ||--o{ PROPERTY_SCOUT_VERIFICATIONS : checklist

  PA_ABSENCES {
    BIGINT id PK
    BIGINT pa_id FK "M"
    DATE absent_from "M"
    DATE absent_to "M"
    NVARCHAR status "M"
    NVARCHAR reason "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
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
```

### PA availability flow

```mermaid
stateDiagram-v2
  [*] --> PRESENT
  PRESENT --> SCHEDULED_ABSENCE: Admin records future range
  SCHEDULED_ABSENCE --> ABSENT: absent_from reached
  ABSENT --> PRESENT: absent_to passed
  SCHEDULED_ABSENCE --> CANCELLED: Admin cancels
  ABSENT --> PRESENT: Authorized early return
```

Do not use `pa_absences.status` as the only source of truth for the user's account. Account activation and availability are separate concepts. `users.is_active` controls account access; the current date and active absence rows determine routing availability.

---

## 6. Generic Admin maker-checker ER diagram

```mermaid
erDiagram
  WORKFLOW_DEFINITIONS ||--o{ WORKFLOW_STEPS : defines
  WORKFLOW_DEFINITIONS ||--o{ WORKFLOW_INSTANCES : instantiates
  WORKFLOW_INSTANCES ||--o{ WORKFLOW_TASKS : creates
  WORKFLOW_STEPS ||--o{ WORKFLOW_TASKS : controls
  USERS ||--o{ WORKFLOW_INSTANCES : starts
  USERS ||--o{ WORKFLOW_TASKS : assigned_user
  ROLES ||--o{ WORKFLOW_TASKS : assigned_role
  WORKFLOW_TASKS ||--o{ APPROVAL_DECISIONS : records
  USERS ||--o{ APPROVAL_DECISIONS : decides
  REASON_CODES ||--o{ APPROVAL_DECISIONS : explains
  WORKFLOW_INSTANCES ||--o{ ENTITY_STATUS_HISTORY : traces
  USERS ||--o{ ENTITY_STATUS_HISTORY : changes

  WORKFLOW_DEFINITIONS {
    BIGINT id PK
    NVARCHAR workflow_code UK "M"
    NVARCHAR workflow_name "M"
    NVARCHAR entity_type "M"
    INT definition_version "M"
    BIT is_active "M"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
  }
  WORKFLOW_STEPS {
    BIGINT id PK
    BIGINT workflow_definition_id FK "M"
    NVARCHAR step_code "M"
    NVARCHAR step_name "M"
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
  REASON_CODES {
    BIGINT id PK
    NVARCHAR reason_code UK "M"
    NVARCHAR reason_type "M"
    NVARCHAR description "M"
    BIT is_active "M"
  }
  ENTITY_STATUS_HISTORY {
    BIGINT id PK
    BIGINT workflow_instance_id FK "O"
    NVARCHAR entity_type "M"
    BIGINT entity_id "M"
    NVARCHAR old_status "O"
    NVARCHAR new_status "M"
    BIGINT changed_by FK "O"
    NVARCHAR reason "O"
    DATETIME2 changed_at "M"
  }
```

### Maker-checker execution flow

```mermaid
flowchart LR
  S[Seller, Content Manager or Operations Maker] --> V[Validate permission and payload]
  V --> WI[Create workflow_instance]
  WI --> MT[Create Maker task]
  MT --> CT[Create/assign Checker task]
  CT -->|Approve| A[approval_decision APPROVED]
  CT -->|Reject| R[approval_decision REJECTED]
  A --> E1[Apply domain status change]
  R --> E2[Retain/revert permitted domain state]
  E1 --> H[entity_status_history]
  E2 --> H
  H --> O[outbox_event]
  O --> N[notification_delivery]
  N -->|Retries exhausted| D[dead_letter_event]
```

### Recommended workflow definitions

| Workflow code | Entity type | Typical actors |
|---|---|---|
| `PROPERTY_PUBLISH` | `PROPERTY` | Content Manager → Operations Checker |
| `PROPERTY_UNPUBLISH` | `PROPERTY` | Content Manager/Seller → Operations Checker |
| `PROPERTY_DELIST` | `PROPERTY` | Authorized Maker → Operations Checker |
| `PROJECT_PUBLISH` | `PROJECT` | Content Manager → Operations Checker |
| `SCOUT_VERIFICATION_REVIEW` | `SCOUT_VERIFICATION` | Scout → Admin Reviewer |
| `BOOKING_EXTENSION` | `BOOKING_EXTENSION` | Buyer/PA → Operations Maker or Checker |
| `PAYMENT_RECONCILIATION` | `PAYMENT_RECONCILIATION` | Customer/CS Agent → Finance/Operations |
| `LOAN_CONFIG_CHANGE` | `LOAN_CONFIGURATION` | Admin Maker → Admin Checker |
| `PA_ASSIGNMENT_CHANGE` | `PROPERTY_PA_ASSIGNMENT` | PA Manager/Admin → Checker when required |

---

## 7. Booking extension and payment reconciliation ER diagram

```mermaid
erDiagram
  BOOKINGS ||--o{ BOOKING_EXTENSIONS : extends
  USERS ||--o{ BOOKING_EXTENSIONS : requests
  USERS ||--o{ BOOKING_EXTENSIONS : decides
  BOOKINGS ||--o{ PAYMENT_RECONCILIATION_QUERIES : concerns
  TRANSACTIONS ||--o{ PAYMENT_RECONCILIATION_QUERIES : concerns
  USERS ||--o{ PAYMENT_RECONCILIATION_QUERIES : raises
  USERS ||--o{ PAYMENT_RECONCILIATION_QUERIES : resolves
  WORKFLOW_INSTANCES ||--o| BOOKING_EXTENSIONS : governs
  WORKFLOW_INSTANCES ||--o| PAYMENT_RECONCILIATION_QUERIES : governs

  BOOKING_EXTENSIONS {
    BIGINT id PK
    BIGINT booking_id FK "M"
    BIGINT requested_by FK "M"
    BIGINT decided_by FK "O"
    BIGINT workflow_instance_id FK "O"
    INT requested_days "M"
    INT approved_days "O"
    NVARCHAR reason "M"
    NVARCHAR status "M"
    DATETIME2 requested_at "M"
    DATETIME2 decided_at "O"
  }
  PAYMENT_RECONCILIATION_QUERIES {
    BIGINT id PK
    BIGINT user_id FK "M"
    BIGINT booking_id FK "O"
    BIGINT transaction_id FK "O"
    BIGINT assigned_to FK "O"
    BIGINT workflow_instance_id FK "O"
    NVARCHAR query_details "M"
    NVARCHAR resolution "O"
    NVARCHAR status "M"
    NVARCHAR crm_query_id "O"
    DATETIME2 resolved_at "O"
    DATETIME2 created_at "M"
    DATETIME2 updated_at "M"
    BIGINT created_by "O"
    BIGINT updated_by "O"
    INT version "M"
  }
```

### Booking-extension rule requiring confirmation

The earlier customer/business baseline specifies **1–2 calendar days**, requested on Day 4 or Day 5. The supplied Admin v24 blueprint specifies **exactly 7 additional days**. These requirements conflict.

Until the Product Owner confirms the final rule:

- Keep `requested_days` and `approved_days` configurable.
- Do not hard-code `7` in Java or the database default.
- Store the allowed values in `system_configurations` or workflow rules.
- Keep the property `RESERVED` while an extension decision is pending.

---

## 8. Admin workflow state diagrams

### Project publication

```mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> REVIEWING: Maker submits
  REVIEWING --> APPROVED: Checker approves
  REVIEWING --> DECLINED: Checker rejects
  DECLINED --> DRAFT: Maker edits
  APPROVED --> PUBLISHED: Publish action completed
  PUBLISHED --> UNPUBLISH_PENDING: Maker requests unpublish
  UNPUBLISH_PENDING --> NOT_PUBLISHED: Checker approves
```

### Property publication

```mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> SUBMITTED: Seller or Content Manager submits
  SUBMITTED --> DUPLICATE_CHECK
  DUPLICATE_CHECK --> REJECTED: CRM duplicate confirmed
  DUPLICATE_CHECK --> UNDER_REVIEW: Scout assigned
  UNDER_REVIEW --> APPROVAL_PENDING: Verification submitted
  APPROVAL_PENDING --> PUBLISHED: Operations Checker approves
  APPROVAL_PENDING --> REJECTED: Operations Checker rejects
  REJECTED --> SUBMITTED: Edited and resubmitted
  PUBLISHED --> UNPUBLISH_PENDING: Maker requests
  UNPUBLISH_PENDING --> UNPUBLISHED: Checker approves
  UNPUBLISH_PENDING --> PUBLISHED: Checker rejects request
```

### Payment reconciliation

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> CRM_SUBMITTED: CRM query created
  CRM_SUBMITTED --> UNDER_REVIEW: Assigned to Operations
  UNDER_REVIEW --> INFORMATION_REQUIRED: More evidence needed
  INFORMATION_REQUIRED --> UNDER_REVIEW: Customer/agent responds
  UNDER_REVIEW --> RESOLVED: Reconciled
  UNDER_REVIEW --> REJECTED: Query invalid
  RESOLVED --> CLOSED
  REJECTED --> CLOSED
```

---

## 9. Admin integration, audit and notification mapping

```mermaid
erDiagram
  USERS ||--o{ AUDIT_LOGS : performs
  USERS ||--o{ INTEGRATION_EXCHANGES : initiates
  INTEGRATION_EXCHANGES ||--|| INTEGRATION_REQUESTS : request
  INTEGRATION_EXCHANGES ||--o{ INTEGRATION_RESPONSES : responses
  INTEGRATION_EXCHANGES ||--o{ CALLBACK_EVENTS : callbacks
  OUTBOX_EVENTS ||--o{ NOTIFICATION_DELIVERIES : dispatches
  NOTIFICATION_TEMPLATES ||--o{ NOTIFICATION_DELIVERIES : formats
  NOTIFICATION_DELIVERIES ||--o{ DEAD_LETTER_EVENTS : fails_to

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
  INTEGRATION_EXCHANGES {
    BIGINT id PK
    NVARCHAR trace_id UK "M"
    NVARCHAR correlation_id "O"
    NVARCHAR integration_code "M"
    NVARCHAR operation_code "M"
    NVARCHAR business_entity_type "O"
    BIGINT business_entity_id "O"
    NVARCHAR status "M"
    INT attempt_count "M"
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
  NOTIFICATION_DELIVERIES {
    BIGINT id PK
    BIGINT outbox_event_id FK "M"
    BIGINT template_id FK "M"
    BIGINT recipient_user_id FK "O"
    NVARCHAR channel "M"
    NVARCHAR destination_masked "M"
    NVARCHAR status "M"
    INT attempt_count "M"
    NVARCHAR last_error "O"
    DATETIME2 next_retry_at "O"
    DATETIME2 sent_at "O"
    DATETIME2 created_at "M"
  }
```

Admin actions that require audit events include role changes, PA assignments, absence changes, project/property publication, checker decisions, booking-extension decisions, loan configuration changes, reconciliation resolution, manual KYC review, and dead-letter replay.

---

## 10. Mandatory constraints and indexes

1. `property_pa_assignments`: filtered unique index on `(property_id, pa_id)` where status is `ACTIVE`.
2. `property_pa_assignments`: filtered unique index on `property_id` where `is_primary = 1` and status is `ACTIVE`.
3. `pa_absences`: check `absent_to >= absent_from`; reject overlapping active ranges for the same PA in application logic or a transaction-safe procedure.
4. `workflow_steps`: unique `(workflow_definition_id, step_code)` and `(workflow_definition_id, sequence_number)`.
5. `workflow_instances`: index `(entity_type, entity_id, status)`.
6. `workflow_tasks`: index `(assigned_user_id, status, due_at)` and `(assigned_role_id, status, due_at)`.
7. `approval_decisions`: index `(workflow_task_id, decided_at)`.
8. `entity_status_history`: index `(entity_type, entity_id, changed_at DESC)`.
9. `booking_extensions`: filtered unique index allowing only one `PENDING` extension per booking.
10. `payment_reconciliation_queries`: index `(status, assigned_to, created_at)` and `(transaction_id)`.
11. `loan_calculator_configurations`: prevent overlapping effective global configurations.
12. `property_loan_config_assignments`: prevent overlapping active configurations per property.
13. `file_assets`: unique `storage_key` and index `checksum_sha256`.
14. All Admin decisions must run with optimistic version checks; property reservation/payment paths additionally use pessimistic locking.

---

## 11. Audit and temporal standard

Mutable Admin tables must include:

| Field | SQL Server type | Rule |
|---|---|---|
| `created_at` | `DATETIME2` | UTC, `NOT NULL` |
| `updated_at` | `DATETIME2` | UTC, `NOT NULL` |
| `created_by` | `BIGINT` | User FK; nullable only for system/migration |
| `updated_by` | `BIGINT` | User FK; nullable only for system operations |
| `version` | `INT` | Optimistic lock, `NOT NULL` |

Recommended temporal tables:

- `users`
- `projects`
- `properties`
- `property_pa_assignments`
- `pa_absences`
- `loan_calculator_configurations`
- `property_loan_config_assignments`
- `bookings`
- `booking_extensions`
- `workflow_instances`
- `workflow_tasks`
- `payment_reconciliation_queries`

Append-only ledgers such as `approval_decisions`, `entity_status_history`, `audit_logs`, integration requests/responses, and outbox events should not be rewritten except for explicit processing-state fields.

---

## 12. Future Admin FSD update method

1. Add new actors through `roles`, `permissions`, and their junction tables.
2. Version workflow definitions instead of altering historical workflow instances.
3. Reuse `workflow_instances`, `workflow_tasks`, and `approval_decisions` for new maker-checker processes.
4. Add controlled rejection/approval reasons to `reason_codes`.
5. Record domain transitions in `entity_status_history`.
6. Publish post-commit integration and notification work through `outbox_events`.
7. Update this document and create a versioned Flyway migration for every approved schema change.
