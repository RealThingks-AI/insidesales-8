

## Backup System - Complete Fix and Improvements

### Issues Found

1. **Scheduled Backups never fire** -- `is_enabled: true` but `last_run_at` and `next_run_at` are both null. No cron job or edge function exists to check the schedule and trigger backups.
2. **Module Backup missing modules** -- UI only shows Contacts, Accounts, Deals, Tasks. Missing: Leads, Notifications.
3. **Scheduled Backups have no scope setting** -- Can't choose full vs module backup for schedules.
4. **Backup History has no scrollbar** -- Uses `max-h-[400px]` with `overflow-auto` but no proper ScrollArea component.
5. **Delete option should be removed** from history per user request.
6. **Restore logic issues** -- `profiles` and `user_roles` are in `BACKUP_TABLES` but missing from `DELETE_ORDER` and `INSERT_ORDER` in restore function. Also `security_audit_log` is not backed up.
7. **Delete dialog and related code** still present but should be removed.
8. **Storage download fails** for non-service-role clients (no storage RLS policies on backups bucket).

---

### Plan

#### 1. Add Leads and Notifications to Module Backup UI

**File: `src/components/settings/BackupRestoreSettings.tsx`**
- Add to `MODULES` array:
  - `{ id: 'leads', name: 'Leads', icon: FileText, color: 'text-blue-500' }`
  - `{ id: 'notifications', name: 'Notifications', icon: Bell, color: 'text-yellow-500' }`
- Add `Bell` to lucide imports
- Update `fetchModuleCounts` to include `leads` and `notifications` tables
- Change grid from `md:grid-cols-5` to `md:grid-cols-6` to fit 6 modules

#### 2. Add `leads` module mapping to edge function

**File: `supabase/functions/create-backup/index.ts`**
- Add `leads: ['leads', 'lead_action_items']` to `MODULE_TABLES`
- Add `security_audit_log` to `BACKUP_TABLES`

#### 3. Fix Restore logic -- add missing tables to DELETE_ORDER and INSERT_ORDER

**File: `supabase/functions/restore-backup/index.ts`**
- Add `profiles` and `user_roles` to both `DELETE_ORDER` and `INSERT_ORDER` in correct positions
- Add `security_audit_log` to both arrays
- For `DELETE_ORDER`: `profiles` and `user_roles` should come after `user_sessions` (they are parent-like)
- For `INSERT_ORDER`: `profiles` and `user_roles` should come before `user_preferences` (they are parents)

#### 4. Scheduled Backups -- create actual cron infrastructure

**a. New edge function: `supabase/functions/scheduled-backup/index.ts`**
- Reads `backup_schedules` where `is_enabled = true` and `next_run_at <= now()`
- For each due schedule, calls the `create-backup` function logic internally (fetches tables, uploads to storage)
- Supports `backup_scope` (full or module name) from the schedule record
- Updates `last_run_at = now()` and computes `next_run_at` based on frequency
- Uses service role key, no JWT verification needed

**b. Database migration:**
- Enable `pg_cron` and `pg_net` extensions
- Add `backup_scope` column (text, default 'full') and `backup_module` column (text, nullable) to `backup_schedules`
- Create hourly cron job that calls `scheduled-backup` edge function

**c. Add to `supabase/config.toml`:**
```
[functions.scheduled-backup]
verify_jwt = false
```

**d. Update Schedule UI** in `BackupRestoreSettings.tsx`:
- Add frequency dropdown (Daily, Every 2 Days, Weekly)
- Add scope dropdown (Full System, or each module name)
- Show last run and next run timestamps
- Save `backup_scope` and `backup_module` to the schedule record

#### 5. Backup History -- ScrollArea and remove delete

**File: `src/components/settings/BackupRestoreSettings.tsx`**
- Import `ScrollArea` component
- Wrap the history table in `<ScrollArea className="h-[400px]">` for proper scrollbar
- Remove the Delete button from each history row (lines 516-527)
- Remove `handleDeleteClick`, `handleDeleteConfirm`, delete-related state, and the Delete AlertDialog
- Add a status badge column showing completed/failed/in_progress with colors
- Add a refresh button to the history header

#### 6. Storage RLS -- allow admin downloads

**Database migration:**
- Add SELECT policy on `storage.objects` for the `backups` bucket allowing authenticated users who are admins (using `is_user_admin()` function) to read/download files

---

### Technical Details

**Files to create:**
- `supabase/functions/scheduled-backup/index.ts`

**Files to modify:**
- `supabase/functions/create-backup/index.ts` -- add `leads` module, add `security_audit_log`
- `supabase/functions/restore-backup/index.ts` -- add `profiles`, `user_roles`, `security_audit_log` to ordering arrays
- `src/components/settings/BackupRestoreSettings.tsx` -- add modules, scheduled backup scope UI, ScrollArea for history, remove delete
- `supabase/config.toml` -- add `scheduled-backup` function config

**Database migrations:**
1. Enable `pg_cron` and `pg_net` extensions
2. Add `backup_scope` and `backup_module` columns to `backup_schedules`
3. Create cron job: hourly call to `scheduled-backup` edge function
4. Add storage RLS policy for backups bucket admin read access

**Scheduled backup edge function logic:**
```text
1. Read backup_schedules where is_enabled = true
2. For each schedule where next_run_at <= now():
   a. Determine tables based on backup_scope/backup_module
   b. Fetch all data, create JSON, upload to storage
   c. Insert backup record with backup_type = 'scheduled'
   d. Update schedule: last_run_at = now(), next_run_at = computed
3. Return summary
```

**Restore ordering fix:**
```text
DELETE_ORDER (children first):
  deal_action_items, lead_action_items, action_items,
  notifications, notification_preferences, saved_filters,
  column_preferences, dashboard_preferences, user_sessions,
  security_audit_log,
  deals, contacts, leads, accounts,
  user_preferences, yearly_revenue_targets, page_permissions,
  keep_alive, profiles, user_roles

INSERT_ORDER (parents first):
  profiles, user_roles,
  accounts, leads, contacts, deals,
  lead_action_items, deal_action_items, action_items,
  notifications, notification_preferences, saved_filters,
  column_preferences, dashboard_preferences, user_sessions,
  user_preferences, yearly_revenue_targets, page_permissions,
  keep_alive, security_audit_log
```
