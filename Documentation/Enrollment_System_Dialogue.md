# Enrollment System — A Narrative Dialogue

> *Following Maria Santos, a first-year BS Computer Science student, through the complete enrollment lifecycle — and the database behind every step.*

---

## Preamble: The Students Table

Before Maria sets foot on campus, the system already knows her — at least in potential. The `students` table is the heart of the entire database. Every person in the system has a row here:

| Column | What It Holds |
|---|---|
| `studentId` | The system's internal ID (auto-increment, 1 → 29,548) |
| `schoolIdNumber` | The visible student ID like **2026-0001** (unique) |
| `lastName`, `firstName`, `middleName`, `suffix` | Legal name components |
| `gender`, `birthdate`, `birthplace`, `citizenship` | Demographics |
| `religionId` | Foreign key → `religions` table |
| `contactNumber`, `email` | Communication channels |
| `username`, `passwordHash` | Portal login credentials |
| `status` | `active` or `inactive` |

Think of this table as the canvas — everything else in the system exists *because* of a student row.

---

## Phase 0 — Application: "Maria Decides to Enroll"

It's February 2026. Maria opens the school's online admission portal and fills out the application form. She enters her personal details, her home address, her parent's contact numbers, and her educational history.

### What happens in the database

**Step 1 — Student record creation**

The system first checks if Maria already has a `students` row. This is a new applicant, so it creates one:

```sql
INSERT INTO students (schoolIdNumber, lastName, firstName, ...) 
VALUES ('2026-0001', 'Santos', 'Maria', ...);
```

Now Maria has a `studentId` — let's say **15001**.

**Step 2 — Addresses**

Maria enters three addresses: her home in the province, her current boarding house, and a permanent family address. Each becomes a row in `addresses`, linked back to her:

```
addresses.addressId = 45001 → studentId = 15001 (home)
addresses.addressId = 45002 → studentId = 15001 (current)  
addresses.addressId = 45003 → studentId = 15001 (permanent)
```

The `addressType` column distinguishes them (`home`, `current`, `permanent`). A student can have multiple addresses — this is a **one-to-many** relationship between `students` and `addresses`.

**Step 3 — Guardians**

Maria's mother and father are recorded in `guardians`:

```
guardians.guardianId = 38001 → studentId = 15001 (mother)
guardians.guardianId = 38002 → studentId = 15001 (father)
```

Each guardian has a `relationship` column (`mother`, `father`, `guardian`, `other`), contact info, and importantly `isEmergencyContact` and `isAuthorizedToActOnBehalf` — boolean flags that control pickup permissions and consent.

**Step 4 — Educational Background**

Maria attended STI College for Senior High. The school might already be in `educationalinstitutions` (89 institutions are pre-loaded). If not, the system adds it, then links it to Maria via `studenteducationalbackgrounds`:

```
institution → STI College (institutionId = 45)
background  → studentId = 15001, institutionId = 45, levelCompleted = 'seniorHigh'
```

**Step 5 — The Admission Record**

Finally, the system creates the admission entry:

```
admissions.admissionId = 12000
  → studentId = 15001 (Maria)
  → termId = 1 (AY 2025-2026, 1st Semester)
  → courseId = 5 (BS Computer Science)
  → applicantType = 'firstYear'
  → applicationMode = 'online'
  → admissionStatus = 'pending'
```

The `admissions` table is the bridge between being a *person* (students) and being an *enrollee*. It tracks `admissionStatus` through the pipeline: `pending` → `approved` → `rejected`.

**Key relationships active here:**

```
students ──1:M──→ addresses
students ──1:M──→ guardians
students ──1:M──→ studenteducationalbackgrounds
students ──1:1──→ admissions (per term)
admissions ──M:1──→ courses
admissions ──M:1──→ academicterms
```

---

## Phase 0.5 — Entrance Exam: "The Gatekeeper"

