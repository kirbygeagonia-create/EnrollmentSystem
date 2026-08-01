# Database Representations & Visual Reference

> *Supplementary documentation for the Enrollment System database — ER diagrams, relationship matrices, phase mappings, data volumes, business rules, and constraints.*

---

## 1. Entity-Relationship Diagram

```mermaid
erDiagram
    %% ===== STUDENT INFORMATION MODULE =====
    students {
        int studentId PK
        varchar schoolIdNumber UK
        varchar lastName
        varchar firstName
        varchar middleName
        varchar suffix
        enum gender
        date birthdate
        varchar birthplace
        varchar citizenship
        int religionId FK
        varchar contactNumber
        varchar email
        varchar username UK
        varchar passwordHash
        enum status
    }

    addresses {
        int addressId PK
        int studentId FK
        enum addressType
        varchar houseBuildingNo
        varchar street
        varchar sitioPurok
        varchar barangay
        varchar cityMunicipality
        varchar province
        varchar region
        varchar zipCode
    }

    guardians {
        int guardianId PK
        int studentId FK
        enum relationship
        varchar fullName
        varchar contactNumber
        varchar email
        bool isEmergencyContact
        bool isAuthorizedToActOnBehalf
    }

    religions {
        int religionId PK
        varchar religionName
    }

    educationalinstitutions {
        int institutionId PK
        varchar institutionName
        enum institutionType
        varchar cityMunicipality
        varchar province
    }

    studenteducationalbackgrounds {
        int backgroundId PK
        int studentId FK
        int institutionId FK
        enum levelCompleted
        varchar strandTrack
        date yearCompleted
        varchar honorsCertifications
        varchar supportingDocumentPath
    }

    %% ===== ACADEMIC REFERENCE MODULE =====
    academicunits {
        int unitId PK
        varchar unitName
        enum unitType
        int parentUnitId FK
    }

    courses {
        int courseId PK
        int unitId FK
        varchar courseName
        varchar courseCode
        bool requiresEntranceExam
        bool requiresQualifyingExam
    }

    majors {
        int majorId PK
        int courseId FK
        varchar majorName
    }

    curriculums {
        int curriculumId PK
        int courseId FK
        int majorId FK
        date effectiveYear
        varchar curriculumName
    }

    curriculumsubjects {
        int curriculumSubjectId PK
        int curriculumId FK
        int subjectId FK
        int prerequisiteSubjectId FK
        int yearLevel
        enum semesterOffered
    }

    subjects {
        int subjectId PK
        varchar subjectCode
        varchar subjectName
        decimal lectureUnits
        decimal labUnits
        enum subjectType
    }

    %% ===== ADMISSION MODULE =====
    academicterms {
        int termId PK
        int academicYearId FK
        enum semester
        date startDate
        date endDate
    }

    academicyears {
        int academicYearId PK
        varchar yearLabel
        date startDate
        date endDate
    }

    admissionrequirements {
        int requirementId PK
        varchar requirementName
        enum appliesTo
        bool isRequired
    }

    admissions {
        int admissionId PK
        int studentId FK
        int termId FK
        int courseId FK
        enum applicantType
        enum applicationMode
        enum admissionStatus
        int evaluatedBy FK
        date evaluatedDate
    }

    studentrequirementsubmissions {
        int submissionId PK
        int admissionId FK
        int requirementId FK
        enum submissionStatus
        date submittedDate
        text remarks
    }

    documents {
        int documentId PK
        int submissionId FK
        text fileUrl
        varchar fileType
        date uploadedDate
        int verifiedBy FK
    }

    examresults {
        int examId PK
        int studentId FK
        int courseId FK
        int termId FK
        enum examStage (entrance, midCourseQualifying, retention)
        enum examType (general = Guidance Office, courseSpecific = department)
        enum examResult
        date examDate
    }

    %% ===== ENROLLMENT CORE MODULE =====
    enrollments {
        int enrollmentId PK
        int studentId FK
        int courseId FK
        int majorId FK
        int termId FK
        int admissionId FK
        enum studentType
        enum academicStanding
        enum enrollmentMode
        enum enrollmentStatus
        int evaluatedBy FK
        int registrarProcessedBy FK
        date enrolledDate
    }

    blocks {
        int blockId PK
        int courseId FK
        int termId FK
        int yearLevel
        varchar blockName
        int maxStudents
    }

    schedules {
        int scheduleId PK
        int blockId FK
        int subjectId FK
        int instructorId FK
        int roomId FK
    }

    schedulemeetings {
        int meetingId PK
        int scheduleId FK
        enum dayOfWeek
        time startTime
        time endTime
    }

    rooms {
        int roomId PK
        varchar roomName
        int capacity
        varchar building
    }

    enrolledsubjects {
        int enrolledSubjectId PK
        int enrollmentId FK
        int subjectId FK
        int blockId FK
        int scheduleId FK
        decimal grade
        enum status
    }

    creditedsubjects {
        int creditedId PK
        int enrollmentId FK
        int transferRecordId FK
        varchar previousSubjectName
        int creditedToSubjectId FK
        decimal creditedUnits
        text remarks
    }

    transferacademicrecords {
        int transferRecordId PK
        int studentId FK
        int institutionId FK
        varchar subjectNameAtOldSchool
        decimal unitsAtOldSchool
        decimal gradeAtOldSchool
        enum passResult
    }

    %% ===== FINANCIAL MODULE =====
    feetypes {
        int feeTypeId PK
        varchar feeName
        decimal defaultAmount
        enum unitBasis
    }

    studentassessments {
        int assessmentId PK
        int enrollmentId FK
        decimal totalAssessedAmount
        decimal totalScholarshipCoverage
        decimal totalWaived
        decimal remainingBalance
        date assessmentDate
    }

    charges {
        int chargeId PK
        int assessmentId FK
        int feeTypeId FK
        decimal amount
        decimal waivedAmount
    }

    scholarshiptypes {
        int scholarshipTypeId PK
        varchar scholarshipName
        enum coverageType
        decimal coveragePercent
    }

    studentscholarships {
        int studentScholarshipId PK
        int studentId FK
        int scholarshipTypeId FK
        int termId FK
        enum status
        int approvedBy FK
        bool awardedBeforeEnrollment
    }

    payments {
        int paymentId PK
        int enrollmentId FK
        varchar orNumber UK
        decimal amount
        date paymentDate
        enum paymentMode
        int processedBy FK
        enum paymentStatus
    }

    %% ===== CLEARANCE MODULE =====
    clearanceperiods {
        int clearancePeriodId PK
        int termId FK
        date clearanceStartDate
        date clearanceEndDate
        enum periodStatus
    }

    clearancerequirements {
        int clearanceRequirementId PK
        int officeId FK
    }

    studentclearances {
        int studentClearanceId PK
        int studentId FK
        int clearancePeriodId FK
        enum overallStatus
        date extendedDeadline
    }

    clearanceapprovals {
        int clearanceApprovalId PK
        int studentClearanceId FK
        int clearanceRequirementId FK
        enum status
        int approvedBy FK
        date approvalDate
        text remarks
    }

    %% ===== CLINIC MODULE =====
    clinicrecords {
        int clinicRecordId PK
        int enrollmentId FK
        decimal heightCm
        decimal weightKg
        varchar bloodPressure
        varchar philhealthNumber
        bool philhealthRegistered
        text assessmentNotes
        text findings
        int clinicStaffId FK
        date assessmentDate
        enum status
    }

    %% ===== WORKFLOW MODULE =====
    offices {
        int officeId PK
        varchar officeName
    }

    enrollmentworkflow {
        int workflowId PK
        int enrollmentId FK
        int currentStep
        enum workflowStatus
    }

    workflowsteps {
        int workflowStepId PK
        int workflowId FK
        int officeId FK
        int stepOrder
        enum stepStatus
        int signedBy FK
        datetime signedDate
    }

    %% ===== ID MANAGEMENT MODULE =====
    idrequests {
        int idRequestId PK
        int enrollmentId FK
        enum requestReason
        varchar emergencyContactName
        varchar emergencyContactNumber
        varchar bloodType
        varchar cardPhotoPath
        varchar producedByVendor
        date requestDate
        enum status
    }

    studentids {
        int idId PK
        int studentId FK
        int idRequestId FK
        varchar qrCode UK
        date issueDate
        enum validationStatus
        varchar securityPhotoPath
        int validatedBy FK
        datetime validatedDate
    }

    %% ===== SYSTEM MODULE =====
    staffusers {
        int userId PK
        int officeId FK
        int unitId FK
        varchar employeeNo UK
        varchar firstName
        varchar middleName
        varchar lastName
        varchar username UK
        varchar passwordHash
        enum role
        varchar email
        varchar contactNo
        enum status
    }

    gradescale {
        int gradeScaleId PK
        decimal minGrade
        decimal maxGrade
        bool isPassing
        varchar description
    }

    documentprintlog {
        int printLogId PK
        int enrollmentId FK
        enum documentType
        datetime printedDate
        int printedBy FK
        int documentNumber
    }

    %% ===== RELATIONSHIPS =====

    %% Student Information
    students ||--o{ addresses : "has"
    students ||--o{ guardians : "has"
    students ||--o{ studenteducationalbackgrounds : "has"
    students }o--|| religions : "belongs to"
    studenteducationalbackgrounds }o--|| educationalinstitutions : "attended"

    %% Academic Reference
    academicunits ||--o{ academicunits : "parent unit"
    academicunits ||--o{ courses : "offers"
    courses ||--o{ majors : "has"
    courses ||--o{ curriculums : "has curriculum"
    curriculums }o--|| majors : "may have"
    curriculums ||--o{ curriculumsubjects : "includes"
    curriculumsubjects }o--|| subjects : "references"
    curriculumsubjects }o--|| subjects : "prerequisite"

    %% Admission
    students ||--o{ admissions : "applies"
    academicterms ||--o{ admissions : "for term"
    courses ||--o{ admissions : "applied to"
    admissions ||--o{ studentrequirementsubmissions : "requires"
    admissionrequirements ||--o{ studentrequirementsubmissions : "required by"
    studentrequirementsubmissions ||--o{ documents : "has files"
    students ||--o{ examresults : "takes exam"
    courses ||--o{ examresults : "exam for"
    academicterms ||--o{ examresults : "exam term"

    %% Enrollment Core
    students ||--o{ enrollments : "enrolls"
    courses ||--o{ enrollments : "enrolled in"
    academicterms ||--o{ enrollments : "during term"
    admissions ||--o{ enrollments : "originates from"
    enrollments ||--o{ enrolledsubjects : "has subjects"
    subjects ||--o{ enrolledsubjects : "enrolled as"
    blocks ||--o{ enrolledsubjects : "block assigned"
    schedules ||--o{ enrolledsubjects : "schedule assigned"
    blocks ||--o{ schedules : "has schedule"
    subjects ||--o{ schedules : "scheduled"
    staffusers ||--o{ schedules : "instructs"
    rooms ||--o{ schedules : "held in"
    schedules ||--o{ schedulemeetings : "meets on"
    enrollments ||--o{ creditedsubjects : "credits from"
    transferacademicrecords ||--o{ creditedsubjects : "based on"
    students ||--o{ transferacademicrecords : "transferred from"

    %% Financial
    enrollments ||--|| studentassessments : "assessed"
    studentassessments ||--o{ charges : "itemized as"
    feetypes ||--o{ charges : "categorized as"
    scholarshiptypes ||--o{ studentscholarships : "awarded as"
    students ||--o{ studentscholarships : "receives"
    academicterms ||--o{ studentscholarships : "award term"
    enrollments ||--o{ payments : "paid as"
    staffusers ||--o{ payments : "processed by"

    %% Clearance
    students ||--o{ studentclearances : "cleared"
    clearanceperiods ||--o{ studentclearances : "in period"
    academicterms ||--o{ clearanceperiods : "for term"
    studentclearances ||--o{ clearanceapprovals : "approved via"
    clearancerequirements ||--o{ clearanceapprovals : "checks"
    offices ||--o{ clearancerequirements : "responsible"

    %% Clinic
    enrollments ||--o{ clinicrecords : "has clinic"
    staffusers ||--o{ clinicrecords : "administered by"

    %% Workflow
    enrollments ||--o{ enrollmentworkflow : "tracked by"
    enrollmentworkflow ||--o{ workflowsteps : "has steps"
    offices ||--o{ workflowsteps : "step office"
    staffusers ||--o{ workflowsteps : "signed by"

    %% ID Management
    enrollments ||--o{ idrequests : "requests ID"
    idrequests ||--|| studentids : "produces"
    students ||--o{ studentids : "has ID"
    staffusers ||--o{ studentids : "validated by"

    %% System
    staffusers }o--|| offices : "belongs to"
    staffusers }o--|| academicunits : "assigned to"
    academicterms }o--|| academicyears : "belongs to"
    documentprintlog }o--|| enrollments : "prints for"
    staffusers }o--|| documentprintlog : "printed by"
```

