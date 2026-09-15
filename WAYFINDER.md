# Save7OS — Wayfinder Map

Living decision record for the Save7 operations OS build. Read this first in any new
session before touching code — it's the map so context doesn't get re-derived or
re-litigated every time the conversation resets. Open questions are tracked as GitHub
issues (linked at the bottom); this file holds what's *already decided*.

## Status

- **Phase:** clickable prototype (structure + role logic only). No live backend yet.
- **Stack decision:** Next.js + Supabase (auth, DB, storage) + Tailwind. Not yet scaffolded.
- **Prototype:** https://claude.ai/artifact/EG4vtr3ySH3A9SQko21ASK — source mirrored at
  [`prototype/index.html`](prototype/index.html) in this repo.
- **Brand:** logo files only in the brand kit (no color/type guideline doc found).
  Colors sampled from the logo: teal `#00B9B5`, pink `#ED186B`.

## Roles

| Role | Scope | Notes |
|---|---|---|
| `admin` | everything | master account, provisions all other accounts |
| `national_exco` | all branches | titled seats (CEO/CFO/COO/CMO…), sets Mission/Strategy/Pillars |
| `advisory_board` | all branches | read/oversight tier |
| `branch_manager` | 1 branch (edit) + all branches (read-only) | assigned to exactly one branch |
| `ops_manager` | 1 branch's ops team (edit) + all branches (read-only) | branch-level; "national ops" = the COO (national_exco) plus whoever the COO delegates access to — not a separate role for now |
| `volunteer` | own dashboard only | no top-nav hub access at all |

## Auth model

Real accounts (no anonymous access). Landing page = **3 public doors**: Branch Manager,
Operations Manager, Volunteer. `admin` / `national_exco` / `advisory_board` accounts are
provisioned directly by the master admin — plain login, no public landing CTA.

Signed-out visitors still see the whole site shape (the fog-of-war map, everything
misted) as a "preview the whole OS" screen that prompts sign-in — not a hidden wall.

## Branch model

Branches = university chapters (e.g. UCT, Stellenbosch, Wits — seed list, more exist).
Admin-managed table, extendable. `branch_manager` and `ops_manager` are each tied to
exactly one branch; either can browse any other branch read-only but can only edit their
own.

## Volunteer onboarding flow

In-app: Landing → create/log in account → email verification → skills, interests &
availability → dashboard.

Outside this project (separate build, learn.save7.org): volunteer takes a course there,
completes levels, earns certificates. Certificates go to **either** the branch's Ops team
or the Branch Manager (either can validate) → they approve → volunteer account activates
in Save7OS. This repo represents that gate as a UI state only ("awaiting validation"
badge) — the course itself is out of scope here.

Volunteer dashboard panels (already mapped from the original flow diagram): Next best
opportunity, My commitments, Impact tracker, Community, Resources, Quick actions, Apply
for a project.

## Top nav hubs

Save7 OKRs · Finance Hub · Compliance Hub · Media Hub · Operations Hub · Branch Managers Hub

**Fog-of-war = literal dev-progress indicator**, not a game mechanic:
- **fogged** — not built yet (queued)
- **scouted** — structure visible, content still locked (currently only OKRs)
- **explored** — actually live for that role

Nothing is "explored" yet except the scaffold shell and the volunteer dashboard.

Draft access matrix — **unconfirmed, needs Exco sign-off**, tracked as an issue:

| Hub | Access (draft) |
|---|---|
| OKRs | all signed-in staff roles (partial visibility) |
| Finance | `branch_manager` (own branch), `national_exco`, `admin` |
| Compliance | `national_exco`, `advisory_board`, `admin` |
| Media | `branch_manager`, `ops_manager`, `national_exco`, `admin` |
| Operations | `ops_manager`, `national_exco`, `admin` |
| Branch Managers | `branch_manager`, `national_exco`, `admin` |
| *(volunteers)* | no hub nav at all — dashboard only |

## OKRs system

Explained in full by the user; not yet deep-designed in the product.

1. National Exco sets **Mission, Strategy, Pillars**.
2. Off those pillars, Branch Managers *and* National Exco set **Objectives** + **Key
   Results**.
3. Objectives split into **Active** vs **On Ice** (parked, revisit later).
4. Key Results carry **status logs** — on track / at risk / off track (exact taxonomy
   still TBD).

## Explicit scope decisions (don't relitigate these)

- This phase is a **prototype**: structure and role logic only, mock data, no Supabase
  wiring. Real backend comes only after the map itself is signed off.
- Finance / Compliance / Media / Operations / Branch-Managers hub **content** is
  deliberately *not* designed yet — deferred by the user's own call, to avoid getting
  lost in granulars before the map is approved. Don't design hub internals without an
  explicit ask — check the open issues first.

## Open work (see Issues tab for the live list)

1. OKRs hub — deep design (pillar structure, objective/KR CRUD, status-log taxonomy)
2. Finance Hub — content & permissions
3. Compliance Hub — content & permissions
4. Media Hub — content & permissions
5. Operations Hub — content & permissions (national vs branch split)
6. Branch Managers Hub — content & permissions
7. Confirm the hub access matrix above with Exco — currently a first-pass guess
8. National-ops delegation mechanic — how the COO grants "national ops" visibility to others
9. Branch data model — full branch list + admin UI to manage it
10. Auth + Supabase schema — turn the prototype's mock state into real accounts/roles/RLS
11. Volunteer ↔ learn.save7.org integration — how certificate data actually reaches Save7OS
