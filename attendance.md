# Automated QR Attendance and Classroom Verification System Flowchart

> **Document Type:** Capstone / Thesis Manuscript Systems Documentation  
> **Target Audience:** Academic Panel, Non-Technical Reviewers, System Evaluators, and Engineering Team  
> **System Scope:** Student Self-Service QR Kiosk (Time-In / Time-Out) &rarr; Attendance Generation &rarr; Teacher Classroom Verification Lifecycle  
> **Codebase Authority:** Grounded strictly in implemented controllers, services, database models, and constraints of the `CIS_CAPSTONE` repository.

---

## 1. Executive Summary of the Attendance Lifecycle

The automated attendance architecture operates across two synchronized operational tiers:

1. **Tier 1: Campus Gate Kiosk (Self-Service Scan):**  
   Students physically present their QR identification badge to a stationary optical reader or USB HID scanner at the entrance kiosk. Without requiring operator intervention, the system matches the scanned token, validates active enrollment, verifies current operating schedules, evaluates anti-spam cooldown constraints, and logs either a **Time-In** or **Time-Out**. Successful scans automatically trigger real-time dashboard updates and queue guardian email alerts.

2. **Tier 2: Classroom Subject-Level Verification (Teacher Audit & Authority):**  
   While gate scans record arrival on school premises, the classroom teacher holds ultimate legal authority over student presence during scheduled class periods. When teachers open their roster, the system pre-populates attendance using gate logs. Teachers can verify students as **Present**, **Late**, **Absent**, **Excused**, or **Not in Classroom** (detecting students who entered the campus gate but skipped class). Any student tagged as "Not in Classroom" is protected by a 60-minute automated grace period before being transitioned to an unexcused absence.

---

## 2. Master System Flowchart

The following evidence-based flowchart details every operational state, decision condition, failure fallback, and automated transition implemented in the codebase. On-page connectors (`((A))`, `((B))`, and `((C))`) link consecutive phases.

