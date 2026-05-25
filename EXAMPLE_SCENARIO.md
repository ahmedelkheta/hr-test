<div dir="rtl">

# 🏢 مثال عملي كامل: شركة "النور" للتدريب

> **الهدف**: شركة وهمية بكل التفاصيل لتشوف بعينك ازاي النظام بيشتغل، مين بيشوف مين، فين، وامتى.  
> الأرقام والـ IDs **حقيقية** في المثال ده — تقدر تحاكيها في الـ DB بتاعتك بالظبط.

---

## 🌟 الشركة في سطر واحد

**شركة النور للتدريب والتطوير** — 4 فروع، 28 قسم، 4 فرق Cross-Branch، 12 موظف نموذجي.

---

## 1️⃣ الفروع (4 Branches)

```sql
INSERT INTO branches (id, name, code, region) VALUES
  (1, 'المقر الرئيسي - القاهرة', 'BR-CAI', 'القاهرة الكبرى'),
  (2, 'فرع الإسكندرية',          'BR-ALX', 'الإسكندرية'),
  (3, 'فرع كفر الشيخ',           'BR-KFS', 'دلتا غرب'),
  (4, 'فرع طنطا',                'BR-TNT', 'دلتا وسط');
```

---

## 2️⃣ الأقسام (Departments - بتخص فرع)

كل فرع عنده 7 أقسام. هنركز على الأقسام اللي هنحتاجها:

```sql
-- HR Departments
INSERT INTO departments (id, branch_id, name, code) VALUES
  (1, 1, 'HR Cairo',         'DEPT-CAI-HR'),
  (2, 2, 'HR Alexandria',    'DEPT-ALX-HR'),
  (3, 3, 'HR Kafr Elsheikh', 'DEPT-KFS-HR'),
  (4, 4, 'HR Tanta',         'DEPT-TNT-HR');

-- Sales Departments
INSERT INTO departments (id, branch_id, name, code) VALUES
  (5, 1, 'Sales Cairo',         'DEPT-CAI-SLS'),
  (6, 2, 'Sales Alexandria',    'DEPT-ALX-SLS'),
  (7, 3, 'Sales Kafr Elsheikh', 'DEPT-KFS-SLS'),
  (8, 4, 'Sales Tanta',         'DEPT-TNT-SLS');

-- IT Departments
INSERT INTO departments (id, branch_id, name, code) VALUES
  (9,  1, 'IT Cairo',         'DEPT-CAI-IT'),
  (10, 2, 'IT Alexandria',    'DEPT-ALX-IT'),
  (11, 3, 'IT Kafr Elsheikh', 'DEPT-KFS-IT'),
  (12, 4, 'IT Tanta',         'DEPT-TNT-IT');

-- وهكذا للأقسام التانية: Marketing, Finance, Operations, Support
```

> 💡 **لاحظ**: "Sales Alexandria" قسم مختلف عن "Sales Kafr Elsheikh". كل فرع له أقسامه المستقلة.

---

## 3️⃣ الفرق (Teams - بتخترق الفروع) ⭐

```sql
INSERT INTO teams (id, name, code) VALUES
  (1, 'Egypt Sales Squad',     'TEAM-EG-SALES'),  -- بيخترق كل فروع Sales
  (2, 'Digital Marketing Team','TEAM-DIG-MKT'),   -- موظفين من فروع مختلفة
  (3, 'Trainers Pool',         'TEAM-TRAINERS'),  -- مدربين Cross-branch
  (4, 'Q1 Product Launch',     'TEAM-Q1-LAUNCH'); -- مؤقت لمدة 3 شهور
```

---

## 4️⃣ الـ Roles المعتمدة

```sql
INSERT INTO roles (id, code, name) VALUES
  (1, 'super_admin',     'مدير النظام الأعلى'),
  (2, 'hr_director',     'مدير HR التنفيذي'),
  (3, 'hr_manager',      'مدير HR'),
  (4, 'hr_specialist',   'أخصائي HR'),
  (5, 'sales_director',  'مدير Sales التنفيذي'),
  (6, 'branch_manager',  'مدير فرع'),
  (7, 'dept_manager',    'مدير قسم'),
  (8, 'employee',        'موظف');
```

---

## 5️⃣ الموظفين النموذجيين (12 موظف)

دول هم اللي هنبني عليهم كل السيناريوهات:

| ID | الاسم | الفرع | القسم | المنصب | عنده Login؟ |
|----|------|------|------|-------|------------|
| 1  | **يوسف الشيخ**     | القاهرة      | Executive       | CEO              | ✅ |
| 2  | **منار حسام**      | القاهرة      | HR Cairo (1)    | HR Director      | ✅ |
| 3  | **أحمد سعيد**      | الإسكندرية   | HR Alex (2)     | HR Manager       | ✅ |
| 4  | **محمد طارق**      | كفر الشيخ    | HR Kafr (3)     | HR Manager       | ✅ |
| 5  | **سلمى العزب**     | الإسكندرية   | Sales Alex (6)  | Sales Director   | ✅ |
| 6  | **خالد فؤاد**      | طنطا         | Executive Tanta | Branch Manager   | ✅ |
| 7  | **منى عبد الله**   | الإسكندرية   | HR Alex (2)     | HR Specialist    | ✅ |
| 8  | **نهى السيد**      | كفر الشيخ    | Sales Kafr (7)  | Sales Dept Mgr   | ✅ |
| 9  | **عمر إبراهيم**    | طنطا         | IT Tanta (12)   | IT Engineer      | ✅ |
| 10 | **علي رضا**        | كفر الشيخ    | Sales Kafr (7)  | Sales Rep        | ✅ |
| 11 | **هند مصطفى**      | الإسكندرية   | Sales Alex (6)  | Sales Rep        | ✅ |
| 12 | **عبد الرحمن أنور**| طنطا         | Operations Tanta| Office Boy       | ❌ |

> 💡 **عبد الرحمن** موجود في `employees` بـ `user_id = NULL`. مفيش له User حساب للـ Login.

---

## 6️⃣ Visual Org Chart

```
                  ╔══════════════════════════════╗
                  ║   شركة النور للتدريب         ║
                  ║   CEO: يوسف الشيخ            ║
                  ╚══════════════╤═══════════════╝
                                 │
              ┌──────────────────┼──────────────────────┐
              │                  │                       │
        ╔═════▼═══════╗   ╔══════▼══════════╗   ╔════════▼═════════╗
        ║HR Director  ║   ║Sales Director   ║   ║ (Other Dirs...) ║
        ║منار حسام    ║   ║سلمى العزب       ║   ║                 ║
        ║scope:company║   ║scope:team:1     ║   ║                 ║
        ╚═════╤═══════╝   ╚════════╤════════╝   ╚═════════════════╝
              │                    │
   ┌──────────┼──────────┐         │ بتدير Team عابر فروع
   ▼          ▼          ▼         │
┌────────┐ ┌────────┐ ┌────────┐  │
│HR Alex │ │HR Kafr │ │HR Tanta│  │
│أحمد    │ │محمد    │ │ (vac.)│  │
│scope:  │ │scope:  │ │       │  │
│branch:2│ │branch:3│ │       │  │
└───┬────┘ └───┬────┘ └───────┘  │
    │         │                  │
┌───▼───┐  ┌──▼────┐          ┌──▼──────────────────┐
│منى    │  │نهى    │          │   Egypt Sales Squad │
│Specia │  │Sales  │          │   ─────────────────  │
│list   │  │DeptMgr│          │   • علي (Kafr)      │
│scope: │  │scope: │          │   • هند (Alex)      │
│dept:2 │  │dept:7 │          │   • (آخرين)         │
└───────┘  └───┬───┘          └─────────────────────┘
               │
            ┌──▼───┐
            │علي   │
            │Sales │
            │Rep   │
            │scope:│
            │self  │
            └──────┘

╔════════════════════════════════════════╗
║  فرع طنطا (Branch Manager: خالد فؤاد)  ║
║  scope: branch:4                       ║
║  ─────────────────────                 ║
║  • عمر (IT) — عنده Login              ║
║  • عبد الرحمن (Office Boy) — مفيش Login║
╚════════════════════════════════════════╝
```

---

## 7️⃣ إسناد الأدوار (user_roles)

دلوقتي السحر — كل user_role صف فيه: **مين + إيه الـ Role + إيه الـ Scope**

