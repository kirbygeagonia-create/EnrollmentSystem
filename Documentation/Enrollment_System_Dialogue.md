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

Maria's chosen course, BS Computer Science, is a board course (`courses.requiresEntranceExam = 1`). Only board courses require an entrance exam — non-board courses skip this phase entirely. The entrance exam is a **two-stage** process.

### Stage 1 — General Entrance Exam (Guidance Office)

Maria goes to the Guidance Office and takes the **general** entrance exam — a standardized test taken by all applicants to board courses. The Guidance Office records the result:

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

### Stage 2 — Course-Specific Exam (College Department)

Before Maria can take the department's course-specific exam, the College of Computer Studies **pulls the Guidance exam results** (`examresults` filtered by `studentId` + `courseId`, `examType = 'general'`) to verify she passed Stage 1. Only after verification does the department administer the **course-specific** board-course exam:

```
examresults.examId = 4501
  → studentId = 15001
  → courseId = 5 (BS CompSci)
  → termId = 1
  → examStage = 'entrance'
  → examType = 'courseSpecific'
  → examResult = 'pass'
  → examDate = '2026-02-22'
```

If she had failed either stage, the `admissionStatus` in `admissions` would flip to `rejected`, ending her journey here. But she passes both.

**What this tells us:** The `examresults` table separates *admission* from *examination* — a course can opt in to requiring exams independently of the admission flow. The `examStage = 'entrance'` is the same for both stages; `examType` distinguishes `general` (Guidance Office) from `courseSpecific` (college department). This decoupling means the school can add or remove exam requirements without changing the admission pipeline logic.

---

## Phase 1 — Clearance Verification: "The Clearance Slip"

Clearance verification checks library books, lab equipment, and financial obligations. **This phase comes first — before the Academic Department Evaluation** — because the clearance slip's student copy is a mandatory requirement every student must hold when enrolling.

### How clearance really works

Clearance is issued **when a semester ends** — usually for continuing students, but every student's clearance is issued by their **college department**. Each student has **one clearance slip in record**, and it is **free**:

1. At the end of the semester, the college department issues each student's clearance slip, listing what must be settled (library books, lab equipment, financial obligations).
2. The student completes it within the clearance period — a **1–2 week window** (`clearanceperiods.clearanceStartDate` → `clearanceEndDate`, `periodStatus = 'open'`).
3. Within those weeks, the student submits the completed slip to the **Registrar desk** — a *different* registrar staff member than the one in Phase 5 — and receives the **student copy**.
4. If the weeks pass without submission, the **registrar will no longer entertain it** — the student must wait. When enrollment starts after the long break (or summer), students who did not complete and submit their clearance **do it first**, before going to their departments for enrollment and evaluation.

The **student copy is mandatory** — it is submitted along at the Registrar phase (Phase 5), where the Registrar will not approve enrollment without it.

**Important:** Clearance is **not part of the Enrollment Workflow Process form** — it is a separate, important requirement tracked in its own tables (`studentclearances`, `clearanceapprovals`), not on the 8-step workflow form.

### What's printed on the clearance slip

The clearance slip is a printed form. Each copy shows:

- **Semester term** (e.g., First Semester)
- **Academic school year** (e.g., 2025–2026)
- **Full name** of the student
- **Course and year** (e.g., BS Computer Science — 1st Year)

It carries **no student ID field** — the printed form relies on the name and course/year for identification. Each student gets **exactly one copy in record, free**; the slip is tied to the clearance record (`studentclearances.studentClearanceId`). A **"date to be signed"** line is printed on the form — the date by which the student must have it signed and submitted.

The slip's **"Received by"** section is printed with the **Registrar in-charge's signature over their printed name** when the student submits the completed slip at the Registrar desk during the clearance period. The registrar staff member who receives it is recorded in `studentclearances.receivedBy` (FK → `staffusers.userId`) with `receivedDate` — the new columns tracking desk receipt.

The **student copy** is the copy the student keeps after submission — it is used for **verification at the Academic Evaluation (Phase 2)** and is a **mandatory submission at the Registrar phase (Phase 5)**.

### The clearance module

```
clearanceperiods.periodId = 8 → termId = 1, periodStatus = 'open'
clearancerequirements.requirementId = 1 → officeId = 7 (Library)
clearancerequirements.requirementId = 2 → officeId = 12 (Science Lab)
```

Each clearance requirement belongs to an `offices` row. The 22 school offices each have predefined clearance items.

For Maria, the system creates a `studentclearances` record:

