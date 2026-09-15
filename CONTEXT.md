# Save7 Operations OS

Internal ops platform for Save7, a student-run organization operating across university branches. This glossary defines the roles, org units, and navigation concepts that gate access throughout the app.

## Language

**Branch**:
A Save7 chapter at one university (e.g. UCT, Stellenbosch, Wits, UP, UJ). Admin-managed table, extendable as new chapters form.
_Avoid_: location, chapter, campus

**National Exco**:
Org-wide leadership seats — CEO, COO, CMO, CFO. Full access to every hub across every branch.
_Avoid_: leadership, management, executives, exec team

**Admin**:
Master account (1-2 held) that provisions user accounts and assigns their role + branch. A distinct role from National Exco, even when the same person holds both.
_Avoid_: superuser, root, master account (as a role name)

**Branch Manager**:
Runs one branch. Edit access to their own branch across hubs; read-only access to other branches.

**Operations Manager**:
Runs one branch's operations team. Branch-scoped, same edit-own/read-others pattern as Branch Manager.

**Branch Function Head**:
A branch-level lead over one hub's function for their branch only (e.g. Finance Head, Media Head). No branch-level equivalent exists for Compliance, which is national-only.
_Avoid_: department head, branch lead

**Volunteer**:
Recruited at one branch. Has their own dashboard only — no top-nav hub access at all.

**Hub**:
A top-nav section of the site (OKRs, Finance, Compliance, Media, Operations, Branch Managers). A hub is hidden entirely from a role's nav if that role can't access it, rather than shown disabled.
