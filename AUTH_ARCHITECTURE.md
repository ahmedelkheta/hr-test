<div dir="rtl">

# 🔐 Authentication & Authorization Architecture
## مرجع شامل لبناء طبقة الأمان في نظام HR Management System

> **النسخة**: 1.0  
> **اللغة**: عربي مصري (بمصطلحات تقنية إنجليزي)  
> **الجمهور المستهدف**: Backend Engineers, Software Architects, Tech Leads  
> **Tech Stack**: Next.js (Full-stack) + PostgreSQL  
> **مستوى التفصيل**: Production-Grade (مش Tutorial)

---

## 📑 جدول المحتويات

0. [🏛️ Core Architecture Principles (المبادئ الأساسية)](#-core-architecture-principles-المبادئ-الأساسية) ⭐ ابدأ من هنا
0.5 [🛣️ Build Order — ترتيب بناء النظام](#-build-order--ترتيب-بناء-النظام) ⭐ خريطة التنفيذ
1. [مقدمة وأهداف التوثيق](#1-مقدمة-وأهداف-التوثيق)
2. [الفرق الجوهري بين Authentication و Authorization](#2-الفرق-الجوهري-بين-authentication-و-authorization)
3. [مفهوم Identity vs Access](#3-مفهوم-identity-vs-access)
4. [Users vs Employees — السؤال الأهم](#4-users-vs-employees--السؤال-الأهم)
5. [ربط Users بـ Employees — Approach Decision](#5-ربط-users-بـ-employees--approach-decision)
6. [Database Design المقترح](#6-database-design-المقترح)
7. [RBAC (Role-Based Access Control) — الشرح العميق](#7-rbac-role-based-access-control--الشرح-العميق)
8. [Scopes & Multi-Tenancy — حل مشكلة الفروع](#8-scopes--multi-tenancy--حل-مشكلة-الفروع)
9. [Branch Access & Department Access](#9-branch-access--department-access)
10. [Authorization Lifecycle — خطوة بخطوة](#10-authorization-lifecycle--خطوة-بخطوة)
11. [Authentication Flow التفصيلي](#11-authentication-flow-التفصيلي)
12. [Session Management في Next.js](#12-session-management-في-nextjs)
13. [Audit Logs & Compliance](#13-audit-logs--compliance)
14. [⭐ أفضل Architecture مقترحة للمشروع ده](#14--أفضل-architecture-مقترحة-للمشروع-ده)
15. [ملحق: Quick Reference Tables](#15-ملحق-quick-reference-tables)

---

## 🏛️ Core Architecture Principles (المبادئ الأساسية)

> ⚠️ **اقرأ القسم ده الأول، حتى لو معندكش وقت لقراءة الباقي.**  
> ده الـ Mental Model اللي كل النظام بيتبني عليه.

النظام مبني على **3 مفاهيم أساسية** لازم تكون واضحة في دماغك قبل ما تكتب أي سطر كود:

### 1. Employee (الكيان الأساسي في HR)

- يمثل **كل شخص في الشركة بدون استثناء**
- بيخزن بيانات HR فقط (مرتب، فرع، قسم، منصب، ...)
- **كل شخص في المؤسسة لازم يبقى Employee**
- **مش كل Employee لازم يبقى User**

### 2. User (كيان المصادقة)

- يمثل صلاحية الدخول للنظام
- بيتعمل **بس** لو الـ Employee محتاج Access للنظام
- كل User **لازم** يكون مربوط بـ Employee واحد بالظبط
- العلاقة: `Employee 1 ──── 0..1 User`

### 3. Authorization Layer (طبقة التحكم في الصلاحيات)

- بتتحكم في:
  - **WHAT**: ايه الأفعال المسموحة (Permissions في الـ Role)
  - **WHERE / ON WHOM**: على فين/مين الأفعال دي تتطبق (Scopes)

### 🎯 الـ Diagram الأساسي

```
┌──────────────────────────────────────────────────────────────┐
│                    HR Management System                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   ╔═══════════════╗  1     0..1   ╔═══════════════╗         │
│   ║   Employee    ╠═════════════════╣     User      ║         │
│   ║  (دايماً       ║                 ║  (اختياري)    ║         │
│   ║  موجود)        ║                 ║              ║         │
│   ╚═══════╤═══════╝                 ╚══════╤═══════╝         │
│           │                                │                 │
│           │ HR Data:                       │ Identity:       │
│           │ - Branch                       │ - Phone         │
│           │ - Department                   │ - Password Hash │
│           │ - Salary                       │ - Sessions      │
│           │ - Position                     │ - Status        │
│           │                                │                 │
│           │                                ▼                 │
│           │                       ╔════════════════╗         │
│           │                       ║ Authorization  ║         │
│           │                       ║    Layer       ║         │
│           │                       ╠════════════════╣         │
│           │                       ║  Roles         ║         │
│           │                       ║  Permissions   ║         │
│           │                       ║  Scopes ⭐     ║         │
│           │                       ╚════════════════╝         │
│           │                                                  │
└───────────┴──────────────────────────────────────────────────┘
```

### 🛡️ القواعد الذهبية اللي بتحكم النظام

#### قاعدة 1: Employee First, User Second (دايماً)

عند إنشاء أي شخص جديد في الشركة:

```
الخطوة 1: اعمل Employee record (إجباري)
الخطوة 2: اسأل: "هل الشخص ده محتاج Login على النظام؟"
         │
         ├─ آه → اعمل User → اربطه بالـ Employee → اسند Roles + Scopes
         │
         └─ لأ → كفاية، الـ Employee بيشتغل كـ HR record بس
```

> **Login is a feature, not a requirement.**

#### قاعدة 2: Don't Encode Scope Inside Roles

> ❌ **خطأ كارثي**: تعمل Roles بأسماء فيها مكان زي:
> - `sales_manager_alexandria`
> - `sales_manager_kafr_elsheikh`
> - `hr_manager_cairo`
>
> ✅ **التصميم الصح**: Role واحد + Users كتير، كل واحد Scope مختلف:
> - Role: `sales_manager` (واحد بس)
> - User أحمد + Role `sales_manager` + Scope `branch:5`
> - User محمد + Role `sales_manager` + Scope `branch:7`

#### قاعدة 3: Scopes Are Dynamic & Multi-Dimensional

الـ Scope مش بُعد واحد. الـ User الواحد ممكن يكون عنده:
- Scope على Department معين
- Scope على Branch تاني
- Scope على Team بيخترق فروع
- كل ده في نفس الوقت

#### قاعدة 4: Separation of Concerns

| الطبقة | المسؤولية |
|--------|----------|
| **Employee** | بيانات HR فقط |
| **User** | المصادقة فقط |
| **Role** | تحديد الـ Capabilities (إيه يقدر يعمل) |
| **Scope** | تحديد الحدود (على إيه يقدر يعمله) |

كل طبقة بمسؤولية واحدة، ومش بتتداخل مع غيرها.

#### قاعدة 5: Authorization Runtime = Permission Check + Scope Check

لما User يطلب data، النظام بيتحقق من **اتنين** بالترتيب:

```
Step 1: Validate Permission
        هل الـ Role بتاع الـ User فيه الـ Permission المطلوب؟
        مثلاً: عنده employee.read؟
        
        ❌ لأ → Deny (403)
        ✅ آه → Continue
        
Step 2: Validate Scope
        هل الـ Target Entity داخل في الـ Scope بتاعه؟
        مثلاً: الموظف ده branch_id = 5، والـ User scope = branch:5؟
        
        ❌ لأ → Deny (403)
        ✅ آه → Allow
```

**القاعدة**: لو **الاتنين** نجحوا → Allow. غير كده → Deny.

---

## 🛣️ Build Order — ترتيب بناء النظام

> ⚠️ **اقرأ القسم ده قبل ما تبدأ تكتب أي كود.**  
> ده الترتيب الصحيح للـ Implementation. أي حاجة بره الترتيب ده هتسببلك dependency issues.

### الفلسفة

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│         Phase 3: Users + RBAC + Scopes                   │
│         ─────────────────────────────                    │
│         Identity + Authentication + Authorization        │
│         (آخر حاجة — Optional Layer)                       │
│                          ▲                               │
│                          │ بيعتمد على                      │
│                          │                               │
│         Phase 2: Employees                               │
│         ──────────────────                               │
│         HR Data Core                                     │
│         (التاني — Foundation HR)                          │
│                          ▲                               │
│                          │ بيعتمد على                      │
│                          │                               │
│         Phase 1: Organization Structure                  │
│         ────────────────────────────                     │
│         Branches + Departments + Teams                   │
│         (الأول — مفيش Dependencies)                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

> 💡 **القاعدة**: ابني من **تحت لفوق**. مفيش Layer يتعمل قبل اللي تحته.

---

### 🟢 Phase 1: Organization Structure (الأساس)

ده اللي بتبدأ بيه. **مفيش حاجة بتعتمد على حاجة قبله**.

#### الجداول المطلوبة

```sql
1. branches            -- الفروع
2. departments         -- الأقسام (FK لـ branches)
3. teams               -- الفرق متعددة الفروع
```

#### اللي بتعمله في الـ Phase دي

- [ ] إنشاء الـ Migrations للجداول التلاتة
- [ ] Seed Data للفروع الأساسية (مثلاً: القاهرة، إسكندرية، كفر الشيخ)
- [ ] CRUD APIs لـ Branches (مفيش Auth لسه!)
- [ ] CRUD APIs لـ Departments
- [ ] CRUD APIs لـ Teams
- [ ] UI بسيط لإدارة الـ Org Structure (Admin Dashboard)
- [ ] Validation: مينفعش تحذف Branch فيه Departments

#### اللي **مش** بتعمله في الـ Phase دي

- ❌ Auth (مفيش login لسه)
- ❌ Permissions
- ❌ User accounts

#### معيار النجاح

> ✅ تقدر تضيف فرع جديد، تضيف فيه قسم، تضيف Team، وكله بيشتغل من غير ما يكون عندك أي user أو موظف في النظام.

---

### 🟡 Phase 2: Employees (الـ HR Core)

دلوقتي وإنت عندك Org Structure شغالة، تقدر تضيف **الموظفين**.

#### الجداول المطلوبة

```sql
4. employees           -- الموظفين (FK لـ branches, departments)
5. employee_teams      -- M:N بين Employees و Teams
```

#### اللي بتعمله في الـ Phase دي

- [ ] Migration لـ `employees` (مع `user_id NULLABLE`)
- [ ] Migration لـ `employee_teams`
- [ ] CRUD APIs لـ Employees (مفيش Auth برضو!)
- [ ] إدارة عضوية الفرق (Add/Remove Member)
- [ ] Employee Code Generation (EMP-2025-001, ...)
- [ ] Validation: National ID فريد، Employee Code فريد
- [ ] UI لإضافة موظف جديد + ربطه بفرع وقسم
- [ ] Self-Referential Manager Relationship (manager_id)

#### اللي **مش** بتعمله في الـ Phase دي

- ❌ Login لسه (الموظف موجود في DB بس)
- ❌ Authorization checks (`employee.user_id` لسه NULL لكل الموظفين)

#### معيار النجاح

> ✅ تقدر تضيف 100 موظف على النظام بدون أي Login، وتربطهم بفروع وأقسام وفرق. تقدر تشوف بياناتهم بـ raw SQL queries.

> 💡 **ملاحظة**: في الـ Phase دي، الـ HR Team بيشتغل مباشرة على الـ DB أو من خلال Admin Tool بسيط. مفيش Authentication بعد، لأن لسه مفيش Users.

---

### 🔵 Phase 3: Users + RBAC + Scopes (طبقة الأمان)

دلوقتي بقى عندك Org + Employees. وقت الـ Auth Layer.

#### الجداول المطلوبة

```sql
6. users                  -- الـ Identity
7. sessions               -- Active sessions
8. password_reset_otps    -- OTPs
9. roles                  -- الأدوار
10. permissions           -- الصلاحيات
11. role_permissions      -- M:N
12. user_roles            -- ⭐ Roles + Scopes (الـ assignment)
13. audit_logs            -- سجل العمليات
```

#### الـ Phase دي بنقسمها لـ 4 Sub-Phases

##### 3.A — Identity & Authentication

- [ ] جدول `users`
- [ ] جدول `sessions` + `password_reset_otps`
- [ ] Endpoint: `POST /api/auth/login`
- [ ] Endpoint: `POST /api/auth/logout`
- [ ] Endpoint: `POST /api/auth/forgot-password` (OTP)
- [ ] Endpoint: `POST /api/auth/reset-password`
- [ ] Endpoint: `POST /api/auth/change-password`
- [ ] Two-Step Employee Creation (Employee → "needs login?" → User)
- [ ] First Login Flow + Force Change Password
- [ ] Middleware: Session Validation
- [ ] Brute Force Protection (5 attempts → lock)

##### 3.B — RBAC Foundation

- [ ] جدول `roles` + Seed Data (super_admin, hr_manager, ...)
- [ ] جدول `permissions` + Seed Data (employee.read, ...)
- [ ] جدول `role_permissions` + Seed (HR Manager = [employee.*, ...])
- [ ] Admin UI لإدارة Roles والـ Permissions

##### 3.C — Scopes Assignment

- [ ] جدول `user_roles` بـ `scope_type` + `scope_id`
- [ ] UI لإسناد Role + Scope لـ User
- [ ] Validation: scope_id لازم يكون موجود في الجدول المناسب
- [ ] Multi-role per user support
- [ ] Role expiration (`expires_at`) support

##### 3.D — Authorization Middleware

- [ ] `authorize()` helper function
- [ ] `getScopeFilter()` helper
- [ ] تطبيق Scope Filter على كل API endpoint بيرجع data
- [ ] Audit Logs على كل عملية حساسة
- [ ] Force Logout لما الموظف يتفصل

#### معيار النجاح

> ✅ تقدر تعمل Login بـ Phone + Password، تشوف بياناتك بناء على Scope بتاعك، وأي محاولة دخول لمكان مش مسموح بترد 403.

---

### 📋 الـ Build Order كـ Checklist مختصر

```
PHASE 1: ORG STRUCTURE
  ☐ branches table + CRUD
  ☐ departments table + CRUD
  ☐ teams table + CRUD
  ☐ Seed initial branches/departments

PHASE 2: EMPLOYEES
  ☐ employees table (user_id NULL)
  ☐ employee_teams junction
  ☐ Employee CRUD (no auth)
  ☐ Team membership management

PHASE 3: AUTH LAYER
  ├ 3.A Identity
  │  ☐ users + sessions + OTPs
  │  ☐ Login/Logout/Reset flows
  │  ☐ Two-Step Creation (Employee → User)
  │  
  ├ 3.B RBAC
  │  ☐ roles + permissions + role_permissions
  │  ☐ Seed system roles
  │  
  ├ 3.C Scopes
  │  ☐ user_roles (with scope_type, scope_id)
  │  ☐ Role assignment UI
  │  
  └ 3.D Authorization
     ☐ authorize() middleware
     ☐ getScopeFilter() helper
     ☐ Audit logging
     ☐ Force logout on termination
```

---

### ⚠️ أخطاء شائعة في الـ Build Order

#### ❌ Anti-Pattern 1: تبدأ بـ Auth قبل ما يبقى عندك Org

**الموقف**: المطور بيكتب `users` table في أول migration.

**المشكلة**: لما تيجي تربطها بـ Employees و Branches، هتلاقي نفسك بترجع تعدل الـ migrations.

**الحل**: ابدأ بـ Org. الـ Auth أصلاً ميتنفعش بدون Employees.

#### ❌ Anti-Pattern 2: تخلط بين الـ Phases

**الموقف**: تعمل `users` و `employees` في نفس الـ Migration، وتربطهم mandatory.

**المشكلة**: مينفعش تضيف Employee بدون Login، وده ضد قاعدة "Login is a Feature, Not a Requirement".

**الحل**: كل Phase في Migration منفصلة. الـ FK `employees.user_id` لازم يكون **NULLABLE**.

#### ❌ Anti-Pattern 3: تسيب Authorization للآخر

**الموقف**: تخلص كل الـ Features وبعدين تيجي تضيف Authorization.

**المشكلة**: هتلاقي نفسك بتعيد كتابة كل الـ Queries عشان تضيف Scope Filters.

**الحل**: ابدأ Authorization مبكراً (في Phase 3) واكتب الـ Query Filtering من الأول.

#### ❌ Anti-Pattern 4: تأخر الـ Audit Logs

**الموقف**: "هضيف الـ logs بعدين."

**المشكلة**: لما تحصل مشكلة في Production، هتبقى مش عارف مين عمل إيه.

**الحل**: الـ Audit Logging جزء من Phase 3.D، **مش optional**.

---

### 🎯 خلاصة الـ Build Order

| Phase | اللي بتعمله | اللي بتأجله | المدة التقديرية |
|-------|-------------|-------------|------------------|
| **1. Org Structure** | Branches, Departments, Teams | Auth, Employees | 1-2 أيام |
| **2. Employees** | HR Data, Team Membership | Auth, Permissions | 3-5 أيام |
| **3.A Identity** | Users, Sessions, Login | RBAC, Scopes | 3-4 أيام |
| **3.B RBAC** | Roles, Permissions | Scopes | 2-3 أيام |
| **3.C Scopes** | user_roles + Assignment UI | Middleware | 2-3 أيام |
| **3.D Authorization** | Middleware + Audit | - | 3-5 أيام |

> 💡 **المجموع التقديري**: 14-22 يوم عمل (3-4 أسابيع) للـ Foundation الكاملة.

> 💡 **بعد كده**: تقدر تبدأ تضيف الـ Features الفعلية (Payroll, Attendance, Leave Management, ...) فوق الـ Foundation دي.

---

### 🔄 Auth Lifecycle — امتى تضيف Permissions و Roles؟

> **السؤال الشائع**: هل أعمل Auth في بداية المشروع، آخره، ولا بالتوازي مع كل Feature؟

**الإجابة**: مفيش لا الأول ولا الآخر — هما **مرحلتين**:

#### المرحلة 1: Auth Foundation (مرة واحدة — Phase 3)

```
بعد Phase 1 + 2 (Org + Employees)
       ↓
ابني Auth Foundation كاملة:
  ✓ users, sessions, password_reset_otps
  ✓ roles + permissions + role_permissions tables
  ✓ user_roles (مع scopes)
  ✓ audit_logs
  ✓ Login / Logout / Reset flows
  ✓ authorize() middleware
  ✓ getScopeFilter() helper
  ✓ Seed: System Roles (super_admin, hr_manager, ...)
  ✓ Seed: Base Permissions (employee.*, branch.*, ...)
```

**معيار النجاح**: تقدر تعمل Login، تشوف بياناتك حسب Scope، وأي عملية بتتسجل في audit_logs.

#### المرحلة 2: Per-Feature Auth (مدى الحياة)

كل Feature جديدة بعد كده **لازم** تيجي مع الـ Auth بتاعتها:

```
لما هتضيف Module جديد (مثلاً Payroll):
       ↓
1. ️حدد الـ Permissions الجديدة:
   payroll.read, payroll.process, payroll.adjust
       ↓
2. ️اعمل Migration: INSERT INTO permissions
       ↓
3. ️اربطها بالـ Roles المناسبة:
   INSERT INTO role_permissions (hr_manager_id, payroll.process_id)
       ↓
4. ️اكتب الـ Feature مع authorize() من أول سطر:
   await authorize(userId, 'payroll.process', { scopeType, scopeId });
       ↓
5. ️ضيف Audit Logs للعمليات الحساسة
       ↓
6. ️قبل ما تـ commit، اسأل نفسك:
   "هل في endpoint بدون authorize()?"
```

#### الـ Anti-Patterns اللي لازم تتجنبها

| ❌ Anti-Pattern | اللي بيحصل | ✅ الحل |
|----------------|-------------|--------|
| "هضيف Auth بعد ما أخلص كل الـ Features" | Refactor 3 شهور + bugs أمنية | Foundation أولاً، قبل أي Feature |
| "كل Permission أعمله لما أحتاجه فقط" | Permissions متناثرة بدون system | كل Module بيـ seed permissions في migration |
| "Authorization في الـ Frontend بس" | Security catastrophe | Authorization في Service Layer دايماً |
| "هاسبق Foundation وأبدأ Features بسرعة" | كل Feature بتاخد 3× وقتها بعدين | استثمر الأسبوع في Foundation |
| "Roles ثابتة في الـ Code (hardcoded)" | تغيير Role = تغيير code + deploy | Roles في DB، تتعدل من Admin UI |

#### مثال عملي: إضافة Leave Management Feature

**قبل ما تكتب أي كود في الـ UI**:

```sql
-- 1. ضيف الـ Permissions في Migration جديدة
INSERT INTO permissions (code, resource, action, description) VALUES
  ('leave.request',         'leave', 'request',         'طلب أجازة'),
  ('leave.read',            'leave', 'read',            'عرض الأجازات'),
  ('leave.approve',         'leave', 'approve',         'اعتماد أجازة'),
  ('leave.reject',          'leave', 'reject',          'رفض أجازة'),
  ('leave.cancel',          'leave', 'cancel',          'إلغاء أجازة'),
  ('leave.policy.manage',   'leave', 'policy.manage',   'إدارة سياسات الأجازات');

-- 2. اربطهم بالـ Roles المناسبة
INSERT INTO role_permissions (role_id, permission_id) VALUES
  -- HR Manager بياخد كل permissions الـ leave
  ((SELECT id FROM roles WHERE code='hr_manager'), (SELECT id FROM permissions WHERE code='leave.request')),
  ((SELECT id FROM roles WHERE code='hr_manager'), (SELECT id FROM permissions WHERE code='leave.read')),
  ((SELECT id FROM roles WHERE code='hr_manager'), (SELECT id FROM permissions WHERE code='leave.approve')),
  -- ...
  
  -- Department Manager بياخد read + approve لقسمه
  ((SELECT id FROM roles WHERE code='dept_manager'), (SELECT id FROM permissions WHERE code='leave.read')),
  ((SELECT id FROM roles WHERE code='dept_manager'), (SELECT id FROM permissions WHERE code='leave.approve')),
  
  -- الموظف العادي بياخد request بس
  ((SELECT id FROM roles WHERE code='employee'),    (SELECT id FROM permissions WHERE code='leave.request'));
```

**بعدها، كل Endpoint في الـ Feature بيستخدم authorize()**:

```typescript
// src/modules/leave/services/leave.service.ts

export async function requestLeave(userId: number, leaveData: LeaveRequest) {
  await authorize(userId, 'leave.request', { scopeType: 'employee', scopeId: userId });
  // ... logic
  await auditLog({ userId, action: 'leave.requested', ... });
}

export async function approveLeave(userId: number, leaveId: number) {
  const leave = await leaveRepo.findById(leaveId);
  await authorize(userId, 'leave.approve', { 
    scopeType: 'department', 
    scopeId: leave.employee.department_id 
  });
  // ... logic
  await auditLog({ userId, action: 'leave.approved', resourceId: leaveId });
}
```

#### الـ Workflow Visualization

```
┌─────────────────────────────────────────────────────────────┐
│  Foundation Build (مرة واحدة، أسبوع)                          │
│  ───────────────────────────────                            │
│  Users + RBAC + Scopes + audit_logs                         │
│  + System Roles + Base Permissions                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Feature Loop (متكرر، مع كل Module جديد)                     │
│  ─────────────────────────                                  │
│                                                             │
│  1. ضيف Permissions جديدة في DB                              │
│  2. اربطهم بالـ Roles المناسبة                                │
│  3. اكتب الـ Service مع authorize() من أول سطر               │
│  4. اكتب الـ UI                                              │
│  5. ضيف Audit Logs                                          │
│  6. Test الـ Auth بـ Roles مختلفة                            │
│                                                             │
│  → افضل تكرر الـ Loop ده لكل Module جديد                     │
└─────────────────────────────────────────────────────────────┘
```

#### Checklist لكل Feature جديدة

قبل ما تـ Merge أي PR لـ Feature جديدة، تأكد من:

- [ ] الـ Permissions الجديدة موجودة في Migration
- [ ] الـ Roles المناسبة عندهم الـ Permissions
- [ ] كل Endpoint بيستخدم `authorize()` قبل أي logic
- [ ] الـ Queries بتفلتر بالـ Scope (مش بتجيب كل البيانات)
- [ ] العمليات الحساسة بتـ log في `audit_logs`
- [ ] في Tests بتجرب Permissions غلط (Negative Tests)
- [ ] الـ UI بتخفي الأزرار اللي مش متاحة للـ User حسب Permissions

#### القاعدة الذهبية

> 🎯 **مفيش Feature بتدخل Production بدون Auth Layer كاملة.**  
> الـ Auth مش "Feature آخر" — هي **Prerequisite** لأي Feature تانية.

---

## 1. مقدمة وأهداف التوثيق

### مين بيقرا الملف ده؟

الملف ده **مرجع تقني** لأي حد هيشتغل على نظام الـ HR من الناحية الأمنية. سواء كنت:

- 👨‍💻 الـ Developer اللي بيكتب الكود
- 🏗️ الـ Architect اللي بيصمم النظام
- 🧪 الـ QA اللي بيختبر الأمان
- 👔 الـ Tech Lead اللي بيراجع الـ PRs

الكل لازم يرجع للملف ده قبل ما يلمس أي حاجة في الـ Auth Layer.

### ليه التوثيق ده مهم قبل ما نكتب أي سطر كود؟

في الـ HR Systems تحديداً، أي خطأ في الـ Authorization ممكن يخليك تتفرج على:

- 📉 موظف يشوف مرتبات زملاؤه
- 💼 مدير فرع يعدّل على موظفين فرع تاني
- 🔓 موظف اتفصل لسه عنده Access على البيانات
- 📰 تسريب بيانات شخصية (National ID, Salary) في الجرايد

النوع ده من البقات **مش بق برمجي عادي**، ده ممكن يبقى **مشكلة قانونية** بسبب قوانين حماية البيانات الشخصية.

> 💡 **القاعدة الذهبية**: في الـ Auth، الـ Default دايماً يبقى **DENY**. لو مفيش Rule صريحة تقول "هذا المستخدم يقدر يعمل كذا"، الجواب لازم يبقى لأ.

### الفلسفة العامة (Clean Architecture Thinking)

إحنا بنبني النظام بناءً على 4 مبادئ أساسية:

```
┌─────────────────────────────────────────────────┐
│  1. Separation of Concerns                      │
│     افصل الـ Identity (مين) عن الـ HR Data (إيه)│
├─────────────────────────────────────────────────┤
│  2. Principle of Least Privilege                │
│     ادي كل واحد أقل صلاحيات ممكنة                │
├─────────────────────────────────────────────────┤
│  3. Defense in Depth                            │
│     طبقات حماية متعددة (مش طبقة واحدة)          │
├─────────────────────────────────────────────────┤
│  4. Auditability                                │
│     كل حاجة تتسجل، حتى المحاولات الفاشلة         │
└─────────────────────────────────────────────────┘
```

---

## 2. الفرق الجوهري بين Authentication و Authorization

### تعريف بسيط

**Authentication (المصادقة)**:
> هي عملية التأكد إن **اللي بيدخل النظام هو فعلاً هو**.  
> يعني الإجابة على سؤال: **"مين انت؟"**

**Authorization (التفويض)**:
> هي عملية التأكد إن **اللي دخل بالفعل ليه الحق إنه يعمل العملية دي**.  
> يعني الإجابة على سؤال: **"إنت مسموح لك تعمل ايه؟"**

### مثال من الحياة الواقعية

تخيل إنك داخل شركة:

| المرحلة | في الحياة | في النظام |
|---------|-----------|-----------|
| الـ Security بيتأكد من بطاقتك على الباب | Authentication ✅ | Login بالـ Phone + Password |
| دخلت الشركة بس مش عارف تدخل غرفة المدير العام | Authorization ❌ | Permission Check |
| محاسب يقدر يفتح برنامج المرتبات لكن مش يقدر يدخل سيرفر روم | Authorization | RBAC |

### جدول مقارنة تفصيلي

| الجانب | Authentication | Authorization |
|--------|---------------|---------------|
| **السؤال** | مين انت؟ | تقدر تعمل ايه؟ |
| **بيحصل امتى** | في بداية الـ Session | في كل Request |
| **بيعتمد على** | Credentials (Password, OTP, Biometric) | Roles & Permissions |
| **النتيجة** | Token / Session | Allow / Deny |
| **بيتخزن فين** | Sessions Table / JWT | في الـ DB (Roles, Permissions) |
| **مين المسؤول عنه** | Auth Service | Authorization Middleware |
| **لو فشل** | 401 Unauthorized | 403 Forbidden |
| **مرات في الـ Lifecycle** | مرة واحدة (Login) | كل Request |

### ASCII Diagram يوضح الـ Flow

```
┌──────────────┐
│   المستخدم    │
└──────┬───────┘
       │
       │ (1) بيدخل Phone + Password
       ▼
┌──────────────────────────────────┐
│  AUTHENTICATION LAYER            │
│  هل ده فعلاً أحمد؟                  │
│  ─────────────────────           │
│  ✓ موجود في users table          │
│  ✓ Password Hash matches         │
│  ✓ Account ليس Disabled         │
└──────┬───────────────────────────┘
       │
       │ ✅ آه ده أحمد فعلاً
       │
       ▼
┌──────────────────────────────────┐
│   إصدار Session / Token           │
└──────┬───────────────────────────┘
       │
       │ (2) بيطلب /api/employees
       ▼
┌──────────────────────────────────┐
│  AUTHORIZATION LAYER             │
│  هل أحمد يقدر يشوف الموظفين؟        │
│  ─────────────────────           │
│  ✓ Role: HR Manager              │
│  ✓ Has Permission: employee.read │
│  ✓ Scope: Branch #5              │
│  → فيلتر النتايج على Branch #5    │
└──────┬───────────────────────────┘
       │
       │ ✅ مسموح بس بفلتر
       ▼
┌──────────────────────────────────┐
│   النتايج (موظفين فرع #5 بس)     │
└──────────────────────────────────┘
```

### أمثلة واقعية من HR System

**سيناريو 1: موظف عادي بيحاول يشوف مرتبه**
- Authentication: ✅ (هو فعلاً نفسه)
- Authorization: ✅ (مسموح يشوف مرتبه هو بس، Scope = Self)

**سيناريو 2: مدير قسم بيحاول يدخل على بيانات قسم تاني**
- Authentication: ✅ (هو فعلاً المدير)
- Authorization: ❌ (Scope بتاعه قسمه بس، مش الأقسام التانية)

**سيناريو 3: موظف اتفصل لسه بيحاول يدخل**
- Authentication: ❌ (الـ account اتعمله Disable)
- مش حتى هيوصل للـ Authorization

---

## 3. مفهوم Identity vs Access

ده مفهوم متقدم بس مهم جداً تفهمه عشان تبني نظام Scalable.

### الفرق

**Identity (الهوية)**:
> هي **مين** أنت كشخص. اسمك، رقم موبايلك، بصمتك، كله Identity.  
> **ثابتة** نسبياً ومش بتتغير بسرعة.

**Access (الصلاحية)**:
> هي **ايه** اللي تقدر تعمله. الصلاحيات بتاعتك، الفروع اللي تقدر تشوفها، الأقسام اللي تتحكم فيها.  
> **متغيرة** باستمرار حسب الترقيات والنقل والفصل.

### ليه الفصل بينهم مهم في Enterprise Systems؟

```
                   Identity                Access
                   ────────                ──────
   ٢٠٢٢:  أحمد محمد (موظف)         →   يشوف بياناته بس
   ٢٠٢٣:  أحمد محمد (Team Lead)    →   يشوف فريقه
   ٢٠٢٤:  أحمد محمد (Manager)      →   يشوف القسم كله
   ٢٠٢٥:  أحمد محمد (HR Director)  →   يشوف كل الشركة
                ▲                              ▲
                │                              │
            ثابت طول العمر               بيتغير كل سنة
```

**الـ Identity هي أحمد**. ده مش بيتغير. لكن **الـ Access بيتغير** مع الترقيات.

لو دمجت الـ Identity مع الـ Access في جدول واحد، هتلاقي نفسك بتغير صف يخص الشخص في كل ترقية، وده **anti-pattern**.

### التطبيق العملي في الـ Schema

```sql
-- Identity Layer (مين هو)
users (id, phone, password_hash, ...)
employees (id, user_id, full_name, national_id, ...)

-- Access Layer (يقدر يعمل ايه)
user_roles (user_id, role_id, scope_type, scope_id)
roles (id, name, ...)
permissions (id, name, ...)
```

**أحمد** صف واحد في `users` و `employees` طول حياته في الشركة.  
**صلاحيات أحمد** ممكن تتغير عشرات المرات في `user_roles` بدون ما تلمس بياناته الشخصية.

> ⚠️ **خطأ شائع**: تخزين الـ Role كعمود في جدول `users` (مثلاً `users.role = 'manager'`). ده يخليك تبني نظام محدود ميقدرش الموظف الواحد ياخد أكتر من Role.

---

## 4. Users vs Employees — السؤال الأهم

### يعني ايه User؟

**User** = حساب رقمي يقدر يعمل **Login** على النظام.  
ده كيان (entity) رقمي مهمته **Authentication**.

**خصائصه**:
- ليه Phone Number فريد
- ليه Password Hash
- ممكن يبقى Active أو Disabled
- بيدخل النظام بشكل مباشر

### يعني ايه Employee؟

**Employee** = شخص بيشتغل في الشركة.  
ده كيان (entity) إداري مهمته **HR Operations**.

**خصائصه**:
- ليه اسم كامل، رقم قومي، تاريخ توظيف
- منتمي لفرع وقسم
- ليه مرتب وحضور وانصراف
- ليه مدير مباشر
- **ممكن يبقى عنده Login، وممكن لأ**

### السؤال الأهم: هل كل Employee لازم يكون User؟

**الإجابة القاطعة: لأ.**

في أي HR System حقيقي، في موظفين مش محتاجين Login أصلاً:

| نوع الموظف | محتاج Login؟ | السبب |
|-----------|--------------|-------|
| مدير الشركة | ✅ أكيد | بيشوف Reports ويصرح بقرارات |
| HR | ✅ أكيد | بيدير النظام |
| Software Engineer | ✅ آه | بيسجل حضور، يطلب أجازات |
| محاسب | ✅ آه | بيدخل على النظام المالي |
| عامل نظافة | ❌ لأ | مفيش حاجة يعملها في النظام |
| Driver | ❌ غالباً لأ | بيتسجل حضوره يدوياً |
| Security Guard | ❌ غالباً لأ | بيتسجل حضوره يدوياً |
| Field Worker | ❓ يعتمد | لو في Mobile App آه، غير كده لأ |

### متى الموظف يحتاج Login ومتى لا؟

**يحتاج Login لما:**
- محتاج يدخل على النظام بنفسه (يطلب إجازة، يشوف مرتبه)
- محتاج يستخدم Self-Service Portal
- ليه صلاحيات إدارية على موظفين تانيين
- بيستخدم أدوات الشركة الرقمية (Email, Tools)

**مش محتاج Login لما:**
- بياناته بتتدار من HR فقط
- مفيش Self-Service محتاجه
- شغله مش متعلق بأي نظام رقمي
- لسه ما اتعينش رسمياً (Probation period - في بعض الحالات)

### 3 سيناريوهات واقعية

**🔹 سيناريو 1: موظف جديد لسه ما اتسلمش الأوراق**
- في `employees`: موجود بـ status = "pending_documents"
- في `users`: مش موجود لسه
- HR لما يخلص ورق، بيعمل له User ويبعتله بياناته

**🔹 سيناريو 2: عامل نظافة بيشتغل بنظام يومية**
- في `employees`: موجود (عشان نسجل حضوره ومرتبه)
- في `users`: ❌ مش موجود (مش محتاج Login)
- Security بيسجل حضوره يدوياً في النظام

**🔹 سيناريو 3: موظف اتفصل**
- في `employees`: موجود بـ status = "terminated" + termination_date
- في `users`: status = "disabled" (مش بنحذفه عشان Audit)
- لما حد يحاول يدخل بـ phone بتاعه → 401

### الخلاصة

> 💡 **القاعدة**: كل **User** بيخص **Employee واحد** فقط (في نظامنا)، لكن **مش كل Employee** لازم يكون عنده User.

العلاقة بين الجدولين هي:

```
Users ──── (1:0..1) ──── Employees
                ▲
                │
       يعني: كل Employee إما عنده User واحد، أو مفيش User خالص.
```

---

## 5. ربط Users بـ Employees — Approach Decision

ده القرار المعماري الأهم في النظام. في 3 طرق معروفة:

### 🅰️ Approach A: Monolithic (User و Employee في جدول واحد)

```sql
-- جدول واحد فيه كل حاجة
CREATE TABLE users_employees (
    id BIGSERIAL PRIMARY KEY,
    phone VARCHAR(20),
    password_hash TEXT,
    full_name VARCHAR(255),
    national_id VARCHAR(20),
    hire_date DATE,
    branch_id BIGINT,
    -- ... كل حاجة هنا
);
```

#### ✅ المزايا
- بسيط جداً للأنظمة الصغيرة
- Query واحد بييجي بكل البيانات
- مفيش Joins

#### ❌ العيوب
- **موظف بدون Login = صفوف فيها NULLs كتير** (phone NULL, password NULL)
- صعب تضيف User ميخصش موظف (زي Vendor User أو Admin External)
- Indexing معقد
- لو احتجت Multi-Auth (موظف ممكن يبقى عنده أكتر من Login) → مستحيل
- بيكسر مبدأ Single Responsibility

#### 💡 امتى تستخدمه
**نادراً**. ممكن في Startups صغيرة جداً أو MVPs.

---

### 🅱️ Approach B: Linked (User و Employee منفصلين بـ Foreign Key) ⭐

```sql
-- جدولين منفصلين
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    phone VARCHAR(20) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    -- Identity-related فقط
);

CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE SET NULL,  -- ⚠️ NULLABLE
    full_name VARCHAR(255) NOT NULL,
    national_id VARCHAR(20) UNIQUE,
    hire_date DATE,
    branch_id BIGINT,
    -- HR Data فقط
);
```

#### ✅ المزايا
- **فصل واضح** بين الـ Identity والـ HR Data
- موظف يقدر يبقى **بدون Login** (user_id = NULL)
- User يقدر يبقى **مش موظف** (مثلاً Admin External) — بس Employee مش موجود ليه
- Indexing نظيف على كل جدول
- سهل توسعته (تضيف External Users, API Users, إلخ)
- **يدعم Multi-Branch RBAC** بكفاءة عالية

#### ❌ العيوب
- Joins ضرورية عشان تجيب بيانات كاملة
- لازم تحافظ على Referential Integrity
- شوية معقد عن Approach A

#### 💡 امتى تستخدمه
**دايماً تقريباً** في الـ HR Systems متوسطة الحجم وكبيرة الحجم.

#### 🎯 ده الـ Approach اللي إحنا هنستخدمه

---

### 🅲 Approach C: Decoupled (Polymorphic Identity)

```sql
-- User يقدر يخص أي حاجة (Employee, Vendor, Customer)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    phone VARCHAR(20) UNIQUE,
    password_hash TEXT,
    identity_type VARCHAR(50),    -- 'employee', 'vendor', 'customer'
    identity_id BIGINT,           -- بيشاور على ID في الجدول المناسب
    -- ...
);

CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    full_name VARCHAR(255),
    -- ... HR Data
);

CREATE TABLE vendors (
    id BIGSERIAL PRIMARY KEY,
    company_name VARCHAR(255),
    -- ...
);
```

#### ✅ المزايا
- مرونة قصوى لو عندك Identities متنوعة
- يدعم Multi-Tenant بشكل ممتاز
- Future-proof لأقصى درجة

#### ❌ العيوب
- **معقد جداً** لـ HR System عادي
- بيكسر Foreign Key Constraints (مفيش FK من users لـ employees بشكل صريح)
- Joins مضحكة
- Over-engineering لمعظم الحالات

#### 💡 امتى تستخدمه
لو بتبني SaaS Platform بتخدم آلاف الشركات، أو System فيه User بيلعب أكتر من دور (موظف + Vendor + Customer).

---

### جدول المقارنة الشامل

| المعيار | Monolithic (A) | Linked (B) ⭐ | Decoupled (C) |
|---------|---------------|---------------|----------------|
| التعقيد | منخفض | متوسط | عالي |
| المرونة | منخفضة | عالية | عالية جداً |
| الـ Performance | عالي (مفيش Joins) | عالي (Joins بسيطة) | متوسط |
| موظف بدون Login | ❌ صعب | ✅ سهل | ✅ سهل |
| Identity تخص حاجة غير Employee | ❌ مستحيل | ⚠️ ممكن بتعديلات | ✅ Native |
| Multi-Branch RBAC | ⚠️ صعب | ✅ ممتاز | ✅ ممتاز |
| للـ HR System العادي | ❌ | ✅ **التوصية** | ❌ Over-engineered |

### 🎯 التوصية النهائية: Approach B (Linked)

**ليه؟**

1. **يحل مشكلتك مباشرة**: عندك موظفين بـ Login وموظفين بدون Login → `user_id NULLABLE` كده تمام
2. **يدعم النمو**: لو يوم احتجت تضيف Admin External أو API User، تقدر تضيفه في `users` بدون `employee`
3. **Performance ممتاز**: Joins بسيطة و Indexing نظيف
4. **Standard في الصناعة**: 90% من HR Systems الناجحة بتستخدمه
5. **يدعم RBAC بكفاءة**: نقدر نربط Roles بـ Users بدون تعقيد

---

## 6. Database Design المقترح

ده الـ Schema الكامل بالتفصيل. كل جدول هنشرحه وهنشرح **ليه** كل عمود موجود.

### Overview Diagram

```
                        ┌──────────────┐
                        │   branches   │
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │ departments  │
                        └──────┬───────┘
                               │
┌──────────┐            ┌──────▼───────┐
│  users   │◄─(1:0..1)──┤  employees   │
└────┬─────┘            └──────────────┘
     │
     │ (1:N)
     ▼
┌──────────────┐       ┌──────────────┐
│  user_roles  ├──────►│    roles     │
└──────────────┘       └──────┬───────┘
                              │ (M:N)
                       ┌──────▼──────────┐
                       │ role_permissions│
                       └──────┬──────────┘
                              │
                       ┌──────▼──────────┐
                       │   permissions    │
                       └─────────────────┘

┌──────────────┐
│   sessions   │ (1:N من users)
└──────────────┘

┌──────────────────────┐
│ password_reset_otps  │ (1:N من users)
└──────────────────────┘

┌──────────────┐
│  audit_logs  │ (1:N من users)
└──────────────┘
```

### 6.1 جدول `users` (Identity Layer)

```sql
CREATE TABLE users (
    id                      BIGSERIAL PRIMARY KEY,
    phone                   VARCHAR(20) UNIQUE NOT NULL,
    password_hash           TEXT NOT NULL,
    
    -- حالة الحساب
    status                  VARCHAR(20) NOT NULL DEFAULT 'pending_first_login'
                            CHECK (status IN ('pending_first_login', 'active', 'disabled', 'locked')),
    
    -- إجبار تغيير الباسورد
    must_change_password    BOOLEAN NOT NULL DEFAULT TRUE,
    
    -- حماية من Brute Force
    failed_login_attempts   INT NOT NULL DEFAULT 0,
    locked_until            TIMESTAMPTZ,
    
    -- تتبع
    last_login_at           TIMESTAMPTZ,
    last_login_ip           INET,
    password_changed_at     TIMESTAMPTZ DEFAULT NOW(),
    
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_status ON users(status);
```

**شرح كل عمود:**

| العمود | الغرض | ملاحظات |
|--------|-------|---------|
| `id` | Primary Key | BIGSERIAL عشان نتسع للمستقبل |
| `phone` | الـ Username بتاعنا | UNIQUE وممنوع NULL |
| `password_hash` | الـ Hash بـ bcrypt | **مفيش plaintext خالص** |
| `status` | حالة الحساب | ENUM-like بـ CHECK |
| `must_change_password` | إجبار تغيير الباسورد عند أول دخول | حل آمن لمشكلة "بنبعت البيانات للموظفين" |
| `failed_login_attempts` | عداد للحماية من Brute Force | بعد 5 محاولات → lock |
| `locked_until` | لحد امتى الحساب مقفول | بنرفعه بعد فترة |
| `last_login_at` + `last_login_ip` | تتبع آخر دخول | مفيد للأمن والـ Audit |
| `password_changed_at` | امتى آخر تغيير للباسورد | لتطبيق Password Rotation Policies |

### 6.2 جدول `employees` (HR Data Layer)

```sql
CREATE TABLE employees (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT UNIQUE REFERENCES users(id) ON DELETE SET NULL,
    
    -- البيانات الشخصية
    full_name           VARCHAR(255) NOT NULL,
    national_id         VARCHAR(20) UNIQUE NOT NULL,
    date_of_birth       DATE,
    gender              VARCHAR(10) CHECK (gender IN ('male', 'female')),
    
    -- بيانات الاتصال
    email               VARCHAR(255),
    personal_phone      VARCHAR(20),
    address             TEXT,
    
    -- بيانات التوظيف
    employee_code       VARCHAR(50) UNIQUE NOT NULL,  -- زي EMP-2025-001
    hire_date           DATE NOT NULL,
    termination_date    DATE,
    job_title           VARCHAR(255) NOT NULL,
    employment_type     VARCHAR(50) CHECK (employment_type IN ('full_time', 'part_time', 'contract', 'intern')),
    
    -- علاقات
    branch_id           BIGINT NOT NULL REFERENCES branches(id),
    department_id       BIGINT NOT NULL REFERENCES departments(id),
    manager_id          BIGINT REFERENCES employees(id) ON DELETE SET NULL,
    
    -- حالة
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'on_leave', 'terminated', 'suspended')),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_employees_user_id ON employees(user_id);
CREATE INDEX idx_employees_branch_id ON employees(branch_id);
CREATE INDEX idx_employees_department_id ON employees(department_id);
CREATE INDEX idx_employees_manager_id ON employees(manager_id);
CREATE INDEX idx_employees_status ON employees(status);
CREATE INDEX idx_employees_employee_code ON employees(employee_code);
```

**نقاط مهمة في الـ Schema:**

1. `user_id NULLABLE` ← ده اللي بيخلي عندك موظف بدون Login
2. `user_id UNIQUE` ← كل User يخص موظف واحد بس
3. `ON DELETE SET NULL` ← لو حذفت User، الموظف يفضل موجود بس بدون Login
4. `manager_id` بيشاور على نفس الجدول (Self-Referencing) ← بيبني الـ Hierarchy
5. Indexes على كل Foreign Key عشان الـ Queries تطير

### 6.3 جدول `branches` (الفروع)

```sql
CREATE TABLE branches (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) UNIQUE NOT NULL,  -- BR-CAI-01
    region          VARCHAR(100),
    address         TEXT,
    phone           VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6.4 جدول `departments` (الأقسام)

```sql
CREATE TABLE departments (
    id              BIGSERIAL PRIMARY KEY,
    branch_id       BIGINT NOT NULL REFERENCES branches(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,  -- DEPT-IT
    parent_dept_id  BIGINT REFERENCES departments(id),  -- لو في Sub-departments
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE(branch_id, code)
);

CREATE INDEX idx_departments_branch_id ON departments(branch_id);
```

> 💡 **ملاحظة**: القسم بيخص فرع معين. يعني فرع القاهرة عنده قسم IT، وفرع الإسكندرية عنده قسم IT تاني (مختلف). لو عايز قسم Global، اعمل `branch_id NULLABLE`.

### 6.4b جدول `teams` (الفرق متعددة الفروع)

عشان ندعم سيناريوهات زي "Sales Team عبر فروع متعددة"، عندنا جدول Teams بيخترق الفروع:

```sql
CREATE TABLE teams (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) UNIQUE NOT NULL,    -- TEAM-SALES-EG
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**الفرق بين Department و Team:**

| الجانب | Department | Team |
|--------|-----------|------|
| Scope جغرافي | داخل فرع واحد | بيخترق فروع متعددة |
| العضوية | كل Employee في department واحد فقط (FK في employees) | Employee ممكن يبقى في أكتر من Team (M:N) |
| الاستخدام | التنظيم الإداري الرسمي | تنظيمات Cross-functional, Cross-branch |

### 6.4c جدول `employee_teams` (M:N بين Employees و Teams)

```sql
CREATE TABLE employee_teams (
    employee_id     BIGINT NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    team_id         BIGINT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    role_in_team    VARCHAR(100),                  -- "Lead", "Member"
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (employee_id, team_id)
);

CREATE INDEX idx_employee_teams_employee_id ON employee_teams(employee_id);
CREATE INDEX idx_employee_teams_team_id ON employee_teams(team_id);
```

> 💡 **استخدام**: لو عندك "Egypt Sales Team" فيه موظفين من فروع إسكندرية وكفر الشيخ وطنطا، كلهم بيكونوا rows في `employee_teams` بنفس `team_id`. الـ Sales Manager بياخد scope = `team:1` فيشوفهم كلهم.

### 6.5 جدول `roles` (الأدوار)

```sql
CREATE TABLE roles (
    id              BIGSERIAL PRIMARY KEY,
    code            VARCHAR(50) UNIQUE NOT NULL,    -- 'hr_manager', 'branch_manager'
    name            VARCHAR(255) NOT NULL,           -- "مدير موارد بشرية"
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,  -- الـ system roles مش بتتحذف
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Seed Data
INSERT INTO roles (code, name, is_system) VALUES
('super_admin',     'مدير النظام الأعلى',  TRUE),
('hr_manager',      'مدير موارد بشرية',     TRUE),
('hr_specialist',   'أخصائي موارد بشرية',   TRUE),
('branch_manager',  'مدير فرع',             TRUE),
('dept_manager',    'مدير قسم',             TRUE),
('team_lead',       'قائد فريق',            TRUE),
('employee',        'موظف',                 TRUE);
```

### 6.6 جدول `permissions` (الصلاحيات)

```sql
CREATE TABLE permissions (
    id          BIGSERIAL PRIMARY KEY,
    code        VARCHAR(100) UNIQUE NOT NULL,   -- 'employee.create'
    resource    VARCHAR(50) NOT NULL,            -- 'employee'
    action      VARCHAR(50) NOT NULL,            -- 'create'
    description TEXT,
    
    UNIQUE(resource, action)
);

-- Seed Data
INSERT INTO permissions (code, resource, action, description) VALUES
('employee.create',     'employee',  'create', 'إضافة موظف جديد'),
('employee.read',       'employee',  'read',   'عرض بيانات الموظفين'),
('employee.update',     'employee',  'update', 'تعديل بيانات موظف'),
('employee.delete',     'employee',  'delete', 'حذف موظف'),
('payroll.process',     'payroll',   'process','معالجة المرتبات'),
('payroll.read',        'payroll',   'read',   'عرض المرتبات'),
('attendance.read',     'attendance','read',   'عرض الحضور'),
('attendance.approve',  'attendance','approve','اعتماد الحضور'),
('leave.request',       'leave',     'request','طلب أجازة'),
('leave.approve',       'leave',     'approve','اعتماد الأجازات'),
('report.financial',    'report',    'financial','عرض التقارير المالية'),
('settings.manage',     'settings',  'manage', 'إدارة إعدادات النظام');
```

### 6.7 جدول `role_permissions` (M:N بين Roles وPermissions)

```sql
CREATE TABLE role_permissions (
    role_id         BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   BIGINT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by      BIGINT REFERENCES users(id),
    
    PRIMARY KEY (role_id, permission_id)
);

CREATE INDEX idx_role_permissions_role_id ON role_permissions(role_id);
CREATE INDEX idx_role_permissions_permission_id ON role_permissions(permission_id);
```

### 6.8 جدول `user_roles` (M:N + Scopes — هنا السحر!) ⭐

```sql
CREATE TABLE user_roles (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         BIGINT NOT NULL REFERENCES roles(id),
    
    -- 🎯 ده الجزء المهم: Scope
    scope_type      VARCHAR(20) NOT NULL 
                    CHECK (scope_type IN ('company', 'branch', 'department', 'team', 'employee')),
    scope_id        BIGINT,  -- بيشاور على Entity من النوع المحدد في scope_type
    
    -- Metadata
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by      BIGINT REFERENCES users(id),
    expires_at      TIMESTAMPTZ,  -- ممكن نخلي Role مؤقت
    revoked_at      TIMESTAMPTZ,
    
    UNIQUE(user_id, role_id, scope_type, scope_id)
);

CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role_id ON user_roles(role_id);
CREATE INDEX idx_user_roles_scope ON user_roles(scope_type, scope_id);
```

**شرح الـ Scope:**

| `scope_type` | `scope_id` | المعنى |
|-------------|-----------|--------|
| `company` | NULL | الـ User ليه Role على الشركة كلها (Super Admin, HR Director) |
| `branch` | `branches.id` | الـ User ليه Role على فرع معين |
| `department` | `departments.id` | الـ User ليه Role على قسم معين |
| `team` | `teams.id` | الـ User ليه Role على Team بيخترق فروع متعددة |
| `employee` | `employees.id` | الـ User ليه Role على موظف واحد (عادةً نفسه = Self) |

> 🎯 **هنا اللي بيحل مشكلة "اتنين HR Manager في فرعين مختلفين"** — هنشرحها بالتفصيل في Section 8.

### 6.9 جدول `sessions` (Active Sessions)

```sql
CREATE TABLE sessions (
    id                      BIGSERIAL PRIMARY KEY,
    user_id                 BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Session Token (Hashed)
    session_token_hash      TEXT NOT NULL UNIQUE,
    refresh_token_hash      TEXT UNIQUE,
    
    -- Device Info
    ip_address              INET,
    user_agent              TEXT,
    device_type             VARCHAR(50),
    
    -- Timing
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_activity_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at              TIMESTAMPTZ NOT NULL,
    
    -- Force Logout Support
    revoked_at              TIMESTAMPTZ
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_token_hash ON sessions(session_token_hash);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

### 6.10 جدول `password_reset_otps` (OTPs لـ Reset)

```sql
CREATE TABLE password_reset_otps (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    otp_hash        TEXT NOT NULL,           -- ⚠️ بنحفظ Hash مش الـ OTP
    expires_at      TIMESTAMPTZ NOT NULL,
    used_at         TIMESTAMPTZ,             -- بنحدد لما يتستخدم
    attempts        INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Defense: مش أكتر من OTP واحد Active في نفس الوقت
    CHECK (attempts <= 5)
);

CREATE INDEX idx_otps_user_id ON password_reset_otps(user_id);
CREATE INDEX idx_otps_expires_at ON password_reset_otps(expires_at);
```

### 6.11 جدول `audit_logs` (سجل العمليات)

```sql
CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id) ON DELETE SET NULL,
    
    -- ايه اللي حصل
    action          VARCHAR(100) NOT NULL,     -- 'employee.created', 'login.success'
    resource_type   VARCHAR(50),                -- 'employee'
    resource_id     BIGINT,                     -- بيشاور على ID الـ resource
    
    -- التفاصيل
    metadata        JSONB,                      -- {old: {...}, new: {...}}
    
    -- المعلومات الجغرافية والتقنية
    ip_address      INET,
    user_agent      TEXT,
    
    -- النتيجة
    status          VARCHAR(20) NOT NULL DEFAULT 'success'
                    CHECK (status IN ('success', 'failed', 'denied')),
    error_message   TEXT,
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes كتيرة عشان الـ Queries على الـ logs بتكون ثقيلة
CREATE INDEX idx_audit_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_action ON audit_logs(action);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_created_at ON audit_logs(created_at DESC);

-- Partitioning بالشهر لو الـ logs هتبقى كبيرة
-- CREATE TABLE audit_logs_2026_05 PARTITION OF audit_logs
--   FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

> ⚠️ **مهم**: الـ `audit_logs` table **Append-only**. مش هنحدّث أو نحذف منها أبداً. ده عشان لو حد حاول يخفي حاجة، الـ logs تكشفه.

---

## 7. RBAC (Role-Based Access Control) — الشرح العميق

### يعني ايه RBAC؟

**RBAC** = نموذج إدارة الصلاحيات اللي بيقول:
> "بدل ما تدي كل User صلاحيات لوحده، اعمل **Roles** فيها مجموعات من الصلاحيات، وادي الـ User الـ Role."

### الفرق بين RBAC و ACL

**ACL (Access Control List)**:
```
User: أحمد
Permissions: [employee.read, employee.update, payroll.read, attendance.approve, ...]

User: محمد  
Permissions: [employee.read, employee.update, payroll.read, attendance.approve, ...]
```
❌ تكرار كبير، صعب الإدارة لما يبقى عندك 1000 موظف.

**RBAC**:
```
Role: HR Manager
Permissions: [employee.read, employee.update, payroll.read, attendance.approve, ...]

User: أحمد → Role: HR Manager
User: محمد → Role: HR Manager
```
✅ تغيير على الـ Role بيتطبق على كل اليوزرز اللي عندهم الـ Role دي.

### يعني ايه Role؟

**Role** = مجموعة من الـ Permissions بتعبر عن دور وظيفي.

أمثلة في الـ HR System:

```
┌─────────────────────────────────────────────────────┐
│ Role: HR Manager                                    │
│ ─────────────────                                   │
│   ✓ employee.create                                 │
│   ✓ employee.read                                   │
│   ✓ employee.update                                 │
│   ✓ payroll.process                                 │
│   ✓ leave.approve                                   │
│   ✓ report.financial                                │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Role: Branch Manager                                │
│ ─────────────────                                   │
│   ✓ employee.read                                   │
│   ✓ employee.update                                 │
│   ✓ leave.approve                                   │
│   ✓ attendance.approve                              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Role: Employee                                      │
│ ─────────────────                                   │
│   ✓ employee.read (own data only - via Scope!)      │
│   ✓ leave.request                                   │
│   ✓ attendance.read (own only)                      │
└─────────────────────────────────────────────────────┘
```

### يعني ايه Permission؟

**Permission** = صلاحية محددة لعمل **Action** على **Resource**.

الـ Format المتعارف عليه:
```
{resource}.{action}
```

أمثلة:
- `employee.create` = أقدر أضيف موظف
- `payroll.process` = أقدر أعمل المرتبات
- `report.financial` = أقدر أشوف التقارير المالية

> 💡 **Best Practice**: استخدم نفس الـ Naming Convention عبر النظام كله. `employee.create` مش `create_employee` ولا `addEmployee`.

### العلاقة Many-to-Many

```
Roles ←──── role_permissions ────→ Permissions
                M:N
```

- الـ Role الواحد ليه Permissions كتير
- الـ Permission الواحد ممكن يبقى في Roles كتير

### جدول Roles مقترح لـ HR System

| Role Code | الاسم | الوصف | Permissions |
|-----------|------|------|-------------|
| `super_admin` | مدير النظام الأعلى | تحكم كامل في كل حاجة | كل الـ permissions |
| `hr_manager` | مدير HR | إدارة شاملة للموظفين | employee.*, payroll.*, leave.approve, ... |
| `hr_specialist` | أخصائي HR | عمليات يومية على الموظفين | employee.read, employee.update, leave.read |
| `branch_manager` | مدير فرع | إدارة فرع | employee.read, leave.approve, attendance.approve |
| `dept_manager` | مدير قسم | إدارة قسم | employee.read, leave.approve (لقسمه) |
| `team_lead` | قائد فريق | إدارة فريق صغير | attendance.approve, leave.read |
| `employee` | موظف | بياناته الشخصية | employee.read (نفسه), leave.request |

### جدول Permissions مقترح

| Permission Code | Resource | Action | الوصف |
|----------------|----------|--------|------|
| `employee.create` | employee | create | إضافة موظف جديد |
| `employee.read` | employee | read | عرض بيانات موظف |
| `employee.update` | employee | update | تعديل بيانات موظف |
| `employee.delete` | employee | delete | حذف موظف |
| `employee.terminate` | employee | terminate | إنهاء خدمة موظف |
| `payroll.process` | payroll | process | معالجة المرتبات |
| `payroll.read` | payroll | read | عرض المرتبات |
| `payroll.adjust` | payroll | adjust | تعديل مرتب |
| `attendance.read` | attendance | read | عرض الحضور |
| `attendance.approve` | attendance | approve | اعتماد الحضور |
| `attendance.edit` | attendance | edit | تعديل بصمات |
| `leave.request` | leave | request | طلب أجازة |
| `leave.approve` | leave | approve | اعتماد أجازة |
| `leave.read` | leave | read | عرض الأجازات |
| `report.financial` | report | financial | تقارير مالية |
| `report.hr` | report | hr | تقارير HR |
| `settings.manage` | settings | manage | إدارة الإعدادات |
| `user.create` | user | create | إنشاء User جديد |
| `user.disable` | user | disable | تعطيل User |
| `role.assign` | role | assign | إسناد Role لـ User |

### ليه RBAC أفضل من ACL في الأنظمة الكبيرة؟

| الجانب | ACL | RBAC |
|--------|-----|------|
| تكرار Permissions | كبير جداً | معدوم |
| إضافة Permission جديد لـ Role | تعديل صف واحد في role_permissions | تعديل آلاف الـ users |
| فهم النظام | صعب | سهل |
| Onboarding موظف جديد | اقعد اربط له Permissions واحد واحد | ادي له Role وخلاص |
| Audit | معقد | بسيط |
| Performance | بطيء (Permissions كتير) | سريع |

---

## 8. Scopes & Multi-Tenancy — حل مشكلة الفروع

### المشكلة اللي بنحلها

السؤال اللي اتسأل:

> "ازاي يبقى في اتنين HR Manager نفس الرول بس واحد في فرع وواحد في فرع تاني؟"

ده سؤال **في صميم Multi-Branch Architecture** ومعظم الأنظمة الفاشلة بتفشل هنا.

### الحل: Scoped Roles

**الفكرة الأساسية**:
> الـ Role لوحدها مش كافية. لازم تبقى مربوطة بـ **Scope** (نطاق).

```
Effective Permission = Role + Scope
```

---

### 🎯 الـ Scope بيعيش فين بالظبط؟ (الفهم الأساسي)

ده أهم سؤال في كل الـ Architecture. خليني أوضحه بشكل ميسبش لبس:

> ❌ **الـ Scope مش مربوط بالـ Role**  
> ✅ **الـ Scope مربوط بالـ Assignment (يعني بربط الـ User بالـ Role)**

#### يعني إيه؟

الـ Role "HR Manager" هو **قالب مجرد** — مفيهوش أي scope. هو مجرد تعريف "إيه الـ Permissions اللي بيدخل تحت الـ Role ده".

الـ Scope بيدخل **لحظة ما أنا باسند الـ Role لـ User**. يعني وأنا بكتب الصف في `user_roles`، باقول:
> "أحمد هياخد Role HR Manager، **بس** في حدود فرع إسكندرية."

#### الـ 4 طبقات بترتيب

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: PERMISSIONS                                    │
│ ─────────────────────                                   │
│ صلاحيات مجردة. لبنات أساسية.                            │
│   • employee.read                                       │
│   • employee.create                                     │
│   • payroll.process                                     │
│                                                         │
│ ❌ مفيش Scope هنا                                       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 2: ROLES                                          │
│ ──────────────                                          │
│ تجميعات من الـ Permissions. زي "وصفة".                  │
│   • HR Manager = [employee.*, payroll.*, leave.*]      │
│   • Sales Manager = [employee.read, sales.*]            │
│                                                         │
│ ❌ مفيش Scope هنا برضو — الـ Role قالب جامد              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 3: ROLE_PERMISSIONS                               │
│ ────────────────────────                                │
│ جدول Many-to-Many. بيربط Roles بـ Permissions.          │
│ (role_id, permission_id)                                │
│                                                         │
│ ❌ مفيش Scope هنا                                       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 4: USER_ROLES ⭐ (هنا الـ Scope بيدخل!)            │
│ ───────────────────                                     │
│ بيقول: "User كذا عنده Role كذا، في Scope كذا."           │
│ (user_id, role_id, scope_type, scope_id)                │
│                                                         │
│   أحمد + HR Manager + branch:Alexandria                 │
│   محمد + HR Manager + branch:KafrElSheikh               │
│   ────────  ───────────  ────────────                   │
│    مين       نفس الرول    Scope مختلف                    │
│                                                         │
│ ✅ هنا الـ Scope بيتحدد لكل Assignment لوحده             │
└─────────────────────────────────────────────────────────┘
```

#### ليه ده مهم؟

##### ❌ لو حطينا الـ Scope على الـ Role
هنحتاج Role منفصل لكل فرع:
- `hr_manager_alexandria`
- `hr_manager_kafr_elsheikh`
- `hr_manager_cairo`
- ... × عدد الفروع

ده **Role Sprawl كارثي**. كل ما تفتح فرع جديد، تعمل 7 Roles جديدة. ومحدش بيقدر يحط بيانات على ده.

##### ✅ مع وضع الـ Scope على الـ Assignment
- Role واحد بس: `hr_manager`
- بتعمل Assignments بعدد الـ HR Managers ومواقعهم
- فتح فرع جديد = صفر تغيير في الـ Roles، بس Assignments جديدة

#### الفرق بين الفهمين بمثال

| الموقف | الفهم الغلط (Scope على Role) | الفهم الصح (Scope على Assignment) |
|--------|---------------------------|-----------------------------------|
| 3 HR Managers في 3 فروع | 3 Roles مختلفة | 1 Role + 3 صفوف في user_roles |
| لما تنقل HR Manager من فرع لفرع | تشيله من Role وتحطه في Role تاني | تعدل صف واحد في user_roles |
| إضافة Permission جديد لكل HR Managers | تحدث 3 Roles | تحدث Role واحد بس |
| إنشاء فرع جديد | تعمل Role جديد لكل وظيفة | مفيش تغيير في الـ Roles |

#### كود مثال يوضح الفرق

```sql
-- ❌ الفهم الغلط — Roles كتير
CREATE TABLE roles (id, code, branch_id);  -- خطأ!
INSERT INTO roles VALUES 
  (1, 'hr_manager', 5),   -- HR Manager - إسكندرية
  (2, 'hr_manager', 7);   -- HR Manager - كفر الشيخ
-- Role متكرر!

-- ✅ الفهم الصح — Role واحد + Assignments متعددة
INSERT INTO roles VALUES (1, 'hr_manager');  -- Role واحد بس

INSERT INTO user_roles (user_id, role_id, scope_type, scope_id) VALUES
  (101, 1, 'branch', 5),    -- أحمد ياخد Role في إسكندرية
  (102, 1, 'branch', 7);    -- محمد ياخد نفس Role في كفر الشيخ
```

---

### الـ 5 Scope Types

```
┌───────────────────────────────────────────────┐
│  COMPANY (الأعلى مستوى)                       │
│  ───────                                      │
│  Super Admin, HR Director                     │
│  بيشوف كل حاجة في الشركة                       │
│  scope_id = NULL                              │
└──────────────────┬────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────┐
│  BRANCH                                       │
│  ──────                                       │
│  HR Manager في فرع، Branch Manager            │
│  بيشوف فرع واحد بس                            │
│  scope_id = branches.id                       │
└──────────────────┬────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────┐
│  DEPARTMENT                                   │
│  ──────────                                   │
│  Department Manager                           │
│  بيشوف قسم واحد بس (داخل فرع)                 │
│  scope_id = departments.id                    │
└──────────────────┬────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────┐
│  TEAM ⭐ (الـ Cross-Branch ساحر)               │
│  ────                                         │
│  Sales Manager, Project Lead                  │
│  بيشوف Team بيخترق فروع                       │
│  scope_id = teams.id                          │
└──────────────────┬────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────┐
│  EMPLOYEE (الأدنى مستوى — Self Access)         │
│  ────────                                     │
│  الموظف العادي                                 │
│  بيشوف بياناته بس (employees.id = نفسه)        │
│  scope_id = employees.id                      │
└───────────────────────────────────────────────┘
```

> 💡 **ملاحظة على الـ Hierarchy**: ده مش tree هرمي صلب. الـ User ممكن يجمع scopes من أي مستوى. مثلاً:
> - HR Manager: scope = company (يشوف كل حاجة)
> - Sales Manager: scope = team + scope = department (مزيج)
> - Branch Manager: scope = branch (واحد بس)

### حل المشكلة بـ user_roles

دلوقتي خلينا نشوف ازاي بنحلها في الـ Database:

#### السيناريو
- **أحمد** = HR Manager في **فرع القاهرة** (branch_id = 5)
- **محمد** = HR Manager في **فرع الإسكندرية** (branch_id = 7)
- الاتنين نفس الـ Role، بس Scope مختلف

#### في الـ Database

```sql
-- أحمد
INSERT INTO user_roles (user_id, role_id, scope_type, scope_id) 
VALUES 
  (101,  -- أحمد user_id
   2,    -- hr_manager role_id
   'branch', 
   5);   -- فرع القاهرة

-- محمد
INSERT INTO user_roles (user_id, role_id, scope_type, scope_id) 
VALUES 
  (102,  -- محمد user_id
   2,    -- hr_manager role_id (نفس الـ Role!)
   'branch', 
   7);   -- فرع الإسكندرية
```

#### النتيجة
- أحمد يدخل `/api/employees` → بيشوف موظفين فرع القاهرة بس
- محمد يدخل `/api/employees` → بيشوف موظفين فرع الإسكندرية بس
- **نفس الكود، نفس الـ Role، نتايج مختلفة!**

### Visualization للموقف ده

```
                    HR Manager Role
                  (نفس الـ Permissions)
                          │
            ┌─────────────┼─────────────┐
            │                           │
            ▼                           ▼
       Scope: Branch 5            Scope: Branch 7
       (القاهرة)                    (الإسكندرية)
            │                           │
            ▼                           ▼
        أحمد                          محمد
            │                           │
            ▼                           ▼
   يشوف موظفين القاهرة            يشوف موظفين الإسكندرية
```

### Edge Case 1: HR Manager له Scope على أكتر من فرع

ايه لو HR Manager مسؤول عن فرعين (القاهرة والإسكندرية)؟

**الحل**: نضيف صفين في `user_roles`:

```sql
INSERT INTO user_roles (user_id, role_id, scope_type, scope_id) VALUES
  (101, 2, 'branch', 5),   -- القاهرة
  (101, 2, 'branch', 7);   -- الإسكندرية
```

دلوقتي أحمد يقدر يشوف الفرعين معاً.

---

### Edge Case 2: Sales Manager وفريقه متفرع على فروع 🔥

ده سيناريو واقعي ومحرج لو الـ Schema مش مرن:

> **الموقف**: عندنا Sales Manager (سلمى) قاعدة في إسكندرية. الـ Sales Team بتاعها فيه ناس شغالة في إسكندرية، وناس في كفر الشيخ، وناس في طنطا. **هتحدد الـ Scope ازاي؟**

عندنا 3 حلول، اختار اللي يناسب طبيعة الشغل:

#### 🔹 الحل 1: Multi-Branch Scope (الأبسط — لو سلمى مديرة فروع بالكامل)

لو سلمى عندها Authority على كل موظفين الفروع دي (مش بس Sales):

```sql
INSERT INTO user_roles VALUES
  (203, 5, 'branch', 5),   -- إسكندرية
  (203, 5, 'branch', 8),   -- كفر الشيخ
  (203, 5, 'branch', 12);  -- طنطا
```

**النتيجة**: سلمى تشوف كل موظفين الفروع التلاتة.

**عيب**: هتشوف **كل** الموظفين، حتى اللي مش في Sales (زي HR والمحاسبة).

#### 🔹 الحل 2: Multi-Department Scope (الأدق — لو الـ Departments per-Branch)

في الـ Schema بتاعنا، الـ `departments.branch_id` = الفرع بتاعه. يعني:
- "Sales Dept إسكندرية" = `department_id = 12`
- "Sales Dept كفر الشيخ" = `department_id = 19`
- "Sales Dept طنطا" = `department_id = 25`

```sql
INSERT INTO user_roles VALUES
  (203, 5, 'department', 12),   -- Sales إسكندرية
  (203, 5, 'department', 19),   -- Sales كفر الشيخ
  (203, 5, 'department', 25);   -- Sales طنطا
```

**النتيجة**: سلمى تشوف موظفين Sales في 3 الفروع، بدون ما تشوف باقي الأقسام.

**عيب**: لو الـ Sales اتفتح في 10 فروع، هتدخل 10 صفوف. وكل ما يفتح فرع جديد، لازم تضيف صف.

#### 🔹 الحل 3: مفهوم Team Cross-Branch (التوصية الرسمية ⭐)

ده الحل المعتمد في الـ Architecture بتاعنا. **الجداول `teams` و `employee_teams` موجودة بالفعل في الـ Schema الأساسي** (Section 6.4b و 6.4c)، و `team` معرف كـ `scope_type` first-class في `user_roles`.

نسند الـ Role بـ Team Scope كده:

```sql
-- موظفين Sales من فروع مختلفة كلهم في نفس الـ Team
INSERT INTO employee_teams (employee_id, team_id, role_in_team) VALUES
  (employee_id_alex_1,   sales_team_id, 'Member'),
  (employee_id_alex_2,   sales_team_id, 'Member'),
  (employee_id_kafr_1,   sales_team_id, 'Member'),
  (employee_id_tanta_1,  sales_team_id, 'Member');

-- سلمى مديرة الـ Team
INSERT INTO user_roles (user_id, role_id, scope_type, scope_id) VALUES
  (203, 5, 'team', sales_team_id);
```

**النتيجة**: سلمى تشوف كل أعضاء الـ Team بغض النظر عن الفرع.

**ميزات**:
- مرن جداً
- موظف يقدر يبقى في أكتر من Team
- لما يجي موظف جديد لـ Sales، تضيفه في الـ Team بصف واحد
- الـ Sales Manager مش هتحتاج تعدل في الـ user_roles خالص

#### مقارنة الحلول

| المعيار | حل 1: Multi-Branch | حل 2: Multi-Department | حل 3: Team ⭐ |
|---------|---------------------|------------------------|----------------|
| تعقيد الـ Schema | جاهز (built-in) | جاهز (built-in) | جاهز (built-in) ✅ |
| لو الـ Manager بيدير قسم بس | ❌ هيشوف الفرع كله | ✅ مظبوط | ✅ مظبوط |
| دعم Cross-Branch بفرع متفرع | ❌ لا يدعم | ⚠️ يحتاج صف لكل فرع | ✅ Native |
| لما يجي موظف جديد للـ Team | تلقائي بحسب الفرع | تلقائي بحسب القسم | لازم تضيفه في `employee_teams` |
| لما يفتح فرع جديد بنفس القسم | تلقائي | لازم تضيف صف في user_roles | تلقائي (الـ Team موجود) |
| الـ Performance | ممتاز | ممتاز | ممتاز |
| الـ Reusability | منخفض | متوسط | **عالي** |

#### إيه التوصية؟

> 💡 **استخدم الحل اللي يناسب طبيعة الـ Reporting Line:**
> - لو المدير بيدير **فرع كامل** → حل 1 (Branch Scope)
> - لو المدير بيدير **قسم في فرع** → حل 2 (Department Scope)
> - لو المدير بيدير **فريق بيخترق فروع** → حل 3 (Team Scope) ⭐

كل الـ 3 حلول **مدعومة Native** في الـ Architecture. اختار حسب الموقف، ومش لازم تختار حل واحد بس — ممكن نفس النظام يجمع بينهم في حالات مختلفة.

#### Visualization للحل 3

```
                  ╔═══════════════════════════╗
                  ║  Egypt Sales Team         ║
                  ║  (team_id = 1)            ║
                  ╚═══════════╤═══════════════╝
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   ┌─────────┐           ┌─────────┐           ┌─────────┐
   │ موظف 1   │           │ موظف 2   │           │ موظف 3   │
   │ branch=  │           │ branch=  │           │ branch=  │
   │إسكندرية  │           │كفر الشيخ │           │ طنطا     │
   └─────────┘           └─────────┘           └─────────┘
        ▲                                            ▲
        │                                            │
        └────────────────────┬───────────────────────┘
                             │
                  ╔══════════▼═══════════╗
                  ║  سلمى (Sales Manager) ║
                  ║  scope: team:1        ║
                  ║  تشوفهم كلهم          ║
                  ╚══════════════════════╝
```

---

### Code Pattern للـ Authorization Check

```typescript
// src/lib/auth/authorize.ts
async function canAccessEmployee(
  userId: number,
  employeeId: number,
  permission: string  // 'employee.read'
): Promise<boolean> {
  // 1. جيب الـ user roles مع الـ scopes
  const userRoles = await db.query(`
    SELECT ur.scope_type, ur.scope_id, r.code as role_code
    FROM user_roles ur
    JOIN roles r ON r.id = ur.role_id
    JOIN role_permissions rp ON rp.role_id = r.id
    JOIN permissions p ON p.id = rp.permission_id
    WHERE ur.user_id = $1
      AND p.code = $2
      AND (ur.expires_at IS NULL OR ur.expires_at > NOW())
      AND ur.revoked_at IS NULL
  `, [userId, permission]);

  if (userRoles.length === 0) return false;

  // 2. جيب بيانات الموظف
  const employee = await db.query(`
    SELECT branch_id, department_id, user_id 
    FROM employees WHERE id = $1
  `, [employeeId]);

  // 3. شيك الـ Scope
  for (const role of userRoles) {
    switch (role.scope_type) {
      case 'company':
        return true;  // الشركة كلها — يشوف الكل
      
      case 'branch':
        if (employee.branch_id === role.scope_id) return true;
        break;
      
      case 'department':
        if (employee.department_id === role.scope_id) return true;
        break;
      
      case 'team':
        // شيك إن الموظف ده عضو في الـ Team
        const isMember = await db.query(`
          SELECT 1 FROM employee_teams 
          WHERE employee_id = $1 AND team_id = $2
        `, [employeeId, role.scope_id]);
        if (isMember.rows.length > 0) return true;
        break;
      
      case 'employee':
        // عادةً self-access: scope_id = الـ employee.id بتاع الـ User نفسه
        if (employee.id === role.scope_id) return true;
        break;
    }
  }

  return false;
}
```

### Query Filtering by Scope (الـ Pattern الأهم)

بدل ما تجيب كل الـ Employees وتفلتر، الـ **Pattern الصحيح** إنك تفلتر في الـ SQL Query نفسه:

```typescript
async function getEmployees(userId: number) {
  return db.query(`
    SELECT DISTINCT e.*
    FROM employees e
    WHERE EXISTS (
      SELECT 1
      FROM user_roles ur
      JOIN role_permissions rp ON rp.role_id = ur.role_id
      JOIN permissions p ON p.id = rp.permission_id
      WHERE ur.user_id = $1
        AND p.code = 'employee.read'
        AND (
             ur.scope_type = 'company'
          OR (ur.scope_type = 'branch'     AND ur.scope_id = e.branch_id)
          OR (ur.scope_type = 'department' AND ur.scope_id = e.department_id)
          OR (ur.scope_type = 'team'       AND ur.scope_id IN (
                SELECT team_id FROM employee_teams WHERE employee_id = e.id
             ))
          OR (ur.scope_type = 'employee'   AND ur.scope_id = e.id)
        )
        AND (ur.expires_at IS NULL OR ur.expires_at > NOW())
        AND ur.revoked_at IS NULL
    )
  `, [userId]);
}
```

> 🎯 **القاعدة الذهبية**: **Filter at the Database Level**. متجبش بيانات وتفلترها في الـ Code، ده أبطأ وأخطر.

---

## 9. Branch Access & Department Access

دلوقتي نشوف السيناريوهات العملية:

### 9.1 مدير القسم يشوف موظفين قسمه فقط

#### الـ Setup
```sql
INSERT INTO user_roles VALUES (..., 'dept_manager_role_id', 'department', 12);
-- 12 = قسم الـ IT في فرع القاهرة
```

#### الـ Query
```sql
SELECT * FROM employees 
WHERE department_id = 12;
```

النتيجة: يشوف موظفين قسم IT في فرع القاهرة بس. لا يشوف موظفين قسم IT في فرع الإسكندرية.

### 9.2 مدير الفرع يشوف موظفين فرعه

#### الـ Setup
```sql
INSERT INTO user_roles VALUES (..., 'branch_manager_role_id', 'branch', 5);
-- 5 = فرع القاهرة
```

#### الـ Query
```sql
SELECT * FROM employees 
WHERE branch_id = 5;
```

### 9.3 HR Manager في فرع معين

نفس فكرة Branch Manager، بس بصلاحيات أكتر:

```sql
-- نفس Scope بس Role مختلف
INSERT INTO user_roles VALUES (..., 'hr_manager_role_id', 'branch', 5);
```

الفرق إن HR Manager عنده `payroll.process` و `employee.create`، بس **في حدود الفرع**.

### 9.4 HR على مستوى الشركة (Company Scope)

```sql
INSERT INTO user_roles VALUES (..., 'hr_manager_role_id', 'company', NULL);
```

ده الـ HR Director اللي بيشوف كل الفروع.

### 9.5 Sales Manager بفريق متفرع على فروع (Team Scope) ⭐

```sql
-- خطوة 1: اعمل الـ Team
INSERT INTO teams (name, code) VALUES ('Egypt Sales Team', 'TEAM-SALES-EG');

-- خطوة 2: ضم الموظفين من الفروع المختلفة
INSERT INTO employee_teams (employee_id, team_id, role_in_team) VALUES
  (101, 1, 'Member'),   -- موظف في إسكندرية
  (102, 1, 'Member'),   -- موظف في كفر الشيخ
  (103, 1, 'Lead');     -- موظف في طنطا

-- خطوة 3: اسند Sales Manager Role بـ Team Scope
INSERT INTO user_roles VALUES (..., 'sales_manager_role_id', 'team', 1);
```

#### الـ Query
```sql
SELECT e.* FROM employees e
WHERE e.id IN (
  SELECT employee_id FROM employee_teams WHERE team_id = 1
);
```

### 9.6 الموظف العادي يشوف بياناته بس (Employee Scope = Self)

```sql
-- scope_id = الـ employee.id بتاع الـ User نفسه
INSERT INTO user_roles VALUES (..., 'employee_role_id', 'employee', $own_employee_id);
```

#### الـ Query
```sql
SELECT * FROM employees e
WHERE e.id = $own_employee_id;
```

> 💡 **ملاحظة مهمة**: كل موظف عنده Login بشكل افتراضي عنده الـ `employee` role بـ `scope_type = 'employee'` و `scope_id = employees.id` بتاعه. ده النواة الأساسية، وبيتعمل تلقائياً وقت إنشاء الـ User.

### Diagram يلخص الكل

```
                     النظام بالكامل
                    ╔════════════════╗
                    ║  Super Admin   ║ (scope: company)
                    ║  HR Director   ║
                    ╚════════╤═══════╝
                             │
            ┌────────────────┼────────────────┐
            │                │                │
        فرع القاهرة      فرع الإسكندرية    فرع الجيزة
        ╔══════════╗     ╔══════════╗     ╔══════════╗
        ║HR Manager║     ║HR Manager║     ║HR Manager║
        ║Branch Mgr║     ║Branch Mgr║     ║Branch Mgr║
        ╚════╤═════╝     ╚══════════╝     ╚══════════╝
             │
   ┌─────────┼─────────┐
   │         │         │
 قسم IT    قسم HR    قسم المالية
 ╔═══════╗ ╔═══════╗ ╔═══════╗
 ║Dept Mgr║ ║Dept Mgr║ ║Dept Mgr║
 ╚═══╤════╝ ╚═══════╝ ╚═══════╝
     │
     ▼
  موظفين (each sees only their own data)
```

---

## 10. Authorization Lifecycle — خطوة بخطوة

كل Request في النظام بيمر بـ Pipeline. خليني أوريك كل خطوة بالتفصيل.

### الـ Pipeline الكامل

```
┌─────────────────────────────────────────────────────────────┐
│  1. INCOMING REQUEST                                        │
│     GET /api/employees                                      │
│     Cookie: session_token=abc123...                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  2. PARSE SESSION TOKEN                                     │
│     من الـ HTTP-Only Cookie                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  3. AUTHENTICATE                                            │
│     SELECT user FROM sessions WHERE token_hash = ...        │
│       AND expires_at > NOW()                                │
│                                                             │
│     ❌ لو مفيش session أو expired → 401                     │
└──────────────────────┬──────────────────────────────────────┘
                       │ ✅
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  4. CHECK USER STATUS                                       │
│     SELECT status FROM users WHERE id = ?                   │
│                                                             │
│     ❌ لو status != 'active' → 401                          │
└──────────────────────┬──────────────────────────────────────┘
                       │ ✅
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  5. LOAD USER CONTEXT                                       │
│     - Roles (مع scopes)                                     │
│     - Permissions (من الـ roles)                            │
│     - Employee info (branch, department)                    │
│                                                             │
│     بيتخزن في memory للـ Request دي بس                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  6. AUTHORIZE — CHECK PERMISSION                            │
│     هل الـ User عنده permission 'employee.read'؟            │
│                                                             │
│     ❌ لو لأ → 403 Forbidden                                │
└──────────────────────┬──────────────────────────────────────┘
                       │ ✅
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  7. APPLY SCOPE FILTERS                                     │
│     ضيف WHERE clauses حسب الـ scope                         │
│                                                             │
│     لو scope=branch=5 →                                     │
│       WHERE employees.branch_id = 5                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  8. EXECUTE BUSINESS LOGIC                                  │
│     شغل الـ Query وارجع النتايج                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  9. AUDIT LOG                                               │
│     INSERT INTO audit_logs (...)                            │
│     مين عمل ايه على ايه إمتى                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  10. RETURN RESPONSE                                        │
│      200 OK + JSON Data                                     │
└─────────────────────────────────────────────────────────────┘
```

### Implementation في Next.js (Middleware Pattern)

```typescript
// middleware.ts (Next.js Middleware)
import { NextResponse } from 'next/server';

export async function middleware(req) {
  // 1. Get session token
  const token = req.cookies.get('session_token')?.value;
  if (!token) return NextResponse.redirect('/login');
  
  // 2-4. Authenticate
  const session = await getSession(token);
  if (!session || session.user.status !== 'active') {
    return NextResponse.redirect('/login');
  }
  
  // 5. Load user context
  const userContext = await loadUserContext(session.user.id);
  
  // 6+7. Authorization و Scope بيتعمل في كل API Handler
  // مش في الـ Middleware
  
  // ضيف الـ context في الـ headers عشان API Routes تستخدمه
  const requestHeaders = new Headers(req.headers);
  requestHeaders.set('x-user-id', String(session.user.id));
  requestHeaders.set('x-user-roles', JSON.stringify(userContext.roles));
  
  return NextResponse.next({ request: { headers: requestHeaders } });
}

export const config = {
  matcher: ['/((?!api/auth|_next/static|favicon.ico|login).*)'],
};
```

### في الـ API Handler

```typescript
// app/api/employees/route.ts
import { authorize } from '@/lib/auth';

export async function GET(req: Request) {
  // 6. Authorize
  const auth = await authorize(req, 'employee.read');
  if (!auth.allowed) {
    return Response.json({ error: 'Forbidden' }, { status: 403 });
  }
  
  // 7. Apply scope - getScopeFilter بيرجع SQL fragment
  const scopeFilter = auth.getScopeFilter('employees');
  
  // 8. Execute
  const employees = await db.query(`
    SELECT * FROM employees WHERE ${scopeFilter}
  `, [auth.userId]);
  
  // 9. Audit
  await auditLog({
    userId: auth.userId,
    action: 'employee.read',
    resource_type: 'employee',
    metadata: { count: employees.length }
  });
  
  return Response.json({ data: employees });
}
```

---

## 11. Authentication Flow التفصيلي

### 11.1 Registration Flow — Two-Step Pattern ⭐

> ⚠️ **القاعدة الذهبية**: 
> 1. **اعمل Employee أولاً (دايماً)**.
> 2. **بعدين** اسأل: "هل الموظف ده محتاج Login؟"

ده Pattern مهم جداً لأنه:
- بيخليك تضيف عمال نظافة وأمن (مش محتاجين Login) بنفس الـ Flow
- بيمنع إنك تربط Employee بـ User بطريق الخطأ
- بيخلي الـ HR Data منفصلة عن الـ Identity من أول لحظة

```
┌──────────────────────────────────────────────────────────────┐
│  STEP 1: إنشاء Employee (إجباري لكل شخص)                     │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  HR Specialist بيدخل صفحة "إضافة موظف"                       │
│  بيدخل بيانات HR:                                            │
│  ─────────────────                                           │
│  • الاسم الكامل                                              │
│  • الرقم القومي                                              │
│  • رقم الموبايل الشخصي (مش بالضرورة Login)                  │
│  • الفرع والقسم                                              │
│  • المنصب وتاريخ التعيين                                     │
│  • نوع التوظيف                                                │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│  النظام:                                                      │
│  INSERT INTO employees (full_name, national_id, ...)         │
│  user_id = NULL  ← الموظف موجود بدون Login (لسه)              │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
              ╔════════════════╧════════════════╗
              ║  السؤال الحاسم:                  ║
              ║  هل الموظف ده محتاج Login؟       ║
              ╚════════════════╤════════════════╝
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼ لأ                              ▼ آه
   ┌──────────────────────┐         ┌──────────────────────────────┐
   │  STEP 2A: خلاص        │         │  STEP 2B: إنشاء User          │
   │  ──────────          │         │  ────────────────             │
   │  الموظف بيشتغل بس     │         │  ➀ ولد Password عشوائي         │
   │  HR record           │         │  ➁ bcrypt hash للـ password    │
   │  (زي عمال النظافة)   │         │  ➂ INSERT INTO users (...)     │
   │                      │         │     status='pending_first_login'│
   │  ✅ Done             │         │     must_change_password=TRUE  │
   └──────────────────────┘         │  ➃ UPDATE employees             │
                                    │     SET user_id = $newUserId   │
                                    │  ➄ INSERT INTO user_roles      │
                                    │     (Role + Scope)             │
                                    │  ➅ ابعت SMS بالـ Initial Pass   │
                                    └──────────────────────────────┘
                                                     │
                                                     ▼
                                    ┌──────────────────────────────┐
                                    │  SMS Content:                │
                                    │  "أهلاً يا أحمد،              │
                                    │   تم إنشاء حسابك.            │
                                    │   موبايل: 01012345678        │
                                    │   باسورد مؤقت: Pa$$-2025-XyZ9│
                                    │   ⚠️ هتغيرها أول ما تدخل."   │
                                    └──────────────────────────────┘
```

#### Pseudo-code Implementation

```typescript
// Step 1: Always create Employee
async function createEmployee(data: EmployeeData) {
  const employee = await db.employees.create({
    full_name: data.fullName,
    national_id: data.nationalId,
    employee_code: data.employeeCode,
    branch_id: data.branchId,
    department_id: data.departmentId,
    hire_date: data.hireDate,
    user_id: null,  // ⚠️ مفيش Login لسه
  });
  
  return employee;
}

// Step 2: Optional - create User if needed
async function enableLoginForEmployee(
  employeeId: number, 
  phone: string,
  roleAssignments: RoleAssignment[]
) {
  return await db.transaction(async (tx) => {
    // 1. ولد + Hash الـ Password
    const initialPassword = generateSecurePassword();
    const passwordHash = await bcrypt.hash(initialPassword, 12);
    
    // 2. اعمل User
    const user = await tx.users.create({
      phone,
      password_hash: passwordHash,
      status: 'pending_first_login',
      must_change_password: true,
    });
    
    // 3. اربط الـ User بالـ Employee
    await tx.employees.update(employeeId, {
      user_id: user.id,
    });
    
    // 4. اسند Roles مع Scopes
    for (const assignment of roleAssignments) {
      await tx.user_roles.create({
        user_id: user.id,
        role_id: assignment.roleId,
        scope_type: assignment.scopeType,  // company | branch | department | team | employee
        scope_id: assignment.scopeId,
      });
    }
    
    // 5. ابعت SMS (خارج الـ transaction)
    return { user, initialPassword };  // ⚠️ ابعت ومتخزنش
  });
}
```

#### حالات الـ Use الواقعية

| الحالة | Step 1 (Employee) | Step 2 (User) |
|--------|-------------------|---------------|
| موظف Software Engineer جديد | ✅ | ✅ |
| عامل نظافة | ✅ | ❌ |
| Driver | ✅ | ❌ |
| Security Guard | ✅ | ❌ |
| HR Manager | ✅ | ✅ |
| Field Worker (مع Mobile App) | ✅ | ✅ |
| موظف لسه في Probation | ✅ | ❌ (لما يثبت) |
| موظف اتفصل | ✅ (status=terminated) | ❌ (user.status=disabled) |

### 11.2 إجابة على سؤال "هل أقدر أشوف الباسوردات؟"

> ❌ **لأ، ومينفعش يحصل!**

السبب: لو هتقدر تشوفهم، يبقى هم متخزنين plaintext أو reversible encryption، وده **anti-pattern أمني خطير**.

#### الحل الصحيح

| المتطلب | الحل |
|---------|------|
| "عايز أبعت بيانات الدخول للموظف" | بعت **Initial Password** وقت الإنشاء فقط، **مرة واحدة**. |
| "الموظف نسي باسورده" | يستخدم **Forgot Password Flow** (OTP على موبايله). |
| "عايز أقفل حساب موظف اتفصل" | تغيير `users.status = 'disabled'`. |
| "عايز أعرف الباسورد بتاع موظف معين" | **مينفعش**. لو محتاج، اعمل له Password Reset. |

#### Pseudo-code للـ Password Generation (الجزء اللي بيهمنا هنا)

```typescript
// ⚠️ ملاحظة: الـ Flow الكامل لإنشاء Employee + User موجود في Section 11.1
// (Two-Step Pattern). الكود هنا بيشرح الـ Password Handling تحديداً.

async function generateAndSendInitialPassword(
  employeeId: number,
  fullName: string,
  phone: string
): Promise<string> {
  // 1. ولد Password عشوائي قوي
  const initialPassword = generateSecurePassword({
    length: 12,
    includeUppercase: true,
    includeLowercase: true,
    includeNumbers: true,
    includeSymbols: true,
  });
  
  // 2. Hash بـ bcrypt (cost = 12)
  const passwordHash = await bcrypt.hash(initialPassword, 12);
  
  // 3. حدّث الـ user record بالـ Hash
  await db.users.update(/* user_id */, {
    password_hash: passwordHash,
    status: 'pending_first_login',
    must_change_password: true,
  });
  
  // 4. ابعت الـ Plaintext password عبر SMS — مرة واحدة بس
  await sendSMS({
    to: phone,
    message: `أهلاً ${fullName}، باسوردك المؤقت: ${initialPassword}`,
  });
  
  // 5. ⚠️ مهم جداً: متخزنش الـ initialPassword في أي حتة بعد كده
  //    حتى في الـ logs! بنخزن hash بس.
  
  return /* nothing — تنسى الـ initialPassword فوراً */;
}
```

### 11.3 First Login Flow

```
1. الموظف بيدخل بـ Phone + Initial Password
2. النظام بيتأكد:
   - users.status = 'pending_first_login' ✅
   - bcrypt.compare يطلع true ✅
3. النظام بيلاقي must_change_password = TRUE
4. يفتح Dialog "اختار باسورد جديد"
5. الموظف بيدخل الباسورد الجديد
6. النظام:
   - Validate (طول، تعقيد، مش password شائع)
   - Hash الجديد
   - UPDATE users SET 
       password_hash = ?,
       status = 'active',
       must_change_password = FALSE,
       password_changed_at = NOW()
7. ينشئ Session ويوديه للـ Dashboard
```

### 11.4 Normal Login Flow

```
1. الموظف بيدخل /login
2. بيدخل Phone + Password
3. النظام بيشيك:
   - users.phone موجود؟ ✅
   - users.status = 'active'? ✅
   - users.locked_until > NOW()? ❌ لو oo، 429 Too Many Attempts
   - bcrypt.compare(password, password_hash) ✅
4. لو حاجة فشلت:
   - UPDATE users SET failed_login_attempts = failed_login_attempts + 1
   - لو failed_login_attempts >= 5:
       SET locked_until = NOW() + INTERVAL '15 minutes'
   - INSERT INTO audit_logs (action='login.failed')
5. لو نجح:
   - INSERT INTO sessions (...)
   - UPDATE users SET 
       last_login_at = NOW(),
       last_login_ip = ?,
       failed_login_attempts = 0
   - SET Cookie 'session_token' (HTTP-Only, Secure, SameSite=Strict)
   - INSERT INTO audit_logs (action='login.success')
   - Redirect to /dashboard
```

### 11.5 Password Reset Flow (عبر OTP)

```
┌────────────────────────────────────────────┐
│ الموظف بيدوس "نسيت الباسورد"                │
└──────────────────┬─────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────┐
│ يدخل رقم الموبايل                          │
└──────────────────┬─────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────┐
│ النظام:                                    │
│ 1. يشيك إن الـ phone موجود                │
│ 2. يولد OTP 6 أرقام                        │
│ 3. INSERT INTO password_reset_otps         │
│    (otp_hash, expires_at = NOW() + 5min)   │
│ 4. ابعت SMS فيه الـ OTP                   │
│                                            │
│ ⚠️ مهم: نفس Response لو الرقم موجود ولا   │
│   لأ، عشان متكشفش الأرقام الصحيحة.        │
└──────────────────┬─────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────┐
│ الموظف بيدخل الـ OTP + الباسورد الجديد    │
└──────────────────┬─────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────┐
│ النظام يتحقق:                             │
│ - OTP صحيح (bcrypt.compare)               │
│ - مش expired                              │
│ - مش used_at لسه                          │
│ - attempts <= 5                            │
│                                            │
│ لو نجح:                                    │
│ - Hash الباسورد الجديد                    │
│ - UPDATE users                             │
│ - UPDATE password_reset_otps               │
│   SET used_at = NOW()                      │
│ - ❌ يلغي كل الـ sessions الحالية         │
│   DELETE FROM sessions WHERE user_id = ?  │
│   (Force Logout من كل الأجهزة)           │
└────────────────────────────────────────────┘
```

### 11.6 Logout Flow

```
1. الموظف بيدوس Logout
2. النظام:
   - يجيب الـ session_token من الـ cookie
   - DELETE FROM sessions WHERE session_token_hash = ?
   - يمسح الـ cookie
   - INSERT INTO audit_logs (action='logout')
3. Redirect to /login
```

---

## 12. Session Management في Next.js

### JWT vs Session Cookies — أيهم أنسب لـ Next.js؟

| المعيار | JWT (Stateless) | Session Cookies (Stateful) |
|---------|-----------------|---------------------------|
| التخزين | Client side | Server side (DB) |
| Force Logout | ❌ صعب جداً (لازم blacklist) | ✅ سهل (DELETE من sessions) |
| Performance | ✅ مفيش DB query | ❌ DB query على كل request |
| Size | كبير (1-2KB) | صغير (Cookie فيه ID بس) |
| Security | لو اتسرب → مصيبة لحد ما يـ expire | لو اتسرب → نقدر نلغيه فوراً |
| HR System | ❌ مش مناسب | ✅ **التوصية** |

> 🎯 **التوصية لـ HR System**: **Session-based مع HTTP-only Cookies**.

#### السبب
- في HR System، الـ "Force Logout" مطلب أساسي لما موظف يتفصل أو الباسورد يتغير.
- مع JWT لازم تعمل Blacklist في Redis، وده بيكسر فكرة الـ Stateless.
- الـ Performance overhead للـ DB query بسيط مع Caching.

### Implementation

#### Session Creation

```typescript
// app/api/auth/login/route.ts
import { cookies } from 'next/headers';
import { randomBytes, createHash } from 'crypto';

export async function POST(req: Request) {
  const { phone, password } = await req.json();
  
  // ... validation, password check
  
  // ولد Session Token
  const sessionToken = randomBytes(32).toString('hex');
  const sessionTokenHash = createHash('sha256').update(sessionToken).digest('hex');
  
  // خزن في DB
  await db.sessions.create({
    user_id: user.id,
    session_token_hash: sessionTokenHash,
    ip_address: req.headers.get('x-forwarded-for'),
    user_agent: req.headers.get('user-agent'),
    expires_at: new Date(Date.now() + 1000 * 60 * 60 * 24 * 7), // 7 days
  });
  
  // عيّن الـ Cookie
  cookies().set('session_token', sessionToken, {
    httpOnly: true,        // مش هيقدر يوصله JavaScript
    secure: true,          // HTTPS only
    sameSite: 'strict',    // CSRF protection
    maxAge: 60 * 60 * 24 * 7,  // 7 days
    path: '/',
  });
  
  return Response.json({ success: true });
}
```

#### Session Validation

```typescript
// lib/auth/getSession.ts
import { cookies } from 'next/headers';
import { createHash } from 'crypto';

export async function getSession() {
  const token = cookies().get('session_token')?.value;
  if (!token) return null;
  
  const tokenHash = createHash('sha256').update(token).digest('hex');
  
  const session = await db.query(`
    SELECT s.*, u.id as user_id, u.phone, u.status
    FROM sessions s
    JOIN users u ON u.id = s.user_id
    WHERE s.session_token_hash = $1
      AND s.expires_at > NOW()
      AND s.revoked_at IS NULL
      AND u.status = 'active'
  `, [tokenHash]);
  
  if (!session) return null;
  
  // Update last_activity (Sliding Session)
  await db.query(`
    UPDATE sessions SET last_activity_at = NOW()
    WHERE id = $1
  `, [session.id]);
  
  return session;
}
```

### Refresh Token Strategy

عشان Session تطول أكتر من 7 أيام بدون ما تخسر الأمان، نستخدم Refresh Tokens:

```
Access Cookie (Session Token)  →  مدته قصيرة (1 ساعة)
Refresh Cookie                 →  مدته طويلة (30 يوم)

كل ساعة:
  لو الـ Session Token expired:
    شيك على الـ Refresh Token
    لو valid → ولد Session Token جديد + Refresh جديد
    لو مش valid → redirect to /login
```

### Multi-Device Sessions

الـ User يقدر يكون عنده Sessions كتير في نفس الوقت (موبايل، لاب توب، تابلت):

```sql
SELECT * FROM sessions 
WHERE user_id = ? 
  AND expires_at > NOW()
  AND revoked_at IS NULL;
```

UI ممكن يعرضله "الأجهزة النشطة" ويسمحله يعمل Logout من جهاز معين:

```typescript
async function logoutFromDevice(sessionId: number) {
  await db.sessions.update(sessionId, {
    revoked_at: new Date(),
  });
}
```

### Force Logout (لو الموظف اتفصل)

```typescript
async function terminateEmployee(employeeId: number) {
  const employee = await db.employees.findById(employeeId);
  
  // 1. Update status
  await db.employees.update(employeeId, {
    status: 'terminated',
    termination_date: new Date(),
  });
  
  // 2. Disable user account
  if (employee.user_id) {
    await db.users.update(employee.user_id, {
      status: 'disabled',
    });
    
    // 3. Force Logout من كل الأجهزة
    await db.sessions.deleteWhere({
      user_id: employee.user_id,
    });
    
    // 4. Revoke كل الـ roles
    await db.user_roles.deleteWhere({
      user_id: employee.user_id,
    });
  }
  
  // 5. Audit log
  await auditLog({
    action: 'employee.terminated',
    resource_type: 'employee',
    resource_id: employeeId,
  });
}
```

---

## 13. Audit Logs & Compliance

### ليه الـ Audit Log مش اختياري في HR System؟

في HR System، الـ Audit Log هو **خط دفاعك الأخير**. لو حصلت مشكلة قانونية أو تسريب بيانات، الـ logs هي اللي بتقولك:
- مين عمل ايه
- إمتى
- من فين (IP)
- على إيه (Resource)

### إيه اللي بنسجله؟

#### Authentication Events
- `login.success` — دخول ناجح
- `login.failed` — محاولة دخول فاشلة
- `login.locked` — الحساب اتقفل بعد محاولات
- `logout` — خروج
- `password.changed` — تغيير الباسورد
- `password.reset.requested` — طلب reset
- `password.reset.completed` — reset اكتمل

#### Authorization Events
- `access.denied` — محاولة دخول مكان مش مسموح
- `role.assigned` — Role اتسند
- `role.revoked` — Role اتشال

#### Data Events
- `employee.created`
- `employee.updated` (مع old و new values)
- `employee.deleted`
- `payroll.processed`
- `salary.adjusted` (حساس جداً!)

### مثال على Log Entry

```json
{
  "id": 123456,
  "user_id": 101,
  "action": "employee.updated",
  "resource_type": "employee",
  "resource_id": 5023,
  "metadata": {
    "fields_changed": ["job_title", "department_id"],
    "old": {
      "job_title": "Software Engineer",
      "department_id": 12
    },
    "new": {
      "job_title": "Senior Software Engineer",
      "department_id": 12
    }
  },
  "ip_address": "197.123.45.67",
  "user_agent": "Mozilla/5.0 ...",
  "status": "success",
  "created_at": "2026-05-25T14:32:11Z"
}
```

### Implementation Helper

```typescript
// lib/audit.ts
type AuditLogEntry = {
  userId?: number;
  action: string;
  resourceType?: string;
  resourceId?: number;
  metadata?: Record<string, any>;
  status?: 'success' | 'failed' | 'denied';
  errorMessage?: string;
};

export async function auditLog(entry: AuditLogEntry, req?: Request) {
  await db.audit_logs.create({
    user_id: entry.userId,
    action: entry.action,
    resource_type: entry.resourceType,
    resource_id: entry.resourceId,
    metadata: entry.metadata,
    ip_address: req?.headers.get('x-forwarded-for'),
    user_agent: req?.headers.get('user-agent'),
    status: entry.status || 'success',
    error_message: entry.errorMessage,
  });
}
```

### Best Practices للـ Audit Logs

1. **Append-only**: متعدلش ولا تحذف من الـ logs
2. **Partitioning**: قسم الـ Table بالشهر لو الحجم كبير
3. **Encryption at Rest**: الـ DB Encryption عشان لو الباك-اب اتسرق
4. **Retention Policy**: احفظ الـ logs لمدة قانونية (3-7 سنين حسب البلد)
5. **Performance**: استخدم Async Insertion لو الـ Logs ثقيلة
6. **Don't log sensitive data**: متخزنش Passwords أو Tokens في الـ logs، حتى في الـ metadata

---

## 14. ⭐ أفضل Architecture مقترحة للمشروع ده

ده الـ Section الأهم. هلخصلك القرارات النهائية في شكل مرجع واضح.

### 14.1 Final Recommended Design

```
╔══════════════════════════════════════════════════════════════╗
║              HR AUTHENTICATION & AUTHORIZATION                ║
║                    Final Architecture                         ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  HR Layer (Foundation)                                       ║
║  ──────────────────                                          ║
║  • employees (everyone) — دايماً بيتعمل الأول                ║
║  • branches, departments — التنظيم الجغرافي/الإداري           ║
║  • teams + employee_teams — للـ Cross-Branch grouping        ║
║                                                              ║
║  Identity Layer (Optional)                                   ║
║  ─────────────────────                                       ║
║  • users (separate من employees) — اختياري لكل Employee      ║
║  • Login بـ Phone + bcrypt password                          ║
║  • Initial password generation + Force change on first login ║
║  • Account states: pending, active, disabled, locked         ║
║  • Two-Step Creation: Employee أولاً → User لو محتاج          ║
║                                                              ║
║  Authorization Layer                                         ║
║  ───────────────────                                         ║
║  • RBAC: Roles + Permissions (Many-to-Many)                  ║
║  • Permissions بصيغة resource.action                         ║
║  • 5 Scope Types: company, branch, department, team, employee║
║  • Scope على الـ Assignment مش على الـ Role                   ║
║  • Multi-role + Multi-scope per user                         ║
║  • Authorization = Permission Check + Scope Check            ║
║                                                              ║
║  Session Layer                                               ║
║  ─────────────                                               ║
║  • HTTP-only Cookies                                         ║
║  • Server-side Sessions في PostgreSQL                        ║
║  • Sliding expiration (7 days)                               ║
║  • Multi-device support                                      ║
║  • Force logout on termination                               ║
║                                                              ║
║  Protection Layer                                            ║
║  ────────────────                                            ║
║  • Brute force lockout (5 attempts → 15 min)                 ║
║  • OTP-based password reset                                  ║
║  • Comprehensive audit logging                               ║
║  • DB-level scope filtering (مش in-memory)                   ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### 14.2 ليه هو الأفضل (5 نقاط)

#### 1️⃣ **Separation of Concerns واضح**
الـ Identity منفصلة عن الـ HR Data. ده بيخليك تقدر:
- تضيف موظفين بدون Login (عمال، أمن)
- تضيف Users مش موظفين (Admins External)
- تغيير في الـ Identity مش بيأثر على الـ HR Data والعكس

#### 2️⃣ **يحل Multi-Branch من اليوم الأول**
بـ `scope_type` و `scope_id`، النظام ميقدرش يخلط بيانات الفروع. **Data Isolation by Design.**

#### 3️⃣ **Scalable في 3 اتجاهات**
- **Horizontal**: ضيف فروع جديدة بدون أي تعديل في الكود
- **Vertical**: ضيف Roles جديدة بدون تعديل في الـ schema
- **Functional**: ضيف Permissions جديدة بدون تأثير على القديم

#### 4️⃣ **Security بالـ Best Practices**
- bcrypt مش MD5
- Server-side sessions مش JWT للـ HR
- DB-level filtering مش In-memory
- Audit logs على كل حاجة

#### 5️⃣ **Future-proof**
لو يوم احتجت:
- SSO مع Google Workspace → ضيف فيلد `google_id` في users
- 2FA → ضيف جدول `user_2fa_secrets`
- API Access للأنظمة الخارجية → ضيف Role + Scope جديد

### 14.3 المشاكل اللي بيتجنبها (Common Pitfalls)

#### ❌ Pitfall 1: Data Leakage بين الفروع

**المشكلة**: مدير فرع القاهرة بيشوف موظفين فرع الإسكندرية بطريق الخطأ.

**الحل في النظام**: Scope filtering على مستوى الـ Query نفسه.

```sql
-- ❌ خطأ: جيب الكل وفلتر في الـ Code
SELECT * FROM employees;  -- بعدين JavaScript filter

-- ✅ صح: فلتر في الـ DB
SELECT * FROM employees WHERE branch_id IN (المسموح بيها);
```

#### ❌ Pitfall 2: Hardcoded Roles

**المشكلة**: الكود فيه `if (user.role === 'manager')` في 50 مكان.

**الحل**: Check الـ permission مش الـ role.

```typescript
// ❌ خطأ
if (user.role === 'hr_manager') { ... }

// ✅ صح
if (await hasPermission(user.id, 'employee.update')) { ... }
```

#### ❌ Pitfall 3: Permission Sprawl

**المشكلة**: مع الوقت بتلاقي الـ Permissions كتير جداً (300+ permission) ومحدش فاهم حاجة.

**الحل**: استخدم Naming Convention صارم وعمل Cleanup دوري.

```
employee.create
employee.read
employee.update
employee.delete
employee.terminate  ← action متخصص بدل update.terminate
```

#### ❌ Pitfall 4: تخزين Plaintext Passwords

**المشكلة**: "بنخزن الباسوردات عشان نبعتها للموظفين."

**الحل**: ابعت الـ Initial Password وقت الإنشاء بس، وخزن hash بس.

#### ❌ Pitfall 5: Missing Audit Trail

**المشكلة**: حصل تعديل على مرتب موظف، ومحدش عارف مين عمله.

**الحل**: Audit logs على كل عملية حساسة، مش الـ login بس.

#### ❌ Pitfall 6: عدم وجود Scope في الـ Queries

**المشكلة**: Developer كتب `SELECT * FROM employees` وفكر إن الـ middleware هيفلتر.

**الحل**: 
1. كل Query على Resource حساس لازم يمر بـ helper function اسمه `getScopeFilter()`
2. عمل ESLint Rule بتمنع الـ raw queries على الجداول دي
3. استخدم Repository Pattern يجبر الفلترة

### 14.4 الأخطاء الشائعة في HR Systems وحلولها

| الخطأ | الحل |
|------|------|
| Role واحد لكل User | Many-to-Many مع scopes |
| Plain text passwords | bcrypt cost ≥ 12 |
| Sessions في localStorage | HTTP-only Cookies |
| مفيش rate limiting على login | Lock بعد N attempts |
| OTP plaintext في DB | Hash الـ OTP بنفس bcrypt |
| نسيت Force logout عند الفصل | Trigger يحذف sessions |
| Scope في الـ Application Layer | في الـ DB Query |
| متابعة Roles كـ String في users | جدول `user_roles` منفصل |
| Hard delete الموظف بعد الفصل | Soft delete + status flag |
| كل المدراء عندهم نفس الـ Role بدون scope | role + scope_type + scope_id |

### 14.5 Future Scalability Checklist

لما النظام يكبر، الـ Architecture دي بتدعم:

- ✅ **مئات الفروع**: Branch isolation موجودة by design
- ✅ **آلاف الموظفين**: Indexes على كل الـ FKs
- ✅ **Reporting الكبير**: استخدم Read Replicas + Materialized Views
- ✅ **Mobile App**: نفس الـ Sessions هتشتغل
- ✅ **SSO Integration**: ضيف external_auth_providers table
- ✅ **2FA**: ضيف user_2fa_secrets table
- ✅ **API Access for Vendors**: API Keys + Scopes
- ✅ **GDPR / قوانين البيانات**: عندك Audit Logs + Soft Delete
- ✅ **Microservices Migration**: الـ Auth يتفصل في Service مستقل بسهولة

---

## 15. ملحق: Quick Reference Tables

### 15.1 الـ Roles والـ Permissions Matrix

| Role / Permission | employee.create | employee.read | employee.update | payroll.process | leave.approve | report.financial |
|------------------|:---:|:---:|:---:|:---:|:---:|:---:|
| Super Admin | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| HR Manager | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| HR Specialist | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Branch Manager | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| Dept Manager | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| Team Lead | ❌ | ✅ (limited) | ❌ | ❌ | ❌ | ❌ |
| Employee | ❌ | ✅ (self) | ❌ | ❌ | ❌ | ❌ |

### 15.2 الـ HTTP Status Codes في Auth

| Status | الاستخدام | مثال |
|--------|-----------|------|
| `200 OK` | كل حاجة تمام | Login نجح |
| `201 Created` | اتعمل حاجة جديدة | User اتعمل |
| `400 Bad Request` | الـ data غلط | Phone بصيغة غلط |
| `401 Unauthorized` | مفيش Authentication | مفيش session token |
| `403 Forbidden` | في Auth بس مفيش Permission | User بيحاول يعمل حاجة فوق صلاحياته |
| `404 Not Found` | الـ Resource مش موجود | Employee ID غلط |
| `409 Conflict` | تعارض | Phone موجود قبل كده |
| `423 Locked` | الحساب مقفول | بعد 5 محاولات فاشلة |
| `429 Too Many Requests` | Rate limiting | Login attempts كتير |
| `500 Internal Server Error` | مشكلة في السيرفر | DB down |

### 15.3 Database Schema Cheat Sheet

```sql
-- Identity (Optional Layer)
users (id, phone, password_hash, status, must_change_password, ...)
sessions (id, user_id, session_token_hash, expires_at, ...)
password_reset_otps (id, user_id, otp_hash, expires_at, ...)

-- HR (Foundation Layer)
employees (id, user_id NULLABLE, full_name, branch_id, department_id, ...)
branches (id, name, code, ...)
departments (id, branch_id, name, ...)
teams (id, name, code, ...)                           ⭐ Cross-Branch
employee_teams (employee_id, team_id, role_in_team)   ⭐ M:N

-- Authorization
roles (id, code, name, ...)
permissions (id, code, resource, action, ...)
role_permissions (role_id, permission_id)
user_roles (user_id, role_id, scope_type, scope_id) ⭐
  -- scope_type ∈ {company, branch, department, team, employee}

-- Audit
audit_logs (id, user_id, action, resource_type, resource_id, metadata, ip, ...)
```

### 15.4 Scope Types Reference

| `scope_type` | `scope_id` يشير إلى | الاستخدام النموذجي |
|-------------|---------------------|---------------------|
| `company` | NULL | Super Admin, HR Director |
| `branch` | `branches.id` | Branch Manager, HR Manager في فرع |
| `department` | `departments.id` | Department Manager |
| `team` | `teams.id` | Sales Manager بفريق متفرع على فروع |
| `employee` | `employees.id` | الموظف العادي (Self-access) |

### 15.4 Quick Decision Tree

```
عايز تعمل feature جديدة بتلمس بيانات الموظفين؟
│
├─ هل عندك User context؟
│   ├─ لأ → ضيف authentication middleware
│   └─ آه → ✅
│
├─ هل عندك Permission Check؟
│   ├─ لأ → ضيف authorize() call
│   └─ آه → ✅
│
├─ هل في Scope filter؟
│   ├─ لأ → ضيف WHERE clause بـ scope
│   └─ آه → ✅
│
├─ هل في Audit Log؟
│   ├─ لأ → ضيف auditLog() call
│   └─ آه → ✅
│
└─ تمام، الـ Feature آمنة ✅
```

---

## 🎓 خلاصة

النظام ده مبني على 4 أعمدة:

1. **Identity (Users)** منفصلة عن **HR Data (Employees)**
2. **RBAC** مع **Scopes** لحل Multi-Branch
3. **Server-side Sessions** للأمان والـ Control
4. **Audit Logs** على كل حاجة حساسة

لو اتبعت الـ Architecture دي من اليوم الأول، هتلاقي نفسك:
- ❤️ النظام بيكبر بدون ألم
- 🛡️ ميفيش Security Bugs أساسية
- 📊 الـ Reports والـ Compliance سهلة
- 🔧 الـ Maintenance منخفض
- 👥 الـ Team بيفهم النظام بسرعة

> **آخر نصيحة**: متبدأش تكتب أي Auth code قبل ما تخلص الـ Schema. الـ DB Design هو الأساس. لو غلطت فيه، كل حاجة فوقه هتقع.

---

**نهاية المرجع** 📘

> اعتبر الملف ده مرجعك الأساسي. كل ما تحتاج تتأكد من قرار معماري، ارجعله.  
> ولو لقيت حاجة محتاجة تتحدث، عدلها وادي الفريق يعرف.

</div>
