# Save7 OS — factual inventory

Source: `github.com/gilbertlieb/save7-os`, cloned read-only to
`/private/tmp/claude-501/-Users-zubayrparak-Desktop-Nerve-Center/6b2c277b-83de-4bd6-8864-a0c907a3ccce/scratchpad/save7-os`.
All paths below are relative to that clone. Every claim is cited. Where a claim could not be
verified from the repository it is marked **unverified**.

---

## Executive summary — how much of this is finance?

1. Finance is where it started and is no longer the bulk of it. 12 of 68 live tables (17.6%) are money tables; 16 are projects, 17 are learning, 6 volunteers, 4 meetings, 4 research, 3 assets.
2. By feature, `docs/map.md` lists 28 features; 6 are money (`docs/map.md:18-25`). 22 are not.
3. By migration, 25 of 103 migrations are finance; volunteers (16) + learning (16) = 32 together outnumber finance.
4. The sidebar has 16 admin destinations. Five sit under the "Money" heading (`src/02-markup.html:71-91`). Eleven do not.
5. The whole volunteer half is a second application — its own build, its own domain, its own auth identity, no `people` row (`build-vol.mjs`, `supabase/migrations/0069_volunteers.sql:93-121`).
6. An entire e-learning platform (the "Transplant Alchemy" course, 17 `learn_*` tables plus content) was migrated *into* this database at `0091`-`0101`.
7. Research, meetings with action points, an asset register, a project tracker with reach/KPIs/deliverables/risks, and a funder (CSR) read-only portal are all here and none of them are accounting.
8. But: **finance holds all the power.** `app_role` is a 4-value enum (`0001_init.sql:24`) and `finance_admin` is the only role with a general write. Every non-finance write is a narrow, individually-granted exception.
9. So the honest answer is: **operationally broad, permissions-wise finance-shaped.** The surface is an operations OS; the authorisation model is a finance app's.
10. CLAUDE.md's own first line still says "The app tracks the money" (`CLAUDE.md:8`) — the document under-describes its own system.

---

## 1. Role & permission model

### 1.1 The only role enum

```sql
create type app_role as enum ('branch_member', 'branch_manager', 'exco', 'finance_admin');
```
`supabase/migrations/0001_init.sql:24`. **Never extended.** No `alter type ... add value` exists in
any of the 103 migrations (grep over `supabase/migrations/*.sql` for `add value` returns nothing).

`people.app_role` is `not null default 'branch_member'` (`0001_init.sql:47`).
`people.role` beside it is **free text job title** — "job title, free text ('Finance lead')"
(`0001_init.sql:46`). That is where CEO / COO / CFO / "Operations admin" / "Workspace
administrator" live, and they carry **no** authority (`0021_team_roster.sql:42-45`,
`0015_naz_operations_admin.sql:11-22`, `0035_admin_account.sql:6-9`).

`0015_naz_operations_admin.sql:20-22` states it outright: *"there is no higher level to grant and no
separate 'operations' permission set — app_role is a four-value enum and finance_admin is the top
of it. The title says operations; the permissions are the same as finance."*

### 1.2 Every distinct actor the system recognises

| Actor | Identity table | How it authenticates | Recognised by |
|---|---|---|---|
| `branch_member` | `people` | Google OAuth, `@save7.org` only | `app_role()` |
| `branch_manager` | `people` | as above | `app_role()`, `app_manages_branch()` |
| `exco` | `people` | as above | `app_can_read_org()` |
| `finance_admin` | `people` | as above | `app_is_finance()`, `app_can_read_org()` |
| Cosignatory (orthogonal flag) | `people.can_cosign` | as above | `app_can_cosign()` (`0023_cosign_outgoing.sql:50-53`) |
| Project lead / Ops Mentor (per-row) | `projects.lead_person_id`, `project_tracker.ops_mentor` | as above | `app_runs_project()` (`0068_project_tracker.sql:482-491`) |
| Project member (per-row) | `project_people` | as above | `app_on_project()` (`0047_projects.sql:184-192`) |
| Research lead (per-row) | `research_items.lead_person` | as above | `app_leads_research()` (`0100_research_tracker.sql:184-191`) |
| Action-point owner (per-row) | `meeting_action_owners` | as above | `app_owes_action()` (`0104_action_points_owed_by_several.sql:52-59`) |
| Funder / CSR stakeholder | `stakeholders` | password account at `@csr.save7.org` | `app_is_stakeholder()` (`0048_csr_stakeholders.sql:164-172`) |
| Volunteer | `volunteers` | password account at `@volunteer.save7.org` | `app_is_volunteer()` (`0073_volunteer_opportunities.sql:85-88`) |
| Learner | `learners` | own account; may or may not be a volunteer | `app_is_learner()` (`0091_learn_course_bank.sql:97-100`) |

Only the first four are enum values. Everything else is either a boolean column, a per-row
membership, or a separate identity table. **There is no role registry** — the actor list above is
assembled from twelve independent mechanisms.

### 1.3 The helper functions, and exactly what each returns

All are `language sql stable security definer set search_path = public`.

| Function | Defined at | Returns |
|---|---|---|
| `app_person_id()` | `0002_rls.sql:28-31` | `people.id` for `auth.uid()`; **null** for funders, volunteers, learners, and any signed-in account with no `people` row |
| `app_branch_id()` | `0002_rls.sql:33-36` | the caller's single `people.branch_id` |
| `app_role()` | `0002_rls.sql:38-41` | the caller's `app_role` enum value |
| `app_is_finance()` | `0002_rls.sql:43-46` | `app_role() = 'finance_admin'` |
| `app_can_read_org()` | `0002_rls.sql:48-51` | `app_role() in ('exco','finance_admin')` |
| `app_is_staff()` | `0070_external_accounts_read_nothing.sql:90-93` | `app_person_id() is not null`. Comment: *"The line between staff and an external account (a CSR funder, a volunteer), which is not the same line as any role check."* (`:95-96`) |
| `app_can_cosign()` | `0023_cosign_outgoing.sql:50-53` | `people.can_cosign`, set explicitly for four addresses (`:44-48`) and never inferred from the job title (`:41-42`) |
| `app_manages_branch(b text)` | `0072_branch_manager_volunteers.sql:64-69` | `app_role()='branch_manager' and b = app_branch_id()`. **False for finance** — deliberately (`:71-72`) |
| `app_on_project(p uuid)` | `0047_projects.sql:184-192` | lead of, or member of, project `p`. Governs **reading** |
| `app_runs_project(p uuid)` | `0068_project_tracker.sql:482-491` | `projects.lead_person_id = me` **or** `project_tracker.ops_mentor = me`. Governs **writing** the tracker. Deliberately narrower than `app_on_project()` (`:493-494`) |
| `app_manages_project_branch(p uuid)` | `0073_volunteer_opportunities.sql:97-105` | the caller manages the branch that owns project `p`; false for an org-level project (null branch) by construction (`:93-96`) |
| `app_is_stakeholder()` | `0048_csr_stakeholders.sql:164-172` | an **active** row in `stakeholders` for `auth.uid()` |
| `app_can_read_csr()` | `0048_csr_stakeholders.sql:174-177` | `app_is_stakeholder() or app_can_read_org()` — so exco and finance can preview exactly what a funder sees (`:162-163`) |
| `app_volunteer_id()` | `0073_volunteer_opportunities.sql:80-83` | `volunteers.id` where `user_id = auth.uid() and active`. **Null for staff** (`:78-79`). Scoping to `active` is where revocation bites (`:90-91`) |
| `app_is_volunteer()` | `0073_volunteer_opportunities.sql:85-88` | `app_volunteer_id() is not null` |
| `app_learner_id()` / `app_is_learner()` | `0091_learn_course_bank.sql:92-100` | the caller's active `learners.id` / whether there is one |
| `app_leads_research(i uuid)` | `0100_research_tracker.sql:184-191` | named lead of research item `i` |
| `app_owes_action(a uuid)` | `0104_action_points_owed_by_several.sql:52-59` | one of an action point's owners |

### 1.4 The three parallel permission mechanisms

CLAUDE.md names three (`CLAUDE.md:877-880`, `CLAUDE.md:899-901`). The code confirms all three exist
and confirms that **only one of them is enforcement**.

**(a) RLS in Postgres — the real control.**
`supabase/migrations/0002_rls.sql:3-20` states the position: *"RLS is the security boundary for this
app. The JavaScript sign-in gate in the prototype is … 'a demonstration, not a control', and nothing
here trusts it."* And `:19-20`: *"Everything keys off auth.uid() resolved through people.user_id.
Nothing keys off a client-supplied role."*

The general write grant in `0002_rls.sql` is `app_is_finance()` and nothing else — `branches_write`
(`:67-68`), `people_write` (`:77-78`), `transactions_write` (`:84-85`), `fuel_finance_write`
(`:104-105`), `spend_finance_write` (`:122-123`), `seats_write` (`:130-131`), `repay_write`
(`:137-138`), `receipts_finance_write/delete` (`:168-171`), `perms_write` (`:176-177`).

**`exco` never appears in a write policy anywhere in the 103 migrations.** A grep for `'exco'`
across `supabase/migrations/*.sql` returns only the enum definition (`0001_init.sql:24`), the
`app_can_read_org()` body (`0002_rls.sql:51`), and the documentation-only `role_permissions` seed
rows (`0004_seed_reference.sql:64-71`). Exco is org-wide **read**, plus the exceptions in 1.6.

**(b) `PERM_SPEC` / `PERMS` / `can()` in `src/06-branch-portal.js` — affordances only, and
role-blind.**

```js
const PERMS = Object.fromEntries(PERM_SPEC.map(([k, , , d]) => [k, d]));
const can = k => !!PERMS[k];
```
`src/06-branch-portal.js:65-66`. `PERM_SPEC` is a 9-entry array of
`[key, title, description, default]` (`:54-64`).

**Finding the CLAUDE.md does not state:** `PERMS` is built from the *defaults column only* and is
never assigned to again anywhere in `src/`, `src-vol/` or `test/` (grep for `PERMS[` / `PERMS =` /
`PERMS.` returns only lines 65 and 66). `can()` takes no person and reads no role. So the eight
non-`approve` keys are **`true` for every user, always**, and `approve` is **`false` for every user,
always** (`:55-63`). The mechanism is a static feature-flag table, not a permission system.
`src/06-branch-portal.js:8-9` describes it accurately — *"a client constant on purpose — it gates
affordances, nothing more"* — but `CLAUDE.md:878` calling it "what the branch portal offers" implies
a per-role variation that does not exist.

Consequence: `can('approve')` is `false` even for `finance_admin`. The one place it is read on the
admin side is `src/07-c-forms.js` (`can('approve')`), so that branch is dead for everybody.
`view_own_projects` and `view_receipts` (`:60,:62`) are read only via the `data-perm` attributes on
the member nav (`src/02-markup.html:164,172`), never by a `can()` call in a render function.

The runtime Permissions drawer that used to toggle these **has been deleted**
(`src/06-branch-portal.js:21-22`, `CLAUDE.md:863-864`) — but `gate()` still prints *"Turn
`<key>` on in Permissions to show it"* (`src/06-branch-portal.js:88`), pointing at UI that no longer
exists.

**(c) `role_permissions` — documentation only. VERIFIED.**

