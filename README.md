# Backend patches for school-erp mobile API

Standalone patches for the `school-erp` Laravel API. Apply them on a deployment that needs the fixes without pulling the full `school-erp` history.

Patches are numbered in the order they were created. Apply them **sequentially** — many later patches build on earlier ones.

## Patches in this repo

### Wave 1 — Initial bug fix sweep (mobile API + parent flow)

| File | What it fixes |
|------|---------------|
| `0001-fix-mobile-timetable-empty.patch` | Teacher/student timetable showed "No classes" every day (server keyed schedule by `day_of_week` int instead of lowercase day name). |
| `0002-fix-teacher-attendance-empty.patch` | Teacher Attendance tab showed 0% with no records (server's `attendance()` had no teacher branch — only resolved student attendance). Adds a `staff_attendances` lookup for teachers. |
| `0003-fix-attendance-date-format.patch` | Attendance records rendered raw ISO datetimes like `2026-04-01T18:30:00.000000Z`. Now formats `records[].date` using the school's `date_format` setting from Admin → Settings → System Config. Also adds reusable `School::dateFmt()` / `School::timeFmt()` helpers. |
| `0004-fix-parent-flow-server.patch` | Three parent-flow fixes: (a) `childList()` returned wrong keys so children showed name only — now returns `class_name`, `section_name`, `admission_number`, `roll_number`, `attendance_pct`. (b) Mobile parent BusTracking screen polled `/api/bus/status` which 404'd — added the route and `MobileApiController::busStatus()` returning running/stopped/idle for the active child's bus. (c) Payment history & receipt now include `payment_date_display` formatted per the admin's `date_format` setting. |
| `0005-fix-admission-default-password.patch` | New parent admissions used to get a random unprintable password — admins couldn't tell parents how to log in. Now sets the default to `parent123`. One-line change to `AdmissionService::admit()`. |

### Wave 2 — User Management bulk actions

| File | What it fixes |
|------|---------------|
| `0006-feat-user-management-bulk-actions.patch` | Admin User Login Management gets four upgrades: (a) "Create Missing Logins" button auto-creates User rows for any students/parents who don't have one (defaults: parent123 / student-DOB). (b) Class + Section filter applies to parents (filters by their children's class). (c) Bulk-reset passwords. (d) Export resulting credentials to Excel or PDF. **Large patch (~1k lines)** — touches the User Management controller, Vue page, routes, plus a new Excel exporter and PDF blade view. **Needs `npm run build`**. |
| `0007-fix-bulk-reset-enum-cast.patch` | Hotfix on patch 6 — bulk-reset 500'd because `(string) $enum` throws on PHP 8 backed enums. Now uses `->value`. **Apply immediately after 6.** |
| `0008-fix-csrf-419-bulk-actions.patch` | Hotfix on patches 6 & 7 — every fetch in `Users/Index.vue` was sending `X-CSRF-TOKEN: undefined` because `usePage().props.csrf_token` isn't a shared Inertia prop. New `csrfHeader()` helper reads the `XSRF-TOKEN` cookie and sends it as `X-XSRF-TOKEN`. Frontend-only — **needs `npm run build`**. |
| `0009-reset-password-default-to-password.patch` | Bulk reset and individual reset now both default to the literal string `password` instead of generating random ones. Admins reading the Excel export can now actually log users in. |
| `0010-create-missing-clickable-password-default.patch` | "Create Missing Logins" button was disabled when the count was 0 — but admins still wanted to click it to verify. Now always clickable; shows a toast if there are no missing accounts. Default password unified with patches 5 + 9. |
| `0011-user-management-respect-page-length.patch` | User Management list ignored the System Config "Page Length" setting (always returned 10). Now reads `SystemConfig::page_length` like the rest of the admin tables. |
| `0012-default-password-everywhere.patch` | Cleanup of patches 5/9/10 — `AdmissionService` and the `portal:create-users` artisan command now both pull the default password from a single config constant. One source of truth. |

