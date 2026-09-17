# Save7 Operations OS

Internal ops platform for Save7, a student-run organization operating across university branches. This glossary defines the roles, org units, and navigation concepts that gate access throughout the app.

## Language

**Branch**:
A Save7 chapter at one university (e.g. UCT, Stellenbosch, Wits, UP, UJ). Open-ended table managed by both Admin and National Exco — added, edited, or deleted outright. Carries a separate active/inactive status reflecting current activity/momentum, not whether it still exists — an inactive branch is still fully editable, just flagged as needing a push to get moving again.
_Avoid_: location, chapter, campus

**National Exco**:
Org-wide leadership seats — CEO, COO, CGO (Chief Growth Officer — field of responsibility still TBD), CFO. Full access to every hub across every branch and department. Provides overall oversight, alignment, and accountability across both the Branch and Department systems. Authors Department Head MOUs directly; reviews and signs off on Branch-Manager-authored Project Lead MOUs — routed to them by the General Liaison. Super Admins by default and irrevocably. Exco alone assigns every seat in the organisation — Exco seats, General Liaison, Department Heads and the branch trio — grants or revokes Super Admin status, and adds branches and departments. Exco also hold every branch-level power across every branch. No one outside Exco, whatever admin tier they hold, can create or alter an Exco seat or override an Exco decision.
_Avoid_: leadership, management, executives, exec team, CMO (no longer an Exco seat — see Department Head)

**General Liaison**:
Single org-wide seat, distinct from National Exco. The routing point between Exco and everyone reporting up to it: Branch-Manager-authored Project Lead MOUs pass through them on the way to Exco sign-off, and Department Heads' interval updates and any Exco-discussion requests (including meeting invites) go through them too. Full hub access across every branch and department, same as National Exco, but doesn't author or sign off MOUs.
_Avoid_: secretary general, secretary, exco liaison

**Admin Dashboard**:
Where organisation-wide administrative work is done — seat assignment, the Branch and Department lists, the master volunteer tracker — rather than a section of the hub nav, mirroring the way a Volunteer has their own dashboard instead of hub access. Reached by National Exco and Super Admins.
_Avoid_: admin hub, admin panel, control panel, back office

**Super Admin**:
The only admin status there is — a flag layered on a seat, not a seat of its own, and never held by a Volunteer. Confers **read** access to every hub across every branch and department; it confers no edit rights, which still come from the underlying seat. The "Super" is measured against National Exco rather than a lesser admin: a Super Admin sees everything and changes nothing structural. One who is not Exco cannot grant or revoke Super Admin, assign any seat, add a branch or department, create or alter an Exco seat, or override an Exco decision. Exco hold it by default and irrevocably, and the Tech Department Head holds it by seat, since maintaining the site requires seeing everything; otherwise granted at Exco's discretion, in practice to seats already carrying organisation-wide responsibility such as the General Liaison. Organisation-wide account upkeep that no branch trio covers falls here.
_Avoid_: admin, superuser, root, owner

**Branch Manager**:
Runs one branch. Edit access to their own branch across hubs; read-only access to other branches. Authors MOUs (responsibilities/expectations/deliverables) for that branch's Project Leads — routed through the General Liaison for Exco review and sign-off. Appointed by Exco. With the Operations Manager and Finance Manager, accepts new volunteers onto the platform and grants Project Lead status — powers that come with the seat itself, not from any admin status. The trio own their branch's volunteers for the whole of that lifecycle: reviewing dormancy, deactivating, reactivating, and deleting an account outright.

**Operations Manager**:
Runs one branch's operations team. Branch-scoped, same edit-own/read-others pattern as Branch Manager. Appointed by Exco; shares the branch trio's powers over their branch's volunteers — see Branch Manager. Oversees all volunteers in the branch, all Project Leads, and new-volunteer onboarding; along with the Branch Manager and COO, funnels volunteers with a specialised interest toward the matching Department.

**Finance Manager**:
Runs one branch's finance function. Branch-scoped, same edit-own/read-others pattern as Branch Manager and Operations Manager. Reports directly to the CFO — there is no national Finance Department standing between them. Appointed by Exco; shares the branch trio's powers over their branch's volunteers — see Branch Manager.
_Avoid_: branch function head, finance head, department head, branch lead

**Department**:
A national/organisation-level portfolio for specialised work: Media, Tech, Research, Education. Open-ended, admin-managed list, extendable the same way Branch is. Distinct axis from the Branch system — operates directly under Exco, not under a Branch Manager. Finance is not a Department — see Finance Manager.
_Avoid_: hub (a Department is an org unit; Hub is a nav section — a Department typically owns a Hub, but they aren't the same concept)

**Department Head**:
Owns, grows, and organises one Department at national scale. MOU authored directly by Exco (no Branch Manager in the loop). Gives Exco in-person or online updates at set intervals, and routes any Exco-discussion requests through the General Liaison. Becomes the national owner of the existing Hub matching their portfolio where one already exists (e.g. the Media Department Head owns the Media Hub — this absorbed the former CMO/Marketing remit, which is no longer a separate Exco seat or department). The Tech Department Head additionally carries Super Admin by seat.
_Avoid_: finance manager, branch lead

**Project Lead**:
Not a distinct account type — a status/upgrade on a Volunteer, granted via a Branch-Manager-authored MOU, scoped to a time-boxed initiative/project (vs. Finance Manager/Operations Manager, which are standing operational seats). Sees the volunteers working under them and their assigned tasks.

**MOU**:
An offline, signed agreement between a role and its supervising structure (Branch Manager ↔ Project Lead, Exco ↔ Department Head) setting responsibilities, expectations, deliverables, and KPIs/targets. Not a UI workflow — referenced in the OKRs hub (KPI/target display only) and uploaded to the Compliance Hub for record-keeping.

**Volunteer**:
Recruited at one branch. Has their own dashboard only — no top-nav hub access at all. Can simultaneously belong to a branch, be a member of one or more Departments, and hold Project Lead status for one or more initiatives — these are not mutually exclusive. Accounts are not created on the platform: a prospective volunteer signs up and qualifies at learn.save7.org, receives a Save7 email address, and is then *accepted* onto the platform by one of their branch's trio. Graduating does not end volunteering — accounts persist, and deletion is a genuine last resort rather than routine cleanup.

**Hub**:
A top-nav section of the site (OKRs, Finance, Compliance, Media, Operations, Branch Managers, Departments — the last with a sub-hub per department). A hub is hidden entirely from a role's nav if that role can't access it, rather than shown disabled.
