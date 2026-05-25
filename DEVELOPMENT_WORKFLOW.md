<div dir="rtl">

# 🛠️ Development Workflow — دليل الشغل اليومي

> **الهدف**: ملف عملي تفتحه وأنت بتشتغل عشان تعرف الخطوة الجاية.  
> **مش نظري** — كل خطوة فيها أوامر، أكواد، وchecklists.  
> **افتحه دايماً** قبل ما تبدأ Feature جديدة.

---

## 📑 المحتويات

1. [Day 0: Project Setup](#day-0-project-setup)
2. [Phase 1: Organization Structure](#phase-1-organization-structure)
3. [Phase 2: Employees](#phase-2-employees)
4. [Phase 3: Auth Foundation](#phase-3-auth-foundation)
5. [⭐ The Feature Loop (الأهم!)](#-the-feature-loop-الأهم)
6. [Templates](#templates)
7. [Pre-Commit Checklist](#pre-commit-checklist)
8. [Anti-Patterns](#anti-patterns)

---

## Day 0: Project Setup

> **مرة واحدة في حياة المشروع** — قبل أي حاجة تانية.

### الخطوات

```bash
# 1. اعمل المشروع
npx create-next-app@latest hr-system \
  --typescript --tailwind --app --src-dir

cd hr-system

# 2. ثبت الـ Dependencies الأساسية
npm install drizzle-orm pg
npm install -D drizzle-kit @types/pg

npm install zod
npm install bcrypt
npm install -D @types/bcrypt

# 3. ثبت UI Library
npx shadcn@latest init

# 4. اعمل الـ Folder Structure (شوف FOLDER_STRUCTURE.md)
mkdir -p src/modules src/lib/db src/lib/crypto src/config
mkdir -p drizzle/migrations drizzle/schema
mkdir -p tests/unit tests/integration
mkdir -p docs

# 5. انقل ملفات التوثيق
mv AUTH_ARCHITECTURE.md SYSTEM_DIAGRAMS.md HR_MEETING_QUESTIONS.md \
   FOLDER_STRUCTURE.md DEVELOPMENT_WORKFLOW.md docs/

# 6. Setup Drizzle config
# اعمل drizzle.config.ts (شوف Templates أسفل)

# 7. Setup .env.local
# DATABASE_URL=postgresql://...
# SMS_API_KEY=...
# SESSION_SECRET=...

# 8. أول Commit
git init
git add .
git commit -m "chore: initial project setup"
```

### Checklist
- [ ] Next.js project working (`npm run dev`)
- [ ] Drizzle config ready
- [ ] `.env.local` فيه DATABASE_URL
- [ ] PostgreSQL DB created و accessible
- [ ] First commit done

---

## Phase 1: Organization Structure

> **المدة**: 1-2 يوم  
> **النتيجة**: تقدر تدير الفروع والأقسام والفرق من Admin UI بسيط بدون Auth.

### الخطوة 1: DB Schema

اعمل ملفات الـ Schema بالترتيب ده:

```typescript
// drizzle/schema/branches.ts
import { pgTable, bigserial, varchar, text, boolean, timestamp } from 'drizzle-orm/pg-core';

export const branches = pgTable('branches', {
  id: bigserial('id', { mode: 'number' }).primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  code: varchar('code', { length: 20 }).notNull().unique(),
  region: varchar('region', { length: 100 }),
  address: text('address'),
  phone: varchar('phone', { length: 20 }),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

```typescript
// drizzle/schema/departments.ts
import { pgTable, bigserial, varchar, bigint, boolean, timestamp, unique } from 'drizzle-orm/pg-core';
import { branches } from './branches';

export const departments = pgTable('departments', {
  id: bigserial('id', { mode: 'number' }).primaryKey(),
  branchId: bigint('branch_id', { mode: 'number' }).notNull().references(() => branches.id),
  name: varchar('name', { length: 255 }).notNull(),
  code: varchar('code', { length: 50 }).notNull(),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  unq: unique().on(t.branchId, t.code),
}));
```

```typescript
// drizzle/schema/teams.ts
// نفس الـ pattern...
```

### الخطوة 2: Generate Migration

```bash
npx drizzle-kit generate

# هيتعمل ملف في drizzle/migrations/0001_*.sql
```

### الخطوة 3: Run Migration

```bash
npx drizzle-kit migrate
```

### الخطوة 4: Modules

اعمل الـ Modules بنفس الـ Pattern:

```
src/modules/organization/branches/
├── actions.ts
├── services/branch.service.ts
├── repositories/branch.repository.ts
├── schemas/branch.schema.ts
└── index.ts
```

(شوف [Templates](#templates) أسفل)

### الخطوة 5: UI

```
app/(dashboard)/branches/
├── page.tsx          # List
├── new/page.tsx      # Create form
└── [id]/page.tsx     # Edit
```

### Phase 1 Checklist
- [ ] جداول `branches`, `departments`, `teams` متعملة
- [ ] Seed data: 2-3 فروع و 3-5 أقسام
- [ ] CRUD APIs شغالة (مفيش Auth لسه)
- [ ] UI بسيط لإدارة Org Structure
- [ ] Validation: مينفعش تحذف Branch فيه Departments
- [ ] Commit: `feat: phase 1 - organization structure`

---

## Phase 2: Employees

> **المدة**: 3-5 أيام  
> **النتيجة**: تقدر تضيف 100 موظف على النظام بدون أي Login.

### الخطوة 1: DB Schema

```typescript
// drizzle/schema/employees.ts
export const employees = pgTable('employees', {
  id: bigserial('id', { mode: 'number' }).primaryKey(),
  userId: bigint('user_id', { mode: 'number' }),  // NULLABLE — مفيش users table لسه
  fullName: varchar('full_name', { length: 255 }).notNull(),
  nationalId: varchar('national_id', { length: 20 }).notNull().unique(),
  employeeCode: varchar('employee_code', { length: 50 }).notNull().unique(),
  hireDate: date('hire_date').notNull(),
  branchId: bigint('branch_id', { mode: 'number' }).notNull().references(() => branches.id),
  departmentId: bigint('department_id', { mode: 'number' }).notNull().references(() => departments.id),
  managerId: bigint('manager_id', { mode: 'number' }),  // Self-reference
  status: varchar('status', { length: 20 }).notNull().default('active'),
  // ... باقي الحقول
});

// drizzle/schema/employee-teams.ts
export const employeeTeams = pgTable('employee_teams', {
  employeeId: bigint('employee_id', { mode: 'number' }).notNull().references(() => employees.id),
  teamId: bigint('team_id', { mode: 'number' }).notNull().references(() => teams.id),
  roleInTeam: varchar('role_in_team', { length: 100 }),
  joinedAt: timestamp('joined_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  pk: primaryKey({ columns: [t.employeeId, t.teamId] }),
}));
```

### الخطوة 2: Migration

```bash
npx drizzle-kit generate
npx drizzle-kit migrate
```

### الخطوة 3: Module

```
src/modules/employees/
├── actions.ts
├── services/
│   ├── employee.service.ts
│   └── employee-code-generator.service.ts
├── repositories/employee.repository.ts
├── schemas/
│   ├── create-employee.schema.ts
│   └── update-employee.schema.ts
└── index.ts
```

### الخطوة 4: UI

```
app/(dashboard)/employees/
├── page.tsx          # List
├── new/page.tsx      # Create
└── [id]/page.tsx     # Detail/Edit
```

### Phase 2 Checklist
- [ ] جدول `employees` متعمل (مع `user_id NULLABLE`)
- [ ] جدول `employee_teams` متعمل
- [ ] Employee Code generation شغّال (EMP-2025-0001)
- [ ] CRUD APIs شغّالة (مفيش Auth لسه)
- [ ] UI لإضافة وتعديل موظفين
- [ ] Validation: National ID فريد، رقم الموبايل صحيح
- [ ] في الـ UI: option لإضافة موظف **بدون Login**
- [ ] Commit: `feat: phase 2 - employees`

---

## Phase 3: Auth Foundation

> **المدة**: 7-10 أيام  
> **النتيجة**: Login شغّال + Authorization Middleware + RBAC جاهز.

### Phase 3.A — Identity (3-4 أيام)

#### Schemas

```
drizzle/schema/
├── users.ts
├── sessions.ts
└── password-reset-otps.ts
```

#### Module Structure

```
src/modules/auth/
├── actions.ts                  # login, logout, forgotPassword, resetPassword
├── services/
│   ├── authentication.service.ts
│   ├── session.service.ts
│   └── password.service.ts
├── repositories/
│   ├── user.repository.ts
│   ├── session.repository.ts
│   └── otp.repository.ts
├── schemas/
│   ├── login.schema.ts
│   └── reset-password.schema.ts
└── index.ts
```

#### UI

```
app/(auth)/
├── login/page.tsx
├── forgot-password/page.tsx
└── reset-password/page.tsx
```

#### Two-Step Employee Creation

عدّل `app/(dashboard)/employees/new/page.tsx`:
- بعد إنشاء Employee، اسأل: "محتاج Login؟"
- لو آه → Generate password + Send SMS + Link to User
- لو لأ → خلاص

#### Phase 3.A Checklist
- [ ] Login بـ Phone + Password شغّال
- [ ] First Login + Force Change Password شغّال
- [ ] Forgot Password Flow (OTP) شغّال
- [ ] Sessions في PostgreSQL
- [ ] HTTP-only Cookies
- [ ] Brute Force Protection (5 attempts → lock 15 min)
- [ ] Two-Step Employee Creation شغّال
- [ ] Commit: `feat: phase 3a - authentication`

### Phase 3.B — RBAC (2-3 أيام)

#### Schemas

```
drizzle/schema/
├── roles.ts
├── permissions.ts
└── role-permissions.ts
```

#### Seed Data

```typescript
// drizzle/seeds/roles.seed.ts
export const systemRoles = [
  { code: 'super_admin',    name: 'مدير النظام الأعلى',  isSystem: true },
  { code: 'hr_manager',     name: 'مدير موارد بشرية',     isSystem: true },
  { code: 'hr_specialist',  name: 'أخصائي موارد بشرية',   isSystem: true },
  { code: 'branch_manager', name: 'مدير فرع',             isSystem: true },
  { code: 'dept_manager',   name: 'مدير قسم',             isSystem: true },
  { code: 'team_lead',      name: 'قائد فريق',            isSystem: true },
  { code: 'employee',       name: 'موظف',                 isSystem: true },
];

// drizzle/seeds/permissions.seed.ts
export const basePermissions = [
  { code: 'employee.create', resource: 'employee', action: 'create' },
  { code: 'employee.read',   resource: 'employee', action: 'read' },
  { code: 'employee.update', resource: 'employee', action: 'update' },
  { code: 'employee.delete', resource: 'employee', action: 'delete' },
  { code: 'branch.manage',   resource: 'branch',   action: 'manage' },
  // ... باقي الـ Base Permissions
];
```

#### Phase 3.B Checklist
- [ ] جداول roles, permissions, role_permissions
- [ ] Seed 7 System Roles
- [ ] Seed Base Permissions
- [ ] Admin UI لإدارة Roles والـ Permissions
- [ ] Commit: `feat: phase 3b - rbac foundation`

### Phase 3.C — Scopes (2-3 أيام)

#### Schema

```
drizzle/schema/user-roles.ts
```

#### Phase 3.C Checklist
- [ ] جدول `user_roles` بـ `scope_type` + `scope_id`
- [ ] UI لإسناد Role + Scope لـ User
- [ ] Validation: scope_id موجود في الجدول المناسب
- [ ] Multi-role per user
- [ ] Commit: `feat: phase 3c - scopes`

### Phase 3.D — Authorization (3-5 أيام)

#### Helpers الأساسية

```
src/modules/authorization/
├── helpers/
│   ├── authorize.ts          # دالة authorize() الأساسية
│   ├── get-scope-filter.ts   # دالة getScopeFilter()
│   └── load-user-context.ts  # تحميل user roles + permissions
├── services/
└── repositories/
```

#### Middleware

```typescript
// middleware.ts (في الـ Root)
// شوف Template أسفل
```

#### Audit Logs

```
src/modules/audit/
├── services/audit-log.service.ts
└── repositories/audit-log.repository.ts
```

#### Phase 3.D Checklist
- [ ] `authorize()` middleware شغّالة
- [ ] `getScopeFilter()` helper شغّالة
- [ ] كل endpoint موجود بيستخدم authorize()
- [ ] Audit logs بيتسجل في كل عملية حساسة
- [ ] Force logout على termination
- [ ] Commit: `feat: phase 3d - authorization`

---

## ⭐ The Feature Loop (الأهم!)

> **ده الـ Workflow اللي هتمشي عليه طول حياة المشروع** بعد ما تخلص الـ Foundation.

### كل Feature جديدة بتمر بـ 7 خطوات

```
┌──────────────────────────────────────────────────────────┐
│  1. PLAN                                                 │
│  ─────                                                   │
│  ✓ اعرف الـ Feature بتعمل إيه                            │
│  ✓ حدد الـ Permissions الجديدة المطلوبة                  │
│  ✓ حدد مين من الـ Roles هياخدهم                          │
│  ✓ حدد الـ Scope Types المناسبة                          │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  2. SCHEMA & MIGRATION                                   │
│  ──────────────────                                       │
│  ✓ اعمل Schema الجديدة لو محتاج جداول                    │
│  ✓ Migration: INSERT INTO permissions                    │
│  ✓ Migration: INSERT INTO role_permissions               │
│  ✓ Run: npx drizzle-kit migrate                          │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  3. MODULE STRUCTURE                                     │
│  ──────────────────                                       │
│  ✓ اعمل src/modules/<feature>/                           │
│  ✓ actions.ts, services/, repositories/, schemas/        │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  4. SERVICE LAYER (مع Auth من أول سطر!)                  │
│  ───────────────────────                                  │
│  ✓ كل function في الـ service بتبدأ بـ authorize()        │
│  ✓ كل Query بيستخدم getScopeFilter()                     │
│  ✓ متجبش بيانات ثم تفلتر — فلتر في الـ DB                │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  5. UI                                                   │
│  ───                                                     │
│  ✓ اعمل الـ pages في app/(dashboard)/<feature>/          │
│  ✓ استخدم Server Components للـ data fetching            │
│  ✓ استخدم Server Actions للـ mutations                   │
│  ✓ خفي الأزرار اللي مش متاحة حسب الـ Permissions         │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  6. AUDIT LOGS                                           │
│  ─────────────                                            │
│  ✓ كل عملية حساسة بتـ log في audit_logs                  │
│  ✓ Log فيه: مين، إمتى، عمل إيه، على إيه                   │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│  7. TEST                                                 │
│  ─────                                                   │
│  ✓ Test الـ Feature بـ User عنده Permission              │
│  ✓ Test بـ User معندوش Permission (Negative Test)        │
│  ✓ Test بـ User عنده Permission بس scope مختلف           │
│  ✓ Test الـ UI بـ Roles مختلفة                           │
└──────────────────────────────────────────────────────────┘
```

### مثال كامل: إضافة Leave Management Feature

#### Step 1: Plan

```markdown
**Feature**: Leave Management
**Permissions Needed**:
- leave.request   → الموظف يطلب أجازة
- leave.read      → عرض الأجازات
- leave.approve   → اعتماد أجازة
- leave.reject    → رفض أجازة
- leave.cancel    → إلغاء أجازة

**Role Assignments**:
- Employee: leave.request, leave.read (self scope)
- Dept Manager: leave.read, leave.approve, leave.reject (department scope)
- HR Manager: كل الـ permissions (branch scope)
- HR Director: كل الـ permissions (company scope)

**Scope Types**: department (للموافقات), employee (للطلبات)
```

#### Step 2: Schema & Migration

```typescript
// drizzle/schema/leaves.ts
export const leaves = pgTable('leaves', {
  id: bigserial('id', { mode: 'number' }).primaryKey(),
  employeeId: bigint('employee_id', { mode: 'number' }).notNull().references(() => employees.id),
  leaveType: varchar('leave_type', { length: 50 }).notNull(),
  startDate: date('start_date').notNull(),
  endDate: date('end_date').notNull(),
  status: varchar('status', { length: 20 }).notNull().default('pending'),
  reason: text('reason'),
  approvedBy: bigint('approved_by', { mode: 'number' }),
  approvedAt: timestamp('approved_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

```sql
-- drizzle/migrations/0010_leave_permissions.sql
INSERT INTO permissions (code, resource, action, description) VALUES
  ('leave.request', 'leave', 'request', 'طلب أجازة'),
  ('leave.read',    'leave', 'read',    'عرض الأجازات'),
  ('leave.approve', 'leave', 'approve', 'اعتماد أجازة'),
  ('leave.reject',  'leave', 'reject',  'رفض أجازة'),
  ('leave.cancel',  'leave', 'cancel',  'إلغاء أجازة');

-- Assign to roles
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id 
FROM roles r, permissions p 
WHERE 
  (r.code = 'employee' AND p.code IN ('leave.request', 'leave.read')) OR
  (r.code = 'dept_manager' AND p.code IN ('leave.read', 'leave.approve', 'leave.reject')) OR
  (r.code = 'hr_manager' AND p.code LIKE 'leave.%');
```

#### Step 3: Module Structure

```bash
mkdir -p src/modules/leave/{services,repositories,schemas}
touch src/modules/leave/actions.ts
touch src/modules/leave/index.ts
```

#### Step 4: Service Layer

```typescript
// src/modules/leave/services/leave.service.ts
import { authorize } from '@/src/modules/authorization/helpers/authorize';
import { auditLog } from '@/src/modules/audit/services/audit-log.service';

export async function requestLeave(userId: number, data: LeaveRequest) {
  // ⭐ Authorize أولاً
  await authorize(userId, 'leave.request', { 
    scopeType: 'employee', 
    scopeId: data.employeeId 
  });
  
  // Business logic
  const leave = await leaveRepo.create(data);
  
  // Audit
  await auditLog({ 
    userId, 
    action: 'leave.requested', 
    resourceType: 'leave',
    resourceId: leave.id,
    metadata: { startDate: data.startDate, endDate: data.endDate }
  });
  
  return leave;
}

export async function approveLeave(userId: number, leaveId: number) {
  const leave = await leaveRepo.findById(leaveId);
  const employee = await employeeRepo.findById(leave.employeeId);
  
  // ⭐ Authorize بالـ scope الصح
  await authorize(userId, 'leave.approve', { 
    scopeType: 'department', 
    scopeId: employee.departmentId 
  });
  
  await leaveRepo.update(leaveId, { 
    status: 'approved', 
    approvedBy: userId, 
    approvedAt: new Date() 
  });
  
  await auditLog({ 
    userId, 
    action: 'leave.approved', 
    resourceType: 'leave',
    resourceId: leaveId
  });
}
```

#### Step 5: UI

```typescript
// app/(dashboard)/leaves/page.tsx
import { getMyLeaves } from '@/src/modules/leave/actions';
import { hasPermission } from '@/src/modules/authorization';

export default async function LeavesPage() {
  const leaves = await getMyLeaves();  // بيـ filter بالـ scope تلقائياً
  const canApprove = await hasPermission('leave.approve');
  
  return (
    <div>
      <LeaveList leaves={leaves} showApproveBtn={canApprove} />
      <RequestLeaveButton />
    </div>
  );
}
```

#### Step 6: Audit Logs
- ✅ تم في الـ service layer

#### Step 7: Test

```typescript
// tests/integration/leave.test.ts
describe('Leave Management', () => {
  it('employee can request own leave', async () => {
    const employee = await createTestUser({ role: 'employee' });
    const leave = await requestLeave(employee.userId, { ... });
    expect(leave).toBeDefined();
  });
  
  it('employee cannot approve leaves', async () => {
    const employee = await createTestUser({ role: 'employee' });
    await expect(
      approveLeave(employee.userId, leaveId)
    ).rejects.toThrow('Forbidden');
  });
  
  it('manager can approve leaves in their department only', async () => {
    const manager = await createTestUser({ 
      role: 'dept_manager', 
      scope: 'department:5' 
    });
    
    // Employee في نفس القسم
    const sameDept = await createLeaveInDepartment(5);
    await expect(approveLeave(manager.userId, sameDept.id)).resolves.toBeDefined();
    
    // Employee في قسم تاني
    const otherDept = await createLeaveInDepartment(7);
    await expect(approveLeave(manager.userId, otherDept.id)).rejects.toThrow();
  });
});
```

---

## Templates

### Migration Template

```sql
-- drizzle/migrations/NNNN_feature_name.sql

-- 1. اعمل الـ tables الجديدة
CREATE TABLE feature_table (...);
CREATE INDEX idx_feature_field ON feature_table(field);

-- 2. ضيف Permissions
INSERT INTO permissions (code, resource, action, description) VALUES
  ('feature.action1', 'feature', 'action1', 'وصف'),
  ('feature.action2', 'feature', 'action2', 'وصف');

-- 3. اربط الـ permissions بالـ roles
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p
WHERE (r.code, p.code) IN (
  ('hr_manager', 'feature.action1'),
  ('hr_manager', 'feature.action2'),
  ('dept_manager', 'feature.action1')
);
```

### Service Template

```typescript
// src/modules/<feature>/services/<feature>.service.ts
import { authorize } from '@/src/modules/authorization/helpers/authorize';
import { auditLog } from '@/src/modules/audit/services/audit-log.service';
import { <feature>Repo } from '../repositories/<feature>.repository';
import { <Feature>Schema } from '../schemas/<feature>.schema';

export async function create<Feature>(userId: number, input: unknown) {
  // 1. Validate
  const data = <Feature>Schema.parse(input);
  
  // 2. Authorize
  await authorize(userId, '<feature>.create', { 
    scopeType: 'branch',  // أو حسب الـ feature
    scopeId: data.branchId 
  });
  
  // 3. Business Logic
  const result = await <feature>Repo.create(data);
  
  // 4. Audit
  await auditLog({ 
    userId, 
    action: '<feature>.created',
    resourceType: '<feature>',
    resourceId: result.id,
    metadata: { ...data }
  });
  
  return result;
}
```

### Server Action Template

```typescript
// src/modules/<feature>/actions.ts
'use server';

import { getSession } from '@/src/modules/auth';
import { create<Feature>, update<Feature> } from './services/<feature>.service';
import { revalidatePath } from 'next/cache';

export async function create<Feature>Action(formData: FormData) {
  const session = await getSession();
  if (!session) throw new Error('Unauthorized');
  
  const data = Object.fromEntries(formData);
  
  try {
    const result = await create<Feature>(session.userId, data);
    revalidatePath('/<feature>');
    return { success: true, data: result };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

### Repository Template

```typescript
// src/modules/<feature>/repositories/<feature>.repository.ts
import { db } from '@/src/lib/db/client';
import { <featureTable> } from '@/drizzle/schema';
import { eq } from 'drizzle-orm';

export async function findById(id: number) {
  return db.select().from(<featureTable>).where(eq(<featureTable>.id, id)).limit(1);
}

export async function create(data: New<Feature>) {
  return db.insert(<featureTable>).values(data).returning();
}

export async function update(id: number, data: Partial<<Feature>>) {
  return db.update(<featureTable>).set(data).where(eq(<featureTable>.id, id)).returning();
}
```

---

## Pre-Commit Checklist

> اطبع الـ Checklist ده وحطه جنبك. **قبل أي PR** اعدي عليه.

### General
- [ ] الـ Code بيـ build (`npm run build`)
- [ ] الـ Linter ماشي بدون warnings (`npm run lint`)
- [ ] الـ Types صحيحة (`npm run typecheck`)
- [ ] Tests شغّالة (`npm test`)

### Security
- [ ] كل endpoint جديد بيستخدم `authorize()`
- [ ] الـ Queries بتفلتر بالـ Scope (مش بتجيب الكل)
- [ ] العمليات الحساسة بتـ log في `audit_logs`
- [ ] مفيش Plaintext Passwords / Secrets في الـ Code
- [ ] الـ Inputs بتـ validate بـ Zod

### Auth & Permissions
- [ ] لو في Feature جديدة، الـ Permissions متعملة في Migration
- [ ] الـ Permissions الجديدة مربوطة بالـ Roles المناسبة
- [ ] في Negative Tests للـ Auth (User بدون permission)

### Database
- [ ] Migration متعملة لأي Schema change
- [ ] Indexes موجودة على الـ FKs
- [ ] الـ ON DELETE rules صحيحة (CASCADE / SET NULL)

### UI/UX
- [ ] الأزرار اللي مش متاحة للـ User مخفية
- [ ] الـ Forms بتعرض Validation Errors واضحة
- [ ] Loading States موجودة

---

## Anti-Patterns

> الأخطاء اللي لو وقعت فيها هتدفع الثمن بعدين.

### ❌ Anti-Pattern 1: Auth في الـ UI بس

```typescript
// ❌ غلط
{user.role === 'admin' && <DeleteButton />}
// المستخدم العادي ممكن يستدعي الـ API مباشرة

// ✅ صح: Auth في الـ Service Layer
export async function deleteEmployee(userId, employeeId) {
  await authorize(userId, 'employee.delete', ...);
  // ...
}
```

### ❌ Anti-Pattern 2: جلب البيانات وفلترتها في الـ Code

```typescript
// ❌ غلط
const allEmployees = await db.select().from(employees);
const filtered = allEmployees.filter(e => e.branchId === userBranchId);

// ✅ صح: فلتر في الـ DB
const employees = await db.select().from(employees)
  .where(eq(employees.branchId, userBranchId));
```

### ❌ Anti-Pattern 3: Hardcoded Roles في الـ Code

```typescript
// ❌ غلط
if (user.role === 'hr_manager') { ... }

// ✅ صح: شيك الـ Permission
if (await hasPermission(user.id, 'employee.update')) { ... }
```

### ❌ Anti-Pattern 4: تجاهل Audit Logs

```typescript
// ❌ غلط
await leaveRepo.approve(leaveId);
return { success: true };

// ✅ صح
await leaveRepo.approve(leaveId);
await auditLog({ userId, action: 'leave.approved', ... });
return { success: true };
```

### ❌ Anti-Pattern 5: متطلبات Auth في الـ End

> "هاسيب Auth للآخر، عايز أخلص الـ Features الأول"

**النتيجة**: لما توصل للنهاية، هتلاقي 50 endpoint بدون Auth، 30 query بدون scope filter، صفر audit logs. الـ refactor هياخد شهر.

### ❌ Anti-Pattern 6: Permission Sprawl

```typescript
// ❌ غلط: Permissions غامضة وكتيرة
'admin_thing_1', 'admin_thing_2', 'do_stuff'

// ✅ صح: Naming Convention واضح
'employee.create', 'employee.read', 'employee.update'
// {resource}.{action}
```

---

## 📚 الـ Docs اللي ترجعلها

| متى ترجع لـ | الـ Doc |
|------------|--------|
| لما تنسى ازاي الـ scopes شغالة | [AUTH_ARCHITECTURE.md](./AUTH_ARCHITECTURE.md) |
| لما تحتاج Diagram يوضح علاقة | [SYSTEM_DIAGRAMS.md](./SYSTEM_DIAGRAMS.md) |
| لما تنسى الـ Folder Structure | [FOLDER_STRUCTURE.md](./FOLDER_STRUCTURE.md) |
| لما تيجي تعمل ميتنج مع HR | [HR_MEETING_QUESTIONS.md](./HR_MEETING_QUESTIONS.md) |
| لما تبدأ Feature جديدة | **DEVELOPMENT_WORKFLOW.md** (الملف ده) |

---

## 🎯 خلاصة الـ Workflow

```
Day 0: Setup (مرة واحدة)
    ↓
Phase 1: Org Structure (1-2 يوم)
    ↓
Phase 2: Employees (3-5 أيام)
    ↓
Phase 3: Auth Foundation (7-10 أيام)
    ↓
─────────────────────────────────────
    من هنا، أي Feature جديدة:
─────────────────────────────────────
    1. PLAN → 2. MIGRATE → 3. MODULE → 
    4. SERVICE (+ Auth) → 5. UI → 
    6. AUDIT → 7. TEST → COMMIT
```

> **القاعدة الذهبية**: كل Feature بتدخل النظام **بـ Auth** من اليوم الأول. مفيش "هضيفها بعدين".

</div>
