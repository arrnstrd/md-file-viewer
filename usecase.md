# System Use Case Diagram Documentation

> **Document Type:** Capstone / Thesis Manuscript Systems Documentation  
> **System Title:** Web-Based QR Student Attendance Monitoring with Centralized Academic Management System for Concepcion Integrated School  
> **Repository Authority:** Grounded strictly in implemented routes, controllers, middleware, and database models of the `CIS_CAPSTONE` codebase.  
> **Diagram Specification:** UML 2.0 Use Case Model (System Boundary, Primary & Secondary Actors, Use Cases, `<<include>>`, and `<<extend>>` relationships).

---

## 1. Actor Identification & System Boundary Justification

Based on the validated codebase audit (`EnsureUserHasRole.php`, `routes/`, and `documentation/05_point_of_views/`), actors are strictly categorized according to actual code capabilities:

### 1.1 Primary Human Actors (Direct Initiators)

| Actor | Access Medium | Authentication / Role Constant | Description & Scope of Authority |
|---|---|---|---|
| **Super Admin** | Web Browser | `role:super_admin` (`User::ROLE_SUPER_ADMIN`) | Apex governance authority. Sole role permitted to provision and manage user accounts (teachers, admins), inspect immutable security audit logs, and track administrative activity. |
| **School Admin** | Web Browser | `role:admin` (`User::ROLE_ADMIN`) | Operational management. Configures academic structures (school years, sections, subjects, teaching assignments, schedule configs), manages student enrollments, runs bulk Excel imports, generates student QR badge cards, monitors gate attendance, and manages email delivery logs. |
| **Teacher** | Web Browser | `role:teacher` (`User::ROLE_TEACHER`) | Instructional authority. Performs classroom period-level attendance verification, enters trimester assessment marks (Written Works, Performance Tasks, Quarterly Assessment), imports DepEd E-Class Record spreadsheets, monitors at-risk students, records academic notes, and generates DepEd SF-9 report cards. |
| **Scanner Operator** | Kiosk Hardware / Browser | `role:scanner_operator` (`User::ROLE_SCANNER_OPERATOR`) | Terminal facilitator. Account dedicated solely to authenticating the campus gate kiosk hardware, keeping the fullscreen QR scanner active, and reviewing daily time-in/time-out entry logs and analytics. |
| **Student** | Physical QR Badge | None (Physical Badge Bearer) | Physical gate participant. Presents issued physical QR badge to the autonomous kiosk optical reader to record campus entry and exit. **Has no web portal, login account, or direct database access.** |

### 1.2 Secondary / Supporting Actors (External Systems & Stakeholders)

| Actor | Role / Integration Type | Code Evidence | Interaction |
|---|---|---|---|
| **Parent / Guardian** | External Stakeholder / Notification Recipient | `guardians.email`, `AttendanceNotificationMail` | Passive recipient. Receives automated transactional email notifications upon student gate scan (arrival/dismissal timestamp, status). **Has no web portal or login account.** |
| **External SMTP Mail Server** | External Technical Infrastructure | `config/mail.php`, `App\Services\EmailLogService` | Technical subsystem. Dispatches transactional attendance alert emails to guardians and password recovery emails to staff, returning delivery confirmation or failure status. |

---

## 2. Master System Use Case Diagram

The following diagram defines the high-level system boundary, depicting how all five primary actors and two secondary actors interact with the core functional modules of the application.