```
studentClearanceId = 17500
  → studentId = 15001
  → clearancePeriodId = 8
  → overallStatus = 'pending'
  → receivedBy = 6 (Registrar in-charge who received the completed slip)
  → receivedDate = '2026-02-20 09:30:00'
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

### Lost clearance slip

Each student has **one copy in record**, free. If the slip is lost, the student **pays ₱100** at Accounting — `feetypes.feeTypeId = 11` (`Clearance Slip Replacement`, flat) — then **shows the receipt to their college department**, which issues a new copy.

---

## Phase 2 — Academic Department Evaluation: "The Enrollment Form and Subject Load"

Maria goes to the College of Computer Studies — with her clearance slip student copy already in hand from Phase 1. She lines up to get the **Enrollment Form** and fills in her personal information, course, major, and student type.

### The enrollment record

The system creates Maria's enrollment record:

```
enrollments.enrollmentId = 15000
  → studentId = 15001
  → courseId = 5 (BS CompSci)
  → majorId = 2 (Software Development)
  → termId = 1
  → yearLevel = 1
  → studentType = 'firstYear'
  → enrollmentType = 'new'
  → enrollmentMode = 'online'
  → enrollmentStatus = 'pending'
```

### The enrollment form (what's on it)

The **Enrollment Form** is the printed form Maria fills at the department — the Academic Evaluation. It has two parts: the **demographic profile** and the **subject load**.

**Part 1 — Complete demographic profile.** Every field must be filled before the form can be submitted:

- **Name** (normalized: Last name, First name, Middle initial) and **suffix** (if any)
- **Sex / gender**
- **Date of birth** and **place of birth**
- **Religion** and **citizenship**
- **Civil status** (`students.civilStatus`: single / married / widowed / separated)
- **Address** — full breakdown: lot/block, street name, purok/sitio, barangay, city/municipality, district, province, region, country (`addresses` table)
- **Current address** — with a **"same as above" checkbox**; when ticked, the current address copies the home address. The system stores both addresses as separate `addresses` rows (`addressType = 'home'` and `'current'`), even when identical
- **Telephone number** and **contact (mobile) number** (`students.telephoneNumber`, `students.contactNumber`)
- **Email address** and **mailing address**
- **Current semester completed** — the total number of semesters attended (`students.semestersCompleted`)
- **Years in institution** (`students.yearsInInstitution`)
- **Father's name and contact number**, **mother's name and contact number**, and **guardian's name and contact number in case of emergency** — stored in the `guardians` table (with the emergency-contact flag)

**Part 2 — The subject load**, attached under the profile. The evaluator lists the subjects the student is allowed to take:

```
subject name   subject code   lecture units   lab units
Introduction to Computing   IT101   3   0
Computer Programming 1      IT102   2   1
... (8 subjects total)
```

Also printed on the form: two **checkboxes — Regular or Irregular** (maps to `enrollments.academicStanding`), the **form issue date** (`enrollments.formIssuedDate` — today's date when the form is handed out), and the signature lines:

- **Evaluator** (instructor of the college department) — signs at this phase
- **Dean / Program Head** — signs at this phase (for Maria, `evaluatedBy = 12` is Dr. Reyes, the program head)
- **Registrar** — signs later at Phase 5 (the signature line is on the form but is only filled when the Registrar approves)
- **Student signature** and **signed date** (`enrollments.formSignedDate`)

The student cannot submit the form — and the evaluator cannot process it — until every profile field is filled in and the subject load is attached.

### Retention Exam Gate (Board Courses Only)

For **continuing** students (2nd/3rd/4th/5th year) of board courses, there is a critical gate **before** they can fill the enrollment form: the **Retention Examination**. This is a real written exam that tests what they learned the previous academic term. If they fail, they cannot proceed to enrollment. Maria is a first-year, so this does not apply to her — but for continuing board-course students, the retention exam is recorded as:

```
examresults.examId = 4560
  → studentId = 15001
  → courseId = 5
  → termId = 1
  → examStage = 'retention'
  → examType = 'courseSpecific'
  → examResult = 'pass'
  → examDate = '2026-02-18'
```

**Important:** The retention exam is **not** part of the Enrollment Workflow Process form — it depends on the course (usually board courses) and is administered by the college department before the enrollment form is even issued. Non-board courses skip this gate entirely.

### Evaluation by Student Type

With the enrollment form filled, the evaluators (instructors of the college department) review Maria's credentials. Evaluation differs by student type:

- **Transferees and shifters:** Their previous subjects are **credited** — `creditedsubjects` and `transferacademicrecords` map old-school subjects to the current curriculum.
- **Continuing students:** Their previous subject load/academic term is checked subject-by-subject against the curriculum to determine **regular** or **irregular** standing (`enrollments.academicStanding` = `'regular'` or `'irregular'` — marked on the enrollment form).
- **First-years (like Maria):** No prior subjects to evaluate — she is automatically `regular`.

### Subject Load from Curriculum

The evaluator consults `curriculumsubjects` to determine Maria's first-semester subjects:

```
curriculumsubjects: curriculumId = 5, yearLevel = 1, semesterOffered = '1st'
  → subjectId = 101 (Introduction to Computing)
  → subjectId = 102 (Computer Programming 1)
  → subjectId = 103 (Mathematics in the Modern World)
  → ... (8 subjects total)
