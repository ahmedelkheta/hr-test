# HRMS Enterprise Architecture

# Overview

This HRMS system is designed as a scalable enterprise-grade platform supporting:

- Multi-branch organizations
- Department hierarchies
- Employee lifecycle management
- Scope-based RBAC authorization
- Organizational hierarchy permissions
- Reporting and analytics
- Approval workflows
- Future scalability

The architecture is designed to support:
- Multiple branches
- Multiple departments
- Employees working across branches
- Hierarchical management structures
- Cross-branch roles
- Future expansion without redesign

---

# Core Principles

## 1. Roles Are Global

Roles define WHAT a user can do.

Examples:
- HR_MANAGER
- DEPARTMENT_MANAGER
- EMPLOYEE
- CEO
- HEAD_OF_HR

Roles NEVER contain branch names.

BAD:
- Cairo_HR_Manager

GOOD:
- HR_MANAGER

---

## 2. Scopes Define WHERE Access Applies

Scopes determine organizational boundaries.

Examples:
- Branch: Cairo
- Branch: Alexandria
- Department: Finance
- Company: ALL

Access is always evaluated using:

```text
ROLE + SCOPE + RESOURCE
```

---

## 3. Organizational Hierarchy Is Dynamic

The organization structure must support future expansion.

Hierarchy Example:

```text
Company
 └── Branch
      └── Department
           └── Position
                └── Employee
```

Future hierarchy support:
- Regions
- Countries
- Subsidiaries
- Business Units
- Divisions

---

# Organizational Structure

## Organizational Units

All hierarchy entities are stored generically.

Table:
- organizational_units

Types:
- COMPANY
- BRANCH
- DEPARTMENT
- DIVISION
- REGION

Example:

| id | name       | type       | parent_id |
|----|------------|------------|------------|
| 1  | Main Co    | COMPANY    | null       |
| 2  | Cairo      | BRANCH     | 1          |
| 3  | HR         | DEPARTMENT | 2          |

---

# Employee Structure

Employees may belong to:
- Multiple branches
- Multiple departments
- Multiple positions

This supports:
- Matrix organizations
- Cross-branch assignments
- Temporary assignments
- Shared management roles

---

# Employee Assignment Model

Example:

| Employee | Branch | Department | Position |
|----------|---------|------------|----------|
| Ahmed    | Cairo   | HR         | Manager  |
| Ahmed    | Alex    | HR         | Manager  |

---

# Authentication vs Authorization

Authentication:
- Identifies the user

Authorization:
- Determines what the user can access

These systems must remain separated.

---

# User vs Employee Separation

User:
- Login identity

Employee:
- HR profile

Reason:
- Some employees may not log in
- External users may exist
- Service accounts may exist

---

# RBAC Architecture

## Access Formula

```text
Permission =
Role Permissions
+
Organizational Scope
+
Hierarchy Rules
```

---

# Roles

Roles define capabilities.

Examples:
- HR_MANAGER
- HEAD_OF_HR
- DEPARTMENT_MANAGER
- PAYROLL_MANAGER
- EMPLOYEE

---

# Permissions

Permissions use:

```text
resource.action
```

Examples:
- employee.read
- employee.update
- payroll.view
- leave.approve
- attendance.manage

---

# Scopes

Scopes define organizational boundaries.

Examples:
- BRANCH:Cairo
- BRANCH:Alex
- COMPANY:ALL
- DEPARTMENT:Finance

---

# Example Authorization

Ahmed:
- Role: HR_MANAGER
- Scope: Cairo, Alex

Ahmed can:
- View employees in Cairo
- View employees in Alex

Ahmed cannot:
- Access Giza employees

---

# Future Scalability Example

Future Role:
- HEAD_OF_HR

Scope:
- COMPANY:ALL

This automatically grants organization-wide visibility without changing business logic.

---

# Authorization Rules

The system must evaluate:

```text
Can USER perform ACTION on RESOURCE within SCOPE?
```

Example:

```text
Can Ahmed edit employee profile for employee in Cairo branch?
```

---

# Database Design

## Core Tables

### users

Authentication identities.

### employees

HR employee profiles.

### organizational_units

Organization hierarchy.

### positions

Employee positions.

### employee_assignments

Employee organizational assignments.

### roles

System roles.

### permissions

System permissions.

### role_permissions

Role-permission mappings.

### user_roles

User-role assignments.

### role_scopes

Scope assignments.

### audit_logs

System activity tracking.

---

# Audit Logging

Every important action must be tracked.

Track:
- Who performed action
- Old value
- New value
- Timestamp
- IP address
- Entity affected

Audit logs are mandatory for enterprise HR systems.

---

# Soft Delete Strategy

HR data must never be physically deleted.

Use:
- deleted_at
- deleted_by

Reason:
- Compliance
- Auditing
- Recovery
- Reporting integrity

---

# Approval Workflow Engine

The system should support:
- Leave approvals
- Salary approvals
- Transfers
- Promotions
- Recruitment approvals

Approval flows should be configurable.

---

# Statistics & Reporting

The architecture must support:
- Workforce analytics
- Branch statistics
- Attendance analytics
- Payroll analytics
- Department KPIs
- Organization growth metrics

Reporting should use:
- Aggregated queries
- Materialized views
- Reporting services if needed

---

# Recommended Architecture Style

Initial architecture:

```text
Modular Monolith
```

Reason:
- Easier development
- Easier transactions
- Easier reporting
- Lower complexity

Future migration:
- Microservices if required

---

# Suggested Technology Stack

## Backend
- NestJS
OR
- ASP.NET Core

## Database
- PostgreSQL

## ORM
- Prisma
OR
- TypeORM

## Authentication
- JWT
- Refresh Tokens
- Scope-based authorization middleware

---

# Golden Rules

## 1. Never Hardcode Organizational Roles

BAD:
- Cairo_HR_Manager

GOOD:
- HR_MANAGER + Scope(Cairo)

---

## 2. Treat Permissions as Data

Permissions must be:
- Configurable
- Dynamic
- Database-driven

NOT hardcoded in application logic.

---

## 3. Build Generic Authorization

Future requirements may include:
- Regional managers
- Cross-branch supervision
- Temporary delegation
- Acting managers
- Matrix management

Authorization must support expansion without redesign.

---

# Final Goal

Build a scalable HRMS platform capable of supporting:
- Small companies
- Multi-branch enterprises
- Regional organizations
- Enterprise hierarchy structures

without requiring major architectural changes.
