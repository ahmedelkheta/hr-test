<div dir="rtl">

# 📁 Folder Structure — هيكل المشروع

> **التقنية**: Next.js 15 (App Router) + TypeScript + PostgreSQL  
> **المبدأ**: Logical Separation داخل Project واحد، **مش** Physical Separation.

---

## 🎯 الفلسفة قبل الهيكل

### ليه مش نكسر FE و BE في رمو منفصلين؟

في Next.js الحديث، الفصل لـ رمو منفصلين بيخسرك:

| اللي بتخسره | السبب |
|------------|------|
| Type Safety end-to-end | TypeScript مش بيعدي boundary الـ HTTP |
| Performance | كل request = HTTP call (بدل function call) |
| Auth Simplicity | CORS, Token sharing, Session sync |
| Developer Experience | Hot reload لـ 2 projects، deployment لـ 2 services |
| Server Components Magic | بتخسر فايدة `async` في الـ UI |

### بدل كده — Internal Layering

```
┌─────────────────────────────────────────────────┐
│  UI Layer (app/, components/)                   │
│  ───────────────────────────                    │
│  Server Components + Client Components          │
│  بتستدعي ↓                                       │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│  Application Layer (src/modules/*/actions.ts)   │
│  ─────────────────────────────────────          │
│  Server Actions + Route Handlers                │
│  بتستدعي ↓                                       │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│  Domain Layer (src/modules/*/services/)         │
│  ──────────────────────────────────             │
│  Business Logic + Validation + Authorization    │
│  بتستدعي ↓                                       │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│  Data Layer (src/modules/*/repositories/)       │
│  ──────────────────────────────────             │
│  Database queries فقط                            │
└─────────────────────────────────────────────────┘
```

> 💡 **القاعدة**: كل Layer بتستدعي اللي تحتها فقط. مفيش UI Component بيكلم Database مباشرة.

---

## 📂 الـ Folder Structure الكامل