```sql
INSERT INTO user_roles (user_id, role_id, scope_type, scope_id) VALUES
  -- يوسف (CEO): super_admin على الشركة كلها
  (1,  1, 'company',    NULL),
  
  -- منار (HR Director): hr_director على الشركة كلها
  (2,  2, 'company',    NULL),
  
  -- أحمد (HR Manager Alex): hr_manager على فرع الإسكندرية
  (3,  3, 'branch',     2),
  
  -- محمد (HR Manager Kafr): hr_manager على فرع كفر الشيخ
  --                       ⭐ نفس الـ Role بتاع أحمد، Scope مختلف
  (4,  3, 'branch',     3),
  
  -- سلمى (Sales Director): sales_director على Team:1 (Egypt Sales Squad)
  --                        ⭐ Cross-Branch via Team
  (5,  5, 'team',       1),
  
  -- خالد (Branch Manager Tanta): branch_manager على فرع طنطا
  (6,  6, 'branch',     4),
  
  -- منى (HR Specialist Alex): hr_specialist على قسم HR Alex
  (7,  4, 'department', 2),
  
  -- نهى (Dept Manager Sales Kafr): dept_manager على قسم Sales Kafr
  (8,  7, 'department', 7),
  
  -- عمر (IT Engineer Tanta): employee scope (self only)
  (9,  8, 'employee',   9),
  
  -- علي (Sales Rep Kafr): employee scope (self only)
  (10, 8, 'employee',   10),
  
  -- هند (Sales Rep Alex): employee scope (self only)
  (11, 8, 'employee',   11);

  -- ⚠️ عبد الرحمن مفيش له user_id، فمفيش له user_roles
```

---

## 8️⃣ عضوية الفرق (employee_teams) — هنا الـ Magic ✨

```sql
-- Egypt Sales Squad: علي + هند + آخرين
INSERT INTO employee_teams (employee_id, team_id, role_in_team) VALUES
  (10, 1, 'Member'),  -- علي (Sales Rep Kafr)
  (11, 1, 'Member');  -- هند (Sales Rep Alex)
  -- المثيرة: علي وهند في فروع مختلفة، بس في نفس الـ Team

-- Digital Marketing Team
INSERT INTO employee_teams (employee_id, team_id, role_in_team) VALUES
  (9, 2, 'Tech Support');  -- ⭐ عمر (IT Tanta) في فريق Marketing برضو!
```

> 💡 **لاحظ**: عمر في الأصل IT في طنطا، **بس** هو كمان عضو في Digital Marketing Team. مدير Marketing يقدر يديله مهام Marketing-related، بس مدير الـ HR/Operations في طنطا هو اللي بيدير شغله اليومي.

---

## 9️⃣ مصفوفة "مين يشوف مين" (The Big Matrix)

السؤال الأهم: لو كل واحد دخل على `/employees`، هيشوف مين؟

| User | الـ Role | الـ Scope | بيشوف موظفين... |
|------|---------|----------|-----------------|
| **يوسف** (CEO) | super_admin | company | **الكل** — 12 موظف |
| **منار** (HR Director) | hr_director | company | **الكل** — 12 موظف |
| **أحمد** (HR Mgr Alex) | hr_manager | branch:2 | موظفين الإسكندرية فقط: سلمى، منى، هند |
| **محمد** (HR Mgr Kafr) | hr_manager | branch:3 | موظفين كفر الشيخ فقط: نهى، علي |
| **سلمى** (Sales Director) | sales_director | team:1 | أعضاء Egypt Sales Squad: علي، هند |
| **خالد** (Branch Mgr Tanta) | branch_manager | branch:4 | موظفين طنطا: عمر، عبد الرحمن |
| **منى** (HR Specialist) | hr_specialist | department:2 | موظفين HR Alex: أحمد، منى نفسها |
| **نهى** (Sales Dept Mgr Kafr) | dept_manager | department:7 | موظفين Sales Kafr: علي |
| **عمر** (IT Engineer) | employee | employee:9 | نفسه فقط: عمر |
| **علي** (Sales Rep) | employee | employee:10 | نفسه فقط: علي |
| **هند** (Sales Rep) | employee | employee:11 | نفسها فقط: هند |

---

## 🎯 سيناريوهات حقيقية (مع SQL)

### سيناريو 1: أحمد بيدخل ليشوف الموظفين

