# Save7 Operations OS

Internal ops platform for Save7, a student-run organization operating across university branches. This glossary defines the roles, org units, and navigation concepts that gate access throughout the app.

## Language

**Branch**:
A Save7 chapter at one university (e.g. UCT, Stellenbosch, Wits, UP, UJ). Admin-managed table, extendable as new chapters form. A branch is deactivated rather than deleted once wound down — Branch Managers, MOUs, and volunteers reference it, so its history persists.
_Avoid_: location, chapter, campus

**National Exco**:
Org-wide leadership seats — CEO, COO, CGO (Chief Growth Officer — field of responsibility still TBD), CFO. Full access to every hub across every branch and department. Provides overall oversight, alignment, and accountability across both the Branch and Department systems. Authors Department Head MOUs directly; reviews and signs off on Branch-Manager-authored Project Lead MOUs.
_Avoid_: leadership, management, executives, exec team, CMO (no longer an Exco seat — see Department Head)

**Admin**:
Master account (1-2 held) that provisions user accounts and assigns their role + branch/department. A distinct role from National Exco, even when the same person holds both.
_Avoid_: superuser, root, master account (as a role name)

**Branch Manager**:
Runs one branch. Edit access to their own branch across hubs; read-only access to other branches. Authors MOUs (responsibilities/expectations/deliverables) for that branch's Project Leads — subject to Exco review and sign-off.

**Operations Manager**:
Runs one branch's operations team. Branch-scoped, same edit-own/read-others pattern as Branch Manager. Oversees all volunteers in the branch, all Project Leads, and new-volunteer onboarding; along with the Branch Manager and COO, funnels volunteers with a specialised interest toward the matching Department.

**Branch Function Head**:
A branch-level lead over one hub's function for their branch only. Currently just Finance Head — the former branch-level Media Head no longer exists; media now sits entirely under the national Media Department. No branch-level equivalent exists for Compliance, which is national-only.
_Avoid_: department head, branch lead

**Department**:
A national/organisation-level portfolio for specialised work (e.g. Media, Tech, Finance, Research, Education). Open-ended, admin-managed list, extendable the same way Branch is. Distinct axis from the Branch system — operates directly under Exco, not under a Branch Manager.
_Avoid_: hub (a Department is an org unit; Hub is a nav section — a Department typically owns a Hub, but they aren't the same concept)

**Department Head**:
Owns, grows, and organises one Department at national scale. MOU authored directly by Exco (no Branch Manager in the loop). Becomes the national owner of the existing Hub matching their portfolio where one already exists (e.g. the Media Department Head owns the Media Hub — this absorbed the former CMO/Marketing remit, which is no longer a separate Exco seat or department).
_Avoid_: branch function head, branch lead

**Project Lead**:
Not a distinct account type — a status/upgrade on a Volunteer, granted via a Branch-Manager-authored MOU, scoped to a time-boxed initiative/project (vs. Finance Head/Operations Manager, which are standing operational seats). Sees the volunteers working under them and their assigned tasks.

**MOU**:
An offline, signed agreement between a role and its supervising structure (Branch Manager ↔ Project Lead, Exco ↔ Department Head) setting responsibilities, expectations, deliverables, and KPIs/targets. Not a UI workflow — referenced in the OKRs hub (KPI/target display only) and uploaded to the Compliance Hub for record-keeping.

**Volunteer**:
Recruited at one branch. Has their own dashboard only — no top-nav hub access at all. Can simultaneously belong to a branch, be a member of one or more Departments, and hold Project Lead status for one or more initiatives — these are not mutually exclusive.

**Hub**:
A top-nav section of the site (OKRs, Finance, Compliance, Media, Operations, Branch Managers, Departments — the last with a sub-hub per department). A hub is hidden entirely from a role's nav if that role can't access it, rather than shown disabled.