Table: `role_permissions (role app_role, permission text, granted bool, scope perm_scope,
primary key (role, permission))` — `0001_init.sql:158-164`. Seeded with 32 rows = 8 permissions × 4
roles (`0004_seed_reference.sql:42-71`, asserted at `0005_verify_seed.sql:30`).

**Nothing queries it.** A repo-wide grep for `role_permissions` outside `.git` returns: the CLAUDE.md
/ HANDOVER.md notes; `docs/supabase-setup.md:59,103`; `docs/auth-design.md:37`; the RLS policies in
`0002_rls.sql:62,174,176`; the seed and verify migrations; `0010_realtime.sql:22`;
`0029_correct_manager_approve.sql`; `0070_external_accounts_read_nothing.sql`; a test at
`test/smoke/09-database-and-assets.mjs:84-87`; and **two UI references only** —
`src/07-a-shell.js:240` (the in-app DB schema inspector, which lists the table with the literal note
`'documentation only — nothing queries it; RLS is what enforces access'`) and
`src/07-a-shell.js:286` (the empty-state message for that inspector). No data-layer read, no `can()`
wiring. **The CLAUDE.md claim is accurate.**

It has already misled once: `0029_correct_manager_approve.sql:1` — *"role_permissions promised
something the database refuses"* — corrects a row that claimed `branch_manager` could `approve`,
which `fuel_finance_write` and `spend_finance_write` refuse (`src/06-branch-portal.js:14-19`).

### 1.5 Non-staff identities

**Funders (`stakeholders`, `0048`).**
Table at `0048_csr_stakeholders.sql:64-87`. Login is a password account whose address is
constrained to `username || '@csr.save7.org'` (`:85-86`). Accounts are minted by an Edge Function
(`supabase/functions/create-funder-account/index.ts`) that holds `service_role` and is a **public
HTTPS endpoint not behind the Google sign-in wall** (`CLAUDE.md:785-789`). A trigger
`link_stakeholder` on `auth.users` claims the row on first sign-in (`0048:156-159`).

What a funder can **read**: only the `csr_*` views, which are `security definer` on purpose — under
`security_invoker=on` a funder has no `people` row so every helper returns null and they read
nothing (`CLAUDE.md:822-829`). Views: `csr_me`, `csr_org_meta`, `csr_projects`,
`csr_project_milestones`, `csr_project_agg`, `csr_project_lines` (`0050_csr_views.sql`),
`csr_meeting_agenda` (`0056`, replaced `0062`, `0067`).
What a funder can **write**: nothing. They cannot even read their own `stakeholders` row —
`stakeholders_read_org` is `app_can_read_org()` only, deliberately, so one funder cannot enumerate
the others or read finance's internal `note` about them (`0048:179-191`).
`0070_external_accounts_read_nothing.sql` exists because before it a funder could read
`org_settings` (opening cash balance, the whole 18A issuer block), the chart of accounts, bank
account names, `role_permissions`, the branch list, **and insert receipt rows and upload arbitrary
files into the private receipt vault** (`HANDOVER.md:376`).

**Volunteers (`volunteers`, `0069`-`0082`).**
Table at `0069_volunteers.sql:93+`. Login address is constrained to
`handle || '@volunteer.save7.org'` (`:120-121`) — described at `:118-119` as *"the load-bearing one
… this is what stops a volunteer row authorising an account at any other domain, @save7.org
included."*

**Confirmed: a volunteer has no `people` row.** `volunteers` has no FK from its identity to
`people`; the only two `people` references are `vetted_by` and `invited_by`, which record *staff*
attributions (`0069:106,109`). Therefore `app_person_id()` → null → `app_is_staff()` → false. The
app says so in prose too: *"A volunteer is not staff and holds no people row, so they read nothing
in this app"* (`src/05-f-pages.js:15`).

**What `branch_manager` can actually write (0072).**
`0072_branch_manager_volunteers.sql:3-4` calls itself *"the largest permission change in this app's
history"* and `:4` states the prior state: *"`branch_manager` currently cannot write to a single
table."*

What it gains (`:11-14`): read / insert / update on `volunteers` **where `branch_id` = their own**;
**delete nothing**.
What it does not gain, enumerated at `:18-21`: *"No fuel claim, no branch spend approval, no ledger
line, no budget, no project, no receipt, no `people` row, no other branch's volunteers, and no
delete on its own."*
Three policies: `volunteers_read_branch` (`:75-77`), `volunteers_manager_insert` (`:81-83`),
`volunteers_manager_update` with both `using` and `with check` (`:87-90`). A `before insert or
update` trigger `volunteer_manager_columns()` (`:93-151`) then restricts *columns*: a manager may
not set `user_id` at all, may not reassign `invited_by`, and `vetted_by` is forced to themselves.

That is the total extent of `branch_manager` write authority in the database. It is one table.

### 1.6 Non-finance writes that do exist (the complete list)

Derived from every `with check` clause in `supabase/migrations/*.sql` that is not `app_is_finance()`:

| Write | Who | Policy |
|---|---|---|
| Lodge own fuel claim, status `pending` only | any staff, own person + own branch | `0002_rls.sql:97-102` |
| Lodge own branch spend, `pending` only | any staff, own person + own branch | `0002_rls.sql:115-120` |
| Upload a receipt | any staff (narrowed from `true` by 0070) | `0002_rls.sql:166-167`, `0070:101` |
| Co-sign an outgoing payment | `can_cosign` holders, as themselves | `0023_cosign_outgoing.sql:128` |
| Own preferences | self | `0054_person_prefs.sql:81-84` |
| Suggest a feature | self; and `app_can_read_org()` after 0066 | `0065:112`, `0066:26` |
| Tick an action point you owe | owner | `0067:183`, `0104:110` |
| Write the project tracker / set project status | `app_runs_project()` = lead or Ops Mentor | `0068:551-574` |
| Keep own branch's volunteer roster | `branch_manager` | `0072:83,90` |
| Open a task to volunteers / decide requests | `app_manages_project_branch()` | `0073:396-401` |
| Volunteer's own requests and hours | the volunteer | `0073:388`, `0074:217-229` |
| Learner's own attempts and progress | the learner | `0091:381-403`, `0096:139` |
| Move / progress a research item you lead | `app_leads_research()` | `0100:225-256` |

### 1.7 Department / executive seat / liaison — the direct answers

**A national or org-level "department" as an org unit: NO.**
The only `department` in the schema is a free-text column on `assets` (`0040_asset_register.sql:116`)
plus a lookup list `asset_lists` with `kind in ('category','department')` (`:54`), seeded with
Operations, Finance, Campaigns, Marketing, Fundraising, Administration, IT, Events, Volunteers,
Board, Other (`:77-87`). It exists solely so an asset code can start with the department's first
letter (`:174-196`). No person, project, budget, policy or permission references it.

**However, `branches` is already doing that job informally.** `branches` is not four campuses. It
holds — beyond Tygerberg / UCT / Stellenbosch / Pretoria (`0004_seed_reference.sql:14-19`) —
programme teams Red Cross, Orgamites, Dialysis support group (`0022_add_programme_teams.sql:19-22`)
and **governance units Exco, Advisory board, Management** (`0026_add_governance_teams.sql:20-23`).
Ten rows, three different kinds of thing, one table, no `kind` column.
Note the CLAUDE.md is stale here: `CLAUDE.md:399-400` still says *"Four branches"*.

**An executive/leadership seat (CEO/COO/CFO/CGO): NO, as a role. YES, as two loose artefacts.**
CEO/COO/CFO/CMO appear as `people.role` free-text job titles (`0021_team_roster.sql:42-45`) and as
comments beside the four seeded `can_cosign = true` addresses (`0023_cosign_outgoing.sql:44-48`).
`0023:3-4` describes them as *"the four exco signatories — CEO, COO, CFO, CMO"*. But `can_cosign` is
*"Set explicitly — never inferred from the role job title, which is display-only free text"*
(`0023:41-42`), and it grants exactly one thing: inserting a row into `tx_approvals`. There is no
CGO anywhere.

**A liaison / routing role: NO.**
Three occurrences of "liaison" in the whole repo, none of them a role: a fixture string
`'Ward liaison and the referral pathway'` (`test/harness.mjs:178`) and twice inside course content
JSON about a SAPS liaison (`0094_learn_content.sql:1054`, `0101_learn_content.sql:1027`). There is
no routing, assignment-to-a-role, or queue concept in the schema.

### 1.8 "Project lead"

`projects.lead_person_id text references people (id) on delete set null` —
`0047_projects.sql:58`, indexed at `:80`.

- **A lead must be a `people` row, i.e. staff.** It is a foreign key to `people.id`.
- **A volunteer cannot lead a project as the schema stands.** A volunteer has no `people` row
  (§1.5), so there is no id to put in `lead_person_id`. `app_runs_project()` additionally opens with
  `app_person_id() is not null` (`0068:486`), so even a hypothetical match would fail.
- A volunteer's connection to project work is a different column entirely:
  `project_tasks.volunteer_id uuid references volunteers (id)` (`0073:111`), i.e. a volunteer owns a
  *card*, never a project.
- The second answerable person is `project_tracker.ops_mentor`, also a `people` reference
  (`0068:476`, `:490`).
- "Lead" is not a status on a person; it is a column on a row. The same shape recurs at
  `research_items.lead_person` (`0100:98`).

---

## 2. Navigation & information architecture

### 2.1 `PAGES`

`PAGES` is a plain object literal declared in `src/05-f-pages.js:2` and **assigned into from two
other files at evaluation time**: `src/06-branch-portal.js:1061-1069` (the member pages plus
`mywork`) and `src/05-g-research.js:255-257` (`research`). The reason is load order — those files
sort after `05-f`, so naming their renderers from `05-f` would be reading forwards
(`src/05-g-research.js:6-10`, `CLAUDE.md:121-125`).

Each entry is a 6-tuple: `[eyebrow, title, subtitle, renderFn, quickAddLabel, quickAddModal]`
(destructured at `src/05-f-pages.js:51`).

**Declared in `src/05-f-pages.js:3-16` (14 admin pages):**

| key | eyebrow | title | renderer | quick add |
|---|---|---|---|---|
| `overview` | Financial overview | Where the money is | `renderOverview` | New entry → `txModal` |
| `finance` | Financial tracker | The books | `renderFinance` | Add transaction |
| `s18a` | Tax certificates | Section 18A | `renderS18A` | Add transaction |
| `fuel` | Fuel remuneration | Fuel claims | `renderFuel` | Log fuel claim |
| `branch` | Team managers | Team spend | `renderBranch` | Log spend |
| `suggestions` | Save7 OS | Feature ideas | `renderSuggestions` | Suggest a feature |
| `meetings` | The diary | Meetings | `renderMeetings` | Add meeting |
| `projects` | Portfolio | Projects | `renderProjects` | Add project |
| `claude` | Claude accounts | Seat reimbursements | `renderClaude` | Add seat |
| `assets` | Asset register | What Save7 owns | `renderAssets` | Add asset |
| `receipts` | Document store | Receipt vault | `renderReceipts` | Log spend |
| `stakeholders` | Setup | Funder access | `renderStakeholders` | Add funder |
| `volunteers` | Setup | Volunteers | `renderVolunteers` | Add volunteer |
| `settings` | Setup | Settings | `renderSettings` | New entry |

**Added by `src/06-branch-portal.js:1061-1069` (7):** `mywork` (Yours / My work),
`mhome` (My portal / My Save7), `mfuel` (My claims), `mbranch` (My team), `mprojects` (My work /
My projects), `mseat` (My account), `mreceipts` (My documents).