```mermaid
flowchart TD
    %% ==========================================
    %% SUBGRAPH 1: KIOSK SELF-SERVICE SCANNING
    %% ==========================================
    subgraph SUB_GATE_SCAN["Phase 1: Student Self-Service QR Kiosk (Gate Check-In / Out)"]
        START(["Start: Student Arrives at Kiosk Terminal"]) --> SCAN_INPUT["Student Presents QR Badge to Optical / USB HID Scanner"]
        SCAN_INPUT --> CHK_SCAN_READ{"Did Scanner Capture<br/>Valid QR String?"}
        
        CHK_SCAN_READ -->|"No / Misaligned"| ERR_SCAN_FAIL["Scanner Emits Error Tone & Prompts Re-Scan"]
        ERR_SCAN_FAIL --> SCAN_INPUT
        
        CHK_SCAN_READ -->|"Yes"| API_DISPATCH["Kiosk Transmits Token & Device ID to Backend API"]
        
        API_DISPATCH --> CHK_STUDENT_EXISTS{"Is QR Code Linked to<br/>an Active Student?"}
        CHK_STUDENT_EXISTS -->|"No / Token Inactive"| ERR_404_STUDENT["Display 'Invalid QR or Student Not Found' (404)"]
        ERR_404_STUDENT --> TERM_END_SCAN(["Terminal: Scan Terminated"])
        
        CHK_STUDENT_EXISTS -->|"Yes"| CHK_ACTIVE_ENROLL{"Does Student Have an Active<br/>Enrollment in Current School Year?"}
        CHK_ACTIVE_ENROLL -->|"No / Unenrolled"| ERR_404_ENROLL["Display 'No Active Enrollment Found' (404)"]
        ERR_404_ENROLL --> TERM_END_SCAN
        
        CHK_ACTIVE_ENROLL -->|"Yes"| RES_SCHEDULE["Resolve Academic Schedule<br/>(Grade Level + Morning/Afternoon Session)"]
        RES_SCHEDULE --> CHK_ACTIVE_SCHED{"Is Current Time Within<br/>an Active Schedule?"}
        CHK_ACTIVE_SCHED -->|"No"| ERR_400_SCHED["Display 'No Active Schedule for Current Time' (400)"]
        ERR_400_SCHED --> TERM_END_SCAN
        
        CHK_ACTIVE_SCHED -->|"Yes"| ATOMIC_LOCK["Initiate Transaction & Lock Daily Attendance Record"]
        ATOMIC_LOCK --> CHK_COOLDOWN{"Is Spam Cooldown Timer<br/>Active for This Student?"}
        
        CHK_COOLDOWN -->|"Yes (Under Cooldown)"| LOG_EXCESS["Increment Spam Offense Count &<br/>Record 'excess_scan' Flag"]
        LOG_EXCESS --> ERR_429_COOLDOWN["Display 'Too Many Scans. Wait for Cooldown' (429)"]
        ERR_429_COOLDOWN --> TERM_END_SCAN
        
        CHK_COOLDOWN -->|"No (Cooldown Cleared)"| CHK_COMPLETED_CYCLE{"Has Student Already Completed<br/>Both Time-In AND Time-Out Today?"}
        CHK_COMPLETED_CYCLE -->|"Yes (Cycle Finished)"| LOG_REJECT_IN["Log 'duplicate_scan' Flag &<br/>Reject Additional Entry"]
        LOG_REJECT_IN --> ERR_400_DUP_CYCLE["Display 'Daily Attendance Cycle Completed' (400)"]
        ERR_400_DUP_CYCLE --> TERM_END_SCAN
        
        CHK_COMPLETED_CYCLE -->|"No"| EVAL_DIRECTION{"Has Student Already<br/>Timed-In for Today?"}
    end

    %% ==========================================
    %% SUBGRAPH 2: SCAN DIRECTION & TIME WINDOWS
    %% ==========================================
    subgraph SUB_DIRECTION_EVAL["Phase 2: Direction Evaluation & Time Window Validation"]
        EVAL_DIRECTION -->|"No Record for Today"| TARGET_IN["Target State: TIME-IN"]
        EVAL_DIRECTION -->|"Active Time-In Exists"| TARGET_OUT["Target State: TIME-OUT"]
        
        %% Time-In Branch
        TARGET_IN --> CHK_IN_WINDOW{"Is Current Time Within<br/>Check-In Window (in_start to in_end)?"}
        CHK_IN_WINDOW -->|"No"| ERR_IN_WINDOW["Display 'Outside Allowed Check-In Window' (400)"]
        ERR_IN_WINDOW --> TERM_END_SCAN
        
        CHK_IN_WINDOW -->|"Yes"| CHK_LATE{"Is Current Time Past<br/>the Late Threshold?"}
        CHK_LATE -->|"Yes"| MARK_LATE["Status: Late Arrival<br/>Record 'late_arrival' Flag"]
        CHK_LATE -->|"No"| MARK_ON_TIME["Status: On-Time Arrival<br/>Normal Gate Status"]
        
        MARK_LATE --> CREATE_IN_LOG["Persist Gate AttendanceLog (Type: IN)"]
        MARK_ON_TIME --> CREATE_IN_LOG
        CREATE_IN_LOG --> LINK_TIME_IN["Update Daily QrAttendance: Set time_in_log_id<br/>Arm Anti-Spam Cooldown Timer"]
        LINK_TIME_IN --> CONN_A1(( "A" ))
        
        %% Time-Out Branch
        TARGET_OUT --> CHK_OUT_WINDOW{"Is Current Time Within<br/>Dismissal Window (out_start to out_end)?"}
        CHK_OUT_WINDOW -->|"No (Premature Scan)"| CHK_DUP_ATTEMPTS{"Has Student Exceeded<br/>Max Duplicate Attempts (3)?"}
        
        CHK_DUP_ATTEMPTS -->|"No"| WARN_EARLY["Record 'duplicate_scan' Flag &<br/>Increment Spam Count"]
        WARN_EARLY --> ERR_EARLY_OUT["Display 'Please Wait for Dismissal Window' (400)"]
        ERR_EARLY_OUT --> TERM_END_SCAN
        
        CHK_DUP_ATTEMPTS -->|"Yes"| COOLDOWN_PENALTY["Record 'excess_scan' Flag &<br/>Apply Cooldown Penalty"]
        COOLDOWN_PENALTY --> ERR_429_PENALTY["Display 'Too Many Scans. Cooldown Applied' (429)"]
        ERR_429_PENALTY --> TERM_END_SCAN
        
        CHK_OUT_WINDOW -->|"Yes"| CREATE_OUT_LOG["Persist Gate AttendanceLog (Type: OUT)"]
        CREATE_OUT_LOG --> LINK_TIME_OUT["Update Daily QrAttendance: Set time_out_log_id<br/>Arm Anti-Spam Cooldown Timer"]
        LINK_TIME_OUT --> CONN_A1
    end

    %% ==========================================
    %% SUBGRAPH 3: GATE POST-PROCESSING & ATTENDANCE GENERATION
    %% ==========================================
    subgraph SUB_GATE_POST["Phase 3: Real-Time Dispatch & Official Attendance Baseline"]
        CONN_A2(( "A" )) --> BROADCAST_WS["Broadcast Real-Time Event (Laravel Reverb)<br/>Instant Kiosk Queue & Dashboard Refresh"]
        BROADCAST_WS --> KIOSK_POPUP["Display Green Confirmation Card on Kiosk Terminal<br/>(Student Name, Photo, Level, Status, Timestamp)"]
        
        KIOSK_POPUP --> CHK_GUARDIAN_EMAIL{"Does Student Have a<br/>Registered Guardian Email?"}
        CHK_GUARDIAN_EMAIL -->|"Yes"| QUEUE_MAIL["Create EmailLog (Status: Pending) &<br/>Dispatch SendGateScanNotification Job"]
        CHK_GUARDIAN_EMAIL -->|"No"| BYPASS_MAIL["Bypass Guardian Email Alert"]
        
        QUEUE_MAIL --> ESTABLISH_BASELINE["Establish Automated Attendance Baseline"]
        BYPASS_MAIL --> ESTABLISH_BASELINE
        
        ESTABLISH_BASELINE --> BASELINE_RULE["Hierarchy Rule:<br/>Gate Time-In Present &rarr; Initial Classroom Baseline: PRESENT<br/>No Gate Time-In &rarr; Initial Classroom Baseline: ABSENT"]
        BASELINE_RULE --> CONN_B1(( "B" ))
    end

    %% ==========================================
    %% SUBGRAPH 4: CLASSROOM TEACHER VERIFICATION
    %% ==========================================
    subgraph SUB_ROOM_VERIFY["Phase 4: Subject Classroom Verification (Teacher Audit)"]
        CONN_B2(( "B" )) --> TEACHER_LOGIN["Subject Teacher Accesses Room Attendance Portal"]
        TEACHER_LOGIN --> CHK_TEACHING_ASSIGN{"Does Teacher Have Active Teaching<br/>Assignment for This Section & Subject?"}
        
        CHK_TEACHING_ASSIGN -->|"No"| ERR_403_TEACHER["Access Denied: 403 Forbidden"]
        ERR_403_TEACHER --> TERM_END_VERIFY(["Terminal: Action Blocked"])
        
        CHK_TEACHING_ASSIGN -->|"Yes"| AUTO_EXPIRE_CHECK["Automated Expiry Routine Runs<br/>(AttendanceVerification::expireNotInClassroomRecords)"]
        AUTO_EXPIRE_CHECK --> CHK_EXPIRED_GRACE{"Did Any 'Not in Classroom' Record<br/>Exceed the 60-Minute Grace Period?"}
        
        CHK_EXPIRED_GRACE -->|"Yes"| AUTO_TRANS_ABSENT["Auto-Transition Status to ABSENT<br/>Resolved By: SYSTEM | Audit Log Created"]
        CHK_EXPIRED_GRACE -->|"No"| LOAD_ROSTER["Load Section Roster Matrix Showing:<br/>1. Gate Time-In Log<br/>2. Current Verification Status"]
        AUTO_TRANS_ABSENT --> LOAD_ROSTER
        
        LOAD_ROSTER --> EVAL_GATE_STATUS{"Did Student Self-Scan<br/>at Gate Kiosk Today?"}
        
        %% Case A: Student Scanned at Gate
        EVAL_GATE_STATUS -->|"Yes (Gate In Record Exists)"| EVAL_PHYSICAL_PRESENCE{"Is Student Physically Sitting<br/>in the Subject Classroom?"}
        
        EVAL_PHYSICAL_PRESENCE -->|"Yes"| TEACHER_MARK_PRES["Teacher Confirms Status: PRESENT<br/>(or LATE if arrived late to period)"]
        TEACHER_MARK_PRES --> CONN_C1(( "C" ))
        
        EVAL_PHYSICAL_PRESENCE -->|"No (Cutting Class / Missing)"| TEACHER_MARK_NIC["Teacher Tags Status: NOT IN CLASSROOM<br/>(Attendance Discrepancy Flagged)"]
        TEACHER_MARK_NIC --> ARM_GRACE["Initiate 60-Minute Grace Period Timer"]
        
        ARM_GRACE --> CHK_STUDENT_ARRIVES{"Does Student Arrive Before<br/>60-Minute Grace Period Expires?"}
        CHK_STUDENT_ARRIVES -->|"Yes (Resolved by Teacher)"| TEACHER_RESOLVE["Teacher Updates Status to PRESENT / LATE<br/>Adds Justification Remarks"]
        TEACHER_RESOLVE --> CONN_C1
        
        CHK_STUDENT_ARRIVES -->|"No (Grace Period Expires)"| SYS_RESOLVE["System Automatically Converts to ABSENT<br/>Resolved By: SYSTEM | Audit Log Created"]
        SYS_RESOLVE --> CONN_C1
        
        %% Case B: Student Did NOT Scan at Gate
        EVAL_GATE_STATUS -->|"No (No Gate Scan Recorded)"| EVAL_ROOM_OVERRIDE{"Is Student Physically Present<br/>in the Classroom Regardless?"}
        
        EVAL_ROOM_OVERRIDE -->|"Yes (Manual Gate Override)"| TEACHER_OVERRIDE["Teacher Marks PRESENT or EXCUSED<br/>(Teacher Authority Overrides Missing Gate Log)"]
        TEACHER_OVERRIDE --> CONN_C1
        
        EVAL_ROOM_OVERRIDE -->|"No (Not in Class)"| CHK_EXCUSED{"Did Student/Guardian Present<br/>Legitimate Excuse Slip?"}
        
        CHK_EXCUSED -->|"Yes"| TEACHER_MARK_EXCUSED["Teacher Marks Status: EXCUSED<br/>(Medical / School Errand Remarks)"]
        CHK_EXCUSED -->|"No"| CONFIRM_ABSENT["Status Confirmed as ABSENT<br/>(Unexcused Absence)"]
        
        TEACHER_MARK_EXCUSED --> CONN_C1
        CONFIRM_ABSENT --> CONN_C1
    end

    %% ==========================================
    %% SUBGRAPH 5: AUDIT TRAIL & SYSTEM FINALIZATION
    %% ==========================================
    subgraph SUB_AUDIT_FINAL["Phase 5: Audit Persistence, Notifications & Final Records"]
        CONN_C2(( "C" )) --> PERSIST_VERIF["Persist in attendance_verifications Table<br/>(Status, Teacher ID, Resolved By, Remarks, Timestamp)"]
        PERSIST_VERIF --> CREATE_AUDIT_HIST["Persist in attendance_verification_histories<br/>(Previous Status, New Status, Changed By User ID, Timestamp)"]
        CREATE_AUDIT_HIST --> DISPATCH_NOTIFICATION["Dispatch In-App Notification to Teacher Dashboard"]
        DISPATCH_NOTIFICATION --> DOWNSTREAM_SYNC["Downstream Integration Ready:<br/>1. SF-2 Monthly DepEd Attendance Register<br/>2. Early Warning At-Risk Analytics Engine"]
        DOWNSTREAM_SYNC --> TERM_SUCCESS(["Terminal: Attendance Cycle Finalized"])
    end
```

