# 🛠️ Proposed Database Refactoring Plan

This document outlines the proposed adjustments and enhancements to the database schema for the SEAIT Enrollment Management System (EMS). These recommendations address logical issues, security concerns, and normalization options. 

*No changes have been made to your original `ems_schema.sql` file yet.*

---

## 1. Unified Accounts Table (Authentication Refactoring)

### Current Design
Currently, login credentials are split across two tables:
- `Students` has: `username`, `passwordHash`, `status`
- `StaffUsers` has: `username`, `passwordHash`, `status`

### The Issue
This structure can lead to username collisions (e.g., a student and a staff user having the same username). It also makes authentication code redundant, as the login API has to query two different tables to verify credentials.

### Proposed Solution
Introduce a unified `Accounts` table that serves as the single source of authentication, and link `Students` and `StaffUsers` to it.

```sql
-- 1. Create the unified Accounts table
CREATE TABLE Accounts (
  accountId INT AUTO_INCREMENT,
  username VARCHAR(150) NOT NULL,
  passwordHash VARCHAR(150) NOT NULL,
  role ENUM('student', 'staff', 'officeHead', 'dean', 'programHead', 'admin') NOT NULL,
  status ENUM('active', 'inactive') NOT NULL,
  PRIMARY KEY (accountId),
  CONSTRAINT uq_accounts_username UNIQUE (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 2. Modify Students table definition
-- Remove username, passwordHash, status columns from Students
-- Add accountId INT NULL reference
ALTER TABLE Students DROP COLUMN username;
ALTER TABLE Students DROP COLUMN passwordHash;
ALTER TABLE Students DROP COLUMN status;
ALTER TABLE Students ADD COLUMN accountId INT NULL;
ALTER TABLE Students ADD CONSTRAINT fk_students_account FOREIGN KEY (accountId) REFERENCES Accounts(accountId) ON DELETE SET NULL ON UPDATE CASCADE;

-- 3. Modify StaffUsers table definition
-- Remove username, passwordHash, status columns from StaffUsers
-- Add accountId INT NULL reference
ALTER TABLE StaffUsers DROP COLUMN username;
ALTER TABLE StaffUsers DROP COLUMN passwordHash;
ALTER TABLE StaffUsers DROP COLUMN status;
ALTER TABLE StaffUsers ADD COLUMN accountId INT NULL;
ALTER TABLE StaffUsers ADD CONSTRAINT fk_staffusers_account FOREIGN KEY (accountId) REFERENCES Accounts(accountId) ON DELETE SET NULL ON UPDATE CASCADE;
```

---

## 2. Relax `EnrolledSubjects.grade` Nullability

### Current Design
`EnrolledSubjects.grade` is defined as `DECIMAL(3,2) NOT NULL`.

### The Issue
During the initial phases of enrollment (proposed and confirmed), students have not yet taken or finished the subject, so no grade exists. Forcing this field to be `NOT NULL` requires frontends/APIs to insert a dummy/placeholder value (like `0.00`), which looks like a failure and risks corrupting GPA/academic standing calculations.

### Proposed Solution
Make the grade column nullable and default to `NULL`:
```sql
ALTER TABLE EnrolledSubjects MODIFY grade DECIMAL(3,2) NULL DEFAULT NULL;
```

---

## 3. Enforce Uniqueness on Crucial Identifiers

### Current Design
Important identity columns like `Students.schoolIdNumber` and `StaffUsers.employeeNo` are marked `NOT NULL` but lack `UNIQUE` constraints.

### The Issue
Duplicate student IDs (`schoolIdNumber`) or employee numbers (`employeeNo`) could accidentally be registered in the database, breaking record integrity and student tracking.

### Proposed Solution
Add explicit unique constraints:
```sql
ALTER TABLE Students ADD CONSTRAINT uq_students_schoolid UNIQUE (schoolIdNumber);
ALTER TABLE StaffUsers ADD CONSTRAINT uq_staff_employeeno UNIQUE (employeeNo);
```

---

## 4. Performance Indexes for Query Optimization

### Current Design
The schema relies heavily on primary keys and foreign keys. MySQL automatically indexes these, but high-frequency search and filter columns have no secondary indexes.

### Proposed Solution
Add non-unique indexes to accelerate database search operations as data scales:
```sql
-- Speed up searching for students by name
CREATE INDEX idx_students_names ON Students (lastName, firstName);

-- Speed up cashier/accounting lookup by Official Receipt number
CREATE INDEX idx_payments_or ON Payments (orNumber);

-- Speed up checking subject assignments in block scheduling
CREATE INDEX idx_schedules_lookup ON Schedules (sectionId, subjectId);
```

---

## 5. Automated Clearance Status Verification (Drift Prevention)

### Current Design
`StudentClearances.overallStatus` is stored as an ENUM column (`'pending'`, `'approved'`, etc.) in the database, alongside individual status entries in `ClearanceApprovals`.

### The Issue
Storing both is redundant and creates database state drift. If a program head revokes a clearance in `ClearanceApprovals` but the system fails to update the overall status column, a student might incorrectly show as fully cleared.

### Proposed Solution
Make the clearance evaluation dynamic using a SQL `VIEW` instead of a hardcoded column:
```sql
CREATE VIEW View_StudentClearanceStatus AS
SELECT 
    sc.studentClearanceId,
    sc.studentId,
    sc.clearancePeriodId,
    CASE 
        WHEN NOT EXISTS (
            SELECT 1 FROM ClearanceApprovals ca 
            WHERE ca.studentClearanceId = sc.studentClearanceId 
            AND ca.status != 'approved'
        ) THEN 'approved'
        ELSE 'pending'
    END AS overallStatus
FROM StudentClearances sc;
```

---

## 6. What about the `Notifications` Table?

### Assessment
- **Necessity:** In a face-to-face first digital enrollment system, the `Notifications` table is **not strictly necessary**. Students receive updates, sign forms, and get physical copies in person.
- **Recommendation:** Keep the schema clean for the MVP by dropping the `Notifications` table. Communication logs can be implemented later as a separate module when online channels (portal, SMS integrations) are introduced.