**Added by `src/05-g-research.js:256` (1):** `research` (Knowledge / Research).

**22 pages total.**

### 2.2 `go(p)` — `src/05-f-pages.js:38-75`

In order:
1. Returns silently if `PAGES[p]` is undefined (`:39`).
2. Calls `settingsBlockLeaving(p)` if that function exists, and aborts if it returns true — the
   unsaved-changes guard; `go()` is described as *"the single chokepoint every navigation goes
   through, including the sidebar, the mobile drawer and the quick-action tiles"* (`:40-45`).
3. **Flips the half of the app** unless the page is shared: `if (!isSharedPage(p) && isMemberPage(p)
   !== (SESSION.role === 'member')) setRole(...)` (`:47-49`). This is the load-bearing line: the page
   you open decides which nav you are looking at.
4. Sets `PAGE`, writes eyebrow / title / subtitle into `#topEyebrow` / `#topTitle` / `#topSub`,
   and repoints the single quick-add button's label and `data-open` (`:50-53`).
5. Un-hides the quick-add button, because an individual page's render may have hidden it (`:54-57`).
6. Toggles `.on` on `.page` sections by `id === 'p-' + p` — **every page is already in the DOM**;
   navigation is a CSS class flip, not a mount (`:58`).
7. Hides the 3/6/12-month period selector for `NO_PERIOD = ['s18a','assets','mywork','meetings',
   'stakeholders','suggestions','volunteers','research']` and for any member (`:64-66`).
8. Repaints nav `.on` state, `updatePills()`, then the renderer, then `paintSortSegs()`, then
   scrolls to top (`:67-74`).

### 2.3 `applyRoleChrome()` — `src/05-f-pages.js:76-101`

Reads one boolean, `SESSION.role === 'member'`, and then:
- shows `#navAdmin` / hides `#navMember`, or the reverse (`:78-79`)
- calls `applyVolunteerChrome()` (`:80`)
- hides the period selector for members (`:81`)
- **hides the Database inspector button** (`[data-open="dbPanel"]`) for members (`:82`)
- resolves the person for the sidebar footer, with three fallbacks because `DB.people` can come back
  empty on live data (`:83-90`)
- writes initials, `First L.`, a role line and an environment line (`:91-98`). The role line is
  hardcoded: a member sees `"<job title> · <branch>"`, an admin always sees the literal string
  **`'Finance admin'`** and `'Admin & exco view'` (`:97-98`) — so an `exco` user is labelled
  "Finance admin" in the sidebar.
- finally: `$$('#navMember a[data-perm]').forEach(a => { a.style.display = can(a.dataset.perm) ? ''
  : 'none'; })` (`:100`) — the only place `PERM_SPEC` touches navigation, and since `can()` is
  role-blind (§1.4) it currently never hides anything.

`setRole(role, target)` (`:102-106`) sets `SESSION.role`, calls `applyRoleChrome()`, and if no target
page was given navigates to `mhome` (member) or `overview` (admin).

### 2.4 `SHARED_PAGES` and the admin/member split

```js
const MEMBER_PAGES = ['mhome','mfuel','mbranch','mprojects','mseat','mreceipts'];
const SHARED_PAGES = ['mywork','volunteers','research'];
```
`src/05-f-pages.js:18,33`.

The split is **two `<nav>` elements, one visible at a time**, and the rule stated in the file is
*"a page tells you which half of the app you are in"* (`:22-23`). `SHARED_PAGES` is the list of
exceptions — pages that must not flip the shell.

The reasoning is recorded inline, and each entry is a bug that was found:
- `mywork` (`:19-24`): the same page for a member and for finance; *"Opening it as a member used to
  switch the whole shell to admin chrome."*
- `volunteers` (`:25-28`): *"A branch_manager is a member-portal role and the roster is theirs to
  keep."*
- `research` (`:29-32`): reading the board is `app_is_staff()` and a question's lead is usually a
  branch member, so a page they can write but cannot reach would make the grant unusable.

**Which half you land in is decided by `app_role` at sign-in, not by a nav choice:**
`const ADMIN_APP_ROLES = ['finance_admin', 'exco']` (`src/08-auth.js:16-17`), and
`authGranted()` sets `SESSION.role = admin ? 'admin' : 'member'` (`src/08-auth.js:108-110`).
So `branch_member` and `branch_manager` get the member half; `exco` and `finance_admin` get the
admin half. There is no third shell.

### 2.5 The full sidebar, as rendered

Markup: `src/02-markup.html:51-176`. Section headings are `div.nav-lbl`.

**`#navAdmin` — seen by `finance_admin` and `exco` (`:51-126`):**

| Heading | Items |
|---|---|
| Dashboards | My work `:53`, Projects `:57`, Research `:63`, Financial overview `:67` |
| Money | Financial tracker `:72`, Section 18A `:76`, Fuel claims `:80`, Team spend `:84`, Claude accounts `:88` |
| Records | Asset register `:93`, Receipt vault `:97`, Meetings `:101` |
| Setup | Funder access `:106`, Volunteers `:114`, Feature ideas `:118`, Settings `:122` |

16 destinations. 5 under Money.
Note `research` sits under **Dashboards**, deliberately: *"research is work Save7 is doing, not a
thing to configure"* (`:61-62`). Projects likewise is under Dashboards rather than Money —
*"it is a tracker that happens to know what things cost, not a second set of books"*
(`CLAUDE.md:186-190`).

**`#navMember` — seen by `branch_member` and `branch_manager` (`:129-176`), `display:none` by default:**

| Heading | Items |
|---|---|
| My portal | Home `:131`, My work `:135`, Volunteers `:142`, Research `:150` |
| Submit | My fuel claims `:155` (`data-perm="log_fuel"`), My team spend `:159` (`view_branch_spend`) |
| Records | My projects `:164` (`view_own_projects`), My Claude seat `:168` (`view_own_seat`), My receipts `:172` (`view_receipts`) |

9 destinations. Only the five in Submit/Records carry `data-perm`; the four in My portal have none
(`:139-141`, `:146-149` explain why: what they offer is decided by `app_role` and by RLS, not by a
flag).

**There is no per-role nav beyond these two lists.** `exco` and `finance_admin` see an identical
sidebar; `branch_member` and `branch_manager` see an identical sidebar. The only role-conditional
item in either is Volunteers.

Pill counters (`updatePills()`, `src/05-f-pages.js:108-197`) are computed client-side from what RLS
already handed over, and several are deliberately "problems" counts rather than totals so they can
return to zero (`:117-127`, `:144-147`, `:158-162`).

### 2.6 Three responsive sidebar modes

`CLAUDE.md:551-561` and `src/07-j-drawer.js:94-131`:

1. **Drawer**, ≤ 860px — the sidebar is an overlay, takes no width when shut; gesture-driven with a
   critically-damped spring and momentum projection (`src/07-j-drawer.js:2-18`, `:196+`).
2. **Icon rail**, 861–1180px — 70px glyph rail.
3. **Full sidebar**, ≥ 1181px.

Plus a **fourth thing that is not a mode**: an explicit user override stored in
`person_prefs.options.nav` as `'rail' | 'wide' | absent` (`src/07-j-drawer.js:100-121`). Absent means
"let the stylesheet decide". `navIsRail()` (`:128-131`) resolves the three-way. Two load-bearing
details at `CLAUDE.md:556-561`: the `.app.rail` rules must live inside `@media (min-width:861px)` or
a stored collapse follows the reader onto their phone; and `options` must be written **merged**
because the My work sheet writes the same column.

### 2.7 `DEFAULT_LANDING`

```js
const DEFAULT_LANDING = 'mywork';
```
`src/07-m-prefs.js:34`. The reasoning is at `:29-33`: *"My work for everybody, whatever their role.
The first question anybody has on opening this is what is on them — not what the organisation's cash
position is, and not, for a member, a greeting. It is the one page that answers that for both halves
of the app."*

`landingPage()` (`:35-39`) prefers `person_prefs.defaultPage`, but falls back to `DEFAULT_LANDING`
if the stored page no longer exists or the person's role can no longer reach it.
`landingChoices()` (`:15-25`) derives the menu from `PAGES` + the role rather than listing it, and
for a member filters out any page whose nav link is currently `display:none`.

### 2.8 Hidden vs disabled

**Navigation items are hidden, not disabled** — three separate mechanisms, all `style.display`:
- the wrong half's whole `<nav>`: `src/05-f-pages.js:78-79`
- member items whose `data-perm` is off: `src/05-f-pages.js:100`
- Volunteers, for roles with no business there: `applyVolunteerChrome()`,
  `src/05-d-projects.js:1053-1059`, shown only for `canKeepVolunteers()` (finance_admin or
  branch_manager, `:910-911`) or `exco`. The reasoning at `:1049-1052`: *"Not left visible and
  empty: RLS would make it render as nothing, and a page that is always blank teaches people the app
  is broken rather than that it is not theirs."*
- the Database inspector button: `src/05-f-pages.js:82`.

**Inside a page, content is disabled/explained rather than hidden** — `gate()` replaces a section
body with a padlock panel and the text *"`<label>` is switched off for this role"*
(`src/06-branch-portal.js:83-92`).

---


## 3. Feature inventory

Source: `docs/map.md:16-45`. The table below reproduces every row of the feature table. Column
headers in the source are `Feature | Schema | Data | Render | Handlers` (`docs/map.md:16`).