---

## 3. Technical Implementation Reference Table

The table below maps every operational step and decision node in the flowchart directly to its implementing backend route, controller, service, Eloquent model, and database table in the codebase.

| Operational Stage | Decision / Process Node | Implementing Code File & Symbol | Database Tables & Columns Involved |
|---|---|---|---|
| **Kiosk Optical Scan** | Scan Submission & CSRF Anchor | `resources/views/pov/scanner-operator/qr-station/station.blade.php`<br/>`public/js/qr-station.js` | Hidden `#scannerInput` DOM buffer capturing USB HID keystroke emulation |
| **API Entry Point** | Request Validation & Routing | `routes/scanner-operator.php`<br/>`ScanController@scan` | Validates `code` (string) or `student_id` (integer) and `device_id` |
| **Student Identification** | Resolve Student & Guardian Join | `QrScanService::resolveStudent()` | Reads `qr_codes.code`, joins `students`, left-joins `guardians.email` |
| **Active Enrollment Check** | Resolve Section & Academic Year | `QrScanService::processScan()` | Query on `enrollments` (`status = 'active'`, `school_year_id`) joined with `sections` (`grade_level`, `session_type`, `level`) |
| **Schedule Window Resolution** | Operating Window Resolution | `App\Services\ScheduleResolver::resolve()` | Matches `schedule_configs` (`in_start`, `in_end`, `late_threshold`, `out_start`, `out_end`) against section level and session type |
| **Anti-Spam Shield & Lock** | Atomic Transaction & Cooldown Check | `QrScanService::processScan()` (lines 85–114) | Table `qr_attendances` (`spam_offense_count`, `cooldown_expires_at`) using `lockForUpdate()` |
| **Excess Scan Blocking** | Cooldown Flagging & 429 Return | `QrScanService::processScan()` (lines 104–113) | Table `flagged_scans` (`flag_type = 'excess_scan'`) |
| **Duplicate Attempt Handling** | Duplicate Warnings & Cooldown Penalty | `QrScanService::processScan()` (lines 131–155) | Table `flagged_scans` (`flag_type = 'duplicate_scan'`), increments `spam_offense_count` up to `max_duplicate_attempts` (default 3) |
| **Arrival Punctuality Check** | Late Evaluation vs. Threshold | `QrScanService::processScan()` (lines 190–214) | Compares current time against `schedule_configs.late_threshold`. Creates `flagged_scans` (`flag_type = 'late_arrival'`) |
| **Gate Log Persistence** | Create Official Gate Record | `AttendanceLog::create()` | Table `attendance_logs` (`enrollment_id`, `scan_type`, `session_type`, `scan_time`, `device_id`) |
| **Daily Pairing Update** | Link In / Out Foreign Keys | `QrAttendance::save()` | Updates `qr_attendances.time_in_log_id` or `qr_attendances.time_out_log_id`, sets `cooldown_expires_at` |
| **Real-Time WebSockets** | Live UI Broadcasting | `App\Events\AttendanceRecorded::dispatch()` | Broadcast via **Laravel Reverb** channel; updates live scanner queue and overview stats |
| **Guardian Notification** | Asynchronous Email Queue | `SendGateScanNotification::dispatch()` | Writes to `email_logs` (`status = 'pending'`), dispatches queue worker job |
| **Teacher Portal Access** | Teaching Assignment Gate | `TeacherRoomAttendanceController@show` | Checks `teaching_assignments` (`teacher_id`, `section_id`, `status = 'active'`) |
| **Automated Grace Expiry** | 60-Minute Expiry Worker | `AttendanceVerification::expireNotInClassroomRecords()` | Queries `attendance_verifications` (`status = 'not_in_classroom'`). Auto-updates to `'absent'` if elapsed time &ge; 60 mins |
| **Classroom Verification** | Single / Bulk Teacher Verification | `TeacherRoomAttendanceController@verify`<br/>`TeacherRoomAttendanceController@bulkVerify` | Table `attendance_verifications` (`status`, `remarks`, `resolved_by = 'teacher'`, `verified_at`) |
| **Audit Log Tracking** | Immutable State History | `AttendanceVerificationHistory::create()` | Table `attendance_verification_histories` (`previous_status`, `new_status`, `changed_by`, `remarks`, `created_at`) |

