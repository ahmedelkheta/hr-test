# HR Management System — Architecture & Documentation

> توثيق production-grade لبناء نظام HR Management متعدد الفروع باستخدام Next.js + PostgreSQL.

---

## 📚 Documentation Index

| الملف | الوصف |
|------|------|
| [`AUTH_ARCHITECTURE.md`](./AUTH_ARCHITECTURE.md) | الـ Architecture الكاملة (Authentication, Authorization, RBAC, Scopes) — المرجع الأساسي |
| [`SYSTEM_DIAGRAMS.md`](./SYSTEM_DIAGRAMS.md) | 12 Mermaid Diagrams تشرح كل العلاقات بصرياً |
| [`EXAMPLE_SCENARIO.md`](./EXAMPLE_SCENARIO.md) | مثال عملي كامل بشركة "النور" (4 فروع، 12 موظف، 15 سيناريو) |
| [`HR_MEETING_QUESTIONS.md`](./HR_MEETING_QUESTIONS.md) | 46 سؤال لميتنج Discovery مع HR Managers |
| [`FOLDER_STRUCTURE.md`](./FOLDER_STRUCTURE.md) | هيكل المشروع المقترح في Next.js |
| [`DEVELOPMENT_WORKFLOW.md`](./DEVELOPMENT_WORKFLOW.md) | Playbook يومي للتطوير (Phases + Feature Loop) |

---

## 🎯 Project Status

**Phase**: Documentation & Planning  
**Next**: Phase 1 — Organization Structure Implementation

---

## 🛠️ Tech Stack

- **Framework**: Next.js 15 (App Router)
- **Language**: TypeScript
- **Database**: PostgreSQL
- **ORM**: Drizzle (recommended)
- **Validation**: Zod
- **UI**: shadcn/ui + Tailwind CSS

---

## 🧭 ابدأ من هنا

1. اقرأ [`AUTH_ARCHITECTURE.md`](./AUTH_ARCHITECTURE.md) — قسم **Core Architecture Principles** و **Build Order**
2. شوف الـ Diagrams في [`SYSTEM_DIAGRAMS.md`](./SYSTEM_DIAGRAMS.md) للفهم البصري
3. اقرأ [`EXAMPLE_SCENARIO.md`](./EXAMPLE_SCENARIO.md) لتشوف ازاي النظام بيشتغل بمثال واقعي
4. لما تيجي تكتب كود، افتح [`DEVELOPMENT_WORKFLOW.md`](./DEVELOPMENT_WORKFLOW.md) جنبك

---

## 📐 Core Principles

```
┌─────────────────────────────────────┐
│  Phase 1: Organization Structure    │  ← Branches, Departments, Teams
├─────────────────────────────────────┤
│  Phase 2: Employees (HR Core)       │  ← الموظفين بدون Login لسه
├─────────────────────────────────────┤
│  Phase 3: Users + RBAC + Scopes     │  ← Auth Foundation
└─────────────────────────────────────┘
            ↓
   من هنا، كل Feature بتيجي مع Auth
   (Permissions + Scopes) من اليوم الأول
```

---

## 🔑 Key Concepts

- **Employee 1 ── 0..1 User** — الموظف ممكن يكون عنده Login، وممكن لأ
- **Scope على Assignment مش على Role** — نفس Role لـ users كتير بـ scopes مختلفة
- **5 Scope Types**: `company`, `branch`, `department`, `team`, `employee`
- **Authorization = Permission Check + Scope Check** — الاتنين لازم ينجحوا

---

*Generated with documentation collaboration via Claude Code.*