Maria's chosen course, BS Computer Science, requires an entrance exam (`courses.requiresEntranceExam = 1`). The system needs to record her exam outcome before proceeding.

### The examresults table

```
examresults.examId = 4500
  → studentId = 15001
  → courseId = 5 (BS CompSci)
  → termId = 1
  → examStage = 'entrance'
  → examType = 'general'
  → examResult = 'pass'
  → examDate = '2026-02-15'
```

If she had failed, the `admissionStatus` in `admissions` would flip to `rejected`, ending her journey here. But she passes.

**What this tells us:** The `examresults` table separates *admission* from *examination* — a course can opt in to requiring exams independently of the admission flow. This decoupling means the school can add or remove exam requirements without changing the admission pipeline logic.

---

## Phase 1 — Academic Department Evaluation: "The Dean Approves"

Maria's application moves to the College of Computer Studies. The Program Head reviews her credentials.

### Database operation

The admissions record is updated:

```sql
UPDATE admissions 
SET admissionStatus = 'approved', evaluatedBy = 12, evaluatedDate = '2026-02-20'
WHERE admissionId = 12000;
```

`evaluatedBy` references `staffusers.userId = 12` — that's Dr. Reyes, the program head. No new tables are created here; the existing `admissions` row transitions from `pending` → `approved`.

### What the system checks

Before approval, the system validates:

1. **Does a curriculum exist for this course + academic year?** → `curriculums` table
2. **Are there required admission documents?** → `admissionrequirements` × `studentrequirementsubmissions`
3. **Have all required documents been submitted?** → `documents` table

### Document submission

Maria uploaded her high school diploma, birth certificate, and good moral character. Each is a row:

```
documents.documentId = 71500 → submissionId = 72590
documents.fileUrl = '/uploads/diploma_santos.pdf'
documents.fileType = 'application/pdf'
documents.verifiedBy = 12 (Dr. Reyes)
```

The `studentrequirementsubmissions` table ties everything together:

```
submissionId = 72590
  → admissionId = 12000
  → requirementId = 1 (High School Diploma)
  → submissionStatus = 'submitted' → 'verified'
```

**Key relationship:** `admissionrequirements` is a *reference data* table — it lists what is required. `studentrequirementsubmissions` is the *transaction* table — it tracks whether each student actually submitted each requirement. `documents` holds the actual uploaded files.

---

## Phase 2 — Clearance Verification: "No Outstanding Obligations"

For continuing students, this phase checks library books, lab equipment, and financial obligations. But Maria is a first-year — her clearance is typically straightforward.

### The clearance module

```
clearanceperiods.periodId = 8 → termId = 1, periodStatus = 'open'
clearancerequirements.requirementId = 1 → officeId = 7 (Library)
clearancerequirements.requirementId = 2 → officeId = 12 (Science Lab)
```

Each clearance requirement belongs to an `offices` row. The 21 school offices each have predefined clearance items.

For Maria, the system creates a `studentclearances` record:

```
studentClearanceId = 17500
  → studentId = 15001
  → clearancePeriodId = 8
  → overallStatus = 'pending'
```

Then for each requirement, a `clearanceapprovals` row:

```
clearanceApprovalId = 376500
  → studentClearanceId = 17500
  → clearanceRequirementId = 1
  → status = 'approved' (waived for first-year)
  → approvedBy = 15
```

**Why three tables?** This design separates *what needs clearance* (`clearancerequirements`) from *who needs it* (`studentclearances`) from *the actual approval* (`clearanceapprovals`). It's a classic database pattern that allows different students to have different clearance statuses for the same requirement.

---

## Phase 3 — Scholarship / Financial Assessment: "Computing the Cost"

Maria applied for a 50% academic scholarship. The Scholarship Office evaluates her eligibility.

### Tables involved

```
scholarshiptypes.scholarshipTypeId = 1
  → scholarshipName = 'Academic Scholar'
  → coverageType = 'partial'
  → coveragePercent = 50.00
```

If approved:

```
studentscholarships.studentScholarshipId = 7500
  → studentId = 15001
  → scholarshipTypeId = 1
  → termId = 1
  → status = 'active'
  → approvedBy = 8
  → awardedBeforeEnrollment = 1
```

### The assessment

The system calculates what Maria owes:

```
studentassessments.assessmentId = 29500
  → enrollmentId = 15000 (Maria's eventual enrollment)
  → totalAssessedAmount = 25000.00
  → totalScholarshipCoverage = 12500.00 (50% reduction)
  → totalWaived = 500.00 (laboratory fee waiver)
  → remainingBalance = 12000.00
  → assessmentDate = '2026-02-25'
```

Each fee component is itemized in `charges`:

```
chargeId = 135000 → assessmentId = 29500 → feeTypeId = 1 (Tuition)      → amount = 15000
chargeId = 135001 → assessmentId = 29500 → feeTypeId = 2 (Miscellaneous) → amount = 5000
chargeId = 135002 → assessmentId = 29500 → feeTypeId = 3 (Laboratory)    → amount = 5000 → waivedAmount = 500
```

`feetypes` stores the *catalog* of possible fees; `charges` stores the *actual* line items per assessment. This separation means fee types can be changed mid-year without affecting historical records.

**The data flow:**

```
fee types (catalog) ──→ charges (per assessment) ──→ studentassessments (summary)
scholarship types ──→ studentscholarships (award) ──→ studentassessments (coverage)
```

---

## Phase 4 — Cashier: "Payment"

With her remaining balance of ₱12,000, Maria goes to the Cashier's Office.

### The payments table

```
payments.paymentId = 30200
  → enrollmentId = 15000
  → orNumber = 'OR-2026-45001' (UNIQUE — every OR is system-wide unique)
  → amount = 12000.00
  → paymentDate = '2026-03-01'
  → paymentMode = 'online'
  → processedBy = 5 (Cashier staff)
  → paymentStatus = 'paid'
```

The `orNumber` has a **UNIQUE constraint** — no two payments can share an OR number. This is a business rule enforced at the database level: every official receipt must be traceable to a single transaction.

**The payment cascade:**

When the payment is recorded, the system checks `studentassessments.remainingBalance`:
- If `payment.amount >= remainingBalance` → status = `paid`
- If `payment.amount < remainingBalance` → status = `partial`
- If no payments yet → status = `pending`

---

## Phase 5 — Registrar Approval: "Official Enrollment"

The Registrar verifies that all prior phases are complete. This is a *validation gate* — no new data is created in the student's personal tables, but the system checks that the preconditions are met.

### The enrollment table

Maria's full enrollment record was created earlier, but now it gets finalized:

```
enrollments.enrollmentId = 15000
  → studentId = 15001
  → courseId = 5 (BS CompSci)
  → termId = 1
  → studentType = 'firstYear'
  → academicStanding = 'regular'
  → enrollmentMode = 'online'
  → enrollmentStatus = 'pending' → 'enrolled'
  → registrarProcessedBy = 3
  → enrolledDate = '2026-03-01'
```

### Subject loading

Maria is assigned her first-semester subjects. The system consults `curriculumsubjects` to find which subjects a first-year BS CompSci student should take:

```
curriculumsubjects: curriculumId = 5, yearLevel = 1, semesterOffered = '1st'
  → subjectId = 101 (Introduction to Computing)
  → subjectId = 102 (Computer Programming 1)
  → subjectId = 103 (Mathematics in the Modern World)
  → ... (8 subjects total)
```

Each subject gets an `enrolledsubjects` row:

```
enrolledSubjectId = 179500
  → enrollmentId = 15000
  → subjectId = 101
  → sectionId = 15 (Section A)
  → scheduleId = 450
  → status = 'confirmed'
```

The `status` column tracks the lifecycle: `proposed` (pre-registration) → `confirmed` (official) → `dropped` (if the student drops).

### The schedule

