<div dir="rtl">

# 🎨 System Diagrams — مخططات الترابط

> ملف بيشرح كل العلاقات في النظام بصرياً.  
> كل Diagram مكتوب بـ Mermaid syntax وبيتعرض تلقائياً في VS Code / GitHub / أي Markdown viewer بيدعم Mermaid.

---

## 1️⃣ The Big Picture — الصورة الكبيرة

```mermaid
flowchart TB
    subgraph ORG["🏢 Organization Structure"]
        B[Branches<br/>الفروع]
        D[Departments<br/>الأقسام]
        T[Teams<br/>الفرق متعددة الفروع]
    end

    subgraph HR["👥 HR Layer"]
        E[Employees<br/>الموظفين]
    end

    subgraph AUTH["🔐 Auth Layer (Optional)"]
        U[Users<br/>حسابات الدخول]
        R[Roles<br/>الأدوار]
        P[Permissions<br/>الصلاحيات]
        UR[User Roles + Scopes<br/>الإسناد بالنطاق]
    end

    B --> D
    B -.-> E
    D -.-> E
    T -.M:N.- E
    E -.0..1.-> U
    U --> UR
    R --> UR
    R --> P

    style ORG fill:#e1f5ff,stroke:#0288d1
    style HR fill:#fff4e1,stroke:#f57c00
    style AUTH fill:#f3e5f5,stroke:#7b1fa2
```

---

## 2️⃣ Entity Relationship Diagram (ERD) الكامل

```mermaid
erDiagram
    BRANCHES ||--o{ DEPARTMENTS : "contains"
    BRANCHES ||--o{ EMPLOYEES : "located in"
    DEPARTMENTS ||--o{ EMPLOYEES : "belongs to"

    EMPLOYEES ||--o| USERS : "may have (0..1)"
    EMPLOYEES ||--o{ EMPLOYEE_TEAMS : "member of"
    TEAMS ||--o{ EMPLOYEE_TEAMS : "has members"

    USERS ||--o{ SESSIONS : "has"
    USERS ||--o{ PASSWORD_RESET_OTPS : "requests"
    USERS ||--o{ USER_ROLES : "assigned"
    USERS ||--o{ AUDIT_LOGS : "performs"

    ROLES ||--o{ USER_ROLES : "granted via"
    ROLES ||--o{ ROLE_PERMISSIONS : "has"
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : "in role"

    BRANCHES {
        bigint id PK
        string name
        string code UK
        string region
    }

    DEPARTMENTS {
        bigint id PK
        bigint branch_id FK
        string name
        string code
    }

    TEAMS {
        bigint id PK
        string name
        string code UK
    }

    EMPLOYEES {
        bigint id PK
        bigint user_id FK "NULLABLE"
        string full_name
        string national_id UK
        string employee_code UK
        bigint branch_id FK
        bigint department_id FK
        bigint manager_id FK
        string status
    }

    EMPLOYEE_TEAMS {
        bigint employee_id FK
        bigint team_id FK
        string role_in_team
    }

    USERS {
        bigint id PK
        string phone UK
        string password_hash
        string status
        bool must_change_password
    }

    ROLES {
        bigint id PK
        string code UK
        string name
        bool is_system
    }

    PERMISSIONS {
        bigint id PK
        string code UK
        string resource
        string action
    }

    ROLE_PERMISSIONS {
        bigint role_id FK
        bigint permission_id FK
    }

    USER_ROLES {
        bigint id PK
        bigint user_id FK
        bigint role_id FK
        string scope_type "company|branch|department|team|employee"
        bigint scope_id
        timestamp expires_at
    }

    SESSIONS {
        bigint id PK
        bigint user_id FK
        string session_token_hash UK
        timestamp expires_at
    }

    PASSWORD_RESET_OTPS {
        bigint id PK
        bigint user_id FK
        string otp_hash
        timestamp expires_at
    }

    AUDIT_LOGS {
        bigint id PK
        bigint user_id FK
        string action
        string resource_type
        bigint resource_id
        jsonb metadata
    }
```

---

## 3️⃣ Build Order — ترتيب البناء

```mermaid
flowchart TB
    subgraph P1["🟢 Phase 1: Organization Structure"]
        direction LR
        B1[branches]
        D1[departments]
        T1[teams]
        B1 --> D1
    end

    subgraph P2["🟡 Phase 2: Employees"]
        direction LR
        E2[employees]
        ET2[employee_teams]
        E2 --> ET2
    end

    subgraph P3["🔵 Phase 3: Auth Layer"]
        direction TB
        P3A[3.A Identity<br/>users, sessions, OTPs]
        P3B[3.B RBAC<br/>roles, permissions]
        P3C[3.C Scopes<br/>user_roles]
        P3D[3.D Authorization<br/>middleware, audit]
        P3A --> P3B --> P3C --> P3D
    end

    P1 ==>|بيعتمد على| P2
    P2 ==>|بيعتمد على| P3

    style P1 fill:#c8e6c9,stroke:#388e3c
    style P2 fill:#fff9c4,stroke:#f9a825
    style P3 fill:#bbdefb,stroke:#1976d2
```