---

## 4. Comprehensive Process Breakdown

### Stage 1: Hardware Ingestion & Kiosk Terminal Scanning
The student self-service workflow begins at a physical kiosk terminal. Students hold their individual QR badges before an optical barcode scanner operating in USB HID (Human Interface Device) keyboard emulation mode. 
- The scanner rapidly streams the decoded alphanumeric string into a hidden, auto-focused input field (`#scannerInput`) on the web application interface.
- If a barcode is unreadable, scratched, or improperly angled, the scanner hardware emits an audible error chirp and refuses transmission, prompting the student to reposition their card.
- Upon receiving a complete newline delimiter, the kiosk frontend dispatches an asynchronous HTTP POST payload containing the scanned token (`code`) and unique terminal identifier (`device_id`) to the backend scanning endpoint.

### Stage 2: Identity, Enrollment, and Academic Schedule Validation
Upon receipt of the scan payload, the backend initiates multi-tier verification before any database mutations occur:
1. **Student Identity:** The system queries the `qr_codes` table to find an active token and retrieves the associated student profile and registered guardian email. If the token is revoked, inactive, or unlinked, an HTTP `404 Not Found` is returned, displaying "Invalid QR or student not found" on the kiosk display.
2. **Active Enrollment:** The system verifies that the student possesses an active enrollment record (`status = 'active'`) for the currently running academic year (`SchoolYear::active()`). If a student is unenrolled or pending transfer, the scan is rejected with an HTTP `404` status.
3. **Schedule Window Evaluation:** The `ScheduleResolver` service identifies the student's grade level and assigned session (`Morning` or `Afternoon`). It retrieves the active `schedule_configs` boundary rules. If the scan takes place outside designated school operating hours, an HTTP `400 Bad Request` ("No active schedule for current time") is issued, terminating the scan.

