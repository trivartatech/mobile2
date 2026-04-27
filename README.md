# Backend patches for school-erp mobile API

Standalone patches for the `school-erp` Laravel API. Apply them on a deployment that needs the fixes without pulling the full `school-erp` history.

## Patches in this repo

| File | What it fixes |
|------|---------------|
| `0001-fix-mobile-timetable-empty.patch` | Teacher/student timetable showed "No classes" every day (server keyed schedule by `day_of_week` int instead of lowercase day name). |
| `0002-fix-teacher-attendance-empty.patch` | Teacher Attendance tab showed 0% with no records (server's `attendance()` had no teacher branch — only resolved student attendance). Adds a `staff_attendances` lookup for teachers. |
| `0003-fix-attendance-date-format.patch` | Attendance records rendered raw ISO datetimes like `2026-04-01T18:30:00.000000Z`. Now formats `records[].date` using the school's `date_format` setting from Admin → Settings → System Config. Also adds reusable `School::dateFmt()` / `School::timeFmt()` helpers for future use. |
| `0004-fix-parent-flow-server.patch` | Three parent-flow fixes: (a) `childList()` returned wrong keys so children showed name only — now returns `class_name`, `section_name`, `admission_number`, `roll_number`, `attendance_pct`. (b) Mobile parent BusTracking screen polled `/api/bus/status` which 404'd — added the route and `MobileApiController::busStatus()` returning running/stopped/idle for the active child's bus. (c) Payment history & receipt now include `payment_date_display` formatted per the admin's `date_format` setting (alongside the existing `payment_date` field for backward compat). |
| `0005-fix-admission-default-password.patch` | New parent admissions used to get a random unprintable password — admins couldn't tell parents how to log in to the mobile app. Now sets the default password to `parent123` (matches the existing `portal:create-users` artisan command). One-line change to `AdmissionService::admit()`. |
| `0006-feat-user-management-bulk-actions.patch` | Admin panel User Login Management gets four upgrades: (a) "Create Missing Logins" button auto-creates User rows for any students/parents who don't have one yet (defaults: parent123 / student-DOB). (b) Class + Section filter now also applies to parents (filters by their children's class). (c) Bulk-reset passwords for selected users with one click. (d) Export resulting credentials to Excel or PDF. **Note**: this patch is large (~1k lines) — touches the User Management controller, Vue page, routes, plus a new Excel exporter and PDF blade view. If the other domain has diverged on these files, expect conflicts. |
| `0007-fix-bulk-reset-enum-cast.patch` | Hotfix on top of patch 6 — bulk-reset 500'd in PHP 8 because `user_type` is a backed enum and `(string) $enum` throws. Now uses `->value`. **Apply patch 7 immediately after patch 6** if you've applied 6. |
| `0008-fix-csrf-419-bulk-actions.patch` | Hotfix on top of patches 6 & 7 — every fetch in `Users/Index.vue` was sending `X-CSRF-TOKEN: undefined` because `usePage().props.csrf_token` isn't a shared Inertia prop in this app. New `csrfHeader()` helper reads the `XSRF-TOKEN` cookie (always fresh, axios-style) and sends it as `X-XSRF-TOKEN`. Frontend-only — needs `npm run build` after applying. |
| `0028-fee-due-reminder-voice-and-button.patch` | Adds **voice channel** to the `fee_due_reminder` system trigger (auto-seeded on first visit to Communication → Templates → Voice) and a manual **"Send Reminder"** button on the Due Report page (per row + bulk). Server recomputes balance before sending so admins can't tamper with amounts. Touches `CommunicationTemplateController.php`, `NotificationService.php`, `Finance/LedgerController.php`, `routes/web.php`, and `Finance/Ledger/DueReport.vue`. **Needs `npm run build`** — modifies a Vue file. |
| `0029-push-notifications-and-new-triggers.patch` | **Backend half** of the mobile push-notification feature. New `ExpoPushService` POSTs to `https://exp.host/--/api/v2/push/send`. Adds `users.expo_push_token`, `device_platform`, `push_token_updated_at` columns. `POST /mobile/device/register` now accepts `expo_push_token` alongside `fcm_token`. `NotificationService::sendPushToUser` dispatches to both transports; data payload carries `{type, id, entity_id}` for deep-linking. New auto-fired triggers: **fee_due push**, **diary**, **assignment** (wired into `AssignmentController`, `StudentDiaryController`, `MobileApiController::storeDiary` / `storeAssignment`). Three new system templates auto-seed (`fee_due_reminder` push, `diary_created`, `assignment_created`) on next visit to Communication → Templates. **Run `php artisan migrate`** after applying. Mobile-app side ships with the next Expo build. |
| `0030-mobile-exam-schedule-endpoint.patch` | Enriches `GET /mobile/exams` so the new mobile **Exam Schedule** screen has the data it needs. Adds a flattened `papers` array (one row per subject paper) carrying date / time / duration / writing_marks / passing_marks / class / section / exam name — alongside the legacy `schedules` array so older clients keep working. Role-aware scoping: parents and students see only their class with `status=published`; teachers see schedules for classes they're assigned to (via `TeacherScopeService`); admins see everything including drafts. Pure controller change — no migration. Mobile screen ships with next Expo build. |
| `0031-fix-mobile-students-current-year.patch` | Hotfix: `GET /mobile/students` was duplicating rows — one per academic history entry — so a student enrolled for 4 years showed up 4 times in search. Switched the `student_academic_histories` join from LEFT to INNER and filtered by `current_academic_year_id`. Side effect: students not enrolled in the current year (graduated / transferred) drop out of the list, which is what the admin "Students" screen actually wants. Pure controller change — no migration. |
| `0032-lock-system-templates-from-delete.patch` | Hardens system-template protection. The Delete button could still appear and `destroy()` could accept deletion for rows that had a system slug but `is_system=false` (legacy rows from older deployments). Now `index()` backfills the flag, `destroy()` checks both `is_system` AND slug-membership in `SYSTEM_TRIGGERS`, and the Vue UI uses an `isSystemTemplate()` helper that does the same. Adds the new `diary_created` and `assignment_created` triggers to the Vue trigger list and variables map. **Needs `npm run build`** — touches a Vue file. |