```sql
-- query تلقائياً بتـ inject الـ scope:
SELECT e.* FROM employees e
WHERE e.branch_id = 2;  -- ⭐ branch:2 (الإسكندرية)

-- النتيجة:
-- 3. أحمد سعيد (HR Manager)
-- 5. سلمى العزب (Sales Director)
-- 7. منى عبد الله (HR Specialist)
-- 11. هند مصطفى (Sales Rep)
```

> ✅ أحمد بيشوف موظفين الإسكندرية فقط. **مش هيشوف** علي اللي في كفر الشيخ ولا عمر اللي في طنطا.

### سيناريو 2: سلمى بتدخل ليشوف فريقها

```sql
SELECT e.* FROM employees e
WHERE e.id IN (
  SELECT employee_id FROM employee_teams WHERE team_id = 1  -- Egypt Sales Squad
);

-- النتيجة:
-- 10. علي رضا (Kafr) ← من فرع تاني!
-- 11. هند مصطفى (Alex)
```

> 🔥 **هنا الـ Magic**: سلمى مقرها الإسكندرية بس بتشوف علي اللي في كفر الشيخ لأنه عضو في فريقها. **بدون** ما تشوف موظفين الإسكندرية التانيين (زي منى).

### سيناريو 3: علي بيدخل ليشوف بياناته

```sql
SELECT e.* FROM employees e
WHERE e.id = 10;  -- ⭐ scope: employee:10 (هو نفسه)

-- النتيجة:
-- 10. علي رضا
```

> ✅ علي يشوف بياناته بس.

### سيناريو 4: علي طلب أجازة، مين يقدر يوافق؟

الـ Permission المطلوب: `leave.approve`

```sql
-- شيك مين عنده permission ده مع scope يطابق علي
SELECT u.id, u.phone, r.code as role, ur.scope_type, ur.scope_id
FROM user_roles ur
JOIN users u ON u.id = ur.user_id
JOIN roles r ON r.id = ur.role_id
JOIN role_permissions rp ON rp.role_id = r.id
JOIN permissions p ON p.id = rp.permission_id
WHERE p.code = 'leave.approve'
  AND (
       ur.scope_type = 'company'  -- يوسف، منار
    OR (ur.scope_type = 'branch'     AND ur.scope_id = 3)         -- محمد (HR Kafr)
    OR (ur.scope_type = 'department' AND ur.scope_id = 7)         -- نهى (Sales Kafr Mgr)
    OR (ur.scope_type = 'team'       AND ur.scope_id IN (
           SELECT team_id FROM employee_teams WHERE employee_id = 10))  -- سلمى (لو Sales Director عنده leave.approve)
  );
```

**النتيجة المحتملة**:
- ✅ يوسف (company)
- ✅ منار (company - HR Director)
- ✅ محمد (branch:3 - HR Manager Kafr)
- ✅ نهى (department:7 - Sales Kafr Manager)
- ❓ سلمى — يعتمد على policy الشركة: هل Sales Director ليه leave.approve؟ غالباً لأ (Sales مش HR)

> 💡 ده بيوضح إن **أكتر من شخص يقدر يوافق** على نفس العملية بسلوك Multi-Approval — وده مرونة كبيرة.

### سيناريو 5: محمد عايز يعدل في مرتب هند (Sales Rep في الإسكندرية)

```sql
-- شيك الـ Authorization
-- محمد: hr_manager, scope = branch:3 (Kafr)
-- هند: branch_id = 2 (Alex)

-- Query: 
SELECT e.* FROM employees e
WHERE e.id = 11  -- هند
  AND e.branch_id = 3;  -- ⭐ scope: branch:3 (محمد)

-- النتيجة: 0 rows
```

> ❌ محمد **مش هيقدر** يعدل في بيانات هند. الـ Query بترجع 0 rows لأن هند مش في فرعه. **Data Isolation اشتغلت تلقائياً**.

### سيناريو 6: عمر (IT Tanta) في Digital Marketing Team — مين يديره؟

عمر عنده دورين موازيين:
- **شغله الأساسي**: IT Engineer في فرع طنطا (تحت إدارة خالد - Branch Manager)
- **شغل إضافي**: عضو في Digital Marketing Team (تحت إدارة مدير Marketing)