```
hr-system/
│
├── 📱 app/                              # UI Layer — Next.js App Router
│   │
│   ├── (auth)/                          # Route Group: صفحات بدون Auth
│   │   ├── layout.tsx                   # Auth pages layout
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── forgot-password/
│   │   │   └── page.tsx
│   │   └── reset-password/
│   │       └── page.tsx
│   │
│   ├── (dashboard)/                     # Route Group: صفحات محتاجة Login
│   │   ├── layout.tsx                   # Sidebar + auth check
│   │   ├── page.tsx                     # Dashboard home
│   │   │
│   │   ├── employees/
│   │   │   ├── page.tsx                 # List
│   │   │   ├── new/
│   │   │   │   └── page.tsx             # Create
│   │   │   └── [id]/
│   │   │       ├── page.tsx             # Detail/Edit
│   │   │       ├── documents/
│   │   │       │   └── page.tsx
│   │   │       └── history/
│   │   │           └── page.tsx
│   │   │
│   │   ├── branches/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   │
│   │   ├── departments/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   │
│   │   ├── teams/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   │
│   │   └── settings/
│   │       ├── roles/page.tsx
│   │       ├── permissions/page.tsx
│   │       └── audit-logs/page.tsx
│   │
│   ├── api/                             # API Routes — للـ Webhooks فقط
│   │   ├── webhooks/
│   │   │   └── sms/route.ts
│   │   └── health/route.ts              # Health check
│   │
│   ├── layout.tsx                       # Root layout
│   ├── globals.css
│   ├── error.tsx                        # Global error boundary
│   └── not-found.tsx
│
├── 🧠 src/                              # Backend Layer — Business Logic
│   │
│   ├── modules/                         # Domain-Driven Modules
│   │   │
│   │   ├── auth/                        # Authentication module
│   │   │   ├── actions.ts               # Server Actions (login, logout)
│   │   │   ├── services/
│   │   │   │   ├── authentication.service.ts
│   │   │   │   ├── session.service.ts
│   │   │   │   └── password.service.ts
│   │   │   ├── repositories/
│   │   │   │   ├── user.repository.ts
│   │   │   │   ├── session.repository.ts
│   │   │   │   └── otp.repository.ts
│   │   │   ├── schemas/                 # Zod validation schemas
│   │   │   │   ├── login.schema.ts
│   │   │   │   ├── reset-password.schema.ts
│   │   │   │   └── change-password.schema.ts
│   │   │   ├── types.ts
│   │   │   └── index.ts                 # Public exports
│   │   │
│   │   ├── employees/                   # Employee module
│   │   │   ├── actions.ts               # createEmployee, updateEmployee, ...
│   │   │   ├── services/
│   │   │   │   ├── employee.service.ts
│   │   │   │   └── employee-status.service.ts
│   │   │   ├── repositories/
│   │   │   │   └── employee.repository.ts
│   │   │   ├── schemas/
│   │   │   │   ├── create-employee.schema.ts
│   │   │   │   └── update-employee.schema.ts
│   │   │   ├── types.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── organization/                # Org Structure module
│   │   │   ├── branches/
│   │   │   │   ├── actions.ts
│   │   │   │   ├── services/
│   │   │   │   ├── repositories/
│   │   │   │   └── schemas/
│   │   │   ├── departments/
│   │   │   │   ├── actions.ts
│   │   │   │   ├── services/
│   │   │   │   ├── repositories/
│   │   │   │   └── schemas/
│   │   │   └── teams/
│   │   │       ├── actions.ts
│   │   │       ├── services/
│   │   │       ├── repositories/
│   │   │       └── schemas/
│   │   │
│   │   ├── authorization/               # RBAC + Scopes module
│   │   │   ├── services/
│   │   │   │   ├── permission-check.service.ts
│   │   │   │   ├── scope-filter.service.ts
│   │   │   │   └── role-assignment.service.ts
│   │   │   ├── repositories/
│   │   │   │   ├── role.repository.ts
│   │   │   │   ├── permission.repository.ts
│   │   │   │   └── user-role.repository.ts
│   │   │   ├── helpers/
│   │   │   │   ├── authorize.ts         # authorize() function
│   │   │   │   └── get-scope-filter.ts
│   │   │   └── types.ts
│   │   │
│   │   └── audit/                       # Audit logging module
│   │       ├── services/
│   │       │   └── audit-log.service.ts
│   │       ├── repositories/
│   │       │   └── audit-log.repository.ts
│   │       └── types.ts
│   │
│   ├── lib/                             # Shared utilities (cross-module)
│   │   ├── db/
│   │   │   ├── client.ts                # DB client instance
│   │   │   ├── schema/                  # Drizzle schema (لو استخدمت Drizzle)
│   │   │   │   ├── users.ts
│   │   │   │   ├── employees.ts
│   │   │   │   ├── branches.ts
│   │   │   │   ├── departments.ts
│   │   │   │   ├── teams.ts
│   │   │   │   ├── roles.ts
│   │   │   │   ├── permissions.ts
│   │   │   │   └── index.ts
│   │   │   └── seeds/
│   │   │       ├── roles.seed.ts
│   │   │       ├── permissions.seed.ts
│   │   │       └── branches.seed.ts
│   │   │
│   │   ├── sms/
│   │   │   ├── client.ts                # SMS provider client
│   │   │   └── templates.ts
│   │   │
│   │   ├── crypto/
│   │   │   ├── password.ts              # bcrypt wrapper
│   │   │   ├── token.ts                 # Session token generation
│   │   │   └── otp.ts                   # OTP generation
│   │   │
│   │   ├── errors/
│   │   │   ├── app-error.ts             # Base error class
│   │   │   ├── auth-error.ts
│   │   │   ├── permission-error.ts
│   │   │   └── validation-error.ts
│   │   │
│   │   └── utils/
│   │       ├── date.ts
│   │       ├── phone.ts                 # Phone formatting
│   │       └── id.ts                    # Employee code generation
│   │
│   ├── config/
│   │   ├── env.ts                       # Validated env vars (zod)
│   │   ├── constants.ts                 # App constants
│   │   └── routes.ts                    # Route paths constants
│   │
│   └── middleware/                      # Reusable middleware
│       ├── auth.middleware.ts
│       └── rate-limit.middleware.ts
│
├── 🎨 components/                       # UI Components Layer
│   │
│   ├── ui/                              # Primitives (shadcn/ui style)
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── select.tsx
│   │   ├── dialog.tsx
│   │   ├── table.tsx
│   │   └── ...
│   │
│   ├── layouts/                         # Layout components
│   │   ├── sidebar.tsx
│   │   ├── header.tsx
│   │   └── breadcrumbs.tsx
│   │
│   ├── forms/                           # Reusable form patterns
│   │   ├── form-field.tsx
│   │   ├── form-error.tsx
│   │   └── form-submit.tsx
│   │
│   ├── tables/                          # Data tables
│   │   ├── data-table.tsx
│   │   ├── pagination.tsx
│   │   └── filters.tsx
│   │
│   └── features/                        # Feature-specific components
│       ├── employees/
│       │   ├── employee-form.tsx
│       │   ├── employee-card.tsx
│       │   └── employee-list.tsx
│       ├── branches/
│       ├── auth/
│       │   ├── login-form.tsx
│       │   └── otp-input.tsx
│       └── shared/
│
├── 🔌 hooks/                            # Custom React Hooks
│   ├── use-current-user.ts
│   ├── use-permissions.ts
│   ├── use-toast.ts
│   └── use-debounce.ts
│
├── 📝 types/                            # Global TypeScript types
│   ├── api.ts                           # API response types
│   ├── auth.ts                          # Auth-related types
│   └── globals.d.ts
│
├── 🧪 tests/
│   ├── unit/                            # Unit tests for services
│   │   ├── auth/
│   │   ├── employees/
│   │   └── authorization/
│   ├── integration/                     # Integration tests with DB
│   └── e2e/                             # Playwright/Cypress
│
├── 🗄️ drizzle/                          # ORM migrations
│   ├── migrations/
│   │   ├── 0001_create_branches.sql
│   │   ├── 0002_create_departments.sql
│   │   ├── 0003_create_employees.sql
│   │   └── ...
│   └── drizzle.config.ts
│
├── 📚 docs/                             # Documentation
│   ├── AUTH_ARCHITECTURE.md             # ← انقل ملفاتك دي هنا
│   ├── SYSTEM_DIAGRAMS.md
│   ├── HR_MEETING_QUESTIONS.md
│   ├── FOLDER_STRUCTURE.md              # ← الملف ده
│   └── api/
│       └── (api-specific docs)
│
├── 🌐 public/                           # Static assets
│   ├── logo.svg
│   └── images/
│
├── 📋 scripts/                          # Build/deployment scripts
│   ├── seed-db.ts
│   └── generate-types.ts
│
├── .env.example
├── .env.local                           # ❌ Gitignored
├── .gitignore
├── middleware.ts                        # Next.js Edge Middleware (root)
├── next.config.js
├── package.json
├── tsconfig.json
├── tailwind.config.ts
└── README.md
```