```

Each subject gets an `enrolledsubjects` row (initially `status = 'proposed'`):

```
enrolledSubjectId = 179500
  → enrollmentId = 15000
  → subjectId = 101
  → status = 'proposed'
```

The `status` column tracks the lifecycle: `proposed` (pre-registration) → `confirmed` (official) → `dropped` (if the student drops). The block and schedule pointers (`blockId`, `scheduleId`) are filled in later — during Phase 6 Blocking and Scheduling.

### Evaluator Signature and Registrar Approval

The evaluators (instructors of the college department) sign the enrollment form with the subject load attached/written — the **Evaluator** and **Dean/Program Head** signature lines, plus the student's own signature with the signed date. The enrollment record is updated:

```sql
UPDATE enrollments 
SET evaluatedBy = 12, academicStanding = 'regular'
WHERE enrollmentId = 15000;
```

`evaluatedBy` references `staffusers.userId = 12` — that's Dr. Reyes, the program head. The enrollment form then goes to the **Registrar** phase for final approval of the subject load.

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

## Phase 3 — Scholarship / Financial Assessment: "Free Tuition School"

This school provides **free tuition** to all college students. Once enrolled, every student automatically becomes a school scholar under the **School Grant** — a 100% full-tuition scholarship.

### The School Grant

```
scholarshiptypes.scholarshipTypeId = 9
  → scholarshipName = 'School Grant (Free Tuition)'
  → coverageType = 'full'
  → coveragePercent = 100.00
```

For Maria, the system automatically awards this:

```
studentscholarships.studentScholarshipId = 7500
  → studentId = 15001
  → scholarshipTypeId = 9 (School Grant)
  → termId = 1
  → status = 'active'
  → approvedBy = 8
  → awardedBeforeEnrollment = 1