Each `enrolledsubjects` row is linked to a `schedule`, which links to a `room`, an `instructor` (staffuser), and a `section`. The actual meeting times are in `schedulemeetings`:

```
schedulemeetings.meetingId = 2000
  → scheduleId = 450
  → dayOfWeek = 'Monday'
  → startTime = '08:00'
  → endTime = '09:30'
```

A schedule can have multiple meetings (e.g., Monday/Wednesday/Friday). This is why meetings are in a separate table — **one-to-many** from `schedules` to `schedulemeetings`.

---

## Phase 6 — Registrar Final: "Documents and Certificates"

The Registrar prints Maria's **Subject Load** (list of enrolled subjects) and **Enrollment Certificate**.

### Document print log

```
documentprintlog.printLogId = 24000
  → enrollmentId = 15000
  → documentType = 'subjectLoad'
  → printedDate = '2026-03-01 14:30:00'
  → printedBy = 3
  → documentNumber = 1
```

The `documentType` enum (`subjectLoad`, `classCard`, `certificate`) controls which format is generated. The `documentNumber` is a simple sequence per enrollment.

### The workflow tracking

At this point, the `enrollmentworkflow` system starts tracking Maria's progress through physical offices. This is a separate mechanism from the enrollment pipeline — it's for the *physical* routing form that Maria carries from office to office.

```
enrollmentworkflow.workflowId = 30000
  → enrollmentId = 15000
  → currentStep = 6 (Registrar Final)
  → workflowStatus = 'inProgress'
```

Each workflow has `workflowsteps`:

```
workflowstep 1: officeId = 17 (Business Admin Dept)  → stepStatus = 'completed'
workflowstep 2: officeId = 18 (CRim Dean)            → stepStatus = 'completed'
...
workflowstep 6: officeId = 1  (Registrar Office)     → stepStatus = 'completed'
workflowstep 7: officeId = 11 (School Clinic)         → stepStatus = 'pending'
workflowstep 8: officeId = ? (ID Office)              → stepStatus = 'pending'
```

The `workflowsteps` table tracks which offices have **signed off** on Maria's physical enrollment form. `signedBy` records the staff user who signed, and `signedDate` timestamps it.

---

## Phase 7 — Clinic: "Health Check and PhilHealth"

*This is where my memory was incomplete before. Here's what actually happens.*

Maria goes to the School Clinic (officeId = 11). The clinic staff:

1. Gives her hard-copy assessment forms to fill out
2. Takes her height, weight, and blood pressure
3. Registers her for PhilHealth (separate government system, but the school records the reference number)
4. Signs off on the enrollment workflow process form

### The clinicrecords table

```
clinicrecords.clinicRecordId = 29000
  → enrollmentId = 15000 (Maria's enrollment)
  → heightCm = 160.0
  → weightKg = 54.5
  → bloodPressure = '120/80'
  → philhealthNumber = '12-345678901-2'
  → philhealthRegistered = 1
  → assessmentNotes = 'Student filled out health assessment form. No major concerns.'
  → findings = 'Fit to attend classes. No restrictions.'
  → clinicStaffId = 22 (Nurse Mendoza)
  → assessmentDate = '2026-03-02'
  → status = 'completed'
```

### Workflow signature

After the clinic, the workflow step is updated:

```sql
UPDATE workflowsteps 
SET stepStatus = 'completed', signedBy = 22, signedDate = NOW()
WHERE workflowId = 30000 AND officeId = 11;
```

The `enrollmentworkflow` advances:

```sql
UPDATE enrollmentworkflow 
SET currentStep = 8
WHERE workflowId = 30000;
```

**Important distinction:** The `enrollmentworkflow`/`workflowsteps` system is separate from the enrollment *status* pipeline. The pipeline (`admissions → enrollments`) controls *system access* — whether Maria can proceed to the next digital step. The workflow tracks *physical signatures* — the paper form that confirms each office has done its part. This dual tracking exists because some steps (like clinic assessments and ID photos) are inherently physical and can't be fully digitized.