```sql
-- مين يقدر يطلع لعمر مهمة Marketing؟
-- (لو في Marketing Director عنده scope: team:2)
SELECT u.id FROM user_roles ur
WHERE ur.scope_type = 'team' AND ur.scope_id = 2;
-- مدير الـ Digital Marketing Team

-- مين يقدر يوافق على إجازته؟
-- (الـ Branch Manager خالد، أو HR Manager طنطا)
SELECT u.id FROM user_roles ur
JOIN roles r ON r.id = ur.role_id
JOIN role_permissions rp ON rp.role_id = r.id
JOIN permissions p ON p.id = rp.permission_id
WHERE p.code = 'leave.approve'
  AND (
       (ur.scope_type = 'branch'     AND ur.scope_id = 4)  -- branch:4 (Tanta)
    OR (ur.scope_type = 'department' AND ur.scope_id = 12) -- IT Tanta
    OR (ur.scope_type = 'company')
  );
```

> 🔥 **Matrix Management في النظام**: عمر له "مديرين" مختلفين لأغراض مختلفة — Marketing لمهام التسويق، HR/Branch لشؤونه اليومية.

---

## 1️⃣0️⃣ Permission Matrix — مين يقدر يعمل إيه

تخيل عندنا الـ Permissions دي. مين عنده كل واحد؟

| Permission | يوسف | منار | أحمد | محمد | سلمى | خالد | منى | نهى | علي |
|-----------|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
| `employee.read`    | ✅ كل الموظفين | ✅ كل الموظفين | ✅ Alex بس | ✅ Kafr بس | ✅ Sales Squad بس | ✅ Tanta بس | ✅ HR Alex بس | ✅ Sales Kafr بس | ✅ نفسه بس |
| `employee.create`  | ✅ | ✅ | ✅ في Alex | ✅ في Kafr | ❌ | ❌ | ❌ | ❌ | ❌ |
| `employee.delete`  | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `payroll.read`     | ✅ | ✅ | ✅ Alex | ✅ Kafr | ❌ | ❓ | ✅ HR Alex | ❌ | ✅ نفسه |
| `payroll.process`  | ✅ | ✅ | ✅ Alex | ✅ Kafr | ❌ | ❌ | ❌ | ❌ | ❌ |
| `leave.approve`    | ✅ | ✅ | ✅ Alex | ✅ Kafr | ❓ | ✅ Tanta | ❌ | ✅ Sales Kafr | ❌ |
| `sales.report.read`| ✅ | ❌ | ❌ | ❌ | ✅ Squad | ❌ | ❌ | ✅ Kafr | ✅ نفسه |
| `report.financial` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

> 💡 **علامة "✅ Alex بس"**: يعني الـ permission موجود، بس مفلتر بالـ scope.

---

## 1️⃣1️⃣ الـ Insanity (المرونة الجنونية)

### مثال 1: نفس الموظف، 3 مدراء مختلفين

**علي رضا** (Sales Rep في كفر الشيخ، عضو في Egypt Sales Squad):

| الجانب | المسؤول | السبب |
|------|---------|------|
| الحضور والإجازة | محمد (HR Manager Kafr) | scope: branch:3 |
| مهام Sales اليومية | نهى (Sales Dept Mgr Kafr) | scope: department:7 |
| Sales Targets والـ Cross-branch initiatives | سلمى (Sales Director) | scope: team:1 |

نفس الموظف، 3 ناس بيشوفوه بس **لأغراض مختلفة**. ده مينفعش في Role-based فقط بدون Scopes.

### مثال 2: نفس الـ Role، تأثيرات مختلفة

أحمد ومحمد الاتنين `hr_manager`. بس:
- أحمد بياخد سكب الإسكندرية → يدير 4 موظفين
- محمد بياخد سكب كفر الشيخ → يدير 2 موظفين

**نفس Permissions، نتائج مختلفة تماماً**.

### مثال 3: ترقية بدون كسر

لو سلمى اترقت لـ "VP of Sales" وبقت مسؤولة عن كل الـ Sales في الشركة (مش Squad واحد):

```sql
-- بدل user_role القديم
UPDATE user_roles 
SET scope_type = 'company', scope_id = NULL
WHERE user_id = 5 AND role_id = 5;
```

**صف واحد اتغير** = سلمى تشوف الـ Sales في كل الفروع. مفيش refactor، مفيش تعديل كود، مفيش Migrations.

### مثال 4: فتح فرع جديد

افتحنا فرع في **أسيوط** (`branch_id = 5`). محتاجين HR Manager له:

```sql
-- 1. اعمل الـ Branch
INSERT INTO branches VALUES (5, 'فرع أسيوط', 'BR-ASY', 'الصعيد');

-- 2. اعمل HR Department للفرع
INSERT INTO departments VALUES (29, 5, 'HR Asyut', 'DEPT-ASY-HR');

-- 3. اعمل الموظف
INSERT INTO employees (id, ..., branch_id, department_id) 
VALUES (13, '...', 5, 29);

-- 4. اعمل User للموظف
INSERT INTO users VALUES (12, '0100xxx', ...);

-- 5. ⭐ اسند الـ HR Manager Role بـ Scope الفرع الجديد
INSERT INTO user_roles VALUES (12, 3, 'branch', 5);
```

**خلاص**. HR Manager جديد في فرع جديد، **بدون** أي تعديل في:
- جدول `roles` (الـ Role موجود من زمان)
- جدول `permissions` (نفس الـ Permissions)
- الكود (مفيش if/else جديد)

### مثال 5: تكوين Squad مؤقت لمشروع

عايزين فريق لإطلاق منتج جديد، يتكون من 5 ناس من فروع مختلفة:

```sql
-- 1. اعمل الـ Team (لمدة 3 شهور)
INSERT INTO teams VALUES (4, 'Q1 Product Launch', 'TEAM-Q1-LAUNCH');

-- 2. ضم الأعضاء
INSERT INTO employee_teams VALUES 
  (10, 4, 'Sales Lead'),    -- علي من كفر الشيخ
  (11, 4, 'Marketing'),     -- هند من الإسكندرية
  (9,  4, 'Tech Support'),  -- عمر من طنطا
  (3,  4, 'HR Liaison'),    -- أحمد من الإسكندرية
  (7,  4, 'Coordinator');   -- منى من الإسكندرية

-- 3. عيّن Team Lead بـ Scope على الـ Team
INSERT INTO user_roles VALUES (1, 8, 'team', 4);
-- يوسف بقى عنده team_lead role على الـ Team ده

-- 4. بعد 3 شهور، فك الـ Team
UPDATE teams SET is_active = FALSE WHERE id = 4;
-- أو
DELETE FROM employee_teams WHERE team_id = 4;
DELETE FROM user_roles WHERE scope_type = 'team' AND scope_id = 4;
```

**فريق مؤقت اتعمل واتلغي بدون أي حاجة في الكود**.

---

## 1️⃣2️⃣ Edge Cases والمصايد

### المصيدة 1: موظف عنده 3 Roles مختلفة

تخيل **أحمد** (HR Manager Alex) اترقى وبقى:
- HR Manager للإسكندرية (الـ Role القديم)
- + عضو في Trainers Pool (يقدر يدرّب في أي فرع)
- + Project Lead لـ Q1 Product Launch

```sql
INSERT INTO user_roles VALUES
  (3, 3, 'branch',     2),    -- HR Manager Alex
  (3, 9, 'team',       3),    -- Member of Trainers Pool
  (3, 8, 'team',       4);    -- Team Lead for Q1 Launch
```

دلوقتي أحمد عنده 3 صفوف. النظام بيـ UNION الصلاحيات تلقائياً.

### المصيدة 2: Permission من Role لكن Scope من Role تاني

عمر:
- `employee` role بـ scope: employee:9 (نفسه) — له permission `leave.request`
- عضو في Digital Marketing Team

ايه اللي يحصل لو الـ Team Lead عايز يديله مهمة marketing؟
- الـ Team Lead عنده permission على Team:2
- عمر داخل الـ Team:2 (عن طريق `employee_teams`)
- ✅ Team Lead يقدر يطلع له مهمة

> 💡 **الـ Authorization بيحاول كل scope على حدة**. لو واحد منهم يطابق، يـ Allow.

### المصيدة 3: تنازع الصلاحيات

أحمد ومحمد الاتنين عندهم `leave.approve`. لو علي طلب أجازة، مين الـ المسؤول الفعلي؟

**الإجابة**: الاتنين يقدروا يوافقوا — النظام مش بيمنع. لكن الـ Business Logic بتاعتك بتختار:
- Workflow بيوجه الطلب لأقرب مسؤول (مدير القسم → مدير الفرع → HR Director)
- لو محمد وافق، الطلب بيقفل، أحمد ميشوفوش في pending

