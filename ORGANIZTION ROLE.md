# HRMS Authorization & Scope System

# Purpose

This document defines the authorization architecture for the HRMS platform.

The system is designed to support:
- Multi-branch organizations
- Cross-branch employee assignments
- Department-based management
- Hierarchical organizational access
- Enterprise-grade scalability
- Dynamic permissions
- Scope-based visibility

This architecture avoids hardcoded authorization logic and ensures future scalability.

---

# Core Authorization Philosophy

The system uses:

```text
RBAC + Organizational Scopes + Hierarchy Policies
```

Authorization is NOT based only on roles.

Access decisions are based on:

```text
ROLE + PERMISSION + SCOPE + POLICY
```

---

# Important Principle

## Permissions Define Actions

Examples:
- employee.read
- employee.update
- payroll.view
- attendance.manage
- leave.approve

Permissions answer:

```text
What can the user do?
```

---

# Scopes Define Boundaries

Examples:
- BRANCH:Cairo
- BRANCH:Alex
- COMPANY:ALL
- DEPARTMENT:Finance

Scopes answer:

```text
Where can the user perform the action?
```

---

# Policies Define Rules

Policies answer:

```text
Under which conditions is access allowed?
```

Examples:
- Same branch
- Same department
- Manager hierarchy
- Company-wide visibility

---

# Enterprise Authorization Formula

```text
Can USER perform ACTION
on RESOURCE
within ORGANIZATIONAL SCOPE ?
```

---

# Organizational Hierarchy

The organization structure is hierarchical.

Example:

```text
Company
 └── Branch
      └── Department
           └── Position
                └── Employee
```

Future hierarchy support:
- Regions
- Divisions
- Subsidiaries
- Business Units

---

# Generic Organizational Unit Model

All hierarchy entities should use a generic organizational unit structure.

Table:
- organizational_units

Example:

| id | name       | type       | parent_id |
|----|------------|------------|------------|
| 1  | Main Co    | COMPANY    | null       |
| 2  | Cairo      | BRANCH     | 1          |
| 3  | HR         | DEPARTMENT | 2          |

Benefits:
- Infinite scalability
- Dynamic hierarchy
- No redesign needed later

---

# Role Design

Roles must remain generic.

GOOD:
- HR_MANAGER
- DEPARTMENT_MANAGER
- HEAD_OF_HR

BAD:
- Cairo_HR_Manager
- Alex_HR_Manager

Branch-specific logic belongs to scopes, NOT roles.

---

# Scope Design

Scopes determine data visibility.

Examples:

| User | Role | Scope |
|------|------|--------|
| Ahmed | HR_MANAGER | Cairo |
| Ahmed | HR_MANAGER | Alex |
| Sara | HR_MANAGER | Giza |
| Omar | HEAD_OF_HR | ALL |

---

# Real Enterprise Example

## Ahmed

Role:
```text
HR_MANAGER
```

Scopes:
```text
BRANCH:Cairo
BRANCH:Alex
```

Ahmed can:
- View Cairo employees
- Manage Alex employees

Ahmed cannot:
- Access Giza branch employees

---

# Future Scalability Example

## Head of HR

Role:
```text
HEAD_OF_HR
```

Scope:
```text
COMPANY:ALL
```

This automatically grants:
- Organization-wide access
- All branch visibility
- Global HR operations

No code changes are required.

---

# Feature Access vs Data Access

This is one of the most important architectural concepts.

---

# Feature Access

Determines:
```text
Can user access this page or endpoint?
```

Controlled by:
- permissions

Example:
```text
employee.read
```

---

# Data Access

Determines:
```text
Which records can the user see?
```

Controlled by:
- scopes
- policies

---

# Example

Page:
```text
Employees List
```

All HR managers can open the page.

However:
- Cairo HR sees Cairo employees
- Alex HR sees Alex employees
- Head HR sees all employees

Same page.
Same endpoint.
Different filtered data.

---

# Authorization Pipeline

The system should evaluate authorization in this order:

```text
1. Authenticate User
2. Load Roles
3. Load Permissions
4. Load Organizational Scopes
5. Check Permission
6. Apply Scope Filters
7. Execute Query
8. Return Authorized Data
```

---

# Row-Level Security

The system uses row-level authorization filtering.

Example:

```sql
SELECT *
FROM employees
WHERE branch_id IN (user.allowed_branches)
```

This ensures users only access authorized data.

---

# Backend Enforcement

Frontend visibility is NOT security.

Frontend may:
- hide pages
- hide buttons
- hide actions

However:
- backend MUST always enforce authorization

---

# Example UI Authorization

Permission:
```text
employee.update
```

Frontend:
- show edit button only if permission exists

Backend:
- validate permission again
- validate scope ownership
- validate organizational access

---

# Scope-Aware Services

All major services must support authorization filtering.

Examples:
- EmployeeService
- PayrollService
- AttendanceService
- LeaveService

Every query must apply:
- permission validation
- scope filtering
- policy checks

---

# Recommended Scope Types

Supported scope categories:

| Scope Type | Example |
|------------|----------|
| COMPANY | ALL |
| BRANCH | Cairo |
| DEPARTMENT | HR |
| TEAM | Recruitment |
| EMPLOYEE | Self |

---

# Future Authorization Features

The architecture should support future advanced access models.

Examples:
- Self access
- Direct reports only
- Regional management
- Temporary delegation
- Acting manager roles
- Matrix organizations

---

# Self Scope Example

Permission:
```text
employee.read.self
```

Behavior:
- Employee only sees own profile

---

# Team Scope Example

Manager sees:
- direct reports only

---

# Hierarchical Scope Example

Regional manager sees:
- all child branches
- all nested departments

---

# Recommended Database Tables

Core authorization tables:

```text
roles
permissions
role_permissions
user_roles
role_scopes
organizational_units
employees
employee_assignments
```

---

# Recommended Permission Naming

Use:

```text
resource.action
```

Examples:
- employee.read
- employee.update
- payroll.view
- attendance.manage
- leave.approve

---

# Golden Rules

## 1. Never Hardcode Authorization

BAD:
```js
if (user.role === "Cairo_HR_Manager")
```

GOOD:
```js
if (user.hasPermission("employee.read"))
```

Then apply scope filtering separately.

---

## 2. Separate Roles from Organizational Boundaries

Roles define:
```text
WHAT
```

Scopes define:
```text
WHERE
```

---

## 3. Treat Permissions as Data

Permissions should:
- live in database
- be configurable
- support dynamic assignment

NOT hardcoded in application logic.

---

# Final Goal

Build a scalable authorization system capable of supporting:
- Small businesses
- Multi-branch organizations
- Enterprise hierarchies
- Regional structures
- Future organizational growth

without requiring authorization redesign.