### Wave 3 — Attendance display fixes + format helpers

| File | What it fixes |
|------|---------------|
| `0013-fix-attendance-date-render.patch` | Student/parent attendance date was returning null in the response because `Attendance.date` has no Eloquent cast — `optional($string)->format()` silently failed. Now uses `Carbon::parse($r->date)->format(...)`. |
| `0014-useformat-composable-and-sweep.patch` | New `useFormat()` Vue composable that exposes `fmtDate`, `fmtDateTime`, `fmtTime` reading from System Config. Sweeps ~12 admin Vue pages to use it instead of hardcoded `Date#toLocaleDateString` calls. **Needs `npm run build`**. |
| `0015-mobile-api-display-fields.patch` | Mobile API responses now include `*_display` variants (e.g. `payment_date_display`, `created_at_display`) formatted per the school's date/time format. Mobile screens drop their own date-formatting code. |

### Wave 4 — Chat + admin polish

| File | What it fixes |
|------|---------------|
| `0016-chat-search-and-start-direct.patch` | Chat — adds search-by-name in the New Chat sheet + a `startDirect` endpoint that finds-or-creates a 1-on-1 conversation. Mobile + web share the same endpoint. |
| `0017-user-management-enhancements.patch` | User Management — class/section filter for parents (filters by their children's class), bulk select-all-on-page, fixed pagination jumping. **Needs `npm run build`**. |
| `0018-fix-empty-username-show-phone.patch` | Staff list showed empty cells when a staff user had no `username` set — now falls back to phone number. |

### Wave 5 — Staff attendance audit & cleanup

| File | What it fixes |
|------|---------------|
| `0019-fix-staff-attendance-display.patch` | Staff attendance dashboard inferred status from a derived join that lit "all present" before any attendance was actually marked. Now reads from `staff_attendances` directly. |
| `0020-fix-dashboard-real-staff-attendance.patch` | Admin dashboard's staff attendance card was reading from the same broken inference as 0019 — now uses the real table. |
| `0021-attendance-audit-cleanup.patch` | 13-issue attendance audit — formula consistency (`(present + late*0.5 + half_day*0.5) / (total - holiday)`) across 5+ controllers, fixes for late mark counting, holiday subtraction, and attendance-summary screen pagination. |

### Wave 6 — Holiday auto-fill + leave system

| File | What it fixes |
|------|---------------|
| `0022-m2-holiday-autofill.patch` | New `php artisan attendance:fill-holidays` command run nightly via the scheduler — auto-marks every student/staff as `holiday` for declared school holidays. Skips dates where attendance was already marked. Also adds the migration to allow `'holiday'` as a value in the attendance status enum (driver-aware: ALTER TABLE on MySQL, no-op on SQLite). |
| `0023-g1-g6-g7-backend.patch` | Three leave-system bug clusters: G1 (leave creation rejected when no leave_type provided), G6 (leaveTypes endpoint returned wrong shape for the picker), G7 (balance calculation double-counted approved leaves). |
| `0024-surface-unmarked-attendance-days.patch` | Adds an `unmarked` count alongside `present/absent/late/holiday` on the attendance summary endpoints. The dashboard "X days not yet marked" chip is wired to this. Doesn't penalise the % calculation; just makes the gap visible. |

### Wave 7 — Staff QR attendance

| File | What it fixes |
|------|---------------|
| `0025-staff-qr-attendance.patch` | Staff can now mark their own attendance by having a teacher scan their QR badge at the gate. New `POST /mobile/staff-attendance/rapid-scan` endpoint mirrors the student rapid-scan flow. New `users.qr_token` column generated on first request. |
| `0026-staff-qr-badges-and-self.patch` | Two pieces: (a) bulk staff QR badge exports — `Staff & HR` page gets "QR Badges PDF" (A4 sheets, 4-up) and "QR Excel" buttons. (b) `GET /mobile/staff-qr/me` endpoint that returns the logged-in staff member's QR data-URI for the in-app My QR Code screen. **Needs `npm run build`** (Vue file). |
| `0027-student-qr-badge-pdf.patch` | Mirror of 0026's badge PDF for students — Students list page gets "QR Badges PDF" + "QR Excel" header buttons. **Needs `npm run build`**. |

### Wave 8 — Recent (current cycle)

| File | What it fixes |
|------|---------------|
| `0028-fee-due-reminder-voice-and-button.patch` | Adds **voice channel** to the `fee_due_reminder` system trigger and a manual **"Send Reminder"** button on the Due Report page (per row + bulk). Server recomputes balance before sending so admins can't tamper with amounts. **Needs `npm run build`** (Vue file). |
| `0029-push-notifications-and-new-triggers.patch` | **Backend half** of the mobile push-notification feature. New `ExpoPushService` POSTs to `https://exp.host/--/api/v2/push/send`. Adds `users.expo_push_token`, `device_platform`, `push_token_updated_at` columns. `POST /mobile/device/register` now accepts `expo_push_token`. `NotificationService::sendPushToUser` dispatches to both Expo + FCM transports; data payload carries `{type, id, entity_id}` for deep-linking. New auto-fired triggers: **fee_due push**, **diary_created**, **assignment_created** (wired into AssignmentController, StudentDiaryController, MobileApiController storeDiary/storeAssignment). Three new system templates auto-seed on next visit to Communication → Templates. **Run `php artisan migrate`** after applying. |
| `0030-mobile-exam-schedule-endpoint.patch` | Enriches `GET /mobile/exams` for the new mobile **Exam Schedule** screen. Adds a flattened `papers` array (one row per subject paper) carrying date / time / duration / writing_marks / passing_marks / class / section / exam name — alongside the legacy `schedules` array. Role-aware scoping: parents/students see their class with `status=published`; teachers see classes via `TeacherScopeService`; admins see everything including drafts. |
| `0031-fix-mobile-students-current-year.patch` | Hotfix: `GET /mobile/students` was duplicating rows — one per academic history entry — so a student enrolled for 4 years showed up 4 times. Switched the `student_academic_histories` join from LEFT to INNER and filtered by `current_academic_year_id`. Side effect: graduated/transferred students drop from the list, which is what the admin "Students" screen wants. |
| `0032-lock-system-templates-from-delete.patch` | Hardens system-template protection. Delete button could appear and `destroy()` could accept deletion for rows with a system slug but `is_system=false` (legacy rows). Now `index()` backfills the flag, `destroy()` checks both flag AND slug-membership in `SYSTEM_TRIGGERS`, and the Vue UI uses `isSystemTemplate()` doing the same. Adds the new `diary_created` and `assignment_created` triggers to the Vue trigger list. **Needs `npm run build`** (Vue file). |
| `0033-driver-mobile-api-fix.patch` | Wires up the driver mobile app's missing API endpoints. Without this patch, every GPS push 404s, the vehicle picker is empty, and drivers can't mark student boarding. Adds `GET /mobile/transport/driver/vehicles`, `POST /mobile/transport/driver-tracking/update`, `POST /mobile/transport/driver-tracking/stop`, plus two transport-attendance endpoints (`GET .../attendance/students` and `POST .../attendance/mark`) that fire SMS/WhatsApp/push to parents. Driver/conductor scope enforced server-side. |
| `0034-fix-driver-dropdown-empty.patch` | Hotfix: the **Edit Vehicle** modal's "Driver" dropdown was empty even when driver users existed. The query required a `staff_designation` row with name='driver', but most installations identify drivers via `users.user_type='driver'`. Now matches staff whose linked user has `user_type='driver'` OR (legacy fallback) whose designation name equals 'driver'. |
| `0035-fix-driver-dashboard-no-vehicles.patch` | Hotfix follow-up to 0033: the driver mobile dashboard kept showing "no vehicles assigned" even after assignment. Root cause was that the dashboard pulled `/mobile/transport/live`, which only returns vehicles with a recent `live_location` row. Fix enriches `assignedVehicles()` to include route + stops + student counts + live status. |
| `0036-gate-keeper-mobile-api.patch` | Wires up the front gate keeper mobile app, which previously had a UI shell but no functional backend (every call 404'd). Adds 7 endpoints under `/mobile/front-office/`: `gate/stats`, `gate-passes/verify/{token}` (QR resolution + next valid action), `gate-passes/{id}/exit`, `gate-passes/{id}/entry` (state-machine transitions), `visitors`, `visitors` POST (walk-in log), `visitors/{id}/exit`. School scope enforced. |

---

## How to apply on another live server

SSH into the server, then from the project root of the school-erp deployment:

```bash
# 1. Download every patch in order
for n in $(seq -w 1 36); do
  curl -sL "https://raw.githubusercontent.com/trivartatech/mobile2/main/00${n}-"*.patch \
    -o "/tmp/p${n}.patch" 2>/dev/null
done

# 2. Dry-run — fails loudly if conflicts (test all together)
for f in /tmp/p*.patch; do
  git apply --check "$f" || echo "CONFLICT in $f"
done

# 3. Apply in order
for f in $(ls /tmp/p*.patch | sort); do git apply "$f" || break; done

# 4. Migrations (patches 22, 29 add columns / enum values)
php artisan migrate --force

# 5. Vue rebuild — required after patches 6, 8, 14, 17, 26, 27, 28, 32
npm install --no-audit --no-fund
npm run build

# 6. Reload PHP and clear caches so the new code + routes take effect
php artisan optimize:clear
php artisan route:cache
php artisan config:cache
sudo systemctl reload php8.3-fpm   # adjust PHP version
```

If `git apply --check` reports a conflict, that file on the server has diverged from `trivartatech/school-erp`. Open the relevant `.patch` file and apply by hand — diffs are small and the headers identify the file + lines.

## Mobile-app changes (informational)

Many backend patches ship with corresponding mobile-app changes in `schools-1/` (the React Native source). Those are **not** in this repo — they ship in the next Expo build. The backend patches keep the API working for any client; the matching UI is just polish.

Mobile changes that pair with backend patches:
- 0029 push notifications → `expo-notifications` setup, NotificationsContext, tab badge, deep linking
- 0030 exam schedule → ExamScheduleScreen wired into Parent/Student/Teacher navigators
- 0033/0035 driver API → Mark Boarding screen, real GPS tracking
- 0036 gate keeper → QR scanner + Log Visitor screen

Plus a navigation redesign (sidebar + 4 fixed bottom tabs + dark mode toggle) that's mobile-only — no backend changes.

## Verify after applying

A few quick smoke tests as different roles:

```bash
# Driver — should now return assigned vehicles + route + stops
curl -H "Authorization: Bearer <DRIVER_TOKEN>" -H "Accept: application/json" \
  https://<domain>/api/mobile/transport/driver/vehicles | head -c 400

# Gate keeper — should return today's stats
curl -H "Authorization: Bearer <GK_TOKEN>" -H "Accept: application/json" \
  https://<domain>/api/mobile/front-office/gate/stats

# Teacher — exam schedule with full data
curl -H "Authorization: Bearer <TEACHER_TOKEN>" -H "Accept: application/json" \
  https://<domain>/api/mobile/exams | head -c 400

# Parent — child list with class/section info
curl -H "Authorization: Bearer <PARENT_TOKEN>" -H "Accept: application/json" \
  https://<domain>/api/mobile/children
```

Get bearer tokens from `tinker`:

```bash
php artisan tinker --execute="
  \$u = App\Models\User::where('email','<their_email>')->first();
  echo 'BEARER='.\$u->createToken('debug')->plainTextToken.PHP_EOL;
"
```