---

## 🎯 الأنماط المهمة في الـ Structure

### 1. Module-Based Organization

كل Domain Concept في Module مستقل في `src/modules/`. كل Module عنده نفس الـ Structure:

```
modules/<domain>/
├── actions.ts          # Server Actions (entry points)
├── services/           # Business logic
├── repositories/       # Database access
├── schemas/            # Validation (Zod)
├── types.ts            # TypeScript types
└── index.ts            # Public exports
```

**ليه ده مهم؟**
- لما تيجي تشتغل على Feature معينة، كل اللي محتاجه في فولدر واحد
- سهل تنقل Module كامل لـ Microservice لو احتجت لاحقاً
- الـ Encapsulation واضح (الـ `index.ts` بيحدد إيه المتاح للـ Public)

### 2. The Service-Repository Pattern

```typescript
// repositories/employee.repository.ts (Database فقط)
export async function findEmployeeById(id: number) {
  return db.select().from(employees).where(eq(employees.id, id));
}

// services/employee.service.ts (Business Logic)
export async function getEmployeeWithPermissionCheck(
  userId: number, 
  employeeId: number
) {
  await authorize(userId, 'employee.read', employeeId);
  return findEmployeeById(employeeId);
}

// actions.ts (Server Action — UI Entry Point)
'use server';
export async function getEmployee(employeeId: number) {
  const session = await getSession();
  return getEmployeeWithPermissionCheck(session.userId, employeeId);
}
```

**ليه ده مهم؟**
- Testing سهل (تـ mock الـ repository وتختبر الـ service)
- لو غيرت من Drizzle لـ Prisma، بتغير الـ repositories بس
- الـ Authorization في الـ Service Layer مش الـ UI

### 3. Server Actions بدل API Routes

```typescript
// ✅ صح: Server Action
// app/(dashboard)/employees/new/page.tsx
'use client';
import { createEmployee } from '@/src/modules/employees/actions';

<form action={createEmployee}>
  <input name="full_name" />
  <button type="submit">حفظ</button>
</form>
```

```typescript
// ❌ تقليدي: API Route (تجنبه لما متاح Server Actions)
// app/api/employees/route.ts
export async function POST(req: Request) {
  // ...
}
```

**ليه Server Actions أفضل؟**
- Type Safety بين الـ Form و الـ Backend Function
- No HTTP layer = no JSON serialization overhead
- Progressive enhancement (يشتغل بدون JavaScript)

> 💡 **استثناء**: استخدم API Routes للـ Webhooks الخارجية، Mobile Apps، أو Third-party Integrations.

### 4. Route Groups للـ Layouts المختلفة

```
app/
├── (auth)/          ← Layout بدون sidebar (login, forgot password)
│   └── layout.tsx
└── (dashboard)/     ← Layout بـ sidebar (الصفحات الداخلية)
    └── layout.tsx
```

الـ Parentheses بتمنع الـ Folder من إنه يبقى جزء من الـ URL. يعني `/login` مش `/auth/login`.

### 5. Middleware للـ Auth Edge

