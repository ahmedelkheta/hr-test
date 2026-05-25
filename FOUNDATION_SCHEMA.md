<div dir="rtl">

# 🏗️ Foundation Schema — الأساس اللي هنبني عليه

> **الفكرة**: قبل أي feature، لازم يبقى عندنا foundation صلب من 5 طبقات.  
> لما الـ Foundation تخلص، تقدر تبني **أي feature** والتحكم يبقى موجود تلقائياً.

---

## 🎯 الـ 5 طبقات للـ Foundation

```
┌─────────────────────────────────────────────┐
│  Layer 5: RBAC + Scopes (الأمان والمرونة)    │  ← مرونة في "مين بيشوف إيه"
├─────────────────────────────────────────────┤
│  Layer 4: Users + Authentication            │  ← الـ Login
├─────────────────────────────────────────────┤
│  Layer 3: Employees                          │  ← الموظفين (يعتمدوا على Org)
├─────────────────────────────────────────────┤
│  Layer 2: Positions (المناصب)                │  ← الوظائف المعتمدة
├─────────────────────────────────────────────┤
│  Layer 1: Organization (Branches + Depts)    │  ← الهيكل التنظيمي
└─────────────────────────────────────────────┘
                    ↑
              Company (الشركة)
              (Tenant واحد أو متعدد)
```

---

## 1️⃣ Company (الشركة)

في النظام الحالي، شركة واحدة (Single Tenant). 

> 💡 لو يوم احتجت Multi-Tenant (نظام يخدم كذا شركة)، تضيف جدول `companies` و `company_id` في كل الجداول. **معماري النظام جاهز للنقلة دي**.

