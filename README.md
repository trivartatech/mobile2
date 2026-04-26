# Backend patches for school-erp mobile API

Standalone patches for the `school-erp` Laravel API. Apply them on a deployment that needs the fixes without pulling the full `school-erp` history.

## Patches in this repo

| File | What it fixes |
|------|---------------|
| `0001-fix-mobile-timetable-empty.patch` | Teacher/student timetable showed "No classes" every day (server keyed schedule by `day_of_week` int instead of lowercase day name). |
| `0002-fix-teacher-attendance-empty.patch` | Teacher Attendance tab showed 0% with no records (server's `attendance()` had no teacher branch — only resolved student attendance). Adds a `staff_attendances` lookup for teachers. |

Both touch the same file: `app/Http/Controllers/Api/MobileApiController.php`. They apply cleanly in order.

## How to apply on another live server

SSH into the server, then from the project root of the school-erp deployment:

```bash
# 1. Download the patches
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0001-fix-mobile-timetable-empty.patch -o /tmp/p1.patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0002-fix-teacher-attendance-empty.patch -o /tmp/p2.patch

# 2. Dry-run — fails loudly if conflicts
git apply --check /tmp/p1.patch /tmp/p2.patch

# 3. Apply
git apply /tmp/p1.patch /tmp/p2.patch

# 4. Reload PHP so OPcache picks up the new code
php artisan queue:restart
sudo systemctl reload php8.3-fpm               # adjust PHP version
```

If `git apply --check` reports a conflict, the file on that server has diverged from `trivartatech/school-erp@da600f7`. In that case open the `.patch` file and apply the change by hand — both diffs are short.

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