### Stage 3: Anti-Spam Shield and State Machine Determination
To prevent students from inadvertently or deliberately creating duplicate records by hovering their badge repeatedly:
1. **Database Row Lock:** The system initiates a database transaction and obtains an exclusive row lock (`lockForUpdate()`) on the student's daily record in `qr_attendances`.
2. **Cooldown Shield:** If a previous scan activated a cooldown timer (`cooldown_expires_at`) and that timestamp is still in the future, the system increments the student's `spam_offense_count`, records an `excess_scan` event in the `flagged_scans` audit table, and returns an HTTP `429 Too Many Requests` status, displaying the remaining cooldown seconds.
3. **Direction State Machine:**
   - If `time_in_log_id` is null, the target scan is categorized as **TIME-IN**.
   - If `time_in_log_id` exists and `time_out_log_id` is null, the target scan is categorized as **TIME-OUT**.
   - If both `time_in_log_id` and `time_out_log_id` are populated, the student has completed their authorized daily check-in/out cycle. Any further scans are logged as `duplicate_scan` and rejected with an HTTP `400` status.

### Stage 4: Window Validation, Punctuality, and Gate Log Creation
- **Time-In Evaluation:**
  - The system checks if the current time falls within `in_start` and `in_end`. If early or late beyond operating hours, the scan is rejected.
  - If the arrival time exceeds `schedule.late_threshold`, the system records a `late_arrival` record in `flagged_scans`. Otherwise, the arrival is deemed On Time.
