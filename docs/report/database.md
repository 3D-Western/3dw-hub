# Database Schema

The 3D Western backend uses a **MariaDB** relational database (`backend_3dw`), managed with **Flyway** for version-controlled schema migrations. The schema is divided into four logical domains: user management, authentication & security, print job lifecycle, and file storage.

---

## Entity Relationship Diagram

```mermaid
erDiagram
    users {
        int student_id PK
        bigint registration_invitation_id FK
        varchar email
        tinyint email_verified
        timestamp email_verified_at
        varchar first_name
        varchar last_name
        varchar password_hash
        int status FK
        int faculty FK
        int experience_level FK
        int account_status FK
        datetime account_status_until
        int failed_mfa_attempts
        datetime mfa_locked_until
        int otp_resend_count
        datetime last_otp_resend_at
        datetime created_at
        datetime updated_at
    }

    user_status {
        int ID PK
        varchar descrip
    }

    account_status {
        int ID PK
        varchar descrip
    }

    faculty {
        int ID PK
        varchar descrip
    }

    experience_level {
        int ID PK
        varchar descrip
    }

    registration_invitations {
        bigint id PK
        int student_id
        varchar email
        varchar invitation_code
        int status FK
        int created_by FK
        timestamp created_at
        timestamp expired_at
        timestamp accepted_at
    }

    registration_invitation_status {
        int id PK
        varchar description
    }

    sessions {
        binary token PK
        int student_id FK
        datetime created_date
        datetime expiry_date
    }

    mfa_sessions {
        binary token PK
        int user_id FK
        datetime created_date
        datetime expiry_date
    }

    email_otp_challenges {
        bigint id PK
        int user_id FK
        varchar otp_hash
        timestamp expires_at
        int attempt_count
        timestamp consumed_at
        timestamp created_at
    }

    email_verification_tokens {
        bigint id PK
        int user_id FK
        varchar token_hash
        timestamp expires_at
        timestamp consumed_at
        timestamp created_at
    }

    password_reset_tokens {
        bigint id PK
        int student_id FK
        varchar token_hash
        datetime created_at
        datetime expires_at
        datetime consumed_at
    }

    audit_logs {
        bigint id PK
        int user_id FK
        varchar action
        varchar ip_address
        text user_agent
        longtext metadata
        timestamp created_at
    }

    files {
        binary id PK
        int owner_user_id FK
        int uploaded_by FK
        varchar original_filename
        varchar mime_type
        varchar s3_bucket
        varchar s3_key
        bigint size_bytes
        char checksum
        enum status
        enum visibility
        enum storage_type
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    jobs {
        binary id PK
        int student_id FK
        binary file_id FK
        binary reprint_id FK
        int status FK
        int category FK
        varchar name
        varchar description
        text form_answers_json
        datetime created_at
        datetime updated_at
        datetime completed_at
    }

    job_category {
        int ID PK
        varchar descrip
    }

    print_status {
        int ID PK
        varchar descrip
    }

    job_audit_logs {
        bigint id PK
        binary job_id
        int user_id FK
        varchar action
        varchar field_name
        varchar old_value
        varchar new_value
        datetime created_at
    }

    presigned_url_generations {
        binary id PK
        binary job_id FK
        int generated_by_student_id FK
        int generation_number
        datetime url_expires_at
        datetime created_at
    }

    training {
        uuid id PK
        varchar email
        longtext status
    }

    %% User lookup relationships
    users ||--o{ user_status : "status"
    users ||--o{ account_status : "account_status"
    users ||--o{ faculty : "faculty"
    users ||--o{ experience_level : "experience_level"
    users ||--o| registration_invitations : "registration_invitation_id"

    %% Registration
    registration_invitations ||--o{ registration_invitation_status : "status"
    registration_invitations }o--o| users : "created_by"

    %% Auth sessions
    users ||--o{ sessions : "student_id"
    users ||--o{ mfa_sessions : "user_id"
    users ||--o{ email_otp_challenges : "user_id"
    users ||--o{ email_verification_tokens : "user_id"
    users ||--o{ password_reset_tokens : "student_id"
    users ||--o{ audit_logs : "user_id"

    %% Files
    users ||--o{ files : "owner_user_id"
    users ||--o{ files : "uploaded_by"

    %% Jobs
    users ||--o{ jobs : "student_id"
    files ||--o| jobs : "file_id"
    jobs ||--o{ print_status : "status"
    jobs ||--o{ job_category : "category"
    jobs ||--o| jobs : "reprint_id"
    jobs ||--o{ job_audit_logs : "job_id"
    jobs ||--o{ presigned_url_generations : "job_id"
    users ||--o{ job_audit_logs : "user_id"
    users ||--o{ presigned_url_generations : "generated_by_student_id"
```