| Feature | Migration(s) | Data layer | Renderer | Handler file | Line |
|---|---|---|---|---|---|
| The ledger | `0001`, `0011`–`0014` | `03-data.js` `DB.transactions` | `05-a-money.js` `renderFinance` | `07-c-forms.js` `#txForm` | `:18` |
| Fuel claims | `0001`, **`0105`** | `DB.fuel_claims`, `fuelRate()`, `fuelAmount()`, `fuelClaimant()`, `isSlipClaim()` | `05-a-money.js` `renderFuel` | `07-c-forms.js` `#fuelForm` | `:19` |
| Team spend | `0001` | `DB.branch_spend` | `05-a-money.js` `renderBranch` | `07-c-forms.js` `#spendForm` | `:20` |
| Claude seats | `0045`, `0059`, `0064` | `DB.claude_seats` | `05-a-money.js` `renderClaude` | `07-c-forms.js` `openSeat`, `#seatForm` | `:21` |
| Section 18A | `0031`–`0039` | `s18a*()` in `03-data.js` | `05-b-assets.js` `renderS18A` | `07-c-forms.js` `openS18a` | `:22` |
| Asset register | `0026`, `0052`, `0058`, **`0106`** | `DB.assets`, `assetGoneOn()`, `assetsGoneInPeriod()` | `05-b-assets.js` `renderAssets` | `07-d-assets.js`, `openAssetGone()` | `:23` |
| Receipt vault | `0001` | `DB.receipts` | `05-b-assets.js` `renderReceipts` | `07-a-shell.js` `viewReceipt` | `:24` |
| Budgets with periods | `0053` | `DB.budgets`, `budgetInWindow()` | `05-c-settings.js` `renderSettings` | `07-l-settings.js` `openBudget` | `:25` |
| Projects | `0047`, `0049` | `project*()` in `03-data.js` | `05-d-projects.js` `renderProjects` | `07-e-projects.js` | `:26` |
| The task board | `0060` | `projectBoard()`, `unifiedBoard()` | `05-d-projects.js` `renderBoard` | `07-g-board.js` | `:27` |
| **The archive** | `0102` (`is_done`, `finished_at`, `board_archive_days`) | `taskArchived()`, `archivedTasks()` — derived, never stored | `05-d-projects.js` `renderArchive` | `07-g-board.js` `reopenTask`, `data-bc-done` | `:28` |
| **The project tracker** | `0068` | `tracker*()`, `canEditTracker()` | `05-e-tracker.js` `renderTracker` | `07-f-tracker.js` | `:29` |
| Meetings | `0056`, `0062`, `0067` | `DB.meetings`, `agendaPoints()` | `05-d-projects.js` `renderMeetings` | `07-h-meetings.js`, `07-i-suggestions.js` | `:30` |
| Feature ideas | `0065`, `0066` | `suggestions*()` | `05-d-projects.js` `renderSuggestions` | `07-i-suggestions.js` | `:31` |
| Funder access | `0048` | `DB.stakeholders` | `05-d-projects.js` `renderStakeholders` | `07-h-meetings.js` | `:32` |
| My work | `0054`, `0055` | `myPrefs()` in `06-branch-portal.js` | `06-branch-portal.js` `MW_CARDS` | `07-m-prefs.js` | `:33` |
| **Their own Google calendar** | none — stores nothing | `CAL` in `07-q-calendar.js` (memory only) | `07-q-calendar.js` `calendarCardHtml` | `07-q-calendar.js` `calendarConnect` | `:34` |
| **Volunteers** | `0069`–`0072` | `DB.volunteers`, `myVolunteers()` | `05-d-projects.js` `renderVolunteers` | `07-o-volunteers.js` | `:35` |
| Opportunities and requests | `0073` | `DB.volunteer_requests`, `reqWaiting()` | `05-d-projects.js` (roster §01, `boardCard`) | `07-o-volunteers.js`, `07-g-board.js` | `:36` |
| Volunteer hours | `0074` | `DB.volunteer_hours`, `volHoursVerified()` | `05-d-projects.js` (roster §02) | `07-o-volunteers.js` | `:37` |
| The volunteer app | `0074` (`vol_*` views) | `src-vol/03-app.js` `loadVol` | `src-vol/03-app.js` | `src-vol/03-app.js`, `04-auth.js` | `:38` |
| **The vetting gate** | `0099` (`volunteer_course.gate_levels`) | `courseGateLevels()`, `volCourseLevels()`, `volCoursePassed()` | `05-d-projects.js` `renderVolunteers` §03 | `07-o-volunteers.js` `#volForm` | `:39` |
| The course, in the OS | `0091`–`0099` (read-only) | `DB.learners`, `DB.learn_levels`, `DB.learn_*_progress` | `05-d-projects.js` §03 | — (written by the course, never from here) | `:40` |
| **The research board** | `0100` | `research*()` in `03-data.js`, `canEditResearch()` | `05-g-research.js` `renderResearch` | `07-p-research.js`, drag in `07-g-board.js` | `:41` |
| Research documents | `0100` (`save7-research` bucket) | `DB.research_files`, `researchDocUrls()` | `07-p-research.js` `renderResearchDocs` | `07-p-research.js`, `uploadResearchDoc()` | `:42` |
| The mobile drawer | — | — | `01-styles.html` | `07-j-drawer.js` | `:43` |
| Collapsing the sidebar | `0054`/`0055` (`person_prefs.options.nav`) | `prefOpts()`, `navRailPref()` | `01-styles.html` `.app.rail` | `07-j-drawer.js` `applyNavRail()` | `:44` |
| Deleting anything | — | — | — | `07-k-deleting.js` `deletePlan()` | `:45` |

**28 feature rows.**

### 3.1 Grouped by domain — how much is NOT finance

| Domain | Features | Count | % |
|---|---|---|---|
| money / finance | ledger, fuel claims, team spend, Claude seats, Section 18A, budgets | 6 | 21.4% |
| volunteers | roster, opportunities/requests, hours, the volunteer app, vetting gate | 5 | 17.9% |
| projects | projects, task board, archive, project tracker | 4 | 14.3% |
| assets / documents | asset register, receipt vault | 2 | 7.1% |
| research | research board, research documents | 2 | 7.1% |
| settings / personalisation | My work, collapsing the sidebar | 2 | 7.1% |
| meetings | meetings | 1 | 3.6% |
| learning | the course, in the OS | 1 | 3.6% |
| other | feature ideas, funder access, Google calendar, mobile drawer, deleting | 5 | 17.9% |

**Finance is 6 of 28 features (21%).** Folding the receipt vault and asset register in as financial
records takes it to 8 of 28 (29%). Volunteers + learning alone is 6 of 28 — equal to or larger than
finance.

### 3.2 The same question, by schema

103 migration files (`0001`–`0106`, skipping `0018`–`0020` — the removed CSR work,
`docs/map.md:96`).

| Domain | Migrations | Count |
|---|---|---|
| finance / money | `0001 0007 0008 0011 0012 0013 0014 0023 0024 0025 0028 0029 0030 0031 0032 0036 0037 0038 0039 0045 0046 0053 0059 0064 0105` | 25 |
| volunteers | `0069 0071 0072 0073 0074 0075 0076 0077 0078 0079 0080 0082 0083 0090 0099 0103` | 16 |
| learning / the course | `0081 0084 0085 0086 0087 0088 0089 0091 0092 0093 0094 0095 0096 0097 0098 0101` | 16 |
| people / roster / admin accounts | `0006 0015 0016 0017 0021 0022 0026 0027 0035 0041` | 10 |
| platform / RLS / auth / realtime | `0002 0003 0004 0005 0009 0010 0033 0034 0057 0070` | 10 |
| assets | `0040 0042 0043 0044 0052 0058 0106` | 7 |
| meetings | `0056 0061 0062 0063 0067 0104` | 6 |
| projects | `0047 0060 0068 0102` | 4 |
| funders / CSR | `0048 0049 0050 0051` | 4 |
| settings / prefs | `0054 0055` | 2 |
| feature suggestions | `0065 0066` | 2 |
| research | `0100` | 1 |

**Volunteers + learning = 32 migrations, more than finance's 25.**

### 3.3 The same question, by table

68 live tables (71 `create table` statements; `claude_seat_reminders` dropped by `0032`, the first
`meeting_agenda_points` dropped by `0063` and re-created by `0067`, `learn_lessons` dropped and
re-created inside `0094`).

| Domain | Count | Tables |
|---|---|---|
| learning | 17 | `learners`, `learn_levels`, `learn_modules`, `learn_lessons`, `learn_resources`, `learn_questions`, `learn_choices`, `learn_attempts`, `learn_answers`, `learn_module_progress`, `learn_level_progress`, `learn_course_progress`, `learn_certificates`, `learn_review_items`, `learner_registrations`, `learn_course`, `learn_events` |
| projects | 16 | `projects`, `project_people`, `project_milestones`, `project_board_columns`, `project_tasks`, `project_tracker`, `project_log`, `project_reach`, `project_deliverables`, `project_budget_lines`, `project_risks`, `project_checklist`, `project_stakeholders`, `project_support`, `project_kpis`, `project_checkins` |
| finance / money | 12 | `transactions`, `fuel_claims`, `branch_spend`, `claude_seats`, `claude_repayments`, `receipts`, `tx_approvals`, `donors`, `budgets`, `org_settings`, `categories`, `accounts` |
| volunteers | 6 | `volunteers`, `volunteer_requests`, `volunteer_hours`, `volunteer_registrations`, `volunteer_course`, `volunteer_quiz_attempts` |
| meetings | 4 | `meetings`, `meeting_agenda_points`, `meeting_action_points`, `meeting_action_owners` |
| research | 4 | `research_columns`, `research_items`, `research_log`, `research_files` |
| assets | 3 | `asset_lists`, `assets`, `asset_code_counters` |
| org core | 3 | `branches`, `people`, `role_permissions` |
| funders / CSR | 1 | `stakeholders` |
| settings | 1 | `person_prefs` |
| suggestions | 1 | `feature_suggestions` |

**Finance is 12 of 68 tables (17.6%).** Learning + volunteers is 23 (33.8%); projects is 16 (23.5%).

Views, for completeness: `csr_*` (7, from `0050`/`0053`/`0056`/`0062`/`0067`), `vol_*`
(`vol_me`, `vol_projects`, `vol_opportunities`, `vol_my_tasks`, `vol_my_hours` from `0074`, replaced
and extended with `vol_course` by `0076`, `vol_my_quiz` by `0088`), and `learn_questions_pub`,
`learn_options_pub`, `learn_my_attempts`, `learn_my_progress`, `vol_my_course_learning` from `0092`.

### 3.4 What `docs/map.md` does not cover

Files on disk with no row in the map: `src/02-markup.html` (2241 lines — the entire DOM),
`src/04-charts.js` (the SVG chart engine), `src/07-b-transfer.js` (CSV import/export),
`src/07-n-unsaved.js`, `src/08-auth.js` (the OS auth gate), `src-vol/01-styles.html`,
`src-vol/02-markup.html`, and **`src-vol/03-learn.js`** — the front end of the whole learning domain.
No path `docs/map.md` names is missing from disk (33 distinct names checked).

Three stale migration citations in the map itself: `:22` (Section 18A cited as `0031`–`0039` when
`0031`–`0035` are seat reminders, auth relinking and the admin account; the real range is
`0036`–`0039`); `:23` (asset register cited as `0026`, which is governance teams, and omitting
`0040`/`0042`/`0043`/`0044`); `:39` and `:58` (the vetting gate described as `0099`'s rule, which
`0103_vetting_needs_the_quizzes_or_prior_learning` has superseded).

---


## 4. The volunteer portal

### 4.1 What it is

`docs/volunteers.md:33-35` states the product in one sentence: *"**It is an assignment-and-reporting
loop.** A branch manager posts work; a volunteer signs in, sees it, does it, ticks it off and logs
the hours; the branch manager verifies."*

Status claim: *"built. Design agreed 15 August 2026, last updated 17 August. Migrations `0069`–`0086`
are applied to production, the OS half is live, and the volunteer app is deployed."*
(`docs/volunteers.md:3-6`).

Explicitly out of scope (`docs/volunteers.md:37-47`): a matching engine / "Next Best Opportunity",
skills / interests / availability, a forum, a community tab, a newsletter, calendar and Slack
integration, achievement milestones.

Two post-design departures called out at the top: volunteers self-register with their own Google
accounts (`0075`, `docs/volunteers.md:11-15`); and the Google Classroom sync was built and then
removed (`0081`/`0084`/`0085`, out in `0086`, `docs/volunteers.md:16-18`).

### 4.2 What a volunteer sees

Shell (`src-vol/02-markup.html`): a full-screen auth overlay (`:3-8`) with a Google Identity
Services button, a fallback OAuth button and a "New to Save7? Join first" link (`:16-33`); a Join
panel with name, Google address, campus `<select>` and "Join Save7" (`:40-61`); then a header bar
(`:73-79`), a greeting region (`:82`), **four nav tabs** (`:84-92`) and four panels (`:94-97`).

The four tabs: **My work**, **Opportunities**, **My hours**, **Learn**.

**Tab visibility is decided by `vetted`.** Opportunities and My hours are `display:none` for an
unapproved volunteer (`src-vol/03-app.js:478-481`); an unvetted volunteer who lands on either is
redirected to Learn (`:452`); before approval the app renders a course gate into My work and lands on
Learn (`:483-494`).