### Data relationships

```
enrollments ──1:1──→ clinicrecords (one clinic record per enrollment per term)
clinicrecords ──M:1──→ staffusers (the clinic staff who administered)
enrollmentworkflow ──1:M──→ workflowsteps
workflowsteps ──M:1──→ offices (21 offices in the system)
```

---

## Phase 8 — ID Evaluation: "The Student ID"

Maria now needs her school ID. She goes to the ID Office, has her photo taken, and provides emergency contact details.

### ID Request

```
idrequests.idRequestId = 21000
  → enrollmentId = 15000
  → requestReason = 'newStudent'
  → emergencyContactName = 'Mr. Santos'
  → emergencyContactNumber = '09171234567'
  → bloodType = 'O+'
  → cardPhotoPath = '/photos/2026-0001.jpg'
  → producedByVendor = 'ID Card Services Inc.'
  → requestDate = '2026-03-02'
  → status = 'pending' → 'cardProduced'
```

### The ID record

Once the card is produced, a `studentids` record is created:

```
studentids.idId = 15000
  → studentId = 15001
  → idRequestId = 21000
  → qrCode = 'QR-2026-0001-A3F2C8' (UNIQUE)
  → issueDate = '2026-03-10'
  → validationStatus = 'pendingValidation' → 'active'
  → securityPhotoPath = '/photos/security/2026-0001.jpg'
  → validatedBy = 25 (ID Officer)
  → validatedDate = '2026-03-10 10:00:00'
```

The `qrCode` is a unique per-ID identifier — it's used for validation via QR scanning at school entrances. The `validationStatus` lifecycle: `pendingValidation` → `active` → `lost` (if reported) or `replaced` (if a new one is issued).

### Why three tables for IDs?

| Table | Purpose | Example |
|---|---|---|
| `idrequests` | The *request* — reason, emergency info, photo | "new student request on March 2" |
| `studentids` | The *physical card* — QR code, validation status | "card #001 with QR xxx" |
| `enrollments` | The *link* — which enrollment triggered it | "for AY 2025-2026 1st Sem" |

The `idrequests → studentids` relationship is **one-to-one**: each request produces exactly one card. But a student can have multiple ID requests over time (for lost cards, renewals, shifted courses) — hence `enrollments → idrequests` is **one-to-many**.

---

## Phase 9 — Mid-Course Qualifying Exam: "The Checkpoint"

Some courses (like BS Computer Science) require a mid-course qualifying exam (`courses.requiresQualifyingExam = 1`). This happens mid-semester, around weeks 6-8.

### Database operation

A second `examresults` record, this time with `examStage = 'midCourseQualifying'`:

```
examresults.examId = 4567
  → studentId = 15001
  → courseId = 5
  → termId = 1
  → examStage = 'midCourseQualifying'
  → examType = 'courseSpecific'
  → examResult = 'pass'
  → examDate = '2026-04-15'
```

**Key difference from entrance exam:** The entrance exam (`examStage = 'entrance'`) gates *admission* to the school. The qualifying exam gates *continuation* in the course — if Maria fails, she may be shifted to a different program but doesn't lose her enrollment entirely.

---

## Cross-Cutting Concerns: Tables Used Throughout

### The academic calendar

`academicyears` and `academicterms` are referenced by almost every transaction table. Their hierarchy:

```
academicyears (yearLabel, startDate, endDate)
  └── academicterms (semester, startDate, endDate)
       ├── admissions (per-term application)
       ├── enrollments (per-term enrollment)
       ├── studentassessments (per-term financial)
       ├── studentclearances (per-term clearance)
       ├── studentscholarships (per-term awards)
       └── examresults (per-term exams)
```

### Staff and offices

`staffusers` and `offices` are the who-and-where of every action:

- `evaluatedBy` → who approved an admission
- `processedBy` → who took the payment
- `registrarProcessedBy` → who finalized enrollment
- `clinicStaffId` → who administered clinic
- `validatedBy` → who validated the ID
- `printedBy` → who printed a document
- `signedBy` → who signed a workflow step

Every action in the system is auditable to a specific staff member in a specific office.

---

## The Grand Picture: All Tables by Functional Module

```
┌─────────────────────────────────────────────────────────────┐
│                    STUDENT INFORMATION                       │
│  students ──→ addresses, guardians, religions               │
│  students ──→ studenteducationalbackgrounds                  │
│  educationalinstitutions (reference)                        │
├─────────────────────────────────────────────────────────────┤
│                    ACADEMIC REFERENCE                        │
│  academicunits ──→ courses ──→ majors, curriculums          │
│  curriculums ──→ curriculumsubjects ──→ subjects            │
│  subjects (203 rows — the full course catalog)              │
├─────────────────────────────────────────────────────────────┤
│                    ADMISSION PIPELINE                        │
│  admissions ──→ studentrequirementsubmissions ──→ documents  │
│  admissionrequirements (reference)                          │
│  examresults (entrance + qualifying)                        │
├─────────────────────────────────────────────────────────────┤
│                    ENROLLMENT CORE                           │
│  enrollments ──→ enrolledsubjects ──→ schedules, subjects    │
│  blocks (sections) ──→ schedules ──→ schedulemeetings, rooms │
│  creditedsubjects, transferacademicrecords                  │
├─────────────────────────────────────────────────────────────┤
│                    FINANCIAL                                 │
│  feetypes ──→ charges ──→ studentassessments ──→ enrollments │
│  scholarshipstypes ──→ studentscholarships                  │
│  payments ──→ enrollments                                   │
├─────────────────────────────────────────────────────────────┤
│                    CLEARANCE                                 │
│  clearancerequirements ──→ clearanceapprovals                │
│  clearanceperiods ──→ studentclearances                      │
├─────────────────────────────────────────────────────────────┤
│                    CLINIC                                    │
│  clinicrecords ──→ enrollments, staffusers                  │
├─────────────────────────────────────────────────────────────┤
│                    WORKFLOW TRACKING                         │
│  enrollmentworkflow ──→ workflowsteps ──→ offices             │
│  workflowsteps ──→ staffusers (signedBy)                     │
├─────────────────────────────────────────────────────────────┤
│                    ID MANAGEMENT                             │
│  idrequests ──→ enrollments                                  │
│  studentids ──→ students, idrequests                         │
├─────────────────────────────────────────────────────────────┤
│                    SYSTEM                                    │
│  academicterms, academicyears, gradescale                    │
│  documentprintlog, offices, staffusers                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Epilogue: What Maria's Data Trail Looks Like

When Maria finishes enrollment, her data footprint across the database:

| Table | Rows Created | Size |
|---|---|---|
| students | 1 | Core profile |
| addresses | 3 | Home, current, permanent |
| guardians | 2 | Mother, father |
| studenteducationalbackgrounds | 1 | STI Senior High |
| admissions | 1 | Application + approval |
| studentrequirementsubmissions | 4 | Diploma, birth cert, etc. |
| documents | 4 | Uploaded files |
| examresults | 1 | Entrance exam pass |
| enrollments | 1 | Official enrollment |
| enrolledsubjects | 8 | This semester's load |
| studentassessments | 1 | Fee computation |
| charges | ~10 | Line-item breakdown |
| payments | 1 | Payment transaction |
| studentclearances | 1 | Clearance record |
| clearanceapprovals | ~5 | Cleared requirements |
| clinicrecords | 1 | Health assessment |
| enrollmentworkflow | 1 | Workflow tracking |
| workflowsteps | ~8 | Office sign-offs |
| idrequests | 1 | ID request |
| studentids | 1 | Physical card |

**~54 rows across 20 tables** — that's the data cost of one successful enrollment.

---

*End of Dialogue Documentation*