```mermaid
flowchart LR

    %% ==========================================
    %% ACTORS
    %% ==========================================
    subgraph ACTORS_LEFT ["Primary Human Actors"]
        direction TB
        SA["fa:fa-user-shield Super Admin"]
        ADM["fa:fa-user-tie School Admin"]
        TCH["fa:fa-chalkboard-teacher Teacher"]
        OP["fa:fa-desktop Scanner Operator"]
        STU["fa:fa-id-badge Student\n(Badge Bearer)"]
    end

    subgraph ACTORS_RIGHT ["Secondary / External Actors"]
        direction TB
        GRD["fa:fa-envelope-open-text Parent / Guardian\n(Email Recipient)"]
        SMTP["fa:fa-server External SMTP Server"]
    end

    %% ==========================================
    %% SYSTEM BOUNDARY
    %% ==========================================
    subgraph SYS ["System Boundary: CIS Capstone Application"]
        direction TB

        %% Super Admin Module
        subgraph MOD_GOV ["1. Governance & Access Control"]
            UC_AUTH(["Log In / Log Out / Password Recovery"])
            UC_USERS(["Manage User Accounts & Roles"])
            UC_AUDIT(["Inspect Security Audit & Activity Logs"])
        end

        %% School Admin Setup & Management
        subgraph MOD_ACAD ["2. Academic Setup & Student Management"]
            UC_ACAD_SETUP(["Configure Academic Year, Sections & Subjects"])
            UC_ASSIGN(["Manage Teaching Assignments"])
            UC_SCHED(["Configure Operating Schedule Windows"])
            UC_STUDENTS(["Manage Student & Guardian Records"])
            UC_BULK_IMPORT(["Import Students via Bulk Spreadsheet"])
            UC_QR_GEN(["Generate & Print Student QR Badges"])
        end

        %% Gate QR Attendance
        subgraph MOD_GATE ["3. Gate QR Attendance Kiosk"]
            UC_KIOSK(["Operate Gate QR Station Kiosk"])
            UC_SCAN(["Self-Scan QR Badge (Entry/Exit)"])
            UC_GATE_LOGS(["Monitor Daily In/Out Logs & Analytics"])
            UC_EMAIL_LOGS(["Monitor & Retry Guardian Notification Emails"])
        end

        %% Classroom Attendance
        subgraph MOD_ROOM ["4. Classroom Verification"]
            UC_ROOM_VERIFY(["Verify Classroom Period Attendance"])
            UC_ROOM_HIST(["View Attendance Verification Roster History"])
        end

        %% Academic Grading & DepEd Compliance
        subgraph MOD_GRADE ["5. Grading, DepEd Records & Analytics"]
            UC_GRADE_ENTRY(["Input Assessment Scores (WW, PT, QA)"])
            UC_DEPED_IMPORT(["Import DepEd E-Class Record Spreadsheet"])
            UC_AT_RISK(["Monitor At-Risk Students & Log Remarks"])
            UC_NOTES(["Record Student Academic Notes"])
            UC_REPORTS(["Generate DepEd Form SF-9 / Class Records"])
        end

    end

    %% ==========================================
    %% ASSOCIATIONS (Actor to Use Case)
    %% ==========================================
    
    %% Super Admin
    SA --- UC_AUTH
    SA --- UC_USERS
    SA --- UC_AUDIT

    %% School Admin
    ADM --- UC_AUTH
    ADM --- UC_ACAD_SETUP
    ADM --- UC_ASSIGN
    ADM --- UC_SCHED
    ADM --- UC_STUDENTS
    ADM --- UC_BULK_IMPORT
    ADM --- UC_QR_GEN
    ADM --- UC_KIOSK
    ADM --- UC_GATE_LOGS
    ADM --- UC_EMAIL_LOGS
    ADM --- UC_ROOM_HIST

    %% Scanner Operator
    OP --- UC_AUTH
    OP --- UC_KIOSK
    OP --- UC_GATE_LOGS

    %% Student (Physical interaction at Kiosk)
    STU --- UC_SCAN

    %% Teacher
    TCH --- UC_AUTH
    TCH --- UC_ROOM_VERIFY
    TCH --- UC_ROOM_HIST
    TCH --- UC_GRADE_ENTRY
    TCH --- UC_DEPED_IMPORT
    TCH --- UC_AT_RISK
    TCH --- UC_NOTES
    TCH --- UC_REPORTS

    %% External Systems & Stakeholders
    UC_SCAN -.->|&lt;&lt;notify&gt;&gt;| GRD
    UC_SCAN -.->|&lt;&lt;dispatch&gt;&gt;| SMTP
    UC_EMAIL_LOGS -.->|&lt;&lt;resend&gt;&gt;| SMTP
    UC_AUTH -.->|&lt;&lt;reset link&gt;&gt;| SMTP
```

---

## 3. Subsystem Detailed Use Case Models

### 3.1 Gate QR Attendance Subsystem (With `<<include>>` and `<<extend>>`)

The QR Attendance scanning workflow represents a multi-step verification pipeline triggered when a student presents their physical QR badge.

