# ECCD Monitoring System — Architecture & Flowcharts

Visual reference for how the **Little Dreamers ECCD portal** is built and how each workflow behaves. Diagrams are [Mermaid](https://mermaid.js.org/) and render natively on GitHub and most Markdown viewers.

- **Roles:** Admin · Teacher · Parent · Visitor
- **Stack:** Flask 3 + MySQL/MariaDB + Bootstrap 5 (see README → Tech stack)
- **Portals:** Public website (`/`), Teacher/Parent login (`/login`), Admin portal (`/admin`)

## Table of contents

1. [High-level system architecture](#1-high-level-system-architecture)
2. [Deployment architecture](#2-deployment-architecture)
3. [Request lifecycle & security pipeline](#3-request-lifecycle--security-pipeline)
4. [Business workflows](#4-business-workflows)
   - [4.1 Login & session routing](#41-login--session-routing)
   - [4.2 Teacher registration & admin approval](#42-teacher-registration--admin-approval-gated-by-admin)
   - [4.3 Parent registration](#43-parent-registration)
   - [4.4 Enrollment request](#44-enrollment-request)
   - [4.5 Classwork & grading loop](#45-classwork--grading-loop)
   - [4.6 Attendance recording](#46-attendance-recording)
   - [4.7 Classroom Stream (posts)](#47-classroom-stream-posts)
   - [4.8 Announcements](#48-announcements)
   - [4.9 Forgot / recover password](#49-forgot--recover-password)
   - [4.10 Notifications](#410-notifications)
5. [Data model (entity-relationship)](#5-data-model-entity-relationship)

---

## 1. High-level system architecture

```mermaid
flowchart TB
    subgraph Clients["Clients (browser)"]
        VIS["Visitor<br/>(public site, contact)"]
        PRN["Parent"]
        TCH["Teacher"]
        ADM["Admin"]
    end

    subgraph FL["Flask Application (Python 3)"]
        direction TB
        SEC["security.py middleware<br/>CSRF · security headers · rate limit · logging"]
        subgraph RTS["Blueprints (routes/)"]
            MAIN["main.py<br/>landing + contact"]
            AUTH["auth.py<br/>login · registrations · forgot-password"]
            TEA["teacher.py<br/>teacher portal"]
            PAR["parent.py<br/>parent portal"]
            ADMX["admin.py<br/>admin portal"]
        end
        DBH["db.py helper<br/>execute_query() + transactions"]
    end

    DB[("MySQL / MariaDB<br/>eccd_db")]
    FS[("uploads/<br/>photos · files · certificates")]

    VIS --> SEC
    PRN --> SEC
    TCH --> SEC
    ADM --> SEC

    SEC --> MAIN
    SEC --> AUTH
    SEC --> TEA
    SEC --> PAR
    SEC --> ADMX

    MAIN --> DBH
    AUTH --> DBH
    TEA --> DBH
    PAR --> DBH
    ADMX --> DBH

    DBH --> DB
    TEA --> FS
    PAR --> FS
    ADMX --> FS
```

- Every request passes through `security.py`: **CSRF** validation on state-changing POSTs, **security headers** on every response, **rate limiting** on the admin login, and **request logging** (query strings omitted to protect PII).
- All five blueprints share one `db.py` query helper, which owns the single MySQL connection pool / transaction handling.
- Uploaded files (teacher photos, parent photos, learner photos, birth certificates, class banners, stream attachments, classwork submissions, website media) live under `uploads/` and are validated by extension **and** magic-byte signature, capped at 16 MB (5 MB for profile pictures, which go through the shared `profile.py` helpers).
- The self-service **My Profile** module (`profile.py` + the `teacher`/`parent` blueprints) lets an account edit only its own row: personal fields plus `users.username`/`users.email`, and a password change that requires the current password and invalidates outstanding reset tokens.

---

## 2. Deployment architecture

```mermaid
flowchart LR
    DEV["Developer / maintainer"] --> GM["git → GitHub<br/>(private: ECCD-Monitoring-System)"]
    GM -->|pull / deploy| SRV["Production server"]

    subgraph SRV["Production server"]
        direction TB
        LBR["Reverse proxy<br/>Nginx / IIS / Caddy (HTTPS + static)"]
        WS["Waitress WSGI<br/>127.0.0.1:8000"]
        APP["Flask app"]
        DB[("MySQL eccd_db<br/>utf8mb4")]
        UP[("uploads/ volume")]
        LOG[("app.log — rotating")]
        LBR --> WS
        WS --> APP
        APP --> DB
        APP --> UP
        APP --> LOG
    end

    U["Users on the internet<br/>visitors · parents · teachers · admins"] -->|HTTPS :443| LBR
```

- Development uses the Flask dev server (`py run.py`); production runs **Waitress** behind a reverse proxy that terminates HTTPS.
- `init_db.py` creates the DB from `schema.sql` (idempotent) and bootstraps the default admin. The private GitHub repo is the single source of truth for code; the live MySQL DB and `uploads/` stay server-local (never committed — see `.gitignore`).

---

## 3. Request lifecycle & security pipeline

```mermaid
flowchart TD
    R["HTTP request"] --> S1{"State-changing<br/>POST?"}
    S1 -->|"yes"| CSRF{"CSRF token<br/>valid?"}
    CSRF -->|"no"| BAD["400 — token validation failed"]
    CSRF -->|"yes"| AUTH{"User logged in?"}
    S1 -->|"no"| AUTH
    AUTH -->|"no"| REDIR["Redirect to login (302)"]
    AUTH -->|"yes"| ROLE{"Role matches<br/>the route?"}
    ROLE -->|"no"| FORBID["Redirect / flash — access denied"]
    ROLE -->|"yes"| OWN{"Ownership check<br/>teacher → own classrooms/learners<br/>parent → own child's data"}
    OWN -->|"no"| FORB
    OWN -->|"yes"| SVC["Run action + write to DB"]
    SVC --> DB[("MySQL")]
    SVC --> FL["Flash message + redirect"]
    R2["Any response"] --> HDR["Apply security headers<br/>CSP · X-Frame-Options:DENY · nosniff · Referrer-Policy"]
    HDR --> RENDER["Render Jinja template / JSON"]
    R --> LOG["Log to app.log (query string omitted)"]
```

---

## 4. Business workflows

### 4.1 Login & session routing

```mermaid
flowchart TD
    S[Enter username + password] --> RL{"Rate limit OK?"}
    RL -->|"no"| WAIT["Blocked — too many attempts, wait"]
    RL -->|"yes"| FIND["Find user by username / email"]
    FIND -->|"not found"| ERR["Invalid credentials (generic message)"]
    FIND --> PW{"Password hash<br/>matches?"}
    PW -->|"no"| ERR
    PW -->|"yes"| ROL{"Role?"}
    ROL -->|"admin"| APORT["Redirect to /admin/login<br/>— public login rejects admins"]
    ROL -->|"teacher"| TS{"Teacher status?"}
    TS -->|"pending"| PEND["Pending approval screen"]
    TS -->|"rejected"| REJ["Rejected — contact admin"]
    TS -->|"active"| TDASH["Teacher session + dashboard"]
    ROL -->|"parent"| PDASH["Parent session + dashboard"]
```

- Public `/login` **rejects administrator accounts** deliberately; admins sign in only through the dedicated admin portal.

### 4.2 Teacher registration & admin approval (gated by admin)

```mermaid
flowchart TD
    REG["Teacher fills registration form<br/>(account + teacher details + security answer)"] --> VAL{"Valid?<br/>strong password · email · duplicates"}
    VAL -->|"no"| BACK["Back to form with errors"]
    VAL -->|"yes"| TX["Transaction — insert users + teachers<br/>status = pending"]
    TX --> SUB["Flash: submitted for approval"]
    SUB -->|"cannot sign in yet"| LOGIN["Teacher blocked at login"]
    ADM["Admin /admin → Teachers → Pending"] --> LIST["List pending applications"]
    LIST --> D1{"Approve?"}
    D1 -->|"yes"| ACT["status = active"]
    ACT --> CANLOGIN["Teacher can now log in"]
    D1 -->|"no"| REJ2["status = rejected<br/>reason stored"]
```

### 4.3 Parent registration

```mermaid
flowchart TD
    P["Parent registration form<br/>(account + parent + child details)"] --> POK{"Valid?"}
    POK -->|"no"| PERR["Back to form with errors"]
    POK -->|"yes"| PTX["Transaction — insert users + parents + learners"]
    PTX --> PDONE["Parent logs in immediately"]
    PDONE --> PROFILE["Sees child profile<br/>(grade · attendance · BMI history)"]
    PROFILE --> REQ["Can request enrollment into a classroom"]
```

### 4.4 Enrollment request

```mermaid
flowchart TD
    EN["Parent: Find & Enroll"] --> BROWSE["Browse active classrooms<br/>(with capacity)"]
    BROWSE --> REQ2{"Request enrollment"}
    REQ2 --> DUPE{"Already enrolled<br/>in this classroom?"}
    DUPE -->|"yes"| MSG["Flash: already enrolled"]
    DUPE -->|"no"| PDUP{"Pending request exists?"}
    PDUP -->|"yes"| MSG2["Flash: already pending"]
    PDUP -->|"no"| INS[("INSERT enrollment_requests<br/>status = pending<br/>(unique classroom + learner)")]
    INS --> TSEES["Teacher sees request in their inbox"]
    TSEES --> DEC{"Accept / Reject"}
    DEC -->|"Accept"| ACC[("INSERT classroom_learners<br/>request → accepted")]
    ACC --> TRY{"Capacity reached?"}
    TRY -->|"yes"| FULL["Request blocked / flash: classroom full"]
    TRY -->|"no"| ENROLLED["Learner enrolled + parent notified"]
    DEC -->|"Reject"| REJ3["Request → rejected<br/>learner not added"]
```

### 4.5 Classwork & grading loop

```mermaid
flowchart TD
    CT["Teacher uploads classwork<br/>→ one of 7 subject folders<br/>(activity / assignment / worksheet / assessment / material)"] --> CINS[("INSERT classwork")]
    CINS --> CNOTIF["Notify parents → 'New Classwork'"]
    CNOTIF --> CPAR["Parent sees item in<br/>Classroom → Classwork & Grades"]
    CPAR --> COPEN["Parent opens classwork"]
    COPEN --> CSUB{"Parent submits work?"}
    CSUB -->|"yes"| CDONE[("INSERT classwork_submissions<br/>(classwork + learner)")]
    CDONE --> CDOCK["Appears in teacher<br/>'Pending to Grade' list"]
    CDOCK --> GR["Teacher grades 0–100 + remarks<br/>(inline or bulk)"]
    GR --> GINS[("INSERT / UPDATE assessments<br/>per learner + classwork")]
    GINS --> GNOTIF["Notify parent → 'Classwork Graded'"]
    GNOTIF --> GSHOW["Parent sees score:<br/>classwork page · child profile · dashboard<br/>report card / analytics"]
    CSUB -->|"no"| GSHOW2["Status stays 'Not graded yet'"]
```

### 4.6 Attendance recording

```mermaid
flowchart TD
    AT["Teacher opens monthly Attendance calendar<br/>(pick classroom + month)"] --> ASEL["Select a date"]
    ASEL --> AMARK["Mark each enrolled learner:<br/>Present / Absent / Tardy"]
    AMARK --> ASAVE["Save the form"]
    ASAVE --> AUPS[("Upsert per (learner, classroom, date)<br/>INSERT … ON DUPLICATE KEY UPDATE")]
    AUPS --> ACAL["Calendar renders P / A / T badges<br/>+ rate summary"]
    ACAL --> APAR["Parent sees child's history + rate<br/>on classroom page / child profile"]
```

### 4.7 Classroom Stream (posts)

```mermaid
flowchart TD
    P1["Teacher posts to Stream<br/>(type 'post'/'pinned-post' + title + content)"] --> PF{"Has file<br/>attachment?"}
    PF -->|"yes"| PUP["Save to uploads/posts"]
    PF -->|"no"| PNOP["no-op"]
    PUP --> PINS[("INSERT posts")]
    PNOP --> PINS
    PINS --> PTCH["Teacher Stream tab lists posts<br/>pinned posts first"]
    PINS --> PPAR["Parent 'Classroom Stream' tab shows the same posts<br/>(pinned first, type badge, download link)"]
```

### 4.8 Announcements

```mermaid
flowchart TD
    AN["Teacher writes announcement<br/>(category + priority)"] --> ANINS[("INSERT announcements")]
    ANINS --> ANARCH["Teacher announcements page<br/>(manage history)"]
    ANINS --> ANPAR["Parent announcements page<br/>school-wide, filterable by category"]
```

### 4.9 Forgot / recover password

```mermaid
flowchart TD
    FR["Forgot Password (/forgot-password)"] --> FRL{"Rate limit OK?"}
    FRL -->|"no"| FWAIT["Try again later"]
    FRL -->|"yes"| F1["Submit registered email"]
    F1 --> FFO{"Account found?"}
    FFO -->|"no"| FMSG["Generic response:<br/>reset link sent if account exists"]
    FFO -->|"yes"| FTOK["Generate single-use token<br/>(URL-safe), hash to DB<br/>expires in 2 hours"]
    FTOK --> SEND["Send reset email with link"]
    SEND --> FMSG
    FMSG --> USER["User clicks reset link"]
    USER --> V{"Token valid, unused,<br/>not expired?"}
    V -->|"no"| INV["Invalid/expired/used<br/>request new link"]
    V -->|"yes"| F3["Set new password + confirm"]
    F3 --> VALP{"Password valid + match?"}
    VALP -->|"no"| FERR["Back with error"]
    VALP -->|"yes"| FUPD[("Update password (hashed)<br/>mark token used, clear CSRF")]
    FUPD --> FDONE["Reset success - redirect to login"]
```

### 4.10 Notifications

```mermaid
flowchart TD
    EV["Application event<br/>new classwork · classwork graded · due reminders · enrollment results"] --> NINS[("INSERT notifications<br/>target = parent user_id")]
    NINS --> NBEL["Unread badge on the bell / dashboard"]
    NBEL --> NOPEN["Parent opens Notifications page"]
    NOPEN --> NRD["POST mark-as-read"]
    NRD --> NLINK["Redirect to the linked page and clear the alert"]
```

---

## 5. Data model (entity-relationship)

Complete schema: [`schema.sql`](../schema.sql) (19 tables). Key entities and their relationships:

```mermaid
erDiagram
    users ||--o| teachers : has
    users ||--o| parents : has
    users ||--o{ notifications : receives
    parents ||--o{ learners : has
    centers ||--o{ teachers : employs
    centers ||--o{ classrooms : runs
    centers ||--o{ learners : enrolls
    teachers ||--o{ classrooms : teaches
    teachers ||--o{ announcements : publishes
    classrooms ||--o{ posts : has_stream
    classrooms ||--o{ classwork : contains
    classrooms ||--o{ attendance : has_records
    classrooms ||--o{ enrollment_requests : receives
    learners ||--o{ classroom_learners : joins
    classrooms ||--o{ classroom_learners : contains
    learners ||--o{ classwork_submissions : submits
    classwork ||--o{ classwork_submissions : gets
    learners ||--o{ assessments : graded
    classwork ||--o{ assessments : graded_by
    learners ||--o{ attendance : has_days
    learners ||--o{ bmi_records : has_bmi
    learners ||--o{ immunization_records : has_vaccines
    learners ||--o{ enrollment_requests : sends

    users {
        int id PK
        string username UK
        string email UK
        string password_hash
        enum role "teacher | parent | admin"
    }
    learners {
        int id PK
        int parent_id FK
        int center_id FK
        string full_name
        enum gender
        date birthdate
        string photo
    }
    classrooms {
        int id PK
        int center_id FK
        int teacher_id FK
        string name
        string subject
        enum status "active | archived | draft"
        int max_learners
    }
    teachers {
        int id PK
        int user_id FK
        string employee_id UK
        string position
        enum status "pending | active | rejected"
    }
    classwork {
        int id PK
        int classroom_id FK
        string subject_folder
        enum type "activity | assignment | worksheet | assessment | material"
        string title
        date due_date
    }
    assessments {
        int id PK
        int learner_id FK
        int classwork_id FK
        int score "0-100"
        string remarks
    }
    attendance {
        int id PK
        int learner_id FK
        int classroom_id FK
        date attendance_date
        enum status "present | absent | tardy"
    }
    enrollment_requests {
        int id PK
        int learner_id FK
        int classroom_id FK
        enum status "pending | accepted | rejected"
    }
```

Notes:

- Every important table uses `CHECK` constraints on allowed status/type values, and referential integrity (`ON DELETE CASCADE` / `SET NULL`).
- `classroom_learners` is a join table with a **unique (classroom_id, learner_id)** pair — one learner per classroom.
- **No LRN (Learner Reference Number)** is in use — the legacy LRN column was removed from the UI and the production DB; only internal FK columns named `learner_id` remain.
- `immunization_records` is schema-reserved (no UI yet).