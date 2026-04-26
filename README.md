# Timetable fix — backend patch

A single backend patch for the school-erp Laravel API. It fixes the mobile teacher/student timetable showing "No classes" on every day.

## What it changes

`app/Http/Controllers/Api/MobileApiController.php` — the `timetable()` method now keys the response by lowercase day names (`monday`..`sunday`) instead of the raw `day_of_week` integer (1..7), matching what the mobile app expects.

12 lines added, 2 lines removed. No migrations, no composer changes, no JS.

## How to apply on another live server

SSH into the server, then from the project root of the school-erp deployment:

```bash
# 1. Download the patch
curl -L https://raw.githubusercontent.com/trivartatech/mobile2/main/0001-fix-mobile-timetable-empty.patch -o /tmp/timetable-fix.patch

# 2. Apply it
git apply --check /tmp/timetable-fix.patch    # dry run, fails if conflicts
git apply /tmp/timetable-fix.patch             # actually applies

# 3. Reload PHP so OPcache picks up the new code
php artisan queue:restart
sudo systemctl reload php8.3-fpm               # adjust PHP version
```

If `git apply --check` complains about conflicts, the file on that server has diverged from the version at commit `10c88ab` of `trivartatech/school-erp`. In that case open the patch file and apply the change by hand — it's small.

## Verify it's live

```bash
curl -H "Authorization: Bearer <teacher_token>" https://<that-domain>/api/mobile/timetable | head -c 200
```

You should see `"schedule":{"monday":[...` — if it still shows `"schedule":{"1":[...`, OPcache is still serving the old code.
