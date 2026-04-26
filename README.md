# Backend patches for school-erp mobile API

Standalone patches for the `school-erp` Laravel API. Apply them on a deployment that needs the fixes without pulling the full `school-erp` history.

## Patches in this repo

| File | What it fixes |
|------|---------------|
| `0001-fix-mobile-timetable-empty.patch` | Teacher/student timetable showed "No classes" every day (server keyed schedule by `day_of_week` int instead of lowercase day name). |
| `0002-fix-teacher-attendance-empty.patch` | Teacher Attendance tab showed 0% with no records (server's `attendance()` had no teacher branch — only resolved student attendance). Adds a `staff_attendances` lookup for teachers. |
| `0003-fix-attendance-date-format.patch` | Attendance records rendered raw ISO datetimes like `2026-04-01T18:30:00.000000Z`. Now formats `records[].date` using the school's `date_format` setting from Admin → Settings → System Config. Also adds reusable `School::dateFmt()` / `School::timeFmt()` helpers for future use. |
| `0004-fix-parent-flow-server.patch` | Three parent-flow fixes: (a) `childList()` returned wrong keys so children showed name only — now returns `class_name`, `section_name`, `admission_number`, `roll_number`, `attendance_pct`. (b) Mobile parent BusTracking screen polled `/api/bus/status` which 404'd — added the route and `MobileApiController::busStatus()` returning running/stopped/idle for the active child's bus. (c) Payment history & receipt now include `payment_date_display` formatted per the admin's `date_format` setting (alongside the existing `payment_date` field for backward compat). |

Patches 1, 2, 4 touch `app/Http/Controllers/Api/MobileApiController.php`. Patch 3 touches the same controller plus `app/Models/School.php`. Patch 4 also adds a route in `routes/api.php`. They apply cleanly in order.

## How to apply on another live server

SSH into the server, then from the project root of the school-erp deployment:

```bash
# 1. Download the patches
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0001-fix-mobile-timetable-empty.patch -o /tmp/p1.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0002-fix-teacher-attendance-empty.patch -o /tmp/p2.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0003-fix-attendance-date-format.patch -o /tmp/p3.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0004-fix-parent-flow-server.patch -o /tmp/p4.patch

# 2. Dry-run — fails loudly if conflicts
git apply --check /tmp/p1.patch /tmp/p2.patch /tmp/p3.patch /tmp/p4.patch

# 3. Apply
git apply /tmp/p1.patch /tmp/p2.patch /tmp/p3.patch /tmp/p4.patch

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