```typescript
// middleware.ts (في الـ Root)
export async function middleware(req) {
  const session = await getSession(req);
  if (!session) return NextResponse.redirect('/login');
}

export const config = {
  matcher: ['/((?!login|forgot-password|_next|api/webhooks).*)'],
};
```

ده بيحمي كل الصفحات تلقائياً ما عدا الـ Public.

---

## 🛠️ التقنيات الموصى بيها

| الطبقة | التقنية الموصى بيها | البديل |
|--------|---------------------|--------|
| **Framework** | Next.js 15 (App Router) | — |
| **Language** | TypeScript (Strict) | — |
| **ORM** | Drizzle ORM | Prisma |
| **Database** | PostgreSQL 16+ | — |
| **Validation** | Zod | Yup, Valibot |
| **UI Library** | shadcn/ui (Radix + Tailwind) | Mantine, Chakra |
| **Forms** | React Hook Form | Conform, Tanstack Form |
| **State (server)** | React Query / SWR | Built-in fetch |
| **State (client)** | Zustand | Jotai, Context |
| **Auth** | Lucia / Custom | NextAuth.js |
| **SMS** | Twilio / Local provider | — |
| **Testing** | Vitest + Playwright | Jest, Cypress |
| **Linting** | ESLint + Prettier | Biome |

> 💡 **التوصية على Drizzle**: لأنه SQL-like وبيتماشى مع الـ Raw SQL في الـ AUTH_ARCHITECTURE.md. Type-safe وخفيف.

---

## 📋 قواعد الـ Code Organization

### ❌ DON'T
- متستوردش من `app/` إلى `src/` (مش لازم الـ UI يكون dependency)
- متستوردش من Module لـ Module في `repositories/` (هتعمل tight coupling)
- متخليش الـ Components في `components/` تكلم الـ DB مباشرة
- متحطش Business Logic في الـ UI (الـ pages)

### ✅ DO
- استورد من `src/modules/*/index.ts` فقط (Public API)
- خلي الـ Business Logic في `services/`
- خلي الـ DB queries في `repositories/`
- استخدم Server Components لـ Data Fetching (مفيش `useEffect + fetch`)

---

## 🚦 ترتيب البناء على الـ Folder Structure

بناءً على الـ Build Order اللي اتفقنا عليه (Phase 1 → 2 → 3):

### Phase 1: Organization Structure
ابني في الترتيب ده:

```
1. drizzle/schema/branches.ts
2. drizzle/schema/departments.ts
3. drizzle/schema/teams.ts
4. drizzle/migrations/0001_*.sql
5. src/modules/organization/branches/
6. src/modules/organization/departments/
7. src/modules/organization/teams/
8. app/(dashboard)/branches/
9. app/(dashboard)/departments/
10. app/(dashboard)/teams/
```

### Phase 2: Employees

```
1. drizzle/schema/employees.ts
2. drizzle/schema/employee_teams.ts
3. drizzle/migrations/0004_create_employees.sql
4. src/modules/employees/
5. app/(dashboard)/employees/
```

### Phase 3: Auth Layer

```
1. drizzle/schema/users.ts, sessions.ts, ...
2. src/modules/auth/
3. src/modules/authorization/
4. src/modules/audit/
5. middleware.ts (root)
6. app/(auth)/login, forgot-password, ...
```

---

## 🎓 خلاصة

| المبدأ | التطبيق |
|--------|---------|
| **Separation** | منطقي (Layers) مش فيزيائي (Repos) |
| **Modules** | كل Domain في فولدر مستقل |
| **Service Pattern** | Service ↔ Repository |
| **Type Safety** | TypeScript على كل الـ Stack |
| **Server-First** | Server Components + Server Actions |
| **Migration Path** | لو احتجت Microservices لاحقاً، الـ Modules جاهزة للنقل |

> 🎯 **القاعدة الذهبية**: ابني Monolith بـ Clean Architecture. لما المشروع يكبر فعلاً ومحتاج Scale، تقدر تكسر Module واحد لـ Microservice. **مش العكس**.

---

## 📌 Action Items

- [ ] انقل ملفات التوثيق الموجودة (AUTH_ARCHITECTURE.md, SYSTEM_DIAGRAMS.md, HR_MEETING_QUESTIONS.md) لـ `docs/`
- [ ] ابدأ بـ `npx create-next-app@latest hr-system --typescript --tailwind --app`
- [ ] ثبت Drizzle: `npm install drizzle-orm pg && npm install -D drizzle-kit`
- [ ] ثبت Zod: `npm install zod`
- [ ] ثبت shadcn/ui: `npx shadcn-ui@latest init`
- [ ] اعمل الـ folders فاضية بالـ structure ده
- [ ] ابدأ Phase 1: `drizzle/schema/branches.ts`

</div>