```mermaid
flowchart LR

    %% Actors
    STU["Student (Badge Bearer)"]
    OP["Scanner Operator"]
    ADM["School Admin"]
    GRD["Parent / Guardian"]
    SMTP["External SMTP Server"]

    subgraph GATE_BOUNDARY ["Subsystem: Gate QR Attendance Kiosk"]
        direction TB

        UC_RUN_KIOSK(["Operate Fullscreen QR Kiosk"]):::uc
        UC_SCAN_ENTRY(["Scan QR Badge for Attendance"]):::uc
        
        %% Included Use Cases
        UC_VAL_QR(["Validate QR Code & Token"]):::uc_inc
        UC_RES_ENROLL(["Resolve Active Student & Enrollment"]):::uc_inc
        UC_EVAL_SCHED(["Resolve Schedule Config & Tardiness"]):::uc_inc
        UC_PERSIST_LOG(["Persist AttendanceLog & QrAttendance"]):::uc_inc
        UC_NOTIF_MAIL(["Queue Guardian Email Notification"]):::uc_inc
        
        %% Extended Use Cases
        UC_COOLDOWN(["Handle Anti-Spam Scan Cooldown"]):::uc_ext
        UC_OFF_SCHED(["Log Off-Schedule / Discrepancy Remark"]):::uc_ext

        %% Log & Analytics
        UC_VIEW_LOGS(["View Daily Time-In / Time-Out Logs"]):::uc
        UC_VIEW_ANALYTICS(["Inspect Attendance Analytics"]):::uc
        UC_EXPORT_PDF(["Export In/Out Logs as PDF"]):::uc
        UC_RETRY_EMAIL(["Retry Failed Email Deliveries"]):::uc
    end

    %% Actor Connections
    OP --- UC_RUN_KIOSK
    OP --- UC_VIEW_LOGS
    OP --- UC_VIEW_ANALYTICS
    OP --- UC_EXPORT_PDF

    ADM --- UC_RUN_KIOSK
    ADM --- UC_VIEW_LOGS
    ADM --- UC_VIEW_ANALYTICS
    ADM --- UC_EXPORT_PDF
    ADM --- UC_RETRY_EMAIL

    STU --- UC_SCAN_ENTRY

    %% Include Relationships
    UC_SCAN_ENTRY -.->|&lt;&lt;include&gt;&gt;| UC_VAL_QR
    UC_SCAN_ENTRY -.->|&lt;&lt;include&gt;&gt;| UC_RES_ENROLL
    UC_SCAN_ENTRY -.->|&lt;&lt;include&gt;&gt;| UC_EVAL_SCHED
    UC_SCAN_ENTRY -.->|&lt;&lt;include&gt;&gt;| UC_PERSIST_LOG
    UC_SCAN_ENTRY -.->|&lt;&lt;include&gt;&gt;| UC_NOTIF_MAIL

    %% Extend Relationships
    UC_COOLDOWN -.->|&lt;&lt;extend&gt;&gt;| UC_SCAN_ENTRY
    UC_OFF_SCHED -.->|&lt;&lt;extend&gt;&gt;| UC_SCAN_ENTRY

    %% External Notifications
    UC_NOTIF_MAIL -.->|Transmits Email| SMTP
    SMTP -.->|Delivers Notice| GRD
    UC_RETRY_EMAIL -.->|Re-dispatches Mail| SMTP

    classDef uc fill:#e1f5fe,stroke:#0288d1,stroke-width:1.5px;
    classDef uc_inc fill:#e8f5e9,stroke:#2e7d32,stroke-width:1.5px,stroke-dasharray: 4 2;
    classDef uc_ext fill:#fff3e0,stroke:#e65100,stroke-width:1.5px,stroke-dasharray: 4 2;
```

---

### 3.2 Classroom Period Attendance Verification Subsystem

Classroom attendance gives the assigned subject teacher legal verification authority over student presence during class hours, cross-referencing campus gate scans.

```mermaid
flowchart LR

    %% Actors
    TCH["Teacher"]
    ADM["School Admin"]

    subgraph ROOM_BOUNDARY ["Subsystem: Classroom Verification"]
        direction TB

        UC_OPEN_ROSTER(["Open Subject Section Attendance Roster"]):::uc
        UC_VERIFY_STUDENT(["Verify Period Attendance Status"]):::uc
        UC_BULK_VERIFY(["Bulk Verify Section Attendance"]):::uc
        UC_AUDIT_HISTORY(["View Verification Audit History"]):::uc

        %% Included Use Cases
        UC_CHECK_GATE(["Cross-Reference Gate Station Time-In"]):::uc_inc
        UC_FLAG_CUTTING(["Flag Discrepancy (Not in Classroom / Cutting)"]):::uc_inc
        UC_RECORD_AUDIT(["Persist Verification History Trail"]):::uc_inc
    end

    TCH --- UC_OPEN_ROSTER
    TCH --- UC_VERIFY_STUDENT
    TCH --- UC_BULK_VERIFY
    TCH --- UC_AUDIT_HISTORY

    ADM --- UC_AUDIT_HISTORY

    %% Dependencies
    UC_OPEN_ROSTER -.->|&lt;&lt;include&gt;&gt;| UC_CHECK_GATE
    UC_VERIFY_STUDENT -.->|&lt;&lt;include&gt;&gt;| UC_RECORD_AUDIT
    UC_VERIFY_STUDENT -.->|&lt;&lt;include&gt;&gt;| UC_FLAG_CUTTING

    classDef uc fill:#e1f5fe,stroke:#0288d1,stroke-width:1.5px;
    classDef uc_inc fill:#e8f5e9,stroke:#2e7d32,stroke-width:1.5px,stroke-dasharray: 4 2;
```

