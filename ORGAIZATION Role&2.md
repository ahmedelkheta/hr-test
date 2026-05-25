# HRMS Build Order & Identity Architecture

# Purpose

This document defines the recommended implementation order and identity architecture for the HRMS platform.

The goal is to build:
- A scalable enterprise HRMS
- A reusable authorization engine
- Flexible employee management
- Multi-branch organizational support
- Scope-based access control
- Future-proof system architecture

---

# Recommended Development Order

The system should be built in layers.

Correct order:

```text
1. Organizational Structure
2. Employee Profiles
3. Authentication System
4. Roles & Permissions
5. Scope System
6. User Management
7. Employee ↔ User Linking
8. Authorization Engine
9. HR Modules
10. Workflows & Reporting
```

This order prevents architectural redesign later.

---

# Step 1 — Organizational Structure

Build the company hierarchy first.

Core entities:
- Company
- Branches
- Departments
- Positions

Hierarchy Example:

```text
Company
 └── Branch
      └── Department
           └── Position
```

Everything in the HRMS depends on this structure.

---

# Organizational Unit Strategy

Use a generic hierarchy table.

Recommended table:
- organizational_units

Example:

| id | name       | type       | parent_id |
|----|------------|------------|------------|
| 1  | Main Co    | COMPANY    | null       |
| 2  | Cairo      | BRANCH     | 1          |
| 3  | HR         | DEPARTMENT | 2          |

Benefits:
- Infinite scalability
- Flexible hierarchy
- Future organizational expansion

---

# Step 2 — Employee Profiles

After organizational structure is ready, build employee profiles.

Employee profiles should contain:
- Personal data
- Contact information
- Contracts
- Salary information
- Attendance settings
- Documents
- Organizational assignments
- Reporting managers

At this stage:
- Employees DO NOT need login accounts yet

This is intentional.

---

# Important Principle

Employee ≠ User

These are separate concepts.

---

# Employee

Represents:
- HR entity
- Organizational identity
- Workforce member

Examples:
- Office worker
- Factory worker
- Archived employee
- Temporary staff

Employees may exist without system login access.

---

# User

Represents:
- Authentication identity
- Login account
- System access

Examples:
- Email
- Password
- MFA
- Security credentials

Users control access to the application.

---

# Why Separation Matters

Enterprise systems must support:
- Employees without login accounts
- External users
- Service accounts
- Contractors
- Disabled accounts
- Archived employees

This is why:
```text
Employee != User
```

---

# Correct Enterprise Relationship

```text
Employee
    may have
        ↓
      User Account
        ↓
      Roles
        ↓
      Scopes
```

NOT:

```text
User = Employee
```

---

# Step 3 — Authentication System

Build the authentication layer.

Recommended features:
- Email login
- Password hashing
- JWT authentication
- Refresh tokens
- MFA readiness
- Session tracking

Recommended table:
- users

Example fields:
- id
- email
- password_hash
- is_active
- last_login_at

---

# Step 4 — Roles & Permissions

Build authorization capabilities.

Roles define:
```text
WHAT a user can do
```

Examples:
- HR_MANAGER
- EMPLOYEE
- DEPARTMENT_MANAGER
- HEAD_OF_HR

Permissions define:
```text
Specific actions
```

Examples:
- employee.read
- employee.update
- payroll.view
- attendance.manage

---

# Step 5 — Scope System

Scopes define:
```text
WHERE permissions apply
```

Examples:
- BRANCH:Cairo
- BRANCH:Alex
- COMPANY:ALL

This enables:
- Multi-branch organizations
- Department-level visibility
- Hierarchical access control

---

# Example Authorization

Ahmed:
- Role: HR_MANAGER
- Scope: Cairo

Ahmed can:
- Access Cairo employees

Ahmed cannot:
- Access Giza employees

---

# Step 6 — User Management Module

Build a dedicated Users Management page.

Purpose:
- Create users
- Assign roles
- Assign scopes
- Activate/deactivate accounts
- Reset passwords
- Link employees to users

This page manages:
```text
System Access
```

NOT employee HR data.

---

# Step 7 — Employee ↔ User Linking

Link employee profiles to login accounts.

Example:

| Employee | User |
|----------|------|
| Ahmed HR | ahmed@company.com |

This enables:
- Employee login
- Personalized access
- Role-based authorization

---

# Example Real Workflow

## Step 1

HR creates employee:

```text
Ahmed Hassan
```

Employee exists in HRMS.

No login account yet.

---

# Step 2

Administrator creates user:

```text
ahmed@company.com
```

Assigns:
- Role: HR_MANAGER
- Scope: Cairo

Links:
```text
employee_id ↔ user_id
```

Now Ahmed can log in.

---

# Step 8 — Authorization Engine

Build centralized authorization logic.

Authorization evaluates:

```text
Permission
+
Scope
+
Hierarchy Rules
```

---

# Authorization Flow

```text
Authenticate User
        ↓
Load Roles
        ↓
Load Permissions
        ↓
Load Scopes
        ↓
Check Permission
        ↓
Apply Scope Filtering
        ↓
Return Authorized Data
```

---

# Scope-Based Data Filtering

The same page should behave differently depending on scopes.

Example:
```text
Employees Page
```

Cairo HR sees:
- Cairo employees only

Head HR sees:
- All employees

Same page.
Same endpoint.
Different filtered data.

---

# Row-Level Security

Queries should automatically apply scope filtering.

Example:

```sql
SELECT *
FROM employees
WHERE branch_id IN (allowed_branches)
```

---

# Important Enterprise Principle

Feature Access != Data Access

---

# Feature Access

Controls:
```text
Can user access this page?
```

Using:
- permissions

---

# Data Access

Controls:
```text
Which records can user see?
```

Using:
- scopes
- hierarchy
- policies

---

# Example UI Authorization

Permission:
```text
employee.update
```

Frontend:
- hide edit button if missing permission

Backend:
- validate permission again
- validate organizational scope
- validate hierarchy rules

Backend authorization is mandatory.

---

# Step 9 — HR Modules

After authorization is stable, build:
- Attendance
- Payroll
- Leaves
- Recruitment
- Assets
- Performance
- Evaluations

All modules reuse:
- roles
- permissions
- scopes
- hierarchy logic

---

# Step 10 — Workflows & Reporting

Build:
- Approval workflows
- Dashboards
- Statistics
- Reporting
- Analytics

Examples:
- Leave approvals
- Promotion approvals
- Transfer approvals
- Salary approvals

---

# Reporting Benefits

Because authorization is centralized:
- dashboards automatically filter data
- statistics become scope-aware
- branch managers see branch data only
- executives see organization-wide data

---

# Recommended Core Tables

```text
organizational_units
positions
employees
employee_assignments

users
roles
permissions
role_permissions
user_roles
role_scopes

audit_logs
approval_workflows
```

---

# Golden Rules

## 1. Never Hardcode Branch Roles

BAD:
```text
Cairo_HR_Manager
```

GOOD:
```text
Role = HR_MANAGER
Scope = Cairo
```

---

# 2. Permissions Define Actions

Examples:
- employee.read
- employee.update

---

# 3. Scopes Define Boundaries

Examples:
- Cairo
- Alex
- All Company

---

# 4. Treat Permissions as Data

Permissions must:
- live in database
- be configurable
- support dynamic assignment

NOT hardcoded in source code.

---

# Final Goal

Build a scalable enterprise HRMS capable of supporting:
- Multi-branch organizations
- Cross-branch management
- Enterprise hierarchies
- Dynamic permissions
- Future organizational growth

without requiring architectural redesign.
