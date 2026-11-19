# Scorecard — Handoff Reference

This is a backup reference, not a spec to follow literally. Its purpose: capture the business rules
that got worked out through prototype iteration, so they don't have to be reverse-engineered from
clicking around the HTML file. Your team owns the real architecture decisions from here.

---

## 1. What the prototype is (and isn't)

The prototype (`kpi_tracker_v17.jsx` / `_standalone.html`) is plain JavaScript + JSX, React 18, compiled
in-browser via Babel Standalone, styled with Tailwind CDN. No backend, no database — all state lives in
React memory and resets on page refresh. Passwords are plaintext, "authentication" is a lookup against
an in-memory array. **None of this should carry into the real system** — it was built purely for fast,
no-rebuild iteration on requirements.

The **business logic and data relationships**, however, were refined over many rounds of real feedback
and are worth preserving.

---

## 2. Core entities

**Company** — top-level tenant. Has `name`, optional `description`, `active` flag (soft delete).

**Department** — belongs to exactly one Company (`companyId`). Has a `colorHex` (admin picks via a real
color plate — no preset palette), `active` flag.

**Employee** — belongs to exactly one Department (and therefore, transitively, one Company). Fields:
`name`, `email`, `password`, `role` (`staff` | `ceo`/admin), `deptId`, `active` flag. Admins additionally
carry `assignedCompanyIds` if company-scoped admin access is reintroduced (built once, then explicitly
removed per client decision — see §7).

**Goal** — belongs to one Department. Two kinds:
- **Annual/recurring** (`isAnnual: true`): repeats monthly across a date range (`startDate`/`endDate`
  spans the *whole* range, e.g. a fiscal year — not just one month). Tabs are generated per month within
  that range.
- **Project/one-off**: fixed `startDate`/`endDate`, no monthly repetition.

A goal can be marked **core** (`isTeamGoal: true`) — starred, cannot be deleted, and *can* still have a
hand-picked member subset (core does NOT mean "auto-assign whole department" by default; only the
originally-seeded department scorecard uses `autoAssignWholeDept: true` to auto-include new hires).

**Task** — belongs to a Goal. Has `text`, `weight` (a point value — tasks do *not* need to sum to 100),
optional `requiresAttachment` and `requiresRemark` flags (independent of each other; a task can require
both, either, or neither).

**GoalRecord** — the join of (employee, goal, task): tracks `done`, `approval` (`none|pending|approved`
— note: **no "rejected" state**, only Approve or Un-tick), `awardedPoints`, `remark` (staff's own, never
admin-editable), `adminRemark` (admin's separate note to staff, never overwrites staff's remark),
`attachment`, `awardedWeight` (see §3, critical).

**RewardScheme** — belongs to a Department, tied to one Goal (for context/tracking, not calculation),
spans a date range. Has `baseAmount` (a starting number, purely a convenience default), `active` toggle.
Each enrolled employee gets a `employeeOverrides[empId] = { updateAmount, remark }` — **`updateAmount` is
a free-form number the admin can set to anything, per employee, with no formula constraining it.** There
is deliberately no "commission" auto-calculation — that was tried and explicitly removed per client
feedback in favor of full manual control.

**Check-in** — GPS + photo capture record, `type: "in"|"out"`, timestamp, `withinWindow` flag.

---

## 3. Critical business rule: frozen approval weight

**This is the single most important rule in the system**, and got the strongest positive client reaction
of anything built. When a task is approved, the task's *current* weight is copied into the record as
`awardedWeight` at that exact moment. If admin later changes the task's live point value:

- **Already-approved records are completely unaffected** — they keep scoring against their original
  `awardedWeight` forever, not the new weight.
- **Pending/untouched records are unaffected too** (no data wipe) — they simply get judged against
  whatever the task's current definition is whenever admin eventually reviews them.

All percentage calculations (`employeePoints`, annual running totals, Approvals table denominators) must
use `awardedWeight ?? task.weight` for approved records, and live `task.weight` for everything else. The
UI surfaces this explicitly when it happens ("Approved under old 40-pt criteria (now 70 pts)").

**Why this matters for your data model:** a task's weight is not simply "current state" — approval
records need their own point-value snapshot, independent of the task's live definition. Don't design a
system where task weight is a single mutable field with no history at the record level.

---

## 4. Annual goal versioning ("edit current / future / both")

Annual goals show one month-tab per month in their date range. For any given month, the goal is one of:
**live** (the actual current month — fully editable, staff can tick), **past** (frozen historical
snapshot, read-only), or **scheduled/future** (a queued change not yet in effect).

When admin edits a *live* month's tasks or assignment, a scope choice is required:
- **This month only** — applies now, but a "revert to old definition" is automatically scheduled for
  next month (so the change is temporary).
- **This month & every future month** — applies now and persists indefinitely.

Past months are immutable once frozen. This is the same pattern reused for Reward Schemes' base-amount
edits.

**Why this matters:** your backend needs an explicit month-scoped versioning table for recurring goals
(and reward schemes), not just a single mutable "current definition" row — otherwise you can't represent
"this changed for August only" vs "this changed from August onward."

---

## 5. Scoring model

Percentage-based, not summed-to-100:

```
percentage = (sum of awarded points across approved tasks) / (sum of relevant weights) × 100
```

Denominator per task: `awardedWeight` if approved (frozen), else current `task.weight`. No cap at 100 —
it's a natural ratio. This applies per-goal, per-employee-overall, and as a running annual total.

---

## 6. Soft delete everywhere

Companies, Departments, Goals, and Employees all use `active: false` rather than hard deletion, so
historical Reports/Approvals data stays referentially intact. Company deletion specifically requires a
reason + an uploaded "letter of proof" before the delete button even enables — this was a deliberate
friction point to prevent accidental company loss, with the reason/letter preserved in an archive view.

**Recommendation:** carry the soft-delete pattern into the real system for the same integrity reasons —
especially for anything Reports or Approvals might reference historically.

---

## 7. Explicitly tried and reverted

Worth knowing so it isn't rebuilt unnecessarily:

- **Super-admin / admin / company-access-restriction hierarchy** was fully built (three-tier login,
  guarded company deletion, per-admin company assignment) and then **explicitly removed** by the client
  in favor of the simpler single-admin-role model. If multi-tenant admin scoping becomes a real
  requirement later, the design is proven out and described above (§2, Employee.assignedCompanyIds) —
  but it is not part of the current direction.
- **Reward "commission" auto-calculation** (base amount × reward% × goal completion%) was built, then
  simplified to plain manual per-employee amounts. Don't reintroduce a formula without confirming it's
  actually wanted.
- **Multi-company employees** (one person working across companies with different departments per
  company) was built, then reverted to "one employee, one company, one department" — simpler and matches
  actual usage.

---

## 8. Suggested target stack (as originally scoped, before prototyping began)

- **Backend:** Node.js + TypeScript, PostgreSQL, Prisma ORM
- **Hosting:** Render/Railway to start, AWS/GCP if it needs to scale
- **Frontend:** proper React build (not CDN/Babel), likely a PWA given check-in/camera usage
- **File storage:** attachments and letters-of-proof are currently base64-in-memory; will need real
  object storage (S3-compatible) in production

This is a starting point, not a mandate — worth revisiting now that the full feature scope (multi-company,
recurring goals with versioning, reward schemes, guarded deletion) is much clearer than when it was first
sketched.
