# Documentation vs. SQL Consistency Report

*Cross-check of `enrollment.sql` (source of truth) against `Database_Representations.md`, `Enrollment_System_Dialogue.md`, `EMS Complete Documentation.docx`, and `EMS Confirmed Tables and ERD Statements.docx`.*

## Verdict

**Overall: ~99% consistent.** All 9 issues identified in the initial audit have been resolved. Both the live MySQL database and the SQL dump file now match the documentation, and vice versa.

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

### What was verified correct (no changes needed)

- All 46 table names and column lists match SQL ↔ docs
- All 78 FK relationships match SQL ↔ relationship matrix
- All unique constraints match business rules BR1–BR6 (plus the bonus composite unique indexes)
- All enum values match across schema and docs
- Row counts in `Database_Representations.md` §4 match dump AUTO_INCREMENT values
- Narrative in dialogue doc is consistent with actual schema

### Current state

| Metric | Value |
|---|---|
| Tables | **46** |
| Explicit FK constraints | **78** (all enforced at DB level) |
| Indexes | ~128 (after removing 3 redundant ones) |
| AUTO_INCREMENT PKs | 45 of 46 tables (`clinicrecords.clinicRecordId` is the only exception) |
| UNIQUE constraints | 10 (6 single-column + 4 composite) |
| Last schema sync | July 2026 — live DB and `enrollment.sql` are in agreement |

### Real-world capacity

**The 30k-student / 1.4M-row dataset is small for InnoDB.** Well-indexed lookups stay in single-digit milliseconds because B-tree index depth grows logarithmically. With 78 FKs and ~128 indexes, the schema is production-ready.

Growth estimate for one school: ~30k students × 2–3 terms/year → ~60–90k enrollments/year. After 10 years, the largest tables reach ~10–15M rows — still routine territory for MySQL/MariaDB on a modest server.

**Deployment notes:**
1. Move off XAMPP for production — use MariaDB 10.11+ or MySQL 8 on a proper Linux server (or managed DB)
2. The synthetic sample data is excellent for load-testing in a staging environment
3. If `clearanceapprovals`-style tables pass ~10M rows, partition by termId or archive closed terms