---

## 2. Table Relationship Matrix

### Tables and Their Foreign Key References

| Source Table | FK Column | References (Table.Column) | Cardinality |
|---|---|---|---|
| **addresses** | `studentId` | students.studentId | M:1 |
| **admissions** | `studentId` | students.studentId | M:1 |
| **admissions** | `termId` | academicterms.termId | M:1 |
| **admissions** | `courseId` | courses.courseId | M:1 |
| **admissions** | `evaluatedBy` | staffusers.userId | M:1 |
| **blocks** | `courseId` | courses.courseId | M:1 |
| **blocks** | `termId` | academicterms.termId | M:1 |
| **charges** | `assessmentId` | studentassessments.assessmentId | M:1 |
| **charges** | `feeTypeId` | feetypes.feeTypeId | M:1 |
| **clearanceapprovals** | `studentClearanceId` | studentclearances.studentClearanceId | M:1 |
| **clearanceapprovals** | `clearanceRequirementId` | clearancerequirements.clearanceRequirementId | M:1 |
| **clearanceapprovals** | `approvedBy` | staffusers.userId | M:1 |
| **clearanceperiods** | `termId` | academicterms.termId | M:1 |
| **clearancerequirements** | `officeId` | offices.officeId | M:1 |
| **clinicrecords** | `enrollmentId` | enrollments.enrollmentId | M:1 |
| **clinicrecords** | `clinicStaffId` | staffusers.userId | M:1 |
| **courses** | `unitId` | academicunits.unitId | M:1 |
| **creditedsubjects** | `enrollmentId` | enrollments.enrollmentId | M:1 |
| **creditedsubjects** | `transferRecordId` | transferacademicrecords.transferRecordId | M:1 |
| **creditedsubjects** | `creditedToSubjectId` | subjects.subjectId | M:1 |
| **curriculums** | `courseId` | courses.courseId | M:1 |
| **curriculums** | `majorId` | majors.majorId | M:1 |
| **curriculumsubjects** | `curriculumId` | curriculums.curriculumId | M:1 |
| **curriculumsubjects** | `subjectId` | subjects.subjectId | M:1 |
| **curriculumsubjects** | `prerequisiteSubjectId` | subjects.subjectId | M:1 |
| **documentprintlog** | `enrollmentId` | enrollments.enrollmentId | M:1 |
| **documentprintlog** | `printedBy` | staffusers.userId | M:1 |
| **documents** | `submissionId` | studentrequirementsubmissions.submissionId | M:1 |
| **documents** | `verifiedBy` | staffusers.userId | M:1 |
| **enrolledsubjects** | `enrollmentId` | enrollments.enrollmentId | M:1 |
| **enrolledsubjects** | `subjectId` | subjects.subjectId | M:1 |
| **enrolledsubjects** | `blockId` | blocks.blockId | M:1 |
| **enrolledsubjects** | `scheduleId` | schedules.scheduleId | M:1 |
| **enrollments** | `studentId` | students.studentId | M:1 |
| **enrollments** | `courseId` | courses.courseId | M:1 |
| **enrollments** | `majorId` | majors.majorId | M:1 |
| **enrollments** | `termId` | academicterms.termId | M:1 |
| **enrollments** | `admissionId` | admissions.admissionId | M:1 |
| **enrollments** | `evaluatedBy` | staffusers.userId | M:1 |
| **enrollments** | `registrarProcessedBy` | staffusers.userId | M:1 |
| **enrollmentworkflow** | `enrollmentId` | enrollments.enrollmentId | 1:1 |
| **examresults** | `studentId` | students.studentId | M:1 |
| **examresults** | `courseId` | courses.courseId | M:1 |
| **examresults** | `termId` | academicterms.termId | M:1 |
| **guardians** | `studentId` | students.studentId | M:1 |
| **idrequests** | `enrollmentId` | enrollments.enrollmentId | M:1 |
| **majors** | `courseId` | courses.courseId | M:1 |
| **payments** | `enrollmentId` | enrollments.enrollmentId | M:1 |
| **payments** | `processedBy` | staffusers.userId | M:1 |
| **schedulemeetings** | `scheduleId` | schedules.scheduleId | M:1 |
| **schedules** | `blockId` | blocks.blockId | M:1 |
| **schedules** | `subjectId` | subjects.subjectId | M:1 |
| **schedules** | `instructorId` | staffusers.userId | M:1 |
| **schedules** | `roomId` | rooms.roomId | M:1 |
| **staffusers** | `officeId` | offices.officeId | M:1 |
| **staffusers** | `unitId` | academicunits.unitId | M:1 |
| **studentassessments** | `enrollmentId` | enrollments.enrollmentId | 1:1 |
| **studentclearances** | `studentId` | students.studentId | M:1 |
| **studentclearances** | `clearancePeriodId` | clearanceperiods.clearancePeriodId | M:1 |
| **studenteducationalbackgrounds** | `studentId` | students.studentId | M:1 |
| **studenteducationalbackgrounds** | `institutionId` | educationalinstitutions.institutionId | M:1 |
| **studentids** | `studentId` | students.studentId | M:1 |
| **studentids** | `idRequestId` | idrequests.idRequestId | 1:1 |
| **studentids** | `validatedBy` | staffusers.userId | M:1 |
| **studentrequirementsubmissions** | `admissionId` | admissions.admissionId | M:1 |
| **studentrequirementsubmissions** | `requirementId` | admissionrequirements.requirementId | M:1 |
| **students** | `religionId` | religions.religionId | M:1 |
| **studentscholarships** | `studentId` | students.studentId | M:1 |
| **studentscholarships** | `scholarshipTypeId` | scholarshiptypes.scholarshipTypeId | M:1 |
| **studentscholarships** | `termId` | academicterms.termId | M:1 |
| **studentscholarships** | `approvedBy` | staffusers.userId | M:1 |
| **transferacademicrecords** | `studentId` | students.studentId | M:1 |
| **transferacademicrecords** | `institutionId` | educationalinstitutions.institutionId | M:1 |
| **workflowsteps** | `workflowId` | enrollmentworkflow.workflowId | M:1 |
| **workflowsteps** | `officeId` | offices.officeId | M:1 |
| **workflowsteps** | `signedBy` | staffusers.userId | M:1 |