---

### 3.3 Academic Grading & DepEd Compliance Subsystem

Teachers record scores, compute DepEd transmuted grades, evaluate early-warning risk scores, and generate official forms.

```mermaid
flowchart LR

    %% Actors
    TCH["Teacher"]

    subgraph GRADING_BOUNDARY ["Subsystem: Grading & Academic Management"]
        direction TB

        UC_INPUT_SCORE(["Record Raw Assessment Scores"]):::uc
        UC_IMPORT_DEPED(["Import DepEd E-Class Record (XLSX)"]):::uc
        UC_VIEW_DASHBOARD(["View Subject Grading Dashboard"]):::uc
        UC_MONITOR_RISK(["Inspect At-Risk Students"]):::uc
        UC_LOG_REMARK(["Log At-Risk Intervention Remarks"]):::uc
        UC_ACAD_NOTE(["Record Student Academic Notes"]):::uc
        UC_GEN_SF9(["Generate DepEd Form SF-9 Report Card"]):::uc

        %% Included Calculations
        UC_WEIGHT_RESOLVE(["Resolve Subject Weights (WW, PT, QA)"]):::uc_inc
        UC_TRANSMUTE(["Apply DepEd Transmutation Table (Pass=75)"]):::uc_inc
        UC_EVAL_RISK(["Calculate Student Risk Index Score"]):::uc_inc
    end

    TCH --- UC_INPUT_SCORE
    TCH --- UC_IMPORT_DEPED
    TCH --- UC_VIEW_DASHBOARD
    TCH --- UC_MONITOR_RISK
    TCH --- UC_LOG_REMARK
    TCH --- UC_ACAD_NOTE
    TCH --- UC_GEN_SF9

    %% Relationships
    UC_INPUT_SCORE -.->|&lt;&lt;include&gt;&gt;| UC_WEIGHT_RESOLVE
    UC_INPUT_SCORE -.->|&lt;&lt;include&gt;&gt;| UC_TRANSMUTE
    UC_INPUT_SCORE -.->|&lt;&lt;include&gt;&gt;| UC_EVAL_RISK
    UC_IMPORT_DEPED -.->|&lt;&lt;include&gt;&gt;| UC_TRANSMUTE
    UC_MONITOR_RISK -.->|&lt;&lt;include&gt;&gt;| UC_LOG_REMARK

    classDef uc fill:#e1f5fe,stroke:#0288d1,stroke-width:1.5px;
    classDef uc_inc fill:#e8f5e9,stroke:#2e7d32,stroke-width:1.5px,stroke-dasharray: 4 2;
```

---

### 3.4 Academic Setup & Student Enrollment Subsystem

School Administrators construct the foundational data required by all downstream subsystems.

```mermaid
flowchart LR

    %% Actors
    ADM["School Admin"]

    subgraph SETUP_BOUNDARY ["Subsystem: Academic Setup & Enrollment"]
        direction TB

        UC_SCHOOL_YEAR(["Manage Active School Year"]):::uc
        UC_SECTIONS(["Manage Grade Levels & Sections"]):::uc
        UC_SUBJECTS(["Manage Subject Curriculum Offerings"]):::uc
        UC_TEACHING_ASSIGN(["Assign Faculty to Section & Subject"]):::uc
        UC_SCHED_CONFIG(["Configure Gate Schedule Windows"]):::uc
        UC_MANAGE_STUDENT(["Manage Student & Guardian Details"]):::uc
        UC_IMPORT_EXCEL(["Bulk Upload Student Roster (XLSX/CSV)"]):::uc
        UC_RESOLVE_ISSUES(["Acknowledge & Resolve Import Issues"]):::uc_ext
        UC_PRINT_QR(["Generate & Print Student QR Cards"]):::uc
    end

    ADM --- UC_SCHOOL_YEAR
    ADM --- UC_SECTIONS
    ADM --- UC_SUBJECTS
    ADM --- UC_TEACHING_ASSIGN
    ADM --- UC_SCHED_CONFIG
    ADM --- UC_MANAGE_STUDENT
    ADM --- UC_IMPORT_EXCEL
    ADM --- UC_PRINT_QR

    %% Extend
    UC_RESOLVE_ISSUES -.->|&lt;&lt;extend&gt;&gt;| UC_IMPORT_EXCEL

    classDef uc fill:#e1f5fe,stroke:#0288d1,stroke-width:1.5px;
    classDef uc_ext fill:#fff3e0,stroke:#e65100,stroke-width:1.5px,stroke-dasharray: 4 2;
```