Patches 1, 2, 4 touch `app/Http/Controllers/Api/MobileApiController.php`. Patch 3 touches the same controller plus `app/Models/School.php`. Patch 4 also adds a route in `routes/api.php`. Patch 5 touches `app/Services/AdmissionService.php`. Patch 6 touches `app/Http/Controllers/School/UserManagementController.php`, `app/Exports/UserCredentialsExport.php` (new), `resources/views/exports/user-credentials.blade.php` (new), `resources/js/Pages/School/Users/Index.vue`, and `routes/web.php`. They apply cleanly in order.

**Patch 6 also requires `npm run build`** on the other server (it modifies a Vue file), unlike the other patches which are pure PHP.

## How to apply on another live server

SSH into the server, then from the project root of the school-erp deployment:

```bash
# 1. Download the patches
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0001-fix-mobile-timetable-empty.patch -o /tmp/p1.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0002-fix-teacher-attendance-empty.patch -o /tmp/p2.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0003-fix-attendance-date-format.patch -o /tmp/p3.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0004-fix-parent-flow-server.patch -o /tmp/p4.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0005-fix-admission-default-password.patch -o /tmp/p5.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0006-feat-user-management-bulk-actions.patch -o /tmp/p6.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0007-fix-bulk-reset-enum-cast.patch -o /tmp/p7.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0008-fix-csrf-419-bulk-actions.patch -o /tmp/p8.patch

# 2. Dry-run — fails loudly if conflicts
git apply --check /tmp/p1.patch /tmp/p2.patch /tmp/p3.patch /tmp/p4.patch /tmp/p5.patch /tmp/p6.patch /tmp/p7.patch /tmp/p8.patch

# 3. Apply
git apply /tmp/p1.patch /tmp/p2.patch /tmp/p3.patch /tmp/p4.patch /tmp/p5.patch /tmp/p6.patch /tmp/p7.patch /tmp/p8.patch

# 3b. Patch 6 modified a Vue file — rebuild the frontend bundle
npm install --no-audit --no-fund
npm run build

# 4. Reload PHP so OPcache picks up the new code
php artisan queue:restart
sudo systemctl reload php8.3-fpm               # adjust PHP version
```

If `git apply --check` reports a conflict, the file on that server has diverged from `trivartatech/school-erp@257c756`. In that case open the `.patch` file and apply the change by hand — all four diffs are short.

## Note on the parent multi-child switcher

Patch 4 includes the backend support for `X-Active-Student-Id` (server already honored it; what changed is `childList()` returning correct keys so the UI can drive a switcher). The corresponding mobile-app changes — `ActiveChildContext`, axios interceptor, tap-to-switch UI on the Children screen — live in the React Native source (`schools-1/`) and ship with the **next Expo build**, not via these patches.

## Verify

Log in as a teacher in the mobile app:
- **Timetable tab** → days with classes should populate (dot indicators appear under those tabs).
- **Attendance tab** → if the teacher has punched in / been marked at least once this month, % and stats should show; otherwise still empty until punch records exist.

You can also probe the endpoint directly:

```bash
curl -H "Authorization: Bearer <teacher_token>" https://<that-domain>/api/mobile/timetable | head -c 200
# expect: "schedule":{"monday":[...

curl -H "Authorization: Bearer <teacher_token>" https://<that-domain>/api/mobile/attendance | head -c 200
# expect: "summary":{"present":N,...
```
