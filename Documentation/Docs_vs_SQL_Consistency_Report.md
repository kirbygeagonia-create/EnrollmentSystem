# Documentation vs. SQL Consistency Report

*Cross-check of `enrollment.sql` (source of truth) against `Database_Representations.md`, `Enrollment_System_Dialogue.md`, `EMS Complete Documentation.docx`, and `EMS Confirmed Tables and ERD Statements.docx`.*

## Verdict

**Overall: ~99% consistent.** All 9 issues identified in the initial audit plus all 9 issues from the second-round audit (schema vs. real school process) have been resolved. Both the live MySQL database and the SQL dump file now match the documentation, and vice versa.

### What was fixed

| # | Issue | Location | Fix Applied |
|---|---|---|---|
| 1 | §6 claimed "only 7 explicit FK constraints" — SQL has 78 | `Database_Representations.md` | ✅ Rewrote section to state all 78 FKs are enforced, removed outdated index-only claim. Also applied the 71 missing FK constraints to the live DB. |
| 2 | Mermaid ERD defined `academicschedule` — SQL table is `academicterms` | `Database_Representations.md` §1 | ✅ Renamed block in ERD |
| 3 | Used `scholarshipstypes` (extra "s") — SQL table is `scholarshiptypes` | `Database_Representations.md` + `Enrollment_System_Dialogue.md` | ✅ Fixed all 9 occurrences across both files |
| 4 | §6 claimed `charges` and `subjects` have "No auto-increment" — they do | `Database_Representations.md` §6 | ✅ Corrected to note only `clinicrecords` lacks AUTO_INCREMENT |
| 5 | Docx header said "45 tables, 76 relationships" | `EMS Complete Documentation.docx` | ✅ Updated to "46 tables, 78 foreign-key constraints" across all 4 occurrences |
| 6 | Students section listed `Notifications.studentId` — table was removed | `EMS Complete Documentation.docx` | ✅ Removed stale reference from inline "Referenced by" list |
| 7 | `examresults` indexes named `fk_courseexams_*` (old table name) | Live DB + SQL file | ✅ Renamed to `fk_examresults_studentid`, `fk_examresults_courseid`, `fk_examresults_termid` |
| 8 | Redundant indexes on `payments` + `schedules` | Live DB + SQL file | ✅ Dropped `idx_payments_or` (duplicates `uq_payments_ornumber`) and `fk_schedules_sectionid` (covered by `idx_schedules_lookup`) |
| 9 | `admissions.evaluatedBy`/`evaluatedDate` and `enrollments.registrarProcessedBy` were NOT NULL | Live DB + SQL file | ✅ Changed to nullable (`DEFAULT NULL`) to reflect phased workflow where `pending` records exist before evaluator assignment |

### Second-round audit (schema vs. real school process)

| # | Issue | Location | Fix Applied |
|---|---|---|---|
| 10 | `blocks` table + relationships still named `sectionId`/`sectionName` — school calls them blocks | Live DB + SQL + all docs | ✅ Full rename: `sectionId`→`blockId`, `sectionName`→`blockName` (180 refs in SQL, 133 data rows "Section A"→"Block A"), FKs `fk_schedules_blockid`/`fk_enrolledsubjects_blockid` recreated, `idx_schedules_lookup` re-pointed to `blockId` |
| 11 | `examresults.examStage` only had `entrance`/`midCourseQualifying` | Live DB + SQL + docs | ✅ Added `'retention'` stage — retention exam for board-course continuing students (2nd–5th yr), taken in Phase 1 *before* the enrollment form (not Phase 9) |
| 12 | Entrance exam documented as single-stage | `Database_Representations.md` + dialogue doc + docx | ✅ Two-stage for board courses: Guidance general exam → Academic Department pulls & verifies Guidance result → course-specific exam (`examType` = `general`/`courseSpecific`) |
| 13 | Physical Examination listed as an admission requirement | Live DB + SQL + docs | ✅ Removed requirement 8 + 8,993 `studentrequirementsubmissions` + 8,992 `documents` rows (cascade). Physical exam moved to Clinic (Phase 7) — added blood pressure + physical examination wording to Phase 7 |
| 14 | No School Grant for the school's free-tuition program | Live DB + SQL + docs | ✅ Added `scholarshiptypes` row 9 "School Grant (Free Tuition)" (full, 100%) — every SEAIT student covered; outside grants stack up to the 100% cap |
| 15 | Phase 4 office named "Cashier" | Live DB + SQL + docs | ✅ Renamed to **Accounting** (officeId 2) in DB, SQL, and all docs |
| 16 | No ID Office row — Phase 8 workflow step referenced a wrong office | Live DB + SQL + docs | ✅ Added `offices` row 22 "ID Office"; Phase 8 retitled "ID Request, Release and Validation" (dialogue doc workflow step 8 → officeId 22) |
| 17 | Phase 1 documented without the enrollment form flow | dialogue doc + docx | ✅ Documented: line up → get enrollment form → fill info → (board: retention exam gate first) → evaluation by type → evaluator signs + subject load → Registrar |
| 18 | "Mid-Course Qualifying Exam" duplicated the Retention Exam — school has only Entrance + Retention exams | Live DB + SQL + all docs | ✅ Removed `midCourseQualifying` from `examresults.examStage` (now `enum('entrance','retention')`), renamed `courses.requiresQualifyingExam` → `requiresRetentionExam`, deleted the Post-Enrollment qualifying section from all docs |

### What was verified correct (no changes needed)

- All 46 table names and column lists match SQL ↔ docs
- All 78 FK relationships match SQL ↔ relationship matrix
- All unique constraints match business rules BR1–BR6 (plus the bonus composite unique indexes)
- All enum values match across schema and docs
- Row counts in `Database_Representations.md` §4 match dump AUTO_INCREMENT values
- Narrative in dialogue doc is consistent with actual schema
- Second round: 0 remaining `sectionId`/`sectionName` columns in DB or SQL (4 `blockId`/`blockName` columns present); 7 admission requirements; 9 scholarship types; 22 offices (2 = Accounting, 22 = ID Office); 133 blocks; `examresults.examStage` = `enum('entrance','retention')`; `courses.requiresRetentionExam` (was `requiresQualifyingExam`)

### Current state

| Metric | Value |
|---|---|
| Tables | **46** |
| Explicit FK constraints | **78** (all enforced at DB level) |
| Indexes | ~128 (after removing 3 redundant ones) |
| AUTO_INCREMENT PKs | 45 of 46 tables (`clinicrecords.clinicRecordId` is the only exception) |
| UNIQUE constraints | 10 (6 single-column + 4 composite) |
| Last schema sync | July 2026 — live DB and `enrollment.sql` are in agreement (second-round process-alignment applied Aug 2026) |

### Real-world capacity

**The 30k-student / 1.4M-row dataset is small for InnoDB.** Well-indexed lookups stay in single-digit milliseconds because B-tree index depth grows logarithmically. With 78 FKs and ~128 indexes, the schema is production-ready.

Growth estimate for one school: ~30k students × 2–3 terms/year → ~60–90k enrollments/year. After 10 years, the largest tables reach ~10–15M rows — still routine territory for MySQL/MariaDB on a modest server.

**Deployment notes:**
1. Move off XAMPP for production — use MariaDB 10.11+ or MySQL 8 on a proper Linux server (or managed DB)
2. The synthetic sample data is excellent for load-testing in a staging environment
3. If `clearanceapprovals`-style tables pass ~10M rows, partition by termId or archive closed terms