- **Time-Out Evaluation:**
  - If a student scans OUT prior to the dismissal window (`out_start` to `out_end`), the system logs a `duplicate_scan` warning. If repeated more than 3 times (`max_duplicate_attempts`), a punitive 300-second cooldown is enforced.
  - If within the dismissal window, the scan is approved.
- **Persistence:**
  - A permanent record is inserted into `attendance_logs` containing the timestamp, scan type, and device ID.
  - The daily `qr_attendances` record links the newly generated log ID (`time_in_log_id` or `time_out_log_id`) and resets the anti-spam cooldown window.

### Stage 5: Real-Time Event Dispatching & Guardian Notification
Upon successful transaction commitment:
1. **Live Dashboard Broadcast:** The system dispatches an `AttendanceRecorded` event broadcast over WebSocket connections via **Laravel Reverb**. This instantly updates the live gate monitoring queue and overview metric counters on the scanner operator and school administrator screens.
2. **Kiosk UI Feedback:** The kiosk screen updates to display an illuminated green confirmation card showing the student's name, photo, grade/section, and punctuality status, accompanied by an automatic reset countdown bar.
3. **Asynchronous Guardian Alert:** If a guardian email address is registered, an entry is created in `email_logs` (`status = 'pending'`), and a `SendGateScanNotification` job is dispatched to the background queue worker, ensuring email latency never slows down physical throughput at the gate.

### Stage 6: Hierarchy of Authority and Baseline Generation
The system establishes a 4-tier **Hierarchy of Authority** to govern attendance records:
1. **Future Dates:** Default to `no_data` (No Data Yet).
2. **Teacher Classroom Verification:** Holds supreme authority over student status during class periods.
3. **Gate Time-In Scan:** Serves as the automated initial baseline (`present` if scanned, `absent` if no scan).
4. **Unverified Absence:** Any student with neither a gate scan nor a teacher verification defaults to `absent`.