### المصيدة 4: موظف بدون User

عبد الرحمن (Office Boy):
- موجود في `employees`
- مفيش له `users` record
- HR بتدير بياناته (الحضور، المرتب) **من غير ما يدخل النظام بنفسه**

```sql
-- HR تقدر تشوف عبد الرحمن
SELECT * FROM employees WHERE id = 12;

-- بس عبد الرحمن ميقدرش يعمل login
SELECT * FROM users WHERE phone = '0100xxx';  -- empty
```

### المصيدة 5: لما الموظف يتنقل

علي اتنقل من كفر الشيخ → الإسكندرية:

```sql
-- 1. حدث الـ Employee record
UPDATE employees 
SET branch_id = 2, department_id = 6
WHERE id = 10;

-- 2. لو لسه في Egypt Sales Squad، خلاص هو موجود
-- (الـ team membership مرتبطة بالـ employee_id مش بالـ branch)

-- 3. الـ user_roles بتاع علي ميتغيرش (لسه self scope)
-- لكن لو علي كان branch_manager في كفر الشيخ، لازم تشيله من scope:branch:3
```

> ⚠️ **مهم**: لما تنقل موظف، **راجع** الـ user_roles بتاعه. أحياناً الـ Scope بقى متعلق بمكان قديم.

---

## 1️⃣3️⃣ ملخص الـ Control والمرونة

### اللي تقدر تتحكم فيه بـ **صف واحد** في DB:

| التغيير | الـ Query |
|---------|----------|
| نقل موظف لفرع جديد | `UPDATE employees SET branch_id = ?` |
| ترقية HR Manager إلى Director | `UPDATE user_roles SET scope_type='company'` |
| إضافة Permission جديدة لكل HR Managers | `INSERT INTO role_permissions` |
| فك الإسناد بدون فصل الموظف | `UPDATE user_roles SET revoked_at = NOW()` |
| تعطيل Login مؤقتاً | `UPDATE users SET status='disabled'` |
| ضم موظف لـ Team موجود | `INSERT INTO employee_teams` |
| Multi-branch HR Manager | `INSERT INTO user_roles` (صفين بـ scopes مختلفة) |

### اللي **مش** بتقدر تعمله (والـ المفروض):

- ❌ موظف يشوف موظف تاني خارج Scope بتاعه
- ❌ نفس User يبقى عنده Role متناقض (هـ employee و super_admin مع بعض — تقني ممكن، عملياً لأ)
- ❌ موظف بدون فرع
- ❌ Department بدون فرع (في الـ schema الحالي)
- ❌ Login بدون رقم موبايل
- ❌ كلمة سر plaintext

---

## 1️⃣4️⃣ خلاصة المثال

| المفهوم | في المثال ده |
|--------|--------------|
| **شركة** | النور للتدريب |
| **فروع** | 4 (القاهرة، الإسكندرية، كفر الشيخ، طنطا) |
| **أقسام** | 28 (7 لكل فرع) |
| **فرق Cross-Branch** | 4 (Sales Squad, Digital Marketing, Trainers, Q1 Launch) |
| **موظفين بـ Login** | 11 |
| **موظفين بدون Login** | 1 (عبد الرحمن) |
| **Roles** | 8 |
| **User-Role Assignments** | 11+ صف |
| **سيناريوهات Scope** | 5 أنواع (company, branch, department, team, employee) |
| **مرونة الإدارة** | Matrix Management — موظف بـ 3 مدراء لأغراض مختلفة |

---

## 1️⃣5️⃣ السؤال الذهبي

> **بعد ما تشوف المثال ده، اسأل نفسك**:  
> "لو شركتي بقت 100 موظف، 10 فروع، 50 قسم، 20 فريق Cross-Branch — هل الـ Architecture ده هيشتغل؟"

**الإجابة**: ✅ **بنفس Schema بالظبط**. مفيش حاجة هتتغير. كل اللي هيحصل = صفوف أكتر في الجداول.

ده معنى **Scalable by Design**.

---

> 🎯 **خد الملف ده كمرجع**: لما تيجي تشتغل وتلخبط في "مين بيشوف إيه"، ارجعله. السيناريوهات هنا بتغطي 95% من المواقف اللي هتقابلك.

</div>