---

## 4️⃣ Two-Step Employee Creation

```mermaid
flowchart TD
    Start([HR بيضيف موظف جديد]) --> Step1[Step 1: اعمل Employee record]
    Step1 --> Save1[(employees table<br/>user_id = NULL)]
    Save1 --> Q{محتاج Login؟}

    Q -->|❌ لأ| End1([خلاص — موظف بدون Login<br/>زي عمال النظافة])
    Q -->|✅ آه| Step2[Step 2: اعمل User]

    Step2 --> Gen[ولّد Password عشوائي]
    Gen --> Hash[bcrypt hash]
    Hash --> Save2[(users table<br/>status=pending_first_login)]
    Save2 --> Link[ربط Employee.user_id]
    Link --> Roles[اسند Roles + Scopes]
    Roles --> SMS[ابعت SMS بالباسورد]
    SMS --> End2([خلاص — Employee + User])

    style Step1 fill:#fff9c4
    style Step2 fill:#bbdefb
    style End1 fill:#c8e6c9
    style End2 fill:#c8e6c9
    style Q fill:#ffe0b2
```

---

## 5️⃣ Authorization Flow

```mermaid
flowchart TD
    Req([Incoming Request<br/>GET /api/employees]) --> Cookie{في Session<br/>Token؟}
    Cookie -->|❌ لأ| Login[Redirect /login]
    Cookie -->|✅ آه| Valid{Session<br/>Valid؟}

    Valid -->|❌ Expired/Revoked| Login
    Valid -->|✅ Valid| Active{User<br/>Active؟}

    Active -->|❌ Disabled| Login
    Active -->|✅ Active| Load[Load User Context<br/>Roles + Scopes]

    Load --> Perm{عنده الـ<br/>Permission؟}
    Perm -->|❌ لأ| Deny[403 Forbidden]
    Perm -->|✅ آه| Scope[Apply Scope Filter<br/>على الـ Query]

    Scope --> Exec[Execute Filtered Query]
    Exec --> Audit[INSERT INTO audit_logs]
    Audit --> Resp([200 OK<br/>Filtered Data])

    style Req fill:#e3f2fd
    style Login fill:#ffcdd2
    style Deny fill:#ffcdd2
    style Resp fill:#c8e6c9
    style Audit fill:#fff9c4
```

---

## 6️⃣ Scope Types

```mermaid
flowchart TB
    Co[🌐 Company Scope<br/>scope_id = NULL<br/>Super Admin, HR Director]
    Br[🏢 Branch Scope<br/>scope_id = branches.id<br/>Branch Manager]
    De[📁 Department Scope<br/>scope_id = departments.id<br/>Dept Manager]
    Te[👥 Team Scope<br/>scope_id = teams.id<br/>Sales Manager Cross-Branch]
    Em[👤 Employee Scope<br/>scope_id = employees.id<br/>الموظف العادي - Self]

    Co --> Br
    Br --> De
    Br --> Te
    De --> Em
    Te --> Em

    style Co fill:#e1bee7,stroke:#6a1b9a
    style Br fill:#bbdefb,stroke:#1565c0
    style De fill:#c5e1a5,stroke:#558b2f
    style Te fill:#ffe082,stroke:#f57f17
    style Em fill:#ffccbc,stroke:#d84315
```

---

## 7️⃣ مثال: اتنين HR Manager في فرعين مختلفين

```mermaid
flowchart TB
    subgraph Roles["جدول roles"]
        HRRole[hr_manager Role<br/>واحد بس]
    end

    subgraph Perms["جدول permissions"]
        P1[employee.read]
        P2[employee.create]
        P3[payroll.process]
        P4[leave.approve]
    end

    HRRole --> P1
    HRRole --> P2
    HRRole --> P3
    HRRole --> P4

    subgraph Assignments["جدول user_roles"]
        A1[أحمد + hr_manager<br/>scope: branch:5<br/>إسكندرية]
        A2[محمد + hr_manager<br/>scope: branch:7<br/>كفر الشيخ]
    end

    HRRole -.تتسند لـ.-> A1
    HRRole -.تتسند لـ.-> A2

    A1 -.يشوف.-> EAlex[موظفين إسكندرية بس]
    A2 -.يشوف.-> EKafr[موظفين كفر الشيخ بس]

    style HRRole fill:#fff9c4
    style A1 fill:#bbdefb
    style A2 fill:#c8e6c9
```

---

## 8️⃣ Sales Manager Cross-Branch — Team Scope