---

## Table Descriptions

### Lookup / Reference Tables

#### `user_status`
Defines the membership lifecycle states a user account can be in (e.g. *Pending*, *Active*, *Suspended*). Referenced by `users.status`.

#### `account_status`
Tracks temporary restriction states on a user account, such as lockouts due to repeated failed attempts. Referenced by `users.account_status` alongside `users.account_status_until` for time-bounded restrictions.

#### `faculty`
A static lookup of university faculties (e.g. *Engineering*, *Science*). Users select their faculty during registration and it is stored as a foreign key on `users.faculty`.

#### `experience_level`
A static lookup describing a user's self-reported 3D printing experience (e.g. *Beginner*, *Intermediate*, *Advanced*). Referenced by `users.experience_level`.

#### `print_status`
Defines the possible states of a print job through its lifecycle (e.g. *Pending*, *In Progress*, *Completed*, *Cancelled*). Referenced by `jobs.status`.

#### `job_category`
Classifies print jobs by type (e.g. *FDM*, *Resin*, *Laser Cutting*). Referenced by `jobs.category`.

#### `registration_invitation_status`
Tracks the lifecycle state of an invitation (e.g. *Pending*, *Accepted*, *Expired*). Referenced by `registration_invitations.status`.

---

### Core User Table

#### `users`
The central entity of the system. Each user is identified by their university **student ID** (integer primary key — not auto-generated). This table stores profile information, authentication credentials, and security state all in one place.

Key fields:
- `password_hash` — BCrypt-hashed password.
- `email_verified` / `email_verified_at` — Tracks whether the user has completed email verification.
- `account_status` / `account_status_until` / `account_status_reason` — Supports temporary suspensions with an optional expiry.
- `failed_mfa_attempts` / `mfa_locked_until` — Per-account MFA brute-force protection; the account is locked after 10 consecutive failures.
- `otp_resend_count` / `last_otp_resend_at` — Rate-limits OTP resend requests to 3 per 5-minute window.
- `password_change_attempts` / `last_password_change_attempt_at` — Rate-limits password change requests.
- `registration_invitation_id` — Links back to the invitation used to create the account.

---

### Registration

#### `registration_invitations`
Implements the invite-only signup model. An admin creates an invitation for a specific `student_id` and `email`, generating a unique `invitation_code`. The invitation has an expiry and tracks when it was accepted. Only one active invitation can exist per `(student_id, email)` pair.

#### `registration_invitation_status`
Lookup table for invitation states: pending, accepted, expired, etc.

---

### Authentication & Security Tables

#### `sessions`
Stores active full-authentication sessions. A session token (UUID stored as `binary(16)`) is issued as an HTTP-only cookie after successful MFA verification. Sessions expire after **7 days** (`expiry_date`). On each authenticated request the backend validates the token and expiry against this table.

#### `mfa_sessions`
A short-lived intermediate session created after the user successfully provides their password but *before* they complete MFA. The token is issued as an `mfaToken` cookie and expires after **15 minutes**. Once the OTP is verified, this record is deleted and a full `sessions` record is created.