### Stage 7: Classroom Period Verification & The 60-Minute Grace Period
When a subject teacher logs into the faculty portal and opens the Room Attendance module:
1. **Teaching Assignment Authorization:** The system verifies the teacher's active assignment (`TeachingAssignment`) for that section and subject; unauthorized teachers receive an HTTP `403 Forbidden`.
2. **Automated Expiry Maintenance (`expireNotInClassroomRecords`):** Before loading the roster, the system automatically identifies any student previously marked as **Not in Classroom** (`not_in_classroom`). If more than 60 minutes have elapsed since the teacher flagged them, the background routine automatically converts their status to **Absent** (`absent`), setting `resolved_by = 'system'` and stamping the audit log.
3. **Discrepancy Identification:** The teacher reviews the live roster matrix, which contrasts the Gate Time-In with physical presence in the room:
   - **Scenario 1 (Normal Presence):** Student scanned at gate and is seated in room &rarr; Teacher confirms **Present** (or **Late**).
   - **Scenario 2 (Cutting Class / Discrepancy):** Student scanned at gate at 7:15 AM but is missing from 9:00 AM Math &rarr; Teacher marks **Not in Classroom** (`not_in_classroom`). The 60-minute countdown grace period activates. If the student reports to class within the hour, the teacher updates them to Present/Late with remarks. If the hour expires without arrival, the system converts them to Absent.
   - **Scenario 3 (Gate Bypass Override):** Student forgot their QR badge but is physically present in class &rarr; Teacher selects **Present** or **Excused**. The teacher's manual determination overrides the absence of a gate scan.
   - **Scenario 4 (Excused Absences):** Student or parent provides a valid medical certificate or official school errand pass &rarr; Teacher marks **Excused** (`excused`) and logs narrative remarks.

### Stage 8: Audit Trail and Downstream System Synchronization
Every manual or automated change to a verification record creates an immutable history row in `attendance_verification_histories`, recording the previous status, new status, changing user ID, timestamp, and explanation remarks. 
- These verified records form the authoritative data source feeding DepEd SF-2 Monthly School Attendance Registers.
- Accumulated unexcused absences and cutting infractions feed directly into the **Student At-Risk Analytics Engine**, triggering academic alerts for counselors and school administrators.

---

## 5. State Transition & Status Summary Matrix

| Verification Status | Code Identifier | Description & Business Trigger | Fallback / Expiry Behavior | Hierarchy Authority |
|---|---|---|---|---|
| **No Data Yet** | `no_data` | Default placeholder for upcoming or future calendar dates. | Cannot be edited until the date arrives. | Level 1 |
| **Present** | `present` | Student is confirmed physically present in the subject classroom. | Set either automatically by Gate Time-In baseline or by manual teacher verification. | Level 2 (Manual) / Level 3 (Gate) |
| **Late** | `late` | Student arrived past the gate late threshold or arrived tardy to class. | Triggered by gate schedule threshold or set manually by teacher. | Level 2 (Manual) / Level 3 (Gate) |
| **Absent** | `absent` | Student did not report to school premises, or failed to report to class. | Applied when no gate scan exists, confirmed by teacher, or auto-converted from grace expiry. | Level 2 (Manual) / Level 4 (Default) |
| **Excused** | `excused` | Student is absent or tardy with an authorized excuse (medical, official permit). | Requires manual teacher verification with mandatory remarks. | Level 2 |
| **Not in Classroom** | `not_in_classroom` | **Discrepancy State:** Student completed Gate Time-In but is absent from classroom. | Protected by a **60-minute grace period**. If unverified after 60 mins, auto-transitions to `absent`. | Level 2 |

---

## 6. Codebase Verification Note

All pathways, database tables, models, and transition branches depicted in this document were verified directly from active source files in the `CIS_CAPSTONE` codebase:
- Gate scanning logic: [`app/Services/QrSystem/QrScanService.php`](file:///home/catsu/Desktop/CIS_CAPSTONE/app/Services/QrSystem/QrScanService.php)
- Schedule resolution: [`app/Services/ScheduleResolver.php`](file:///home/catsu/Desktop/CIS_CAPSTONE/app/Services/ScheduleResolver.php)
- Scanner terminal interface: [`resources/views/pov/scanner-operator/qr-station/station.blade.php`](file:///home/catsu/Desktop/CIS_CAPSTONE/resources/views/pov/scanner-operator/qr-station/station.blade.php)
- Classroom verification controller: [`app/Http/Controllers/Teacher/Attendance/TeacherRoomAttendanceController.php`](file:///home/catsu/Desktop/CIS_CAPSTONE/app/Http/Controllers/Teacher/Attendance/TeacherRoomAttendanceController.php)
- Verification models & grace period mechanics: [`app/Models/AttendanceVerification.php`](file:///home/catsu/Desktop/CIS_CAPSTONE/app/Models/AttendanceVerification.php)