```mermaid
flowchart TB
    subgraph Team1["Egypt Sales Team (team_id=1)"]
        TM[Team Definition]
    end

    subgraph Members["employee_teams (M:N)"]
        M1[موظف 1<br/>branch: إسكندرية]
        M2[موظف 2<br/>branch: كفر الشيخ]
        M3[موظف 3<br/>branch: طنطا]
    end

    TM --> M1
    TM --> M2
    TM --> M3

    Salma[سلمى<br/>Sales Manager]
    Salma -->|user_roles<br/>scope: team:1| TM
    Salma -.تشوفهم كلهم.-> M1
    Salma -.تشوفهم كلهم.-> M2
    Salma -.تشوفهم كلهم.-> M3

    style Salma fill:#fff9c4,stroke:#f57f17,stroke-width:3px
    style TM fill:#e1bee7,stroke:#6a1b9a
```

---

## 9️⃣ Layered Architecture

```mermaid
flowchart TB
    subgraph L1["📋 HR Layer (Foundation)"]
        L1a[branches]
        L1b[departments]
        L1c[teams]
        L1d[employees]
        L1e[employee_teams]
    end

    subgraph L2["🔐 Identity Layer (Optional)"]
        L2a[users]
        L2b[sessions]
        L2c[password_reset_otps]
    end

    subgraph L3["🛡️ Authorization Layer"]
        L3a[roles]
        L3b[permissions]
        L3c[role_permissions]
        L3d[user_roles + Scopes]
    end

    subgraph L4["📊 Audit Layer"]
        L4a[audit_logs]
    end

    L1 -.بيستخدمها.-> L2
    L2 -.بتستخدمها.-> L3
    L3 -.بتسجل في.-> L4

    style L1 fill:#c8e6c9
    style L2 fill:#bbdefb
    style L3 fill:#e1bee7
    style L4 fill:#ffe0b2
```

---

## 🔟 Login Flow (Sequence)

```mermaid
sequenceDiagram
    actor User as الموظف
    participant UI as Login Page
    participant API as Auth API
    participant DB as PostgreSQL

    User->>UI: يدخل Phone + Password
    UI->>API: POST /api/auth/login
    API->>DB: SELECT users WHERE phone=?
    DB-->>API: user record

    alt User مش موجود أو Disabled
        API-->>UI: 401 Unauthorized
    else User Locked
        API-->>UI: 423 Locked
    else bcrypt.compare fails
        API->>DB: UPDATE failed_login_attempts++
        API-->>UI: 401 Unauthorized
    else نجح
        API->>DB: INSERT INTO sessions
        API->>DB: INSERT INTO audit_logs
        API-->>UI: Set HTTP-only Cookie
        
        alt First Login
            UI-->>User: غيّر الباسورد إجباري
        else Normal
            UI-->>User: Redirect /dashboard
        end
    end
```

---

## 1️⃣1️⃣ Password Reset Flow

```mermaid
sequenceDiagram
    actor User as الموظف
    participant UI as Forgot Password
    participant API as Auth API
    participant DB as PostgreSQL
    participant SMS as SMS Service

    User->>UI: نسيت الباسورد
    User->>UI: يدخل رقم الموبايل
    UI->>API: POST /api/auth/forgot-password
    API->>API: ولّد OTP 6 أرقام
    API->>DB: INSERT INTO password_reset_otps
    API->>SMS: ابعت OTP للموبايل
    SMS-->>User: SMS: "كود: 123456"
    API-->>UI: 200 OK

    User->>UI: يدخل OTP + باسورد جديد
    UI->>API: POST /api/auth/reset-password
    API->>DB: SELECT otp WHERE NOT used
    
    alt OTP غلط
        API-->>UI: 400 Bad Request
    else نجح
        API->>DB: UPDATE password_hash
        API->>DB: DELETE FROM sessions (force logout)
        API->>DB: INSERT INTO audit_logs
        API-->>UI: 200 OK
    end
```

---

## 1️⃣2️⃣ Termination Flow

```mermaid
flowchart TD
    Start([HR بيفصل موظف]) --> S1[UPDATE employees<br/>status=terminated]

    S1 --> Q{الموظف عنده user_id؟}

    Q -->|❌| End1([خلاص])
    Q -->|✅| S2[UPDATE users<br/>status=disabled]

    S2 --> S3[DELETE sessions<br/>Force Logout]
    S3 --> S4[DELETE user_roles<br/>Revoke كل الـ Roles]
    S4 --> S5[INSERT audit_logs<br/>employee.terminated]
    S5 --> End2([خلاص])

    style S1 fill:#fff9c4
    style S2 fill:#ffccbc
    style S3 fill:#ffcdd2
    style S4 fill:#ffcdd2
    style End2 fill:#c8e6c9
```

---

## 🛠️ ازاي تشوف الـ Diagrams دي؟

### في VS Code
1. ثبّت Extension: `bierner.markdown-mermaid`
2. افتح الملف واضغط `Ctrl+Shift+V`

### في GitHub
- الملف هيتعرض تلقائياً بالـ Diagrams لما تـ commit.

### Online
- [mermaid.live](https://mermaid.live) — انسخ أي Diagram وشوفه.

</div>