#### `email_otp_challenges`
Stores the OTP codes sent to a user's email as part of the MFA flow. The actual code is never stored — only its **SHA-256 hash** (`otp_hash`). Each row tracks:
- `attempt_count` — Failed guesses for this specific challenge (locked after a threshold).
- `consumed_at` — Set when the OTP is successfully used; prevents replay.
- `expires_at` — OTPs are short-lived and expire after a fixed window.

#### `email_verification_tokens`
Stores tokens for the new-user email verification step. Similar to OTP challenges, only the SHA-256 hash is stored. The token link expires after **72 hours**. `consumed_at` is set once the link is clicked to prevent reuse.

#### `password_reset_tokens`
Stores tokens for the password reset flow. Only the SHA-256 hash is persisted. A token expires after a short window and is single-use (`consumed_at`).

#### `audit_logs`
An append-only security event log. Every significant auth action (MFA login attempts, OTP successes and failures, session creation, etc.) is recorded here with the user's IP address, user agent, and a JSON `metadata` blob for event-specific context. Indexed on `user_id`, `action`, and `created_at` for efficient querying by security tooling.

---

### File Storage

#### `files`
Tracks all uploaded files. Each file is identified by a UUID (`binary(16)`) and stores metadata such as MIME type, original filename, file size, and a SHA-256 `checksum`. The actual binary content is never stored in the database — instead `s3_bucket` and `s3_key` point to the object in the configured storage backend.

Key fields:
- `status` — `UPLOADING`, `COMPLETED`, or `FAILED`; reflects the upload lifecycle.
- `visibility` — `PUBLIC` or `PRIVATE`; controls access.
- `storage_type` — `SEAWEEDFS` (temporary/local dev) or `S3` (persistent production).
- `owner_user_id` — The user who owns the file.
- `uploaded_by` — The user who performed the upload (may differ from owner in admin scenarios).
- `deleted_at` — Soft-delete; the record is retained for audit purposes.

---

### Print Jobs

#### `jobs`
The primary print job table. Each job is identified by a UUID and belongs to a `student_id`. A job has a single associated `file_id` (enforced unique — one file per job), a `status`, and a `category`. The `form_answers_json` field stores the dynamic submission form data captured at creation time.

Key fields:
- `reprint_id` — Self-referential FK; points to the original job if this job is a reprint of a prior submission.
- `created_at` / `updated_at` / `completed_at` — Full lifecycle timestamps.

#### `job_audit_logs`
A change log for print jobs, recording every admin action taken on a job. Each row captures `field_name`, `old_value`, and `new_value`, making it possible to reconstruct the complete edit history of any job. Indexed on `job_id`, `user_id`, `action`, and `created_at`.

#### `presigned_url_generations`
Tracks every time an admin generates a presigned download URL for a job's file. Each row records who generated it (`generated_by_student_id`), which job it was for, the expiry of the URL, and a monotonically increasing `generation_number` per job. This provides a full audit trail of file access by staff.

---

### Training

#### `training`
Stores training and certification records keyed by `email`. The `status` column is a JSON blob that encodes which training modules have been completed and their outcomes. This table is independent of the `users` table — records may exist for individuals before they have a system account.

---

### Internal

#### `flyway_schema_history`
Managed automatically by **Flyway**. Records every applied database migration script, its checksum, execution time, and success status. This table should not be modified manually.

---

## Schema Notes

- **`job` (legacy)** — A legacy print job table exists alongside `jobs`. It shares similar columns but lacks the `category`, `created_at`/`updated_at` audit columns, and the reprint relationship. New development targets `jobs` exclusively.
- **Token storage** — All security tokens (session tokens, MFA tokens, OTP hashes, verification tokens, reset tokens) are stored as hashes only. Raw token values are never persisted.
- **Soft deletes** — `files` uses a `deleted_at` timestamp for soft deletion rather than hard `DELETE` statements, preserving the audit trail.
- **UUID primary keys** — `files`, `jobs`, `mfa_sessions`, `sessions`, and `presigned_url_generations` use `binary(16)` UUIDs as primary keys for global uniqueness and efficient indexing.