```sql
CREATE TABLE companies (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  legal_name VARCHAR(255),
  tax_id VARCHAR(50),
  industry VARCHAR(100),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 2️⃣ Branches (الفروع)

كل شركة لها فروع جغرافية.

```sql
CREATE TABLE branches (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  code VARCHAR(20) UNIQUE NOT NULL,    -- BR-CAI, BR-ALX
  region VARCHAR(100),
  address TEXT,
  phone VARCHAR(20),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**مثال**: المقر الرئيسي - القاهرة، فرع الإسكندرية، فرع كفر الشيخ، فرع طنطا.

---

## 3️⃣ Departments (الأقسام)

كل قسم بيخص فرع (في الـ Schema بتاعنا).

```sql
CREATE TABLE departments (
  id BIGSERIAL PRIMARY KEY,
  branch_id BIGINT NOT NULL REFERENCES branches(id),
  name VARCHAR(255) NOT NULL,
  code VARCHAR(50) NOT NULL,           -- DEPT-ALX-HR, DEPT-KFS-SLS
  parent_dept_id BIGINT REFERENCES departments(id),  -- للـ Sub-departments
  is_active BOOLEAN DEFAULT TRUE,
  UNIQUE(branch_id, code)
);
```

**مثال**:
- HR في الإسكندرية = `dept_id 2` (قسم مستقل)
- HR في كفر الشيخ = `dept_id 3` (قسم آخر مستقل)

> ⚠️ **قرار معماري**: لو الـ HR في القاهرة بيدير الـ HR في كل الفروع، يبقى عندك خيارين:
> 1. كل فرع له HR Dept خاص بيه (الحالي) + HR Director في القاهرة عنده `scope: company`
> 2. HR مركزي بـ `branch_id NULLABLE` (مش موصى بيه)

---

## 3.5️⃣ Teams (الفرق Cross-Branch) ⭐

عشان ندعم فرق بتخترق فروع متعددة.

```sql
CREATE TABLE teams (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  code VARCHAR(50) UNIQUE NOT NULL,
  description TEXT,
  is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE employee_teams (
  employee_id BIGINT NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
  team_id BIGINT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  role_in_team VARCHAR(100),           -- "Lead", "Member"
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (employee_id, team_id)
);
```

**مثال**: "Egypt Sales Squad" فيه موظفين من 3 فروع مختلفة.

---

## 4️⃣ Positions (المناصب) — ضرورية! ⭐

ده اللي كنا حاطينه string في `employees.job_title`. الأفضل نخليه جدول مستقل.

```sql
CREATE TABLE positions (
  id BIGSERIAL PRIMARY KEY,
  code VARCHAR(50) UNIQUE NOT NULL,    -- POS-SALES-REP
  title VARCHAR(255) NOT NULL,         -- "Sales Representative"
  level VARCHAR(50),                   -- "Junior", "Senior", "Manager"
  job_grade INT,                       -- درجة وظيفية (للمرتب)
  description TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### ليه نعمل جدول Positions منفصل؟

| السبب | البيان |
|------|--------|
| **توحيد المسميات** | "Sales Rep" مش "Sales Representative" مش "مندوب مبيعات" — كله يبقى واحد |
| **سهولة الـ Reporting** | "كم Sales Rep عندنا؟" بيبقى Query واحد بسيط |
| **ربط بدرجات الرواتب** | Position له job_grade مرتبط بسلم الرواتب |
| **معرفة الـ Career Path** | "Sales Rep → Senior Sales Rep → Sales Lead" |

---

## 5️⃣ Employees (الموظفين) — Core HR Entity

الكيان الأهم. بيربط كل اللي فوقه مع بعضه.

```sql
CREATE TABLE employees (
  id BIGSERIAL PRIMARY KEY,
  
  -- Identity (Optional)
  user_id BIGINT UNIQUE REFERENCES users(id) ON DELETE SET NULL,  -- ⚠️ NULLABLE
  
  -- Basic Info
  full_name VARCHAR(255) NOT NULL,
  national_id VARCHAR(20) UNIQUE NOT NULL,
  employee_code VARCHAR(50) UNIQUE NOT NULL,  -- EMP-2025-001
  date_of_birth DATE,
  gender VARCHAR(10),
  marital_status VARCHAR(20),
  
  -- Contact
  email VARCHAR(255),
  personal_phone VARCHAR(20),
  address TEXT,
  emergency_contact VARCHAR(255),
  
  -- Organizational (الربط بالـ Org)
  branch_id BIGINT NOT NULL REFERENCES branches(id),
  department_id BIGINT NOT NULL REFERENCES departments(id),
  position_id BIGINT NOT NULL REFERENCES positions(id),       -- ⭐ ربط بـ Positions
  manager_id BIGINT REFERENCES employees(id),                 -- Self-reference
  
  -- Employment
  hire_date DATE NOT NULL,
  termination_date DATE,
  employment_type VARCHAR(50),    -- full_time, part_time, contract, intern
  contract_end_date DATE,
  
  -- Status
  status VARCHAR(20) DEFAULT 'active',  -- active, on_leave, suspended, terminated
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### العلاقات (Relationships)

```
Employee ──────► Branch       (واحد لواحد)
Employee ──────► Department   (واحد لواحد)
Employee ──────► Position     (واحد لواحد)
Employee ──────► Manager      (Self-reference)
Employee ──────► User         (واحد لـ 0..1 — اختياري)
Employee ──◄M:N►── Teams      (Many-to-Many via employee_teams)
```

---

## 6️⃣ Users + Auth (الحسابات)

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  phone VARCHAR(20) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  status VARCHAR(20) DEFAULT 'pending_first_login',
  must_change_password BOOLEAN DEFAULT TRUE,
  failed_login_attempts INT DEFAULT 0,
  locked_until TIMESTAMPTZ,
  last_login_at TIMESTAMPTZ,
  last_login_ip INET,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sessions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_token_hash TEXT UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  ip_address INET,
  user_agent TEXT
);
```

---

## 7️⃣ RBAC + Scopes (السحر الفعلي) ⭐

```sql
-- الـ Roles المعرّفة
CREATE TABLE roles (
  id BIGSERIAL PRIMARY KEY,
  code VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL
);

-- الـ Permissions
CREATE TABLE permissions (
  id BIGSERIAL PRIMARY KEY,
  code VARCHAR(100) UNIQUE NOT NULL,    -- employee.read
  resource VARCHAR(50) NOT NULL,
  action VARCHAR(50) NOT NULL
);

-- ربط Roles بـ Permissions
CREATE TABLE role_permissions (
  role_id BIGINT REFERENCES roles(id),
  permission_id BIGINT REFERENCES permissions(id),
  PRIMARY KEY (role_id, permission_id)
);

-- ⭐ السحر: User Role + Scope
CREATE TABLE user_roles (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  role_id BIGINT NOT NULL REFERENCES roles(id),
  scope_type VARCHAR(20) NOT NULL CHECK (scope_type IN ('company', 'branch', 'department', 'team', 'employee')),
  scope_id BIGINT,
  expires_at TIMESTAMPTZ,
  UNIQUE(user_id, role_id, scope_type, scope_id)
);
```

---

## 🎯 إجابة سؤالك: "هل ده يكفي عشان نبني features؟"

### ✅ آه، **ده الـ Foundation الكامل**.

بعد ما الـ 7 طبقات دي تخلص، تقدر تبني **أي Feature** بـ Pattern موحد:

```
لإضافة Feature جديدة (مثلاً Leave Management):

1. ضيف جداول الـ Feature (leaves table)
2. ضيف Permissions جديدة (leave.request, leave.approve)
3. اربطهم بالـ Roles المناسبة
4. اكتب الـ Service مع authorize() من أول سطر
5. ✅ كل قواعد "مين يعمل إيه فين" بتشتغل تلقائياً
```

### إجابة "مين يدير مين":

| المرونة المطلوبة | الحل في النظام |
|------------------|----------------|
| مدير قسم يدير قسمه بس | `scope_type='department', scope_id=X` |
| مدير فرع يدير فرعه بس | `scope_type='branch', scope_id=Y` |
| مدير Sales في فروع متعددة | عضوية في Team + `scope_type='team'` |
| HR Director يشوف كل الشركة | `scope_type='company', scope_id=NULL` |
| موظف يشوف نفسه بس | `scope_type='employee', scope_id=own_employee_id` |
| اتنين HR Manager في فرعين مختلفين | نفس الـ Role × scopes مختلفة |
| موظف عنده دور إضافي مؤقت | INSERT صف جديد في user_roles بـ `expires_at` |

> 🎯 **كل سيناريو فيه بياناتك بيتحل بـ SQL واحد بدون تغيير في الكود**.

---

## 🎨 ادوات لرسم الـ DB Schema (الأفضل ترتيباً)

### 1. **dbdiagram.io** ⭐ (الأقوى للبداية)

- 🌐 [dbdiagram.io](https://dbdiagram.io)
- **مجاني** للاستخدام الأساسي
- بتكتب الـ Schema بـ DBML (لغة بسيطة جداً)
- بيرسم تلقائياً Diagram احترافي
- بتقدر تصدّر SQL أو PNG أو PDF
- مشاركة بـ link

**مثال DBML**:
```dbml
Table branches {
  id bigint [pk]
  name varchar
  code varchar [unique]
}

Table departments {
  id bigint [pk]
  branch_id bigint [ref: > branches.id]
  name varchar
}

Table employees {
  id bigint [pk]
  user_id bigint [ref: - users.id, null]
  branch_id bigint [ref: > branches.id]
  department_id bigint [ref: > departments.id]
}
```

### 2. **ChartDB** (الأحدث والأذكى)

- 🌐 [chartdb.io](https://chartdb.io)
- **مجاني ومفتوح المصدر**
- بيـ Connect مع PostgreSQL مباشرة ويستورد Schema تلقائياً
- بيرسم Diagram جميل جداً
- يدعم Dark Mode

### 3. **drawSQL**

- 🌐 [drawsql.app](https://drawsql.app)
- GUI Visual (تسحب وتفلت)
- مجاني للـ Public diagrams
- مناسب لو مش مرتاح بكتابة كود

### 4. **Lucidchart**

- 🌐 [lucidchart.com](https://www.lucidchart.com)
- احترافي جداً، بيستخدم في الشركات الكبيرة
- مدفوع (في trial)
- مناسب لو هتعمل Diagrams معقدة بتاخد وقت

### 5. **DataGrip / JetBrains** (لو عندك)

- IDE من JetBrains
- بيتصل بـ PostgreSQL مباشرة
- بيرسم ER Diagram من الـ DB الحقيقية تلقائياً

### 6. **pgAdmin ERD Tool**

- مجاني (مع pgAdmin)
- مدمج في pgAdmin مباشرة
- بسيط بس وظيفي

### 7. **Mermaid** (اللي إحنا مستخدمينه)

- 🌐 [mermaid.live](https://mermaid.live)
- نص بسيط بيتحول لـ Diagram
- مدمج في GitHub markdown
- مفيد لتوثيق الـ Schema في README

---

## 🎯 ترشيحي ليك

| لو عايز | استخدم |
|--------|--------|
| **تبدأ بسرعة وتشارك مع الفريق** | `dbdiagram.io` (الأقوى) |
| **تشوف الـ DB الحقيقية بعد ما تتبني** | `ChartDB` أو `DataGrip` |
| **توثيق في GitHub** | `Mermaid` (إحنا عاملينها في SYSTEM_DIAGRAMS.md) |
| **Diagrams معقدة + Team Collaboration** | `Lucidchart` |

### الـ Workflow اللي أنصحك بيه:

```
1. ابدأ بـ dbdiagram.io
   ├─ اكتب الـ Schema بـ DBML
   ├─ شوف الـ Diagram يطلع تلقائياً
   └─ صدّر SQL وحطه في drizzle/migrations

2. لما الـ DB تنبني فعلاً، استخدم ChartDB
   ├─ Connect على PostgreSQL
   └─ اعرض الـ Schema الحقيقية وقت ما تحب

3. في الـ Docs، استخدم Mermaid
   └─ Diagrams بسيطة بتعيش جنب الكود في GitHub
```

---

## 🏁 الخلاصة

| الطبقة | الجداول | الغرض |
|--------|---------|------|
| **1. Company** | `companies` (لو Multi-tenant) | الشركة |
| **2. Org Structure** | `branches`, `departments`, `teams`, `employee_teams` | الهيكل التنظيمي |
| **3. Positions** | `positions` | المناصب الموحدة |
| **4. Employees** | `employees` | الكيان الأساسي |
| **5. Identity** | `users`, `sessions`, `password_reset_otps` | الـ Login |
| **6. Authorization** | `roles`, `permissions`, `role_permissions`, `user_roles` | RBAC + Scopes |
| **7. Audit** | `audit_logs` | سجل العمليات |

> 🎯 **بعد ما الـ 7 طبقات دي تخلص**، تقدر تبني أي feature (Attendance, Leave, Payroll, Performance) **بثقة كاملة** إن الأمان والمرونة موجودين تلقائياً.

</div>