---

### 3.5 System Governance & Security Subsystem

The Super Admin exercises exclusive control over institutional credentials and audit traceability.

```mermaid
flowchart LR

    %% Actors
    SA["Super Admin"]
    SMTP["External SMTP Server"]

    subgraph GOV_BOUNDARY ["Subsystem: System Governance & Security"]
        direction TB

        UC_PROV_TEACHER(["Provision New Teacher Accounts"]):::uc
        UC_MANAGE_USERS(["Manage User States (Deactivate/Reactivate)"]):::uc
        UC_RESEND_INVITE(["Resend Onboarding Invitation Email"]):::uc
        UC_AUDIT_LOGS(["Inspect Security Audit Logs"]):::uc
        UC_ACTIVITY_LOGS(["Review Recent Admin Activity Trail"]):::uc
        UC_EXPORT_AUDIT(["Export Audit Logs as PDF or Excel"]):::uc
    end

    SA --- UC_PROV_TEACHER
    SA --- UC_MANAGE_USERS
    SA --- UC_RESEND_INVITE
    SA --- UC_AUDIT_LOGS
    SA --- UC_ACTIVITY_LOGS
    SA --- UC_EXPORT_AUDIT

    UC_RESEND_INVITE -.->|Dispatches Link| SMTP

    classDef uc fill:#e1f5fe,stroke:#0288d1,stroke-width:1.5px;
```

---

## 4. Comprehensive Use Case Traceability Matrix

Every use case is grounded in specific Laravel routes, controllers, middleware, and database models:

| Subsystem | Use Case Title | Primary Actor | Trigger / Route | Controller / Service Grounding | Target Models / Stores |
|---|---|---|---|---|---|
| **Auth** | User Login | All Staff Roles | `POST /login` | `AuthController@login` | `users`, `login_logs` |
| **Auth** | Password Recovery | All Staff Roles | `POST /forgot-password` | `ForgotPasswordController@sendResetLinkEmail` | `password_reset_tokens` |
| **Gov** | Provision Teacher Account | Super Admin | `POST /teachers` | `TeacherProvisioningController@store` | `users`, `teachers` |
| **Gov** | Manage User Accounts | Super Admin | `GET /users`, `POST /api/users/*` | `UserManagementController` | `users`, `admin_activity_logs` |
| **Gov** | Inspect Security Audit Logs | Super Admin | `GET /security-audit-log` | `SecurityAuditLogController@index` | `login_logs` |
| **Gov** | Export Audit Reports (PDF/Excel) | Super Admin | `GET /security-audit-log/download[-pdf]` | `SecurityAuditLogController@download*` | `login_logs` |
| **Setup** | Configure School Year | School Admin | `POST /school-years` | `SchoolYearController@store` | `school_years` |
| **Setup** | Manage Sections & Advisors | School Admin | `POST /sections`, `PUT /sections/{id}/advisor`| `SectionController` | `sections`, `teachers` |
| **Setup** | Manage Subject Catalog | School Admin | `POST /subjects` | `SubjectController@store` | `subjects` |
| **Setup** | Configure Teaching Assignments | School Admin | `POST /teaching-assignments` | `TeachingAssignmentController@store` | `teaching_assignments` |
| **Setup** | Configure Scan Schedule Windows | School Admin | `POST /schedule-configuration` | `ScheduleConfigController@store` | `schedule_configs` |
| **Setup** | Manage Student Profiles | School Admin | `POST /students`, `PUT /students/{id}` | `StudentManagementController` | `students`, `guardians` |
| **Setup** | Bulk Upload Student Roster | School Admin | `POST /import/upload` | `BulkImportController@upload`, `BulkImportService` | `bulk_imports`, `students`, `enrollments` |
| **Setup** | Resolve Bulk Import Issues | School Admin | `GET /import/{id}/issues` | `BulkImportController@issues` | `bulk_import_issues` |
| **Setup** | Generate & Download QR Cards | School Admin | `GET /students/{id}/qr/download` | `QrCodeController@download` | `qr_codes` |
| **Gate** | Operate Kiosk QR Station | Scanner Operator, Admin | `GET /qr-station` | `QrStationController@index` | `qr_codes`, `schedule_configs` |
| **Gate** | Process Self-Scan QR Check-In/Out| Student (Badge) | `POST /qr-station/scan` | `ScanController@scan`, `QrScanService` | `attendance_logs`, `qr_attendances`, `scan_remarks` |
| **Gate** | Inspect In/Out Logs & Analytics | Scanner Operator, Admin | `GET /time-in-time-out-history` | `AttendanceLogController@index` | `attendance_logs`, `qr_attendances` |
| **Gate** | Monitor & Retry Guardian Emails | School Admin | `GET /emails`, `POST /emails/{id}/retry` | `EmailLogController` | `email_logs` |
| **Room** | Open Period Attendance Roster | Teacher | `GET /teacher/room-attendance/{section}` | `TeacherRoomAttendanceController@show` | `teaching_assignments`, `enrollments` |
| **Room** | Verify Student Period Presence | Teacher | `POST /teacher/room-attendance/{sec}/{enr}/verify` | `TeacherRoomAttendanceController@verify` | `attendance_verifications`, `attendance_verification_histories` |
| **Room** | Bulk Verify Section Attendance | Teacher | `POST /teacher/room-attendance/{sec}/bulk-verify` | `TeacherRoomAttendanceController@bulkVerify` | `attendance_verifications` |
| **Grade** | Input Raw Assessment Scores | Teacher | `POST /teacher/grading-system/grade-sheet/score` | `GradeSheetController@storeScore`, `GradingService` | `student_assessment_scores`, `term_grades` |
| **Grade** | Import DepEd E-Class Record | Teacher | `POST /teacher/grading-system/import-data/process`| `DepEdClassRecordImportController@process` | `assessments`, `student_assessment_scores` |
| **Grade** | Monitor At-Risk Students | Teacher | `GET /teacher/grading-system/at-risk` | `AtRiskController@index`, `RiskScoreService` | `student_assessment_scores`, `attendance_verifications` |
| **Grade** | Log At-Risk Remarks | Teacher | `POST /teacher/grading-system/at-risk/{enr}/remarks`| `AtRiskController@storeRemark` | `at_risk_remarks` |
| **Grade** | Record Student Academic Notes | Teacher | `POST /teacher/grading-system/student-profile/{enr}/notes`| `StudentProfileSearchController@storeAcademicNote`| `academic_notes` |
| **Grade** | Generate DepEd SF-9 Report Card | Teacher | `GET /teacher/grading-system/reports/student-academic-record`| `ReportsController@studentAcademicRecord` | `term_grades`, `students`, `subjects` |

---

## 5. Architectural Defense Notes (For Capstone Panel Presentation)

During an academic thesis or capstone defense, the following technical defenses must be articulated:

1. **Why is the Student not modeled as a web user?**
   - The repository explicitly omits any student login controller or authentication guard.
   - The student interacts with the system solely as a physical QR token bearer at the autonomous optical scan kiosk terminal.
2. **Why is the Parent/Guardian modeled as a Secondary Actor?**
   - Parents do not log in or manage credentials. They act strictly as external consumers receiving automated transactional notification emails dispatched via the external SMTP relay upon gate check-in/out.
3. **What makes the Scanner Operator distinct from a Guard?**
   - In code, `ScannerOperator` is a standard web account restricted to `/qr-station` and `/time-in-time-out-history`.
   - Its primary function is to initialize and monitor the autonomous self-service station terminal without exposing curriculum or student grading data.
4. **Why are Super Admin and School Admin separate actors?**
   - Separation of administrative duties: In `routes/super-admin.php`, only `super_admin` can hit `POST /teachers` or access `SecurityAuditLogController`. School administrators are restricted to curriculum orchestration and student enrollment.
5. **How is DepEd Compliance verified without an online API?**
   - DepEd integration is implemented via local parsing of standard official Excel spreadsheets (`DepEdClassRecordImportController` and PhpSpreadsheet) adhering to DepEd Order No. 8, s. 2015/2016 for weight calculations (Written Works, Performance Tasks, Quarterly Assessment) and grade transmutations. No external DepEd Web API is invoked.
