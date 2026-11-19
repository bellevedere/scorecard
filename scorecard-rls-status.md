## New — per-task Description field (requested 2026-11-17 by Jean/Liyun to match Vimigo's layout, built same day, **confirmed fully deployed 2026-11-18**)

Liyun wanted each task to carry a long-form, multi-line description (matching Vimigo's own "DESCRIPTION" column next to its "VIMIGOAL" column) — Scorecard tasks previously had only a single-line `text` (the task's short name/title), nothing for detailed steps.

**Built into `index.html` (2026-11-17)**:
- New optional `description` field on every task, editable wherever tasks are created/edited: `NewGoalForm` (brand-new goals), the project-goal task list/add-task row, and the annual-goal monthly task editor (`AnnualGoalCard`'s draft tasks) — a textarea under the task name in each case.
- New `updateTaskDescription(goalId, taskId, description)` function, same pattern as `updateTaskText`/`updateTaskWeight`.
- `addTaskToGoal` and `addGoal` both take/persist `description` now; every place a task gets copied, snapshotted, or reloaded (live load from Supabase, `goal_month_history` snapshots, `goal_future_versions` snapshots, `applyLiveGoalDefinition`'s update/insert, `scheduleFutureVersion`/`commitCurrentMonthEdit`'s `tasks_snapshot`) carries `description` through — checked exhaustively by grepping every site that builds a task object, not just the obvious ones.
- Staff see it: `TaskRow` now shows the description as small grey text directly under the task title, always visible (not gated behind the "add remark" expand).
- Phase 9 Excel export: the Details Report's `Description` column (previously always blank, see the Phase 9 section below) now pulls this real field instead.
- Validated via a real Babel (`@babel/preset-react`) transform pass before delivery, same as every other `index.html` change.

**DB migration — confirmed run 2026-11-18**:
```sql
alter table goal_tasks add column if not exists description text;
```

**`rollover_annual_goals()` Postgres function — also needed a fix, found during this pass and confirmed run 2026-11-18**: the function explicitly names columns in two places (the `jsonb_build_object` that freezes a month's tasks into `goal_month_history`, and the `insert into goal_tasks` that applies a scheduled future version) rather than using `select *` — so without this fix, `description` would have silently vanished from annual/recurring goals the first time they rolled over a month, both in frozen history and in the live task list. Fixed by adding `'description', coalesce(description, '')` to the snapshot build and `description` to the future-version re-insert's column list and `select`. Full corrected function given to Jean; assumed name `rollover_annual_goals()` with no arguments based on earlier notes in this doc — **not independently re-verified against the actual Dashboard function signature**, worth a quick confirm next time this function comes up if anything looks off.

**Still needs porting to the MySQL/PHP migration (`scorecarddev.ingenious.sg`)** once that environment is reachable again (see "scorecarddev PHP execution blocked" below) — same `description` column on the PHP backend's `goal_tasks` table, same field in its task-creation/edit forms, same display to staff, same export column. Flagged, not yet started on that side.

## scorecarddev.ingenious.sg — PHP execution completely broken, escalated to hosting support (2026-11-16/17)

The MySQL/PHP migration's dev subdomain stopped executing any `.php` file at all — a brand-new, blank test file at the document root itself (`public_html/scorecard-mysql/test2.php`) returned a generic LiteSpeed 404, same as every real endpoint. Static files (`index.html`, CSS, JS, images) in the same folder load fine.

**Ruled out, in order, each confirmed not the cause**: wrong file path/case, file permissions (0644 files, 0755 on every parent folder up through `public_html` — found `scorecard-mysql` and `public_html` both wrongly at `0750`, fixed to `0755`, no change), a hidden `.htaccess` deny rule inside `api/` (none found), ModSecurity (disabled for the domain, no change), PHP version not assigned (confirmed PHP 8.5 explicitly assigned via MultiPHP Manager). Even with everything above checked and fixed, the exact same 404 persists for any `.php` file anywhere under the document root.

**Escalated**: the only account contact available (via Liyun) turned out to be a plain cPanel-level login, not WHM/root — they couldn't find anything either. **Not yet resolved.** Next step given to Jean: try deleting and recreating the `scorecarddev` subdomain itself in cPanel (same name, same document root) to force a fresh vhost regeneration — a common fix for a subdomain whose UI-level settings look right but whose underlying Apache/LiteSpeed vhost never got built correctly — and failing that, contact the actual hosting company (not just the cPanel-level account contact) directly, since this needs real server-side vhost config / error log access neither of us has. **Status as of 2026-11-18: unresolved, work on the MySQL side paused until this clears.**

A client reported two related problems on the live site:

1. **No edit function for Department or Employee** — only "add" existed. To fix a mistake, her only option was to archive and re-add.
2. **Re-adding an employee under the same email failed** after she archived the old department and the employee in it, then tried adding that same person (same real email) into a new department.

**Corrected diagnosis**: my first write-up of this (logged earlier the same day) guessed the cause was a hard delete leaving an orphaned `auth.users` row — that was wrong, written before actually reading `index.html`. Once read, the code showed `archiveEmployee`/`restoreEmployee` and `archiveDepartment`/`restoreDepartment` already existed and already used the same safe `active:false` archive pattern as companies — nothing is ever hard-deleted, so no auth user was ever orphaned. The real gap was simpler: `updateEmployee` (already able to patch `name`/`role`/`deptId`) existed in the code but wasn't wired to any button, and there was no `updateDepartment` function at all. So the only way to move an employee to a different department was archive + brand-new row — which collided on the unique email the moment she tried it.

**First fix (2026-11-06, part 1) — real Edit**:
- New `updateDepartment(deptId, { name, colorHex })`, wired to a pencil icon per department inside "Manage departments" — inline rename + recolor, Save/Cancel.
- `updateEmployee` is now actually used: a pencil icon on each employee row in People management opens an inline edit form (name, access rights, company, department) that calls it directly. Email is deliberately left read-only there — changing it would also need a matching Supabase Auth update, which this form doesn't attempt.
- Net effect: "move this employee to a different department" is now a direct edit — no archive/re-add, no email collision, ever.
- The client's immediate stuck case was already resolved separately — she deleted the stale record directly in the Supabase dashboard.

**Follow-up ask from Jean (2026-11-06, same day)**: even with Edit now covering the original scenario, archiving an employee *still* leaves their real email permanently stuck on the archived row forever (by design — archive never changes `employees.email` or the Auth login). Jean's ask: when an employee is archived, actually free up the email so it can be reused right away, without needing a manual Supabase dashboard fix.

**The data-safety tradeoff, and the decision made**: freeing an email by deleting the Supabase Auth user outright would cascade-delete (`employees.id` FK's to `auth.users(id) on delete cascade`) — almost certainly wiping that employee's row and, by further FK chains, their check-in/goal/approval history too, making Restore meaningless. Given the choice between (a) that destructive true-delete, (b) freeing the email non-destructively, or (c) leaving archive as-is and adding a separate "permanently delete" action — **chose (b)**: keep all history, just free the email.

**Second fix (2026-11-06, part 2) — archive frees the email, non-destructively**:
- Archiving/restoring an employee now goes through two new **Edge Functions** (`archive-employee`, `restore-employee`) instead of a plain client-side `.update()`, because only the service-role key can rename someone else's Supabase Auth email — same reason `create-employee`/`admin-reset-password` had to be Edge Functions.
- `archive-employee`: sets `active:false` **and** renames the employee's Auth login email to a unique placeholder (`localpart+archived-<timestamp>@domain`) — frees the real address immediately. `employees.email` is deliberately left untouched, so the real email still displays correctly everywhere (People list, exports) — only the login credential changes.
- `restore-employee`: sets `active:true` and tries to hand the real email back to the Auth login — but only if no other employee row has since claimed it. If it's been reused elsewhere (most likely scenario), restore still proceeds for history/visibility, but the frontend shows a warning that this person needs a new login email/password before they can sign in again (`emailRestored:false` in the response).
- Both Edge Functions were written to match the exact pattern already used by the existing `create-employee` function (found as a reference file in this session's outputs: `CORS_HEADERS`/`json()` helper, trying `SUPABASE_SERVICE_ROLE_KEY`/`SUPABASE_SECRET_KEY`/`SUPABASE_SECRET_KEYS` in order since this project may be on either Supabase key-naming generation, same caller-is-ceo check via a direct `employees` table read).
- **Deployed 2026-11-17**: both functions created via Supabase Dashboard → Edge Functions → New function → pasted `index.ts` → Deploy, done in the correct order (both functions live before the matching `index.html` was uploaded to cPanel), so Archive/Restore never broke for live users during the rollout.

## Result: RLS verified complete (2026-09-17)

Supabase project "Scorecard", `main` (production) branch. All 13 `public` schema tables have RLS enabled and policies applied, proven by SQL tests run directly in the SQL editor (`begin; set local role ...; set local "request.jwt.claims" = ...; ... rollback;` pattern).

## Role model (confirmed with Belle)
Two tiers only — no third tier needed:
- **staff**: self-scoped via `employee_id = auth.uid()` (own records) or `my_company_id()` (own company's shared data). Cannot self-approve `goal_records` — `approval <> 'approved'` is baked into both the INSERT `with_check` and UPDATE `using`/`with_check` clauses, so once a record is approved it becomes physically un-editable by its owner (frozen-approval-weight pattern).
- **ceo**: global superadmin, intentionally NOT scoped by company. This is the Ingenious internal team's role — deliberate design, not a bug. Approval of KPIs is centralized through this role; there is currently no per-company manager/admin tier (client companies like ZHotel do not get their own approver).

Enforced via two `SECURITY DEFINER` helper functions: `my_role()` and `my_company_id()`.

## Bootstrapping the first admin account (clarified 2026-09-28)

The app has **no public sign-up form**, and the `create-employee` Edge Function (the only way to create a new login from inside the app) explicitly checks that the caller is already signed in as a `ceo` before it will do anything — so there's a chicken-and-egg problem on a brand-new setup: nobody can log in to create the first account, because creating an account requires already being logged in as one.

The first admin on any fresh Supabase project must be created manually, outside the app:
1. Supabase Dashboard → Authentication → Users → Add User — enter email + password, check "Auto Confirm User."
2. Copy that new user's UUID from the Users list.
3. Supabase Dashboard → Table Editor → `employees` → insert a row: `id` = that UUID, `name`, `email` (matching the login), `role` = `ceo`, `company_id`/`dept_id` set appropriately, `active` = `true`.

Once that row exists, that person can log in normally, and everything else — more admins, staff, companies, departments, goals, rewards — can be created from inside the app itself, since `ceo` isn't scoped to one company. This only needs doing once per fresh Supabase project. The current live setup already has this done (see "Test Admin" under Known test identities below); it only matters again if a brand-new environment is ever spun up from scratch.

## Tables
Original 13: `check_ins`, `companies`, `departments`, `employees`, `goal_assignments`, `goal_future_versions`, `goal_month_history`, `goal_records`, `goal_tasks`, `goals`, `notifications`, `reward_enrollments`, `reward_schemes`.

Added since (2026-09-20): `reviews` and `rewards`. **Both are confirmed dead weight, not planned features — see "Reviews is not a real feature" below and the Phase 6 notes.** Neither has any UI planned; both stay unused. Candidates to actually drop in Phase 10 rather than just leave alone, since we now know for certain nothing will ever consume them.

`employees.id` is FK'd to `auth.users(id) on delete cascade` — every employee row (including test/seed rows) needs a real Supabase auth user first.

`goals.live_month` (text) — server-side-only bookkeeping for annual/recurring goals; see "Monthly rollover" section below. The frontend never writes it directly (only reads it back as `liveMonth`, currently unused by any UI).

`goal_records.approval` is plain `text` (confirmed 2026-09-28 via `information_schema.columns` — not a Postgres enum), currently guarded by a check constraint allowing `none`/`pending`/`approved`/`rejected` — see "Rejection is now a real state" below for why a 4th value was added and the exact migration run.

`goal_records` has **no synthetic `id` column** (confirmed 2026-09-29 via `information_schema.columns`) — its natural key is the composite `(employee_id, goal_id, task_id)`, matching the `upsert(..., { onConflict: "employee_id,goal_id,task_id" })` the frontend already uses. Full column list: `employee_id`, `goal_id`, `task_id`, `done`, `approval`, `remark`, `admin_remark`, `submitted_at`, `awarded_points`, `awarded_weight`, `attachment_url`, `no_attachment_reason`.

## Reviews is not a real feature (corrected 2026-09-28)

A `reviews` table + `review-attachments` bucket were added to the schema on 2026-09-20 mirroring a `StaffReview` mock component that existed in the original prototype (a dedicated "submit a client review" form, never mounted in the app's nav). This got built out into a full "Phase 7" (a Reviews tab for staff + a new admin Reviews screen, wired to real Supabase storage/inserts) — **this was a misread of what Belle actually needed, corrected the same day, and fully reverted from `index.html`.**

Belle's actual model: "monthly Google review" is just a **regular goal task**, using the task system that's existed since Phase 3/4 — admin ticks both `requires_attachment` and `requires_remark` on a task (e.g. "Monthly Google review from customers"), and staff get exactly the flow this needs for free: upload a screenshot, write a remark (which client, what work), or tick "no attachment" and give a reason if there was no review that month (`noAttachmentReason` — already fully wired end-to-end, including in the admin Approvals view). **Nothing new needed building for this — it already worked.**

Separately, "admin reviews" in the sense of the admin's own commentary already has two existing, real mechanisms — no third one needed:
- `adminRemark` on `goal_records` — the "Note to staff…" field admin fills in when approving/rejecting a task submission (Phase 3, `AdminApprovals`).
- `remark` on `reward_enrollments` — the per-employee note admin can attach when setting a reward override (Phase 6, `updateRewardEmployeeOverride`).

Net result: the `reviews` table / `review-attachments` bucket / `StaffReview` component were dead code with no real use case behind them. **`StaffReview` and its `addReview`/`updateReview`/`rewards`(mock)/`addReward` plumbing were fully deleted from `index.html` on 2026-09-29 as part of Phase 10** (see below) — no longer just reverted to an unmounted state, actually removed. The `reviews` table and `review-attachments` bucket still need dropping from Supabase itself — SQL given in the Phase 10 section below.

### `reward_schemes` / `reward_enrollments` columns (confirmed 2026-09-28 via `information_schema.columns`)
- `reward_schemes`: `id`, `dept_id`, `company_id`, `goal_id`, `start_date`, `end_date`, `base_amount`, `active`, `future_base_amount`, `future_effective_month`.
- `reward_enrollments`: `scheme_id`, `employee_id`, `update_amount`, `remark` — **no separate "enrolled" boolean column**. A row's mere presence for a given `(scheme_id, employee_id)` pair IS the enrollment; that same row carries the employee's override amount/remark. Un-enrolling deletes the row outright rather than flipping a flag.
- **Action needed from Belle, not yet confirmed run**: a unique index is required on `reward_enrollments (scheme_id, employee_id)` for the frontend's upsert (`onConflict: "scheme_id,employee_id"`) to work:
  ```sql
  create unique index if not exists reward_enrollments_scheme_employee_uidx
    on reward_enrollments (scheme_id, employee_id);
  ```
  Without this, `updateRewardEmployeeOverride` (editing an amount/remark in the admin Rewards screen) will fail — flag this to Belle if that action starts erroring.

## Rejection is now a real state (fixed 2026-09-28, confirmed working live 2026-09-28)

Belle tested the Phase 9 Excel export against real data and found a genuine bug, not a report-formatting nitpick: when admin hits "Un-tick" on a submitted task (`adminUntick`), the code reset `approval` all the way back to `"none"` — identical to a task the employee never touched at all. Two consequences: staff got zero feedback that their submission had been reviewed and turned down (the UI silently reverted to looking untouched), and every export/report showed "not started" for what was actually rejected work, with no visibility into the admin's own review remark.

Fixed by giving rejection its own real value:
- `adminUntick` now sets `approval: "rejected"` (previously `"none"`) — still resets `done`/`awardedPoints` so staff can freely redo and resubmit (submitting flips it back to `"pending"`, same as before). Notification text changed from "was un-ticked" to "was rejected by management."
- **Staff now see it**: `TaskRow`'s not-done view shows a rose "Rejected — please redo and resubmit" banner plus the admin's remark, where before a rejected task looked completely identical to a fresh unassigned one.
- **Admin now sees it distinctly too**: `AdminApprovals`'s status column and `AdminPeople`'s per-employee quick-view both got a "Rejected" label/color, previously falling through to the generic empty-state styling. Rejected rows also correctly stay visible in the Approvals table (the row filter is `rec.done || rec.approval !== "none"` — since rejected records are no longer `"none"`, they don't disappear like they did under the old bug).
- **Excel export**: `scoreTask`'s bucket is now `approved`/`rejected`/`open` (previously just `approved`/`open`, with no way to tell rejected from never-touched). Report 1's `Rejected` column is now a real count instead of always `0`. Report 2's `Approval Status` now says `"Rejected"` for these rows instead of the misleading `"Not Started"`.
- **Column swap, per Belle's request**: the Details Report's `"User Submitted Points"` column (a Vimigo self-report field Scorecard never had a real equivalent for) is now `"Admin Review"`, populated by the same `adminRemark` shown in Approvals — since un-tick/reject is a real decision admin makes and annotates, that's more useful in this slot than a synthetic self-claim number.
- **DB migration**: `goal_records.approval` is plain `text` (not an enum), so the fix was a check-constraint swap: dropped whatever constraint restricted the column to the old 3 values, added one allowing `none`/`pending`/`approved`/`rejected`. Belle confirmed this ran successfully.

**First live re-test (2026-09-28) initially still failed** — but the code itself wasn't the problem. Belle tested on the live `ingenious.sg/scorecard/` site and saw the old behavior (notification still said "was un-ticked," rejected tasks still vanished from Approvals, export still said "Not Started"). The giveaway was the exact old notification wording still showing up — that only happens if the live site is running the pre-fix `index.html`. Root cause: **the fixed file had been delivered but not yet uploaded to cPanel**, so the live site was still serving the old build; the SQL migration alone doesn't change frontend behavior. Once Belle uploaded the latest `index.html` to cPanel and re-tested, everything worked correctly: Approvals shows "Rejected" status, Details Report shows "Rejected" with Admin Review populated. **Confirmed fully working live by Belle 2026-09-28.**

**Lesson for future updates**: since deployment is cPanel-upload-based (not an auto-deploying pipeline), any code fix only takes effect once Belle actually uploads the new `index.html` to cPanel — a DB migration alone, or Claude having "sent the file," is not the same as it being live. Worth explicitly confirming the upload step when handing off a fix, not just the SQL/code change.

## Corrupted attachment on an already-approved task, and the real "stuck edit" bug (found + fixed 2026-09-29)

Belle reported an attachment that wouldn't open in the admin Approvals view — clicking it threw `Failed to fetch`/`net::ERR_FAILED` in the console, and the label showed garbled text ending in `"isImage":true}` instead of a filename. Traced to one specific row: `employee_id = da7a56b0-7606-4657-ae85-a29579f1ca50` (Test Staff), `task_id = 8db96626-5c89-4ed2-a30d-78482ec5039d` ("Contact customer and note down reviews"). Its `attachment_url` literally contained a raw JSON-stringified object — `{"name":"WhatsApp Image...","dataUrl":"data:image/jpeg;base64,<huge blob>","isImage":true}` — instead of a real Supabase Storage path. `createSignedUrl()` tried to use that whole blob as a path, producing an invalid, oversized request URL that failed outright.

That shape (`{name, dataUrl, isImage}`) is a fingerprint: it's exactly how the *original pre-Supabase mock prototype* held a picked file in memory, confirmed by checking that code in git history. So this row predates (or bypassed) the real Phase 4 upload wiring — legacy/seed data, not something the current upload code can produce (`submitTaskWithDetails` always saves a clean `uploadData.path` or `null`, never the raw file object).

**Belle then reported the value kept reappearing even after she deleted the attachment and uploaded a new one in the app**, and worried it was a caching bug ("if yes, will kinda be a disaster as management can't see it"). It isn't caching — it's the frozen-approval RLS policy working exactly as designed, combined with the app giving no feedback when that policy blocks a write:

- This task was already showing **Approved** in her screenshot. Once `approval = 'approved'`, `TaskRow` renders the fully read-only view (`rec.done` branch) — there has never been an "Attach file"/edit control shown for an approved task, only a static `📎 filename` line. So no UI action on an approved task could have re-uploaded anything through the normal attach flow.
- The only way to *appear* to edit it was via the checkbox itself: tapping it on a done task called `toggleTask`, which optimistically updated local state to look unticked/editable *before* checking whether the database would actually accept the change. But staff's own RLS policies require `approval <> 'approved'` on both the read-modify and the write — so that `UPDATE` was silently rejected by Postgres. The local screen showed success; the database never changed. On the next login, the app re-fetched from Supabase and the untouched old row reappeared — exactly what Belle saw, and not caching at all.
- `persistRecord` (the single function all task writes funnel through) only ever `console.error`'d a failed save — it never told the person at the keyboard that anything had gone wrong. That silence is what made a blocked write indistinguishable from a successful one until the next reload.

**Fixed in `index.html` (2026-09-29), delivered to Belle, needs uploading to cPanel to go live:**
1. `toggleTask` now checks `cur.approval === "approved"` up front and refuses with a real toast — "This task has already been approved — ask your admin to un-tick it if you need to change or resubmit anything" — instead of optimistically flipping local state into a dead end.
2. `persistRecord` now shows a toast ("Couldn't save that change — please refresh the page and try again.") on *any* database error, not just this one — so a blocked/failed save can never again look identical to success. This is a general safety net, not specific to attachments.

**Still needed to actually clear the corrupted row** — the code fix above stops *new* instances of this, but the existing bad value in the database needs a one-time cleanup once admin un-ticks/rejects that specific task (which resets `done: false`, unlocking a real re-submission) — or directly via SQL:
```sql
-- find any rows with this corrupted shape
select employee_id, task_id, attachment_url
from goal_records
where attachment_url like '{%';

-- clear them once confirmed
update goal_records set attachment_url = null
where attachment_url like '{%';
```
**Not yet confirmed run by Belle.** After running it (or after admin un-ticks the specific task and staff resubmits a real file), the attachment should open normally in Approvals.

## RLS fixes applied
1. **`my_role()` / `my_company_id()` search_path**: both were `SECURITY DEFINER` with `proconfig = NULL` (no pinned `search_path`) — a search_path-hijack risk since every policy on every table depends on them. Fixed with `alter function my_role() set search_path = public, pg_temp;` (same for `my_company_id()`).
2. **Missing staff policies**: `reward_schemes` and `reward_enrollments` originally had only the `ceo full access` policy — staff had zero visibility into rewards. Added:
   - `staff read own company reward schemes` (select, using `company_id = my_company_id()`)
   - `staff read own reward enrollments` (select, using `employee_id = auth.uid()`)
   Verified with a real cross-company test: seeded one `reward_schemes` row each for Ingenious and ZHotel, queried as Ingenious staff (`da7a56b0-7606-4657-ae85-a29579f1ca50`), got back exactly the Ingenious row.
3. **`my_role()`/`my_company_id()` executable by `anon`**: database linter flagged these `SECURITY DEFINER` functions as callable by unauthenticated requests (default Postgres `EXECUTE` grant to `PUBLIC`). Revoked from `public`, granted only to `authenticated`.

## Known test identities
- Ingenious staff: `da7a56b0-7606-4657-ae85-a29579f1ca50` ("Test Staff")
- Ingenious ceo: `05bcf1f8-06e8-48f2-8c48-42a499a7ca14` ("Test Admin") — this is the account that was originally bootstrapped manually per "Bootstrapping the first admin account" above.
- ZHotel staff: created during this pass (new auth user + matching `employees` row, dept = Front Office) — id not recorded here, check `employees` table if needed again.

## Storage — verified complete (2026-09-20)

Four private buckets, all `public: false`:
- `task-attachments` — screenshots of completed tasks (this is what the "monthly Google review" screenshot actually uploads to — it's a task attachment like any other)
- `checkin-photos` — location photo taken at check-in
- `letters-of-proof` — PDF evidence
- `review-attachments` — **dead, see "Reviews is not a real feature" above** — nothing uploads here and nothing should

Folder convention: `{company_id}/{employee_id}/{record_id}/{filename}`. Ownership enforced via `storage.foldername(name)[2] = auth.uid()::text` (note the `::text` cast — folder segments are `text[]`, unlike the `uuid` columns used in table RLS).

Staff get `select`/`insert` only on their own files (no `update`/`delete` — matches the frozen-approval philosophy, evidence shouldn't be swappable after upload). CEO gets one `for all` policy covering all four buckets via `my_role() = 'ceo'` (policy was dropped and recreated to add the 4th bucket).

**check_ins table**: added `address text` column to hold a reverse-geocoded, Malaysian-format address (e.g. "33, Jln Harmonium, Taman Desa Tebrau, 81100 Johor Bahru, Johor") — populated client-side/edge-function at check-in time from `lat`/`lng`, not by the database itself.

**Auto-delete after 90 days**: implemented as a Supabase Edge Function (`clever-handler` — deployed via Dashboard, kept its auto-generated name rather than being renamed to `cleanup-old-files`; renaming the "Name" field does NOT change the slug/URL) scheduled via `pg_cron` + `pg_net`, daily at 03:00 UTC (11:00 AM MY time). Job name: `scorecard-cleanup-old-files`.

Key implementation detail: the function does NOT query `storage.objects` directly via `supabase.schema("storage").from("objects")` — that schema isn't exposed via PostgREST by default and silently fails. Instead uses a narrow `SECURITY DEFINER` SQL function `public.get_old_storage_object_names(bucket, cutoff)` called via `.rpc()`, then deletes via `supabase.storage.from(bucket).remove(paths)` (the correct way — deleting `storage.objects` rows directly via raw SQL does not reliably remove the underlying file bytes from the storage backend).

Service role key is stored in Supabase Vault (`service_role_key` secret), never hardcoded in the cron job SQL.

Verified working: manual trigger of `net.http_post(...)` against the function returned `{"task-attachments":0,"checkin-photos":0,"letters-of-proof":0}` (confirmed via `select * from net._http_response`) — all zero as expected since nothing is 90+ days old yet. The job also already fired once on its own schedule (03:00 UTC).

**Note**: the cleanup only deletes files, not the referencing rows — after 90 days, `check_ins.photo_url` / equivalent columns on `goal_records` will point at deleted files unless the frontend/function is later extended to null those out too. Flagged, not yet built. Does **not** need extending to cover `review-attachments` — that bucket is dead (see above), not a real gap.

## Known linter warnings — status
- `pg_net` extension in public schema (WARN, cosmetic) — **left as-is**. `ALTER EXTENSION ... SET SCHEMA` isn't supported for `pg_net`; fixing it requires drop-and-recreate, which risks breaking the now-working cron job for a purely cosmetic flag. Not worth it.
- `my_role()`/`my_company_id()` executable by anon — **fixed** (see RLS fixes above).
- Leaked password protection disabled — **not yet resolved**; couldn't locate the toggle (should be under Authentication → Attack Protection), may be gated behind a paid plan. Low priority, revisit later if needed.

## Notes for later
- `anon` role gets a hard `permission denied` (not empty rows) on all tables — no `GRANT` exists for `anon` at all. Stronger than RLS alone; fine as-is unless a future feature needs public/pre-login read access (e.g. a public leaderboard), which would need an explicit `GRANT` + permissive policy just for that case.
- `relforcerowsecurity = false` on all tables — expected/fine, only matters for table-owner/superuser connections, which the app never uses (this is also what lets `rollover_annual_goals()`, below, read/write across every employee's rows despite RLS).
- **Deployment is via cPanel now, not GitHub Pages** (clarified by Belle 2026-09-28). A local git clone of `github.com/NotJean729/scorecard` still sits in the working environment (commit history from prior sessions, several commits ahead of origin, never pushed) — per Belle, leave it alone; do not commit or push from here unless she explicitly asks. She still uploads code to GitHub herself as a backup/reference, but it is **not** what serves the live site anymore — `ingenious.sg/scorecard/` is now served via cPanel, so "where does this update actually go live" is cPanel, not a GitHub Pages rebuild. **Confirmed 2026-09-28**: a delivered file/SQL migration is not "live" until Belle actually uploads it to cPanel — this caused a false-alarm re-test failure once already (see "Rejection is now a real state" above). Worth flagging the upload step explicitly on every future handoff.
- **2026-11-06**: the local git clone referenced above was gone (fresh session environment) — re-cloned fresh from `github.com/NotJean729/scorecard` to make the Edit-function fix. That remote copy was current as of its last push (2026-10-01, Phase 10 + rejection-state fixes all present) — worth a quick diff/eyeball against whatever's actually live on cPanel before trusting it blindly next time, same caution as always with this repo.

## Frontend migration progress (last confirmed against actual index.html source: 2026-11-06)

Single-file `index.html` (~4700+ lines, no-build React via CDN + in-browser Babel), source backed up at `github.com/NotJean729/scorecard`, live at `ingenious.sg/scorecard/` served via **cPanel** (not GitHub Pages — see deployment note above). Belle has no local git/push access — every update is delivered as a file, she uploads it herself (to cPanel to go live; to GitHub separately, just as a backup).

Supabase JS client added via `<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2">`, initialized with the project URL and the **publishable/anon key** (hardcoded client-side — this is intentional and safe, it is not the service_role key).

**A caution about this doc going forward**: this pass found that both Notifications and the Excel export were already substantially real (wired to Supabase / using live data) despite this doc having tracked them as "not started" — most likely built in an earlier session whose work never got written back here. Before scoping a "next phase" from this doc alone, it's worth a quick grep of the actual `index.html` to confirm the doc still matches reality, same as was done this pass. **Repeated again 2026-11-06**: the "no edit function" bug report led to assuming archive/restore for employees/departments didn't exist yet — it actually already did, just undocumented and half-wired. Always grep the real source before writing up a fix plan.

**Phase 1 — Auth: done.** Real `signInWithPassword`, session restore via `getSession`/`onAuthStateChange`, `signOut`. Extended 2026-09-27 with a working forgot-password flow: `resetPasswordForEmail` (with explicit `redirectTo` pointing at the app's own URL) → email a one-time code → `verifyOtp({ type: "recovery" })` → `updateUser({ password })`. Note: this Supabase project's `mailer_otp_length` is 8 (not Supabase's usual 6) — the frontend's `OtpBoxes` component and `verifyCode` length check are both hardcoded to 8 to match.
- Also confirmed working this pass: custom SMTP via Gmail (`smtp.gmail.com`) — switching to a real Gmail account + matching Gmail app password resolved both an auth mismatch and connection timeouts hit with other providers.
- **No public sign-up flow exists** — every login is created either by the manual bootstrap process above (first admin only) or via the `create-employee` Edge Function called by an already-signed-in `ceo` (everyone after that).

**Phase 2 — Core reference data (companies/departments/employees): done**, including the employee-creation gap, and now including real Edit for both Department and Employee (fixed 2026-11-06 — see "Fixed" section at the top of this doc).
- `companies`, `departments`, `employees` fetched live from Supabase after login (old `COMPANIES_SEED`/`DEPARTMENTS_SEED`/`buildEmployees()` demo generators — **deleted from `index.html` 2026-09-29**, see Phase 10 below).
- **Real employee creation**: `create-employee` Edge Function (service-role-backed, atomically creates both the `auth.users` row and the `employees` row) is called from the admin UI. Requires the caller to already be signed in as `ceo` — see "Bootstrapping the first admin account" above for the one-time manual exception.
- **Admin password reset**: `admin-reset-password` Edge Function lets a signed-in `ceo` force-set any employee's password directly (`auth.admin.updateUserById`) — built for staff who have no real email address.
- **Employee/department archive+restore**: never hard-deletes — `active:false`/`active:true` toggles, same pattern as companies. Wired to Trash2/Undo2 buttons in People management and the Manage Departments modal.
- **Employee/department edit**: `updateEmployee(empId, { name, role, deptId })` and `updateDepartment(deptId, { name, colorHex })`, each wired to a pencil icon that opens an inline edit form. Email is not editable from here on purpose — see "Fixed" section above.

**Phase 3 — Goals: done** (structure, tasks, assignment, staff completion, admin approval) — **this is also what covers "monthly Google review"-style tasks**, via `requires_attachment`/`requires_remark`/`noAttachmentReason` on `goal_tasks`/`goal_records`. No separate reviews feature needed (see correction above).

**Phase 3b — Monthly rollover for annual/recurring goals: done and confirmed live (2026-09-27).** All five pieces confirmed present via a status-check query.

**Phase 4 — Task attachments: done.** Uploads go to the private `task-attachments` bucket, signed URLs (`createSignedUrl`, 300s) used for admin viewing. This is the bucket "Google review screenshot"-style tasks actually use. **Caveat found 2026-09-29**: once a task is approved, staff have no way to replace a bad/corrupted attachment themselves — see "Corrupted attachment on an already-approved task" above. That's RLS working as designed (frozen-approval philosophy), but the app previously gave no feedback when this blocked a save; now fixed to show a clear message instead of failing silently.

**Phase 5 — Check-in + reverse geocoding: done.** Check-in photos go to the `checkin-photos` bucket; lat/lng are reverse-geocoded server-side via a `reverse-geocode` Edge Function.

**Phase 6 — Rewards: done and confirmed working (2026-09-28).** Scoped down deliberately after checking with Belle: the ad-hoc `rewards` table has no consuming UI and never will (see correction above — same fate as `reviews`). Only `reward_schemes` (which has a full working admin screen — `AdminRewards`/`RewardSchemeDetail`/`NewRewardSchemeForm`) got wired to Supabase.
- Data-loading effect fetches `reward_schemes` and `reward_enrollments`, reshaping enrollment rows into the `enrolledEmployeeIds`/`employeeOverrides` shape the UI already expected.
- `addRewardScheme` inserts into `reward_schemes` first (async, mirrors `addGoal`) so local state holds the real DB id before any enroll/override actions reference it.
- `archiveRewardScheme`/`restoreRewardScheme`/`toggleRewardActive` persist `active` via a plain `update`. `toggleRewardEnrollment` inserts/deletes the `reward_enrollments` row (row-presence-as-enrollment). `updateRewardEmployeeOverride` upserts onto `(scheme_id, employee_id)`.
- **Known pre-existing gap carried over as-is**: nothing ever reads `future_base_amount`/`future_effective_month` back to actually apply the "reverts next month" promise — no rollover job exists for `reward_schemes`. Flagged for later.
- **Bug found and fixed during first real test**: "Create scheme" silently did nothing — `NewRewardSchemeForm`'s `goalId` state got stuck stale. Fixed with a `useEffect` resync + a visible toast instead of a silent no-op. **Confirmed working by Belle.**
- **Action needed from Belle**: run the `reward_enrollments` unique-index SQL above before relying on per-employee override/remark editing in the admin Rewards screen.

**"Phase 7" does not exist as previously scoped.** What was tracked as "Phase 7 — Reviews" (a dedicated `reviews` table/bucket/UI) was a misunderstanding, corrected and reverted the same day it was built — see "Reviews is not a real feature" above.

**Phase 8 — Notifications: turned out to already be done, discovered 2026-09-28.** This doc had it listed as "still fully local/mock," but the actual code (`index.html` lines ~1360-1412 as of this pass) shows it's real: `notifications` table loaded per-employee on login, kept live via a Supabase Realtime `postgres_changes` subscription (staff see a new notification the instant admin approves/edits something, no refresh needed), `addNotification` inserts into the table, `markNotificationsRead` persists read-state via `update`. No work needed — this was a stale doc, not a real gap. Unclear which earlier session actually built this; the doc just never got updated to reflect it.

**Phase 9 — Reports/Excel export: done and confirmed fully working live (2026-09-28).** The export already existed and was already using real Supabase-backed data (`employeePoints`/`employeeGoals`/`getRecord`, all wired since Phase 3) — but its columns were a simplified ad-hoc shape (Period/Department/Employee/Score/Remarks/Approved/Pending), not either of the two formats in `vimigo-reference-report-formats.md`. Per Belle's choice, rebuilt to produce **both** Vimigo formats in one workbook, replacing the old simplified sheet entirely:
- **"Performance by Month" sheet** (Vimigo's "Performance by Month, Grouped by vimiGoal"): one row per employee per goal per month — `Month, Full Name, Department, vimiGoal Name, Period, Completion (%), Completed Score, Total Score, Completed Goals, Total Goals, Approved, Pending, Rejected`.
- **"Details Report" sheet**: one row per task per employee per month — `Period Start Date, Period End Date, Period, Full Name, Department, vimiGoal Set, Goal, Description, Achievable Points, Admin Review, Remarks, Admin Approved Points, Approval Status, Completed Date, Completed Time, Completion (%), Rewards, vimiGoal Settings Status`. (`Admin Review` replaces Vimigo's `User Submitted Points` — see "Rejection is now a real state" above for why.)
- Both sheets are generated from a shared helper (`monthRowsForGoal` + `scoreTask`) that reuses `AdminApprovals`'s exact month-resolution logic — for annual/recurring goals this means `getMonthView(goal, isoMonth)` (skipping "scheduled"/"inherited" future months), for one-off goals a direct `goal.startDate.slice(0,7) === isoMonth` check. This matters because the old simplified report used `employeeGoals()`/`goal.tasks` directly, which only reflects an annual goal's *current live* task set — picking a past month in the exporter would have silently shown this month's tasks mislabeled as that past month. The new version reads the correct frozen snapshot from `goal_month_history` for past months instead.
- Scoring reuses `employeePoints`'s own rule: approved tasks count at their frozen weight/points (`awardedWeight`/`awardedPoints`), so a later change to a task's live weight never retroactively rewrites a closed month's numbers.
- **Real bug found on first live test (2026-09-28), fixed same day, confirmed resolved after re-test**: Belle tested with real ZHotel data and found un-ticking a task ("rejecting" it, in her terms) showed as "Not Started" in the export with no visibility into the admin's review — see "Rejection is now a real state" above for the full fix (new `rejected` approval state, DB constraint migration, staff/admin UI updates, Details Report column swap to `Admin Review`) and for the deployment gotcha that made the first re-test attempt look like it had failed (fix wasn't uploaded to cPanel yet). **Belle confirmed the full flow works correctly after uploading the updated file: Approvals shows "Rejected," Details Report shows "Rejected" status with Admin Review populated.**
- Remaining judgment calls, flagged for Belle to correct if wrong:
  - "Description" (per-task) and "Rewards" (per-task) have no Scorecard equivalent — left blank rather than guessing at a value or duplicating another column into them.
  - "Total Score"/"Achievable Points" show Scorecard's real per-goal totals rather than Vimigo's always-100 sample pattern, per the 2026-09-21 decision to keep task weights flexible.
- On-screen preview has a toggle between the two report shapes (trimmed columns for screen width); the **download always produces both sheets** in one `.xlsx` regardless of which one is being previewed.
- The Leaderboard screens (`AdminLeaderboard`/`StaffLeaderboard`) were already fully real (built on `employeePoints`, wired since Phase 3) — nothing needed there.

**Phase 10 — Final cleanup: code side done 2026-09-29, DB side still needs Belle to run one SQL pass.**

Code cleanup done in `index.html` (2026-09-29), delivered to Belle, verified with a real Babel transform pass (not just eyeballed) before sending — **needs uploading to cPanel to take effect live, same as any other code change**:
- Deleted entirely (all confirmed dead — zero remaining call sites, checked by grep before removal): `COMPANIES_SEED`, `DEPARTMENTS_SEED`, `DEPT_COUNT`, `FIRST`/`LAST` name lists, `DEMO_PASSWORD`, `buildEmployees()`, `BASELINE_TASKS`, `EXTRA_GOAL`, `buildSeedState()` — the entire old in-memory demo-data generator, superseded since Phase 1–3 by real Supabase fetches.
- Deleted the dead `reviews`/`rewards` mock state (`useState`), `addReview`/`updateReview`/`addReward` mutators, their entries in `sharedProps`, and the never-mounted `StaffReview` component. (Careful distinction made here: the *mock* `rewards`/`setRewards` state is gone, but the *real* `rewardSchemes`/`setRewardSchemes` state — the actual Phase 6 Rewards feature — is untouched and still fully wired.)
- Kept `nextId`/`idCounter` — initially filed under the same "seed data" section but actually still live, used by the real task-creation draft UI (`AdminGoalDetail`'s "add task" flow) and as a fallback id when mapping loaded tasks. Confirmed via grep before touching anything.
- Bonus fix, same pass: `manifest.json`'s `theme_color`/`background_color` were a generic teal/light-slate placeholder that never matched the actual app icon (a blue/purple neon checkmark on a near-black background). Sampled the icon's real colors and updated both to match (`theme_color: #2fb8fe`, `background_color: #0b0b12`) — purely cosmetic (affects PWA splash screen / OS chrome color only), flag to Belle in case she'd prefer different exact shades.

**Still needs a DB-side pass from Belle** — three independent, unrelated fixes bundled into one SQL editor session since they're all "run once and done":

```sql
-- 1. Required for the admin Rewards screen's per-employee override/remark editing to work at all
--    (Phase 6 — updateRewardEmployeeOverride upserts onto this pair and needs a real unique constraint to target).
create unique index if not exists reward_enrollments_scheme_employee_uidx
  on reward_enrollments (scheme_id, employee_id);

-- 2. Clears the one legacy corrupted attachment (Test Staff / "Contact customer and note down reviews")
--    that was saved as a raw JSON blob instead of a real storage path — see "Corrupted attachment on an
--    already-approved task" above for the full diagnosis. Safe to run even if already clean (matches nothing).
update goal_records set attachment_url = null
where attachment_url like '{%';

-- 3. Drops the dead reviews table now that its code (StaffReview) is gone from index.html too.
--    Confirmed nothing reads or writes this table anywhere in the current source.
drop table if exists reviews;

-- 4. Required for the new "archive a company" flow (reason + letter of proof) — see
--    "Company archiving now requires a reason + letter of proof" below. Without this, archiving
--    any company will error immediately.
alter table companies
  add column if not exists archived_reason text,
  add column if not exists archived_proof_url text,
  add column if not exists archived_at timestamptz;
```

The `review-attachments` storage bucket can't be dropped via SQL — that's a Dashboard action: Storage → `review-attachments` → delete bucket (only safe once the table above is dropped too, so nothing could be pointing at files in it).

## Company archiving now requires a reason + letter of proof (built 2026-09-29)

`letters-of-proof`'s purpose, clarified by Belle: in the multi-company model, "deleting" a company (really archiving — Scorecard never hard-deletes, see `archiveCompany` below) is meant to be gated. Admin must state a reason and attach a letter of proof before a company can be archived; the company's departments/goals/people all stay intact, just hidden, and can be brought back with Restore.

**Previously**, `archiveCompany` in `AdminCompanies` was a single click on the trash icon — no reason, no file, no confirmation at all. That gap is now closed:
- `archiveCompany(companyId, { reason, proofFile })` now requires both a non-empty reason and a file; it uploads the file to the `letters-of-proof` bucket first (path `{company_id}/{timestamp}-{filename}`, same sanitization pattern as task attachments), and only flips the company to `active: false` after that upload succeeds. Stores `archived_reason`, `archived_proof_url`, `archived_at` on the `companies` row.
- `AdminCompanies`'s trash-icon click now opens an inline confirm panel (reason textarea + required file attach) instead of archiving instantly; "Confirm archive" stays disabled until both are filled in.
- An archived company's card now shows the stored reason, archive date, and a "View letter of proof" link (signed URL, same pattern as `AdminApprovals`'s `openAttachment`).
- **Restoring a company deliberately does NOT clear** `archived_reason`/`archived_proof_url`/`archived_at` — kept as a permanent record of the last archival, per the "letter of proof" being compliance documentation, not a transient UI state.
- No new storage RLS needed — the existing `ceo`-role `for all` policy already covers all four buckets including `letters-of-proof`, and this whole flow is admin-only (`AdminCompanies`).
- **Drive-by fix in the same touched code**: `companies` loaded from Supabase were never getting their `description` mapped into local state (only companies added mid-session had one) — a company's description silently vanished after every page reload. Fixed alongside this change since it's the exact same loading code.

**DB migration needed from Belle** (not yet confirmed run) — add to the SQL pass below:
```sql
alter table companies
  add column if not exists archived_reason text,
  add column if not exists archived_proof_url text,
  add column if not exists archived_at timestamptz;
```
Without this, `archiveCompany`'s `update` will error on the unrecognized columns and no company can be archived until it's run.

## Roadmap position
✅ Supabase project created
✅ Schema (all tables; `reviews`/`rewards` turned out to be unneeded scaffolding, see above)
✅ Auth configured, test account created, forgot-password flow working (2026-09-27); first-admin bootstrap process documented (2026-09-28)
✅ Companies/departments seeded, test employee linked to real login
✅ Row Level Security policies — verified 2026-09-17
✅ Storage buckets (photos, attachments, letters of proof) — verified 2026-09-20; `review-attachments` unused by design
✅ Rewire the frontend to actually use all this instead of memory — Phase 1 (Auth) ✅, Phase 2 (core reference data + employee creation + edit) ✅, Phase 3 (goals, including review-style tasks) ✅, Phase 3b (monthly rollover) ✅, Phase 4 (attachments) ✅, Phase 5 (check-in + geocoding) ✅, Phase 6 (reward schemes) ✅, Phase 8 (notifications) ✅ confirmed already done 2026-09-28, Phase 9 (Vimigo-format reports/export) ✅ built and fully confirmed working live 2026-09-28 (including the rejection-state fix)
🔄 **Phase 10 (final cleanup)**: code-side cleanup done 2026-09-29 (dead mock generators + reviews/rewards-mock code removed, manifest colors fixed) — needs cPanel upload to go live. DB-side still needs one SQL pass from Belle (reward_enrollments unique index, corrupted-attachment cleanup, drop `reviews` table) plus dropping the `review-attachments` bucket via Dashboard and clarifying `letters-of-proof`'s purpose.
✅ Test end-to-end (reject a task, confirm it saves and shows correctly in both the app and the export) — confirmed working live 2026-09-28
✅ Redeploy (via cPanel) — confirmed done 2026-09-28
🔄 **2026-09-29 fixes delivered in one `index.html`** (attachment-lock error feedback + Phase 10 dead-code removal + manifest colors) — needs cPanel upload; DB-side SQL pass (above) still needs running.
✅ **2026-11-06 fixes, confirmed fully deployed 2026-11-17** — one `index.html` (real Edit for Department/Employee, plus archive/restore now freeing/restoring the employee's email) + two new Edge Functions `archive-employee`/`restore-employee` (see "Fixed" section at top). Both Edge Functions deployed to Supabase first, `index.html` uploaded to cPanel after — correct order followed, so Archive/Restore stayed working for live users throughout.