A signed-in account on no roster gets a warning; if the address ends `@save7.org` it says you are
staff and points at `os.save7.org` (`src-vol/03-app.js:236-252`), and the nav is hidden entirely
(`:461-465`).

A **gate strip** sits under the greeting on every tab except Learn: "N of M required levels
finished", a "Carry on" button, the sentence *"No work can be given to you until the first N levels
of the course are finished"*, and a progress bar (`src-vol/03-app.js:204-231`).

| Tab | Contents |
|---|---|
| **My work** (approved) | one card per assigned task — project · stage, title, detail, badges for `time_band` / `Due` / `Done`; and a per-card **log-hours form** (date capped at today, hours step 0.25, optional note). `src-vol/03-app.js:270-297`. Nothing here ticks a task done (`:264-266`) |
| **My work** (unapproved) | the course gate: status badges, a "Continue — <module>" button, one card per gate level, and a link to `learn.save7.org`. `src-vol/03-app.js:400-446` |
| **Opportunities** | a time-band filter built from the bands actually present (`:307-313`), one card per open task, and either an `I'll do this` button (vetted), an "Asked — waiting on a manager" badge, or the line *"Applying opens up once you are vetted."* (`:328-333`). A standing note explains that asking is not being given it, that cross-branch work needs both managers, and that an unanswered request lapses in seven days (`:335-343`) |
| **My hours** | two figures side by side that are **never summed** — Verified ("Checked by a branch manager") and Waiting ("Not counted until somebody verifies them") (`:350-357`) — then one card per claim with a status badge and, when queried, the manager's question (`:360-376`). No edit or delete control is rendered |
| **Learn** | three screens in `src-vol/03-learn.js`: a levels list (`:303-373`), one module rendered as a single page of lessons (`:375-416`), and inline self-checks (`:184-230`) that say *"They are not a test: nothing is scored and nothing waits on them"* |

**Every write a volunteer can make, exhaustively:**

| Action | Call | Cite |
|---|---|---|
| Register (unauthenticated) | POST `/functions/v1/register-volunteer` | `src-vol/04-auth.js:80-85` |
| List campuses for the Join form | same endpoint, `{list_branches:true}` | `src-vol/04-auth.js:50-55` |
| Sign in (ID-token) | `sb.auth.signInWithIdToken` | `src-vol/04-auth.js:133-135` |
| Sign in (redirect fallback) | `sb.auth.signInWithOAuth` | `src-vol/04-auth.js:222-228` |
| Sign out / hard reset | `sb.auth.signOut` | `src-vol/04-auth.js:235-251` |
| Apply for an opportunity | `insert volunteer_requests` (task + self, nothing else) | `src-vol/03-app.js:526` |
| Log hours | `insert volunteer_hours` | `src-vol/03-app.js:559-566` |
| Enrol on the course | `rpc('learn_claim_me')` | `src-vol/03-learn.js:245`, `:469` |
| Mark a module read | `rpc('learn_complete_module')` | `src-vol/03-learn.js:479` |
| Answer an inline check | `rpc('learn_grade_check')` | `src-vol/03-learn.js:253` |

Everything it reads, in one `Promise.all` (`src-vol/03-app.js:82-99`): `vol_me`, `vol_course`,
`vol_my_tasks`, `vol_opportunities`, `vol_my_hours`, `vol_projects`, `vol_my_quiz`, `learn_levels`,
`learn_modules`, `learn_lessons`, `learn_level_progress`, `learn_module_progress`,
`vol_my_course_learning`; plus `learn_questions_pub` / `learn_options_pub` lazily per module
(`src-vol/03-learn.js:152-161`).

### 4.3 The `src-vol/` boundary and how `build-vol.mjs` enforces it

Why it exists (`build-vol.mjs:9-15`): *"a volunteer must never download a file containing the
ledger, the asset register and the section 18A machinery. Row level security would return them
nothing, but the code would still be sitting in the HTML on their phone."*

It concatenates every `src-vol/` file matching `/^\d\d-/`, sorted (`build-vol.mjs:48`, `:131-132`) —
today `01-styles.html`, `02-markup.html`, `03-app.js`, `03-learn.js`, `04-auth.js` — and writes
`dist-vol/Save7-Volunteers.html` plus an identical `dist-vol/index.html` (`:46`, `:213-217`). The
font and mark are inlined as base64; supabase-js UMD is inlined from `node_modules`; the client is
created with `storageKey: "save7-volunteers-auth"` (`:94-102`). The **only** non-inlined dependency
in either app is Google Identity Services from `accounts.google.com` (`:20-35`).

**Five build-time guards, all throwing.** The load-bearing one is the forbidden-table list
(`build-vol.mjs:116-128`) — `transactions`, `budgets`, `receipts`, `claude_seats`,
`claude_repayments`, `fuel_claims`, `branch_spend`, `assets`, `donors`, `stakeholders`,
`org_settings`, `people`, `project_tracker`, `project_milestones`, `project_budget_lines`,
`project_risks`, `project_kpis`, `project_checkins`, `meetings`, `volunteers`, …, `learn_choices`,
`learners` — checked at `build-vol.mjs:203-209`:

```js
for (const t of FORBIDDEN) {
  const m = html.match(new RegExp(`from\\(['"]${t}['"]\\)`));
  if (m) throw new Error(
    `the volunteer build queries \`${t}\`, which is not part of a volunteer's surface — ` +
    'their reads are the vol_* views, and their writes are volunteer_requests and volunteer_hours'
  );
}
```

**Precisely what that is:** a regex for the literal string `from('<table>')` over the concatenated
HTML, run before the Supabase runtime is inlined. It is a **deny-list, not an allow-list** — a query
against a table not on the list passes. There is no module system, so there is nothing to stop
`src-vol/` importing `src/` other than the fact that imports do not exist here; the boundary is
"these two directories are concatenated separately".

The other four guards: unreplaced `__TOKEN__` / missing `__SUPABASE_RUNTIME__` (`:139-141`); a
`service_role`-looking key (`:84`); a backtick inside an HTML comment in a `.js` source
(`:192-199`); and an orphan-id check requiring every `id="…"` in `02-markup.html` to have a matching
`#id` selector (`:163-190`).

**No test covers the volunteer build's output.** `npm test` runs `build && build:vol && node
test/smoke.mjs` (`package.json:12`), so a *build failure* fails the suite, but grepping `test/` for
`dist-vol`, `Save7-Volunteers` or `build-vol` returns nothing; `test/smoke/16-volunteers.mjs`
exercises the OS-side Volunteers page.

### 4.4 Opportunities, requests, hours — the exact definitions

**An opportunity is not a table.** `0073_volunteer_opportunities.sql:6-18`: *"`project_tasks.person_id`
is already nullable… posting an opportunity is ticking a flag on a card, not creating a parallel
object."* Three columns on `project_tasks` (`0073:113-117`): `volunteer_id uuid references
volunteers(id) on delete set null`, `open_to_volunteers boolean not null default false`, and
`time_band` (enum `'15 minutes' | '2 hours' | 'Ongoing'`, `0073:108`). Plus `check (person_id is null
or volunteer_id is null)` — a card has at most one owner (`0073:118-120`). A task **is** an
opportunity iff `open_to_volunteers` and both owner columns are null (`vol_opportunities`,
`0074:277-281`). There is no status machine on an opportunity.

**`volunteer_requests`** (`0073:136-177`): `task_id`, `volunteer_id`, `requested_at`,
`home_ok/home_by/home_at`, `host_ok/host_by/host_at`, `lapses_on date not null default (current_date
+ 7)`, `status`, `note`.
Status is the enum `('pending','approved','declined')` (`0073:135`) and is **never typed by a
client** — a `before` trigger derives it from the two flags (`0073:291-298`), and the constraint
`volunteer_requests_status_agrees` makes the same rule structural (`:169-177`).
Home = the volunteer's own branch manager or finance; host = the manager of the project's branch or
finance, and finance alone for an org-level project whose `branch_id` is null (`0073:243-259`). When
one manager holds both sides, the trigger mirrors one flag onto the other (`:261-270`).
**Lapsing is derived, not stored** (`0073:46-60`): a request is lapsed when still `pending` with
`lapses_on < current_date`, and the trigger refuses a late approval — *"this request lapsed on % and
can no longer be approved — the volunteer must ask again"* (`:235-242`).
Approval is what assigns: `volunteer_request_assign()` writes `project_tasks.volunteer_id` on the
transition into `approved` and refuses if a staff member already holds the card (`0073:313-337`).

**`volunteer_hours`** (`0074:71-100`): `volunteer_id`, `task_id`, `project_id (not null)`, `on_date`,
`hours numeric(5,2) check (hours > 0)`, `note`, `status`, `decided_by`, `decided_at`,
`decision_note`. Constraints: `(status='pending') = (decided_at is null)` (`:90-93`) and
`on_date <= current_date` (`:95-97`).
Status enum `('pending','verified','queried')` (`0074:69`). A trigger stamps `decided_at`/`decided_by`
on leaving `pending` and clears them on return (`:150-160`); a volunteer may not edit a decided claim
(`:163-168`) nor change the status at all (`:170-173`).
Rules stated in the header (`0074:20-41`): the total is derived, never stored; **pending is never
added to verified**; nothing here posts to the ledger; more than twelve hours in a day is *"flagged,
not refused"* (`:31-35`).

Five read views for the portal, all `security_invoker` **off** on purpose because a volunteer has no
`people` row (`0074:44-52`): `vol_me`, `vol_projects`, `vol_opportunities`, `vol_my_tasks`,
`vol_my_hours` (`0074:237-313`), rewritten by `0076` to require `vetted`, plus `vol_course`
(`0076:93-99`), `vol_my_quiz` (`0088:111-122`) and `vol_my_course_learning` (`0092:105-118`).
`verify_vol_views()` asserts each is guarded by `app_is_volunteer()`, scoped to `app_volunteer_id()`,
carries no `security_invoker=on`, names no forbidden table or column, and requires `vetted` unless
allowlisted (`0074:326+`, rewritten `0092:351+`).

### 4.5 How a volunteer is authenticated

Account admission is `auth_enforce_save7_domain()`, a security-definer trigger on `auth.users`, with
**four** accept branches after `0095` (`0095:76-120`): `@save7.org`; an active
`stakeholders.login_email`; an active `volunteers.login_email`; an active `learners.email`. Anything
else raises `42501` (`:115-119`). `0095:216-231` asserts the count is exactly four.

`auth_link_volunteer()` fires `after insert on auth.users` and claims the row by matching
`login_email` (`0069:212-228`); `volunteer_link_auth()` handles the case where the auth account
already exists (`0082:48-83`); `0083` backfills rows that predate it, after a real incident
(`0083:9-22`).

The login address was originally derived — `check (login_email = handle ||
'@volunteer.save7.org')` (`0069:120-121`), corrected to the plural domain by `0071:37-45`.
**`0075` dropped the derived constraint** (`:67`), made `handle` nullable (`:69`), and replaced it
with `check (login_email !~* '@save7\.org$')` (`:75-79`) plus a two-way trigger forbidding any
overlap between `volunteers.login_email` and `people.email` (`0075:90-124`). The mirror on `people`
is `0069:160-162` / `0071:59-64`.

Registration goes through `supabase/functions/register-volunteer/index.ts`, *"THE ONLY
UNAUTHENTICATED ENDPOINT IN THE PROJECT"* (`:6-29`). It holds `service_role`, serves the campus list
from `branches where volunteer_joinable` (`:130-136`), throttles on a salted IP hash counting
refusals and successes separately, logs every attempt to `volunteer_registrations`, and inserts one
row with `self_registered: true, active: true, vetted: false, course_done: false` (`:240-249`). An
already-registered address gets the same answer as a fresh one, deliberately (`:231-238`).

Client-side, the preferred path is Google Identity Services + `signInWithIdToken` with a
SHA-256/raw nonce pair and the portal's **own** Google client ID
(`src-vol/04-auth.js:123-129`, `:151-192`); the fallback is `signInWithOAuth` with
`prompt: 'select_account'` and deliberately no `hd` (`:206-233`). The file's own header: *"This gate
is a convenience, not a control… the policies are the boundary"* (`:14-17`).

**A volunteer has no `people` row — verified three ways.** The `volunteers` table has no identity FK
to `people`; its only `people` references are `vetted_by` (`0069:106`), `invited_by` (`0069:109`) and
`course_done_by` (`0103:38-40`), all describing staff who acted. `0069:138-139` says it: *"Volunteers.
Not staff: no `people` row, so every helper in `0002_rls.sql` returns null for them and every base
table denies them by construction."* And the two namespaces are held disjoint by a check constraint
(`0069:160-162` / `0071:62-64`) and a two-way trigger (`0075:116-124`).

### 4.6 Deployment state

| Claim | Source |
|---|---|
| "Still open, and blocking nobody: the DNS record for `volunteers.save7.org`, which is why the app currently answers at `save7-volunteers.pages.dev`." | `docs/volunteers.md:20-21` |
| "~~Nobody has pointed a Cloudflare Pages project at `volunteers.save7.org`.~~ **Done.** … **Only the DNS is outstanding:** a `CNAME` for `volunteers` at **xneelo/konsoleH**, not Cloudflare." | `HANDOVER.md:414-418` |
| "The build boundary was checked **against the deployed artifact**, not just the source." | `docs/session-handover.md:75-80` |
| "`save7.org` is not on Cloudflare DNS. Its nameservers are xneelo's, so adding the custom domain in Pages does *not* create the record — it will sit on *Verifying*." | `docs/volunteer-portal-runbook.md:176-184` |
| Phase 1 (the OAuth split) "done 17 August 2026" | `docs/volunteer-portal-runbook.md:40-46` |

`docs/volunteers.md:394-397` is **stale** and contradicts the above — it still lists the Pages
project as not created.

Repo evidence: `scripts/setup-volunteers-deploy.sh` (472 lines) is a manual wizard —
*"Everything here is a dashboard action — Cloudflare has no way to create a Pages project from a repo
without the browser flow"* (`:191-193`) — and records no completion state. There is **no deploy
script, no `wrangler.toml`, no CI config** for Pages anywhere in the tree. `README.md` does not
mention volunteers at all. Both Edge Functions hard-code *both* hostnames as allowed origins
(`register-volunteer/index.ts:49-54`, `submit-quiz/index.ts:40-41`).

**Answer: `volunteers.save7.org` is not confirmed live.** The docs agree a Pages project exists and
serves at `save7-volunteers.pages.dev`, and that the xneelo CNAME was outstanding as of the last
handover (17 Aug 2026; HEAD is 14 Sep 2026). Whether it resolves today is **unverified** — no network
request was made.

### 4.7 A gap worth recording

`0103_vetting_needs_the_quizzes_or_prior_learning.sql` now requires, for vetting, the gate levels
**and** two passed quizzes, unless a named prior-learning waiver (`course_done AND
course_done_source = 'manual'`) applies (`0103:100-106`, `:171-175`). It adds
`volunteer_mark_gate(p_gate, p_answers)` as the new marker (`0103:164+`).

**No file under `src-vol/` calls `volunteer_mark_gate`, and there is no gate-quiz screen in the
portal.** `submit-quiz/index.ts:9-10` still names `src-vol/03-quiz.js` as the source of the
questions; that file does not exist. So as the tree stands a volunteer cannot sit either gate quiz,
and the prior-learning waiver is the only route through vetting. The HEAD-adjacent commit `ad25d8b`
records this as outstanding: *"Still to build: the quiz screens in the volunteer portal."*
`docs/volunteers.md:263-270` still describes the pre-`0103` rule.

Dead code in the portal, observed: `bothQuizzesPassed()` references `QUIZ_KEYS`, defined nowhere in
`src-vol/` (`src-vol/03-app.js:178`); `bestAttempt` (`:174`) and `quizPassMark` (`:179`) are defined
and never called; `VDB.quiz` is loaded and never rendered.

---

## 5. The learning / course tables

### 5.1 Confirmed: the course moved onto this project

`0091_learn_course_bank.sql:1-13`: *"Transplant Alchemy moves its backend here, and the two quizzes
join one bank … `learn.save7.org` is the Transplant Alchemy 101 course. It was built on its own
SQLite/D1 database with its own password login and its own copy of the question bank; the volunteer
portal had a second, hard-coded copy of forty of them… This migration makes **this** project the
course's backend: the structure, the bank, the attempts and the certificates."*

`learn_*` tables exist (16 of them, plus `learners` and `learner_registrations`).
`vol_my_course_learning` exists — `0092_learn_views_and_marking.sql:105-118`, read by the portal at
`src-vol/03-app.js:98`.

"Transplant"/"Alchemy" appears in `CLAUDE.md`, `HANDOVER.md`, `test/harness.mjs`,
`src-vol/03-learn.js`, `src-vol/01-styles.html`, `src-vol/03-app.js`, `docs/build-notes.md`,
`docs/session-handover.md`, `docs/volunteers.md`, `docs/volunteer-quiz-key.md`, `scripts/gen-quiz.py`,
`supabase/functions/register-learner/index.ts`, `supabase/functions/submit-quiz/key.ts`, and
migrations `0013`, `0091`, `0092`, `0094`, `0095`, `0096`, `0099`, `0101`. "learn.save7.org" appears
23 times, including `src-vol/03-learn.js:136` (`const LEARN_HOME = 'https://learn.save7.org'`).

### 5.2 Every table and view

| Table | Created | Purpose | RLS |
|---|---|---|---|
| `learners` | `0091:46-84` | the third principal — anybody taking the course, staff / volunteer / public. `volunteer_id → volunteers` **unique** is what joins course progress to the portal. Deliberately **no** rule against a `@save7.org` address (`:77-83`) | own row or `app_is_staff()`; update narrowed by `0092:322-324` to five columns |
| `learn_levels` | `0091:106-115`, reshaped `0093:36-52` | the three levels; `pass_mark_pct` default 70 | read = any session |
| `learn_modules` | `0091:117-126`, `0093:56-62` | modules in a level; `is_mandatory` drives every completion count | read = any session |
| `learn_lessons` | `0091:128-137`, **recreated** `0094:33-46` | lessons; PK is `(module_slug, slug)` because slugs repeat — `0094:19-31` records the bug ("the 97 lessons upserted into 26 rows") | read = any session |
| `learn_resources` | `0091:139-152`, `0093:74-88` | the reading list; `is_stub` marks an incomplete citation | read = any session |
| `learn_questions` | `0091:161-199` | one question, course or gate. `scope` enum `PRE\|POST\|CHECK\|GATE`, `kind` enum `SINGLE\|MULTI\|TRUE_FALSE\|SCENARIO` | **revoked from `anon` and `authenticated`** (`0091:370`) — read only via `learn_questions_pub` |
| `learn_choices` | `0091:208-227` | the options **and `is_correct`** | RLS on with **zero policies for any role** (`0091:358-361`) — *"the only thing that reads it is `learn_mark()`"* (`:230-231`) |
| `learn_attempts` | `0091:238-265` | one sitting; only attempt 1 counts toward reported improvement (`:246-249`) | own row or staff |
| `learn_answers` | `0091:269-277`, `0096:110-111` | what was chosen per question | own attempt or staff |
| `learn_module_progress` | `0091:282-288`, `0096:73-81` | per learner per module | own select + **own insert + own update** — the only progress table a client may write |
| `learn_level_progress` | `0091:290-296`, `0096:83-88` | per learner per level; `completed_at` is **what the vetting gate reads** | select only; written by `learn_complete_module()` |
| `learn_course_progress` | `0091:298-306`, `0096:90-98` | baseline in, final out, per learner | select only |
| `learn_certificates` | `0091:311-320`, `0096:102-107` | issued certificates; `code` e.g. `S7-2026-B-000123` (`0098:104-106`); name/title/score are snapshots | select only — **no insert or update policy**; issuing is `learn_issue_certificate()` |
| `learn_review_items` | `0091:326-339`, `0096:146-148` | the content-review register: every medical/legal/statistical claim and whether Save7 signed it off | staff read / staff write |
| `learn_course` | `0096:41-49` | the single course row, `slug = 'transplant-alchemy-101'` | `using (true)` — **the only publicly readable table**, because the landing page names the course before sign-in (`0096:52-56`) |
| `learn_events` | `0096:116-127` | engagement log; `learner_id` nullable so signed-out reading still counts | anybody inserts their own; **only staff read** |
| `learner_registrations` | `0095:49-60` | throttle ledger for `register-learner`; salted hashes, never raw values | RLS on with **no policy at all** — service_role only |

Views: `learn_questions_pub` (every question **without `explanation`**, `0092:32-47`),
`learn_options_pub` (options **without `is_correct`**, `:51-61`), `learn_my_attempts` (`:64-76`),
`learn_my_progress` (own baseline/final plus a computed `improvement_pct`, `:78-93`), and
`vol_my_course_learning` (the pull-through to the portal, `:105-118`).

Functions, all `security definer` with a pinned `search_path`: `app_learner_id()` / `app_is_learner()`
(`0091:89-101`); `learn_mark()` (`0092:131`, dropped at `0097:34`); `learn_start_attempt()`,
`learn_attempt_detail()`, `learn_submit_attempt()`, `learn_read_attempt()`, `learn_grade_check()`
(`0097`); `learn_view_lesson()`, `learn_complete_module()` (`0096`); `learn_claim_me()`
(`0095:161-207`); `learn_issue_certificate()`, `learn_verify_certificate()` (**granted to `anon`**,
`0098:150`), `learn_set_name()` (`0098`); `verify_learn_isolation()` (`0092:273-306`);
`volunteer_mark_gate()` (`0103:164`).

`verify_learn_isolation()` asserts: RLS on for every `learn_*` / `learners` table; `learn_choices`
has **exactly zero** policies; no `learn_*` or `vol_*` view definition contains the string
`is_correct`; `anon` can read neither `learn_choices` nor `learn_questions`. Called at the end of
`0092`, `0094`, `0095`, `0096`, `0098`, `0101`.

Two structural rules: gate quizzes **cannot** be marked by `learn_mark` / `learn_start_attempt` —
*"Gate quizzes are marked by submit-quiz, not here"* (`0092:150-154`, `0097:51-54`); and inline
checks gate nothing — *"Formative: no pass mark, and it gates nothing"* (`0097:296-298`).

### 5.3 Is learn.save7.org's content now inside this database?

**Yes — the full content bank is committed as seed INSERTs, twice** (`0094_learn_content.sql`, then
re-emitted as `0101_learn_content.sql`). Both are generated, not hand-written: *"GENERATED by
`scripts/emit-supabase-content.ts` in the transplant-alchemy repo, from the authoring files under
`prisma/content`"* (`0101:10-12`, `0094:4-6`), and both are upserts keyed on the authoring identifier
so re-applying updates in place rather than orphaning answers (`0101:4-8`).

Statement counts in `0101`: 3 levels, 13 modules, **97 lessons**, 27 resources, **131 questions**,
**512 choices**, 129 review items. `0094` is the same with 23 resources and 123 review items.

Each file ends with an assertion block that aborts the transaction on a partial load —
`0101_learn_content.sql:1084-1104` checks `learn_levels = 3`, `learn_lessons = 97`,
`learn_questions = 131`, and `learn_questions where scope = 'GATE'` = 40, plus an integrity check
that every non-`MULTI` question has exactly one correct choice.

A real row, not a placeholder (`0101:27-41`): level `'beginner'`, *"Start the Conversation"*, tier
`BEGINNER`, 30–45 minutes, certificate title *"Conversation Starter"*, pass mark 70. And the course
row itself (`0096:58-69`): `'transplant-alchemy-101'`, *"Transplant Alchemy 101"*, subtitle *"Save7
Organ Donation & Transplantation Awareness Course"*, `is_published = true`.

The 40 gate questions carry *"the option keys the volunteer portal has always sent, so historical
`volunteer_quiz_attempts.answers` keep resolving against them"* (`0101:20-22`).

**What has not moved is the presentation layer.** `src-vol/03-learn.js:17-27`: 53 of the 97 lessons
are markdown and render as written; the other 44 carry a `component_key` and a JSON payload because
the course app draws them as React components. *"That is a deliberate downgrade, and it is why
`learn.save7.org` still matters: the public course is the richer telling."* The portal implements
nine of those component renderers (`TakeawayList`, `ResourceList`, `PathwayJourney`, `ComparePanel`,
`ScenarioDialogue`, `EligibilityMatrix`, `OrganExplorer`, `MythFlip`, `TeamRoster`, `ChapterVideo` —
`src-vol/03-learn.js:88-134`) and degrades an unknown key to a note plus "Open it on
learn.save7.org" (`:285-288`).

### 5.4 The two learning Edge Functions

**`supabase/functions/register-learner/index.ts`** (217 lines) — public enrolment. *"Transplant
Alchemy is a public awareness course, so this endpoint is open: anybody may register, and no
signed-in caller vets them"* (`:5-8`). Holds `service_role` because `learners` has no insert policy,
and *"a client that could write that table could write the fourth accept path of
`auth_enforce_save7_domain()` and mint itself an account"* (`:21-24`). CORS allows
`learn.save7.org`, `*.transplant-alchemy.pages.dev` and localhost — **not** `volunteers.save7.org`
(`:38-57`). Throttles 8 refusals / 25 accepted per salted IP hash per hour (`:92-93`, `:129-145`).
**POPIA consent is required, not defaulted** (`:165-170`). A `@save7.org` address is refused with
*"Save7 staff sign in directly with their Save7 account"* (`:172-178`). Idempotent on an existing
active row; a revoked row is not silently reactivated (`:180-194`). Sets `volunteer_id` when the
address matches an active volunteer (`:196-210`).

**`supabase/functions/submit-quiz/index.ts`** (189 lines) — marks a volunteer's gate quiz. *"THIS
FUNCTION EXISTS BECAUSE THE ANSWERS MUST NOT SHIP TO A BROWSER"* (`:6-20`); `key.ts` beside it holds
the answers and *"is never imported by anything under `src-vol/`"* (`:9-11`). Unlike
`register-volunteer` it is **authenticated**: it demands a Bearer token, resolves the user
server-side and looks the volunteer up by `user_id`, never from the body (`:22-25`, `:76-95`). It
validates that every question of the named quiz is answered and refuses rather than marking a partial
(`:110-122`); it is the **only** writer to `volunteer_quiz_attempts`, which has no write policy for
anybody (`0088:101-103`). It ticks `course_done` / `course_done_source = 'quiz'` only when both
quizzes pass, and never un-ticks or overwrites a manual tick (`:165-177`). Its per-question `review`
is deliberately not stored and not queryable (`:128-131`, `:179-187`).

**Unresolved in the tree:** `0103:164+` introduces `volunteer_mark_gate()` explicitly as the
replacement for `submit-quiz` — *"that function carries its own copy of the forty questions and their
answers in `key.ts`, and the same forty are now rows in `learn_questions` (0091). Two stores meant
two answers to 'did they pass'"* — but nothing in `src/` or `src-vol/` calls it, and `submit-quiz` is
still listed as deployed (`docs/volunteer-portal-runbook.md:24-29`). Which is live is **unverified**.

---

## 6. Current state & open threads

HEAD at time of clone: `8709cc25d5d44e76b2f97397c605168dbace677c`, 14 Sep 2026, author
`Save7 OS <claude@save7.org>`. Working tree clean.

### 6.1 What is deployed and live

| Thing | State | Evidence |
|---|---|---|
| **Save7 OS at `os.save7.org`** | **Live, real data.** Cloudflare Pages from `main` | `CLAUDE.md:11`, `CLAUDE.md:14-15`, `HANDOVER.md:10` — "It is in production at `os.save7.org` and real people sign in to it." `HANDOVER.md:167` — "Work on `main` and push there — that is what Cloudflare Pages deploys from." `docs/session-handover.md:18-20` — "pushing is publishing." |
| **Supabase project `zbaoziisqroqxfwcnhlb`** | Live. **No local Postgres — every migration goes straight to production** | `HANDOVER.md:163`, `HANDOVER.md:189-190` |
| **Volunteer portal** | Built and **deployed to `save7-volunteers.pages.dev`**. `volunteers.save7.org` is **NOT live** — DNS outstanding | `docs/session-handover.md:76-79`; `HANDOVER.md:414-418` — "**Only the DNS is outstanding:** a `CNAME` for `volunteers` at **xneelo/konsoleH**, not Cloudflare — `save7.org` is not on Cloudflare DNS." |
| **`learn.save7.org`** | Its **backend** moved into this Supabase project (`0091`–`0098` + `register-learner`) | `docs/session-handover.md:207-213` |
| **`csr.save7.org`** | **Scaffold only.** The CSR front-end code was deleted from this repo 8 Aug 2026 and is not recoverable from git | `docs/session-handover.md:34`; `HANDOVER.md:107-113` |
| Edge Functions | 4 deployed: `create-funder-account`, `register-learner`, `register-volunteer`, `submit-quiz` | `supabase/functions/`; `docs/session-handover.md:60-61` — `register-volunteer` "is the only unauthenticated endpoint in the project" |
| `~/protoype/save7` (campaign site) | Separate repo, Astro/Svelte/Vercel/Neon. **Out of scope** | `docs/session-handover.md:35`; `docs/supabase-setup.md:14-16` — "Two databases, deliberately" |

### 6.2 What is half-built

1. **The gate quizzes cannot be sat.** Commit `ad25d8b` body: *"Still to build: the quiz screens in
   the volunteer portal."* Confirmed: `grep -rn "submit-quiz" src src-vol` returns nothing;
   `src-vol/` shows attempts read-only (`src-vol/03-app.js:174-179`). Until then the prior-learning
   waiver is the only route through vetting.
2. **41 GATE questions are dead data** — the `0087`/`0088` quizzes were superseded and their
   questions *"sit in `learn_questions` with `scope = 'GATE'`, unread by anything"*
   (`docs/volunteers.md:265-271`).
3. **Module "Check" questions gate nothing** on either surface (`docs/volunteers.md:258-262`).
4. **Nothing past approval has ever run on real data.** `docs/session-handover.md:130-131` —
   *"Vetting, opportunities, applying, hours and verification … have never had a real person walk
   through them. That is the largest untested surface in the project."*
5. **One unexplained bug, one occurrence** — *"After a successful registration, `volunteers` read
   zero rows once."* No root cause (`docs/session-handover.md:133-136`).
6. **The joinable campus list is a guess** — *"The five branches currently marked joinable were a
   guess"* (`docs/session-handover.md:65-67`).
7. **Google Drive as a page: nothing built** (`HANDOVER.md:443-444`, `docs/google-drive.md:3`).
8. **Google Calendar beyond the My work card: nothing built** (`HANDOVER.md:454-455`,
   `HANDOVER.md:495-496`).
9. **Payfast: decision not made** (`HANDOVER.md:471`).
10. **No end-to-end test with a real signed-in session** — the suite stubs `window.sb`, so auth, RLS
    and receipt upload are exercised against a fake (`CLAUDE.md:881-882`, `HANDOVER.md:556-558`).
11. **`vol_*` views have no event trigger** — a migration must call `verify_vol_views()` by hand, and
    this has already silently broken once at `0088` (`HANDOVER.md:124-138`).
12. **No dark mode** — `prefers-color-scheme` appears nowhere (`HANDOVER.md:550-551`). **The type
    scale is px, not rem**, so browser text size is ignored (`HANDOVER.md:546-549`).
13. Leftover test rows: `glieb@volunteers.save7.org` (`docs/session-handover.md:127-128`),
    `test@csr.save7.org` (`docs/volunteer-portal-runbook.md:209-210`).

Data the org still has to enter: opening cash balance is 0 (`HANDOVER.md:500`); almost no budget
allocation entered (`HANDOVER.md:502`); `project_tracker.ops_mentor` unset on every real project
(`HANDOVER.md:493-494`); `assets.invoice_total` null on every real row (`HANDOVER.md:519-522`).

### 6.3 Open questions, quoted

From `HANDOVER.md` §7 ("Waiting on the user", `HANDOVER.md:436`):
- `HANDOVER.md:450-452` — "Verify first: Google notes service accounts do not belong to the
  Workspace domain".
- `HANDOVER.md:460-461` — "Four decisions are listed at the end of that doc, and the second one is
  whether anybody wants this at all."
- `HANDOVER.md:471` — "Decision not yet made." (Payfast.)
- `HANDOVER.md:472-473` — "Section 18A certificate format. The user said they would supply one."
- `HANDOVER.md:474-475` — "'Jordan Christiaans' or 'Jordan Christians'? … Flagged, not changed."

Accountant review (`HANDOVER.md:524-528`): the 18A eligibility classification; the certification
paragraph wording; whether the R5 995 in-kind sponsored asset is itself an 18A donation in kind.

From `CLAUDE.md` "Open questions" (`CLAUDE.md:894-907`) — **the two most relevant to a role model**:
- `CLAUDE.md:899` — "The real permission list, and whether branch managers get more than members."
- `CLAUDE.md:907` — "Who owns adding rows to `people` — a Workspace mailbox alone must not grant
  access to the books."

Also: `CLAUDE.md:902` "Real team budget allocations; annual or per term."; `CLAUDE.md:903` "Whether
fuel claims should be capped per person per month."; `CLAUDE.md:906` "Does Workspace have alias or
secondary domains?"

From `docs/auth-design.md:89-95`: "Session length, and whether exco needs re-auth for sensitive
actions (approving payments)."; "Who administers the `people` table — adding a Workspace account
must not silently grant access to the books".

From `docs/session-handover.md:125-126`: "Two are genuinely undecided — the Dialysis support group
and Red Cross." / "Somebody at Save7 has to say whether a volunteer may join those."

From `docs/volunteers.md:270-271`: "Whether vetting should require passing them again is a decision
about who may be given work, not a bug to fix quietly."

### 6.4 Code TODOs

A grep for `TODO|FIXME|XXX` across `*.md`, `*.js`, `*.sql`, `*.mjs`, `*.html` returns **zero
matches.** This repo carries no literal code TODOs; open work is recorded in prose in `HANDOVER.md`
and `docs/session-handover.md`. The only genuine in-schema gaps are two `is_stub` course citations
(`0094_learn_content.sql:1900`, `:7718` and the same in `0101`): *"Citation incomplete: authors, year
and a stable public link are still needed."*

### 6.5 Build, deploy and test pipeline

`package.json:7-17`: `build` → `node build.mjs` → `dist/Save7-OS.html`; `build:vol` →
`node build-vol.mjs` → `dist-vol/Save7-Volunteers.html`; `test` → build both, then
`node test/smoke.mjs`; `shots` → `scripts/screenshots.mjs`; `serve` → `scripts/serve.mjs` on 4322,
`serve:vol` on 4323. Deps: `playwright` (dev) and `@supabase/supabase-js`. Node ≥ 20
(`package.json:21-23`), `.node-version` pins 22.

`.claude/launch.json:3-22` defines two local servers, `save7-os` (4322) and `save7-volunteers`
(4323). **`file://` no longer works** for local dev because Google will not redirect OAuth to a file
origin (`docs/supabase-setup.md:9-10`) — `CLAUDE.md:61` still says it does, a stale line.

Tests: `test/smoke.mjs` (52 lines) is the runner, deliberately importing section files **in a loop
with `await`** so they run sequentially (`test/smoke.mjs:18-22`). `test/harness.mjs` (~84 KB) holds
fixtures and the Supabase stub, shared with `scripts/screenshots.mjs` so the two cannot drift;
`test/harness.mjs:16-18` — *"Nothing here proves RLS works; that is enforced in the database."*
Seventeen section files: `01-auth-and-boot`, `02-my-work`, `03-portal-and-documents`,
`04-deleting-and-live`, `05-teams-and-budgets`, `06-projects-and-forms`,
`07-section-18a-and-ledger`, `08-claude-seats`, `09-database-and-assets`, `10-funders-and-ideas`,
`11-sorting-and-meetings`, `12-tracker`, `13-board`, `14-projects-lifecycle`,
`15-mobile-and-settings`, `16-volunteers`, `17-research`.

Assertion counts disagree across the repo: `HANDOVER.md:7` and `docs/session-handover.md:9` say
1641; `CLAUDE.md:45` and `:849` say 1897; the HEAD commit body says "1897 assertions, 0 failed".
Not run here — **unverified**.

### 6.6 Recent work (last 30 commits, 17 Aug – 14 Sep 2026)

Three clusters:
- **17–21 Aug — the Transplant Alchemy course moves onto this project.** `e91ae38` two server-marked
  quizzes gating who can be given work; `a983574` "Host the Transplant Alchemy course on this
  project"; `2adaa69` "Complete the course backend: parity, marking, certificates, enrolment";
  `7b540f5` "A Learn tab: the course inside the volunteer portal"; `152c9ca` "The course is the gate".
- **21–25 Aug — OS surface.** `ef69d9c` the research board (`0100`); the sidebar-collapse trio;
  `3fa65d6` course content to the database (`0101`); `f0bd557` Google calendar on My work;
  `2d55775` the board archive (`0102`).
- **10–14 Sep — three schema changes.** `ad25d8b` (`0103`+`0104`) vetting needs both levels and both
  quizzes, or a named prior-learning waiver; `meeting_action_owners` replaces the single `assignee`
  so an action point can be owed by several people. `4bc8bd1` (`0105`) **fuel claimed per kilometre
  at the SARS rate**, reversing the original slip-based decision and making `receipt_id` nullable —
  the first loosening of "no payment without its document". `8709cc2` (`0106`) asset disposal in one
  press, with the delete policy still deliberately withheld.

---

## 7. What a merge would collide with

Stances stated as deliberate and defended, with the citation.

**7.1 The concat single-file build.**
`build.mjs:5-13` — `src/*` concatenated in filename order into one ~630 KB HTML file; the library is
inlined rather than pulled from a CDN. The numeric prefix **is** the load order and `.sort()` on the
filename is the whole rule (`CLAUDE.md:78-79`). Two positions are load-bearing: `05-f-pages.js` must
sort after every `05-*` file whose renderer it names, and `08-auth.js` must stay last because it
holds the closing tags (`CLAUDE.md:121-125`). Every file is its own `<script>`, so cross-file
references work **at call time only** — passing a later file's function by name to
`addEventListener` is an evaluation-time read and throws (`CLAUDE.md:114-119`).
The offline requirement it served was dropped August 2026 (`CLAUDE.md:17-19`, `build.mjs:9-11`), and
the stated migration path is Vite + `vite-plugin-singlefile`, *"Do not switch to a multi-file bundle
or a CDN dependency"* (`CLAUDE.md:884-890`).
Collision: there is no module system. Anything transplanted must become a numbered, prefix-ordered
global-scope `<script>` fragment.

**7.2 No framework, whole-page innerHTML re-render.**
`CLAUDE.md:130-136`: every view is a `render*()` that writes `innerHTML` into a mount point, re-run
wholesale via `renderAll()` after any mutation; **all** events go through one delegated `click`
listener in `src/07-c-forms.js` keyed on `data-*` attributes. *"Add behaviour by adding a `data-`
attribute and a branch in that listener, not by attaching listeners in render functions — they'd
leak on re-render."*
Collision: any component that owns its own listeners, or any virtual DOM, is incompatible with
`renderAll()`.

**7.3 No `localStorage`, by rule.**
`CLAUDE.md:759-761`: *"Nothing persists across reloads, by design — which is why a preference has to
be a row too. `person_prefs` (0054) is where per-person state goes; do not reach for `localStorage`
to avoid a migration."* Enforced culturally, not mechanically; there is a standing test
(`test/smoke/15-mobile-and-settings.mjs:315`). One exception exists in the volunteer build —
`src-vol/04-auth.js:249` calls `localStorage.removeItem('save7-volunteers-auth')`, i.e. clearing
Supabase's own session key on sign-out, not storing app state.
Collision: a UI that remembers a hub, a tab or a collapse state locally must instead ship a
migration and a write path.

**7.4 RLS is the boundary; the client is affordances.**
`0002_rls.sql:6-9,19-20`; `src/06-branch-portal.js:8-19`. Stated consequence: if `can()` is ever
pointed at a table, *"it has to agree with the policies or it will offer buttons whose writes are
refused"* (`CLAUDE.md:877-880`). The advisor position is the same: seven `security_definer_view`
ERRORs on the `csr_*` views are *known and the remedy must not be applied* (`CLAUDE.md:822-829`),
and five more on `learn_*` (`:813-821`); the replacement check is `verify_csr_isolation()` and
`verify_learn_isolation()`, and **every migration that touches a `csr_*` view must end with
`select verify_csr_isolation();`** (`CLAUDE.md:832-838`).
Collision: a role model expressed in application code rather than in policies would be, by this
project's standard, not a role model at all. Any new actor needs an enum value or a helper function
plus policies on every table it touches.

**7.5 The brand font divergence.**
`CLAUDE.md:523-530`: *"The display face here is Bebas Neue, not the Anton the org brand kit
specifies. A deliberate, Save7-OS-only divergence: Anton was too heavy for a screen this dense."*
The two are not drop-in: cap heights `.700em` vs `.859em`, natural line heights `1.200em` vs
`1.505em`, corrected by `size-adjust:123%` on the `@font-face`. *"If the face is ever changed again,
recompute that number from the new font's cap height — do not carry it over."*
Tokens: pink `#ED0E69` hero (used with restraint), teal `#16B9B4` accent, cream `#F7F3EE`
background, ink `#111111` (`CLAUDE.md:517-521`). The font is embedded as a data URL at build time
(`build.mjs:84`).

**7.6 Things a hub-based top nav would specifically fight.**
- **There is no top nav.** The top bar holds only eyebrow/title/subtitle, the period selector, the
  quick-add button and the Database button (`src/05-f-pages.js:52-57`, `:82`). All navigation is the
  left sidebar.
- **Every page is pre-rendered into the DOM** and shown by toggling `.on` on `.page`
  (`src/05-f-pages.js:58`). There is no router, no URL, no history. `go()` does not push state; a
  reload always returns to `landingPage()`.
- **The shell is binary.** `SESSION.role` is `'admin' | 'member'` and the page you open flips it
  (`src/05-f-pages.js:47-49`). `SHARED_PAGES` exists because three pages broke that rule; a hub model
  with several cross-cutting surfaces would make most pages shared and dissolve the rule.
- **One quick-add button, repointed per page** (`src/05-f-pages.js:53`). Every page must declare a
  label and a modal id.
- **Group headings are hardcoded markup**, not data: `Dashboards / Money / Records / Setup`
  (`src/02-markup.html:52,71,92,105`). Reordering into hubs is an edit to markup, and
  `landingChoices()` reads `Object.keys(PAGES)` order for the landing-page picker
  (`src/07-m-prefs.js:23`), so the two would need to stay in step.
- **`docs/map.md` is the index** and is expected to gain a row per feature
  (`CLAUDE.md:127-128`, `docs/map.md:93-118` sets migration conventions).
- **The desktop layout is frozen by a byte-for-byte screenshot test**: `npm run shots` twice across
  a change and the desktop PNGs must be identical (`CLAUDE.md:534-537`).

**7.7 Where CLAUDE.md and the code disagree (findings).**

| CLAUDE.md says | The code does | Citation |
|---|---|---|
| "The app tracks the money: income and spend, fuel reimbursements, branch spend with receipts, and recovery of Claude subscription costs" (`CLAUDE.md:8-9`) | also: projects+tracker, meetings, assets, research, volunteers, a whole LMS, a funder portal | §3 below |
| "`fuel_claims` — `receipt_id` is **NOT NULL**" (`CLAUDE.md:151`) | nullable since 0105; the same file says so 200 lines later (`CLAUDE.md:410-420`) | `0105_fuel_at_the_sars_rate.sql` |
| "Four branches: Tygerberg, UCT, Stellenbosch, Pretoria … There is deliberately no fake head-office branch" (`CLAUDE.md:399-403`) | ten branch rows, including programme teams and Exco / Advisory board / Management | `0022:19-22`, `0026:20-23` |
| `PERM_SPEC` is "what the branch portal offers" per role (`CLAUDE.md:878`) | `PERMS` is a static constant with no role input; identical for every user | `src/06-branch-portal.js:65-66` |
| "`role_permissions` is documentation, not enforcement. Nothing queries it." (`CLAUDE.md:877`) | **true** — verified | §1.4(c) |
| `npm test` is "1897 assertions" (`CLAUDE.md:45`, `:841`) | unverified — not run | — |
| "Three sidebar modes, not two" (`CLAUDE.md:551`) | three CSS modes **plus** a stored two-value override | `src/07-j-drawer.js:100-131` |

Also stale, in the repo's own index: `docs/map.md:22` cites `0031`-`0039` for Section 18A when
`0031`-`0035` are seat reminders, auth relinking and the admin account; `docs/map.md:23` cites `0026`
(governance teams) for the asset register and omits `0040`/`0042`/`0043`/`0044`; and `docs/map.md:39`
describes the vetting gate as `0099`'s rule, which `0103_vetting_needs_the_quizzes_or_prior_learning`
has superseded.