### Most-Referenced Tables

| Table | Referenced By (# FKs) | Key Role |
|---|---|---|
| **students** | 10 | Central hub — most tables link back to the student |
| **enrollments** | 8 | The enrollment event everything connects to |
| **staffusers** | 8 | Every action is auditable to a specific staff member |
| **academicterms** | 5 | Time dimension for enrollment, exam, assessment, clearance |
| **courses** | 5 | Academic structure reference |
| **subjects** | 5 | Subject catalog referenced by curriculum, enrollment, schedule |
| **offices** | 3 | Workflow steps and clearance requirements |
| **schedules** | 3 | Schedule meetings, enrolled subjects, instructor assignment |

---

## 3. Phase-to-Table Mapping Grid

| Phase | Description | Tables Read | Tables Written | Key DB Operations |
|---|---|---|---|---|
| **0 — Application** | Student applies for admission | students (dupe check), courses, academicterms | students, addresses, guardians, studenteducationalbackgrounds, educationalinstitutions, admissions | INSERT student profile, check uniqueness, create admission pending |
| **0.5 — Entrance Exam** | Two-stage entrance exam for board courses: Guidance general exam → department verifies Guidance results → course-specific board-course exam | courses (exam requirement), admissions, examresults (Guidance results check) | examresults, admissions (status update) | INSERT exam result (general, then courseSpecific after verification), UPDATE admissionStatus on pass/fail |
| **1 — Dept Evaluation** | Department reviews credentials; board-course continuing students take Retention Exam before the enrollment form; evaluation by student type (credit transfer, regular/irregular); evaluator signs form + subject load | admissions, studentrequirementsubmissions, documents, admissionrequirements, curriculums, curriculumsubjects, creditedsubjects, transferacademicrecords | admissions (status→approved/rejected), documents (verifiedBy), examresults (retention), enrollments, enrolledsubjects (proposed) | UPDATE admissionStatus, verify document submissions, INSERT retention exam result, create enrollment + proposed subject load |
| **2 — Clearance** | Check outstanding obligations | clearancerequirements, clearanceperiods, offices, students | studentclearances, clearanceapprovals | INSERT clearance + approval rows per requirement |
| **3 — Assessment** | Compute tuition and fees | feetypes, scholarshiptypes, students, courses | studentassessments, charges (per fee), studentscholarships | INSERT assessment with computed amounts, create itemized charges |
| **4 — Accounting** | Payment processing (₱500 enrollment fee, receipt, workflow form signature) | studentassessments (remainingBalance) | payments, studentassessments (balance update), workflowsteps (sign off) | INSERT payment (unique orNumber), UPDATE remainingBalance recalc |
| **5 — Registrar Approval** | Final enrollment validation | admissions (status), enrolledsubjects, schedules, blocks, curriculums, curriculumsubjects | enrollments (status→enrolled), enrolledsubjects (status→confirmed), documentprintlog | UPDATE enrollmentStatus, confirm subject enrollment, print documents |
| **6 — Registrar Final** | Print documents | enrollments, enrolledsubjects, subjects, schedules | documentprintlog, enrollmentworkflow, workflowsteps | INSERT document print log, initialize workflow tracking |
| **7 — Clinic** | Physical examination (height, weight, BP), hard-copy assessments, PhilHealth registration, findings; workflow form signed | enrollments, enrollmentworkflow, staffusers | clinicrecords, workflowsteps (sign off), enrollmentworkflow (advance) | INSERT clinic record (incl. physical exam results), UPDATE workflow step + advance counter |
| **8 — ID Request, Release and Validation** | Student ID request → card production → release + QR validation | enrollments, students | idrequests, studentids, workflowsteps | INSERT ID request + student ID, update workflow sign-off |
| **9 — Qualifying Exam** | Mid-course exam | courses (exam requirement), enrollments | examresults | INSERT exam result with midCourseQualifying stage |

### Data Flow Through Phases (Simplified)

```
Phase 0:   [student input] → INSERT students, addresses, guardians, admissions
Phase 0.5: [exam result]   → INSERT examresults → UPDATE admissions.status
Phase 1:   [review]        → UPDATE admissions.status
Phase 2:   [clearance]     → INSERT studentclearances + clearanceapprovals
Phase 3:   [assessment]    → INSERT studentassessments + charges + studentscholarships
Phase 4:   [payment]       → INSERT payments → recalc assessment balance
Phase 5:   [register]      → UPDATE enrollments.status → INSERT enrolledsubjects
Phase 6:   [documents]     → INSERT documentprintlog + enrollmentworkflow
Phase 7:   [clinic]        → INSERT clinicrecords → UPDATE workflowsteps
Phase 8:   [ID]            → INSERT idrequests + studentids → UPDATE workflowsteps
Phase 9:   [qualifying]    → INSERT examresults
```

---

## 4. Data Volume Heat Map

### Table Row Counts (sorted by size)

```
████████████████████████████████████ clearanceapprovals      376,516
███████████████████████████          enrolledsubjects         179,790
████████████████████████             charges                 135,375
███████████████████████              workflowsteps            144,475
██████████████████                   studentrequirementsub.    72,590
█████████████████                    documents                 71,446
████████████████                     addresses                 59,514
███████████████                      guardians                 55,309
█████████████                        studenteducationalbg.    42,266
████████████                         transferacademicrecords  38,913
██████                                enrollments              30,125
██████                                enrollmentworkflow       30,366
██████                                payments                 30,253
██████                                studentassessments       29,531
██████                                students                 29,548
██████                                clinicrecords            29,923
█████                                 idrequests               21,042
████                                  documentprintlog         24,174
███                                   studentids               15,721
███                                   studentclearances        17,492
██                                    admissions               11,968
██                                    studentscholarships       7,515
                                      examresults               4,569
                                      schedulemeetings          2,389
                                      schedules                   796
                                      curriculumsubjects        1,199
                                      blocks                     133
                                      educationalinstitutions     89
                                      staffusers                  40
                                      rooms                       40
                                      curriculums                 25
                                      academicunits               25
                                      offices                     22
                                      clearancerequirements       21
                                      clearanceperiods            16
                                      religions                   15
                                      subjects                   203
                                      majors                      13
                                      feetypes                    10
                                      gradescale                  10
                                      academicterms               18
                                      academicyears                6
                                      creditedsubjects             0
```

### Size by Module

| Module | Total Rows | % of DB | Tables |
|---|---|---|---|
| **Clearance** | ~394,008 | 33% | clearanceapprovals, studentclearances, clearanceperiods, clearancerequirements |
| **Enrollment Core** | ~210,286 | 18% | enrolledsubjects, enrollments, creditedsubjects |
| **Financial** | ~205,159 | 17% | charges, payments, studentassessments, studentscholarships |
| **Student Info** | ~187,585 | 16% | addresses, guardians, studenteducationalbackgrounds, students |
| **Workflow** | ~174,841 | 15% | workflowsteps, enrollmentworkflow |
| **Admission** | ~85,028 | 7% | studentrequirementsubmissions, documents, admissions, examresults |
| **Clinic** | ~29,923 | 2.5% | clinicrecords |
| **ID Management** | ~36,763 | 3% | idrequests, studentids |
| **Academic Reference** | ~1,720 | 0.15% | curriculums, curriculumsubjects, subjects, courses, majors, academicunits |
| **System** | ~114 | 0.01% | staffusers, offices, rooms, etc. |

### Insights from Data Volume

1. **clearanceapprovals dominates** (376k rows) — each student has many clearance requirements, each generating an approval row. With 17k studentclearances × ~21 requirements each = ~360k expected.
2. **enrolledsubjects** (180k) is the largest *operational* table — every subject per student per term is tracked here.
3. **charges** (135k) grows fast — ~10 fee line items per assessment × 29.5k assessments.
4. **students** has 29.5k rows, but **enrollments** has 30.1k — meaning some students enrolled multiple times (across terms).
5. **clinicrecords** matches **enrollments** closely (29.9k vs 30.1k) — nearly every enrollment goes through the clinic.
6. **creditedsubjects** has 0 rows — the transfer credit feature exists but hasn't been used yet.
7. **blocks** has 133 blocks — an average of ~22 students per block.

---

## 5. Business Rules Catalog

### Identity & Uniqueness Rules

| # | Rule | Enforced By | Tables Involved |
|---|---|---|---|
| BR1 | No two students can share a `schoolIdNumber` | UNIQUE index | students |
| BR2 | No two students can share a `username` (portal login) | UNIQUE index | students |
| BR3 | No two staff can share an `employeeNo` | UNIQUE index | staffusers |
| BR4 | No two staff can share a `username` (staff portal login) | UNIQUE index | staffusers |
| BR5 | No two payments can share an `orNumber` (official receipt) | UNIQUE index | payments |
| BR6 | No two IDs can share a `qrCode` | UNIQUE index | studentids |

### Process Flow Rules

| # | Rule | Phase | Tables |
|---|---|---|---|
| BR7 | First-year and transferee applicants **must** have an admission record before enrollment | 0→5 | admissions, enrollments |
| BR8 | Continuing students **must** have cleared obligations before enrollment | 2→5 | studentclearances, clearanceapprovals |
| BR9 | Entrance exam is required if `courses.requiresEntranceExam = 1` — two-stage for board courses: general exam (Guidance Office) then course-specific exam (department), which verifies the Guidance result first | 0.5 | courses, examresults |
| BR10 | Qualifying exam is required if `courses.requiresQualifyingExam = 1` | 9 | courses, examresults |
| BR10a | Board-course continuing students (2nd–5th year) must pass the **Retention Examination** (`examStage = 'retention'`) before receiving the enrollment form; not part of the Enrollment Workflow Process form | 1 | examresults, enrollments |
| BR11 | An assessment must exist before payment can be recorded | 3→4 | studentassessments, payments |
| BR12 | Enrollment cannot be marked `enrolled` without passing through all prior phases | 5 | enrollments (status) |
| BR13 | Workflow steps must be signed in order (`stepOrder`) | 6→8 | workflowsteps |
| BR14 | An enrollment workflow advances one step at a time | 6→8 | enrollmentworkflow (currentStep) |

### Data Integrity Rules

| # | Rule | Tables |
|---|---|---|
| BR15 | A student can have at most one `home`, one `current`, and one `permanent` address per term | addresses (addressType) |
| BR16 | Enrolled subject status transitions: `proposed` → `confirmed` → `dropped` (no reversal) | enrolledsubjects (status) |
| BR17 | Student type determines which phases apply (firstYear→skip clearance, continuing→skip admission) | enrollments (studentType) |
| BR18 | Academic standing (`regular`/`irregular`) affects how subjects are assigned | enrollments (academicStanding) |
| BR19 | Scholarship coverage cannot exceed 100% in total combined awards | scholarshiptypes (coveragePercent) |
| BR20 | A clearance period must be `open` before clearances can be processed | clearanceperiods (periodStatus) |

### Reference Data Rules

| # | Rule | Tables |
|---|---|---|
| BR21 | Each curriculum belongs to a course and optionally a major | curriculums |
| BR22 | Curriculum subjects must specify which semester they're offered in | curriculumsubjects (semesterOffered) |
| BR23 | Subjects can have prerequisites (self-referencing FK) | curriculumsubjects (prerequisiteSubjectId→subjects) |
| BR24 | Fee types can be `perUnit` (multiplied by enrolled units) or `flat` rate | feetypes (unitBasis) |
| BR25 | Admission requirements specify which student types they apply to | admissionrequirements (appliesTo) |

### Audit Rules

| # | Rule | Tables |
|---|---|---|
| BR26 | Every `evaluatedBy` and `registrarProcessedBy` must reference an active staff user | enrollments, admissions |
| BR27 | Every payment must record who processed it | payments (processedBy) |
| BR28 | Document prints must record who printed and when | documentprintlog |
| BR29 | ID validation must be timestamped with validatedBy + validatedDate | studentids |
| BR30 | Workflow step sign-offs must be timestamped | workflowsteps (signedDate) |

---

## 6. Index and Constraint Summary

### UNIQUE Indexes (Explicit Uniqueness Constraints)

| Table | Column(s) | Purpose |
|---|---|---|
| students | `schoolIdNumber` | Every student has a unique visible ID |
| students | `username` | Portal login uniqueness |
| staffusers | `employeeNo` | Employee number uniqueness |
| staffusers | `username` | Staff portal login uniqueness |
| payments | `orNumber` | Official receipt number traceability |
| studentids | `qrCode` | QR code uniqueness for physical ID scanning |
| **studentclearances** | **(studentId, clearancePeriodId)** | One clearance record per student per clearance period |
| **studentscholarships** | **(studentId, scholarshipTypeId, termId)** | One scholarship award per type per student per term |

### AUTO_INCREMENT Primary Keys

45 of 46 tables use `int` auto-increment primary keys. The only exception is **`clinicrecords.clinicRecordId`** — it has no AUTO_INCREMENT and uses a manually-assigned PK (recommended to add AUTO_INCREMENT for consistency). The current auto-increment values indicate data volume:

| Table | Next AI Value | Notes |
|---|---|---|
| enrollments | 30,001 | 30k enrollments served |
| students | 29,549 | ~29.5k students in the system |
| enrolledsubjects | 180,110 | 180k subject enrollments |
| clinicrecords | — | No auto-increment (only table without it) |

### Composite Indexes

| Table | Index Name | Columns | Purpose |
|---|---|---|---|
| schedules | `idx_schedules_lookup` | blockId, subjectId | Fast lookup of schedule by block + subject |

### FOREIGN KEY Constraints

All **78 FK relationships** documented in the relationship matrix (above) are enforced as actual database-level FOREIGN KEY constraints, all defined via `ALTER TABLE ADD CONSTRAINT`. This means referential integrity is guaranteed at the database engine level — no orphan child rows can exist.

---

## 7. Schema Change Log

| Date | Change | Rationale |
|---|---|---|
| Jul 2026 | Added `clinicrecords` table | Phase 7 — Clinic process formalized |
| Jul 2026 | Added `schedulemeetings` table | Separated meeting times from schedule metadata (normalization) |
| Jul 2026 | Renamed `sections` to `blocks` | Reflected that sections = blocking sections, not class sections |
| Jul 2026 | Completed block rename: `sectionId`→`blockId`, `sectionName`→`blockName`, FK/index names updated | Column-level terminology aligned with `blocks` table name |
| Jul 2026 | Added `retention` stage to `examresults.examStage` | Phase 1 board-course retention examination gate |
| Jul 2026 | Removed Physical Examination from `admissionrequirements` | Physical exam moved to Phase 7 Clinic (`clinicrecords`) |
| Jul 2026 | Added `School Grant (Free Tuition)` scholarship type | Free-tuition school — all college students are school scholars once enrolled |
| Jul 2026 | Renamed office 2 to `Accounting` (was `Cashier`), added `ID Office` (officeId 22) | Office terminology matches school usage; Phase 8 ID office formalized |
| Jul 2026 | Added `enrollmentworkflow` + `workflowsteps` | Physical office routing form tracking |
| Jul 2026 | Added `studentids` table | Decoupled ID cards from ID requests (1 request = 1 card but cards have lifecycle) |

---

*Generated: July 2026 — Matches enrollment database schema (46 tables)*