```

### Outside Scholarships

Continuing students and shifters are passed automatically by this step — the School Grant covers them. However, **first-years and transferees** may also apply for **outside scholarship grants** (e.g., government, private, or NGO scholarships) on top of the School Grant. The scholarship stacking rule **BR19** still applies: combined scholarship coverage cannot exceed 100%.

### The assessment

The system still performs a formal tuition assessment — even though the School Grant covers it, the computation is recorded for audit and reporting:

```
studentassessments.assessmentId = 29500
  → enrollmentId = 15000 (Maria's eventual enrollment)
  → totalAssessedAmount = 25000.00
  → totalScholarshipCoverage = 25000.00 (100% School Grant)
  → totalWaived = 500.00 (laboratory fee waiver)
  → remainingBalance = 0.00
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

## Phase 4 — Accounting: "Payment"

Maria goes to the **Accounting Office**. She lines up, pays the usual enrollment fee of **₱500**, and receives a receipt. The Accounting staff signs her Enrollment Workflow Process form.

### The payments table

Payments ARE recorded in the system — this is the official record; the receipt is the student's proof:

```
payments.paymentId = 30200
  → enrollmentId = 15000
  → orNumber = 'OR-2026-45001' (UNIQUE — every OR is system-wide unique)
  → amount = 500.00
  → paymentDate = '2026-03-01'
  → paymentMode = 'cash'
  → processedBy = 5 (Accounting staff)
  → paymentStatus = 'paid'
```

The `orNumber` has a **UNIQUE constraint** — no two payments can share an OR number. This is a business rule enforced at the database level: every official receipt must be traceable to a single transaction.

### Workflow signature

After payment, the Accounting workflow step is signed:

```sql
UPDATE workflowsteps 
SET stepStatus = 'completed', signedBy = 5, signedDate = NOW()
WHERE workflowId = 30000 AND officeId = 2; -- Accounting Office
```

**The payment cascade:**

When the payment is recorded, the system checks `studentassessments.remainingBalance`:
- If `payment.amount >= remainingBalance` → status = `paid`
- If `payment.amount < remainingBalance` → status = `partial`
- If no payments yet → status = `pending`

---

## Phase 5 — Registrar Approval: "Official Enrollment"

The Registrar verifies that all prior phases are complete. This is a *validation gate* — the system checks that the preconditions are met, the student's data is **recorded or updated**, and the Registrar issues the official documents.

**Mandatory requirement:** Maria submits her **clearance slip student copy** (obtained in Phase 1) here — the Registrar will not approve enrollment without it. Note this is a *different* registrar staff member than the one who received the completed clearance slip at the desk during Phase 1.

### Recording vs. updating student data

The Registrar first checks the subject load from the enrollment form, approves the subjects, then enrolls the student. Depending on the student type, the system either **records** or **updates** the student's data:

- **First year** — the student's profile is *new*: `enrollmentType = 'new'`. The Registrar records the full student data (demographics, address, guardians) from the enrollment form.
- **Transferee** — also a *new* enrollment: `enrollmentType = 'new'`. Previous school records are credited (`transferacademicrecords`, `creditedsubjects`).
- **Continuing** — the student is *already in the system*: `enrollmentType = 'old'`. The Registrar **updates** the existing student data (address, contact numbers, semesters completed, years in institution).
- **Shifter** — also an *old* enrollment: `enrollmentType = 'old'`. The student's existing data is updated with the new course.

The rule is derived automatically: `firstYear` and `transferee` → `'new'`; `continuing` and `shifter` → `'old'`.

### The enrollment table

Maria's full enrollment record was created earlier, but now it gets finalized:

```
enrollments.enrollmentId = 15000
  → studentId = 15001
  → courseId = 5 (BS CompSci)
  → termId = 1
  → yearLevel = 1
  → studentType = 'firstYear'
  → enrollmentType = 'new'
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
  → status = 'confirmed'
```

The `status` column tracks the lifecycle: `proposed` (pre-registration) → `confirmed` (official) → `dropped` (if the student drops).

### The schedule

Each `enrolledsubjects` row is linked to a `schedule`, which links to a `room`, an `instructor` (staffuser), and a `block`. The actual meeting times are in `schedulemeetings`:

```
schedulemeetings.meetingId = 2000
  → scheduleId = 450
  → dayOfWeek = 'Monday'
  → startTime = '08:00'
  → endTime = '09:30'
```

A schedule can have multiple meetings (e.g., Monday/Wednesday/Friday). This is why meetings are in a separate table — **one-to-many** from `schedules` to `schedulemeetings`.

### Document print log

The Registrar prints Maria's **Subject Load** (list of enrolled subjects) and **Enrollment Certificate**, each print recorded in the system:

```
documentprintlog.printLogId = 24000
  → enrollmentId = 15000
  → documentType = 'subjectLoad'
  → printedDate = '2026-03-01 14:30:00'
  → printedBy = 3
  → documentNumber = 1
```

The `documentType` enum (`subjectLoad`, `classCard`, `certificate`) controls which format is generated. The `documentNumber` is a simple sequence per enrollment.

### The Enrollment Certificate print

The **Enrollment Certificate Form** ("Student Subject Load") is the official proof of enrollment — printed with the complete subject loads. What's printed:

**Header:** the school logo, school name, address, and telephone number.

**Student information (pre-filled from the system):**
- Full name — **Last name, First name, Middle initial**
- Course and year
- Student ID (school ID number)
- School year (academic year) and **semester**
- **Type** — `new` or `old` (`enrollments.enrollmentType`), printed as "New" or "Old" student

**The subject table** — one row per enrolled subject:

```
No.   Subject Code   Description             Lecture Units   Lab Units   Total
1     IT101          Introduction to Computing    3             0          3
2     IT102          Computer Programming 1       2             1          3
...   (all subjects)
                                          Total Units              24
```

**Below the table:**
- **Date enrolled** (`enrollments.enrolledDate`)
- **Evaluated by** — the printed name of the registrar evaluator. Every staff member has their own login account (`staffusers` — username/password) and signs in when processing; the evaluator's name is taken from their profile, so it's always correct
- **Processed by** — the same name as Evaluated by: for the certificate, the registrar evaluator who evaluated the enrollment also processes the print
- **Student Copy** indication — this print is the student's copy
- The **"SEAIT ENROLLED"** stamp — applied by the printer as part of the print output

Each certificate print is logged: `documentprintlog.documentType = 'certificate'`.

### The Class Card print

A **Class Card** is printed **per subject** for the semester. Each card shows:

**Header:** the school logo, school name, address, telephone — plus **"Office of the Registrar"**, the title **"Class Card"**, the semester, and the academic/school year.

**Student block:**
- Last / First / Middle name
- Course and year
- Subject code, descriptive title, and **units** — the lecture and lab units **summed** (e.g., IT102 → 3 units)

**The empty boxes** — filled in by the instructor during the term:
- **Set** (the block the student belongs to — e.g., "Block A")
- **Time** (class meeting time, from `schedulemeetings`)
- **Day** (class meeting days)
- **Grade** (final grade given at term end)
- **Name and Signature of Instructor**, with **Date** — signed by the instructor when the card is used

**Footer:** **Issued by** — the evaluator's printed name (the registrar evaluator) — and **Date** — when the card was processed/given.

Each card print is logged: `documentprintlog.documentType = 'classCard'` (one log row per card).

---

## Phase 6 — Blocking and Scheduling: "The Block Assignment"

After the Registrar, the Academic Department assigns Maria to a **block** — a fixed group of students for her year level, course, and term.

### Blocks and Schedules

Each block has fixed schedules, subjects, instructors, and rooms. Schedules are organized under blocks: `schedules.blockId` FK → `blocks.blockId`. Maria is assigned to Block A (`blocks.blockId = 15`, `blockName = 'Block A'`), which has a fixed set of subjects, meeting times, and rooms. The block and schedule pointers on her confirmed subjects are now filled in:

```sql
UPDATE enrolledsubjects 
SET blockId = 15, scheduleId = 450
WHERE enrollmentId = 15000;
```

### The Block and Schedule print

The department also prints the **Block and Schedule** — the printed list of subjects with their meeting times, days, rooms, and instructors for the whole block (see `Documentation/Images/Class Block and Schedule.jpg` for the reference layout). It shows each subject with:

- Subject code and descriptive title
- **Day** and **Time** (from `schedulemeetings` — a subject can meet on multiple days, e.g., Mon/Wed/Fri)
- **Room** (`rooms` — e.g., Room 201)
- **Instructor** (the staff user assigned to the schedule)

This print is what tells the student where to go for each class — it complements the class card, which carries the same meeting info in per-subject boxes.

### The workflow tracking

The **Enrollment Workflow Process form** is given to Maria as a guide — every phase requires its signature as verification/proof of progress. This form IS tracked in the system:

```
enrollmentworkflow.workflowId = 30000
  → enrollmentId = 15000
  → currentStep = 6 (Blocking and Scheduling)
  → workflowStatus = 'inProgress'
```

Each workflow has `workflowsteps`:

```
workflowstep 1: officeId = 17 (Business Admin Dept)  → stepStatus = 'completed'
workflowstep 2: officeId = 18 (CRim Dean)            → stepStatus = 'completed'
...
workflowstep 5: officeId = 1  (Registrar Office)     → stepStatus = 'completed'
workflowstep 6: officeId = 17 (Business Admin Dept)  → stepStatus = 'completed' (blocking)
workflowstep 7: officeId = 11 (School Clinic)         → stepStatus = 'pending'
workflowstep 8: officeId = 22 (ID Office)            → stepStatus = 'pending'
```

The `workflowsteps` table tracks which offices have **signed off** on Maria's physical enrollment form. `signedBy` records the staff user who signed, and `signedDate` timestamps it.

**Safety net:** The `workflowStatus` column supports a `'lost'` status — if the paper form is lost, the system still has the full digital record of every step, every signature, and every timestamp. No data is ever truly lost.

---

## Phase 7 — Clinic: "Physical Examination, Health Check, and PhilHealth"

*This is where my memory was incomplete before. Here's what actually happens.*

Maria goes to the School Clinic (officeId = 11). The clinic staff:

1. Conducts a **physical examination** — height, weight, blood pressure, and other vital signs
2. Gives her hard-copy assessment forms to fill out (health history, medical conditions)
3. Registers her for PhilHealth (separate government system, but the school records the reference number)
4. Signs off on the enrollment workflow process form

**Note:** The physical examination is NOT an admission requirement — it is done at the Clinic phase, not during admissions. It was moved here from the `admissionrequirements` table.

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
workflowsteps ──M:1──→ offices (22 offices in the system)
```

---

## Phase 8 — ID Request, Release and Validation: "The Student ID"

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
│  examresults (entrance + retention)                          │
├─────────────────────────────────────────────────────────────┤
│                    ENROLLMENT CORE                           │
│  enrollments ──→ enrolledsubjects ──→ schedules, subjects    │
│  blocks ──→ schedules ──→ schedulemeetings, rooms │
│  creditedsubjects, transferacademicrecords                  │
├─────────────────────────────────────────────────────────────┤
│                    FINANCIAL                                 │
│  feetypes ──→ charges ──→ studentassessments ──→ enrollments │
│  scholarshiptypes ──→ studentscholarships                  │
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
