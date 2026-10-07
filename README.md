# docs-v2

This is a monorepo for hosting the content for the documentation of various projects in Auth0.

We use [Mintlify](https://mintlify.com/) for our documentation needs.

## Directories

- `main`: Main Documentation for Auth0: https://auth0.com/docs
  - Contains the primary Auth0 documentation content
  - Includes `docs/`, `snippets/`, and `ui/` subdirectories
- `auth4genai`: Documentation for Auth0 for AI Agents features in Auth0: https://auth0.com/ai/docs
  - Contains content for AI-specific Auth0 features
  - Includes sections for quickstart guides, integrations, SDKs, MCP, and more
- `ui`: Shared UI components and tooling
  - React/Vite-based component library used across documentation sites

## Local Development

Use the [Mintlify CLI](https://mintlify.com/docs/installation) to preview and edit documentation locally.

### Prerequisites

- [Node.js](https://nodejs.org/en) v19 or higher

### Installation

Install the Mint CLI globally:

```bash
npm i -g mint
```

Or using pnpm:

```bash
pnpm add -g mint
```

> **Note for VPN Users**
>
> When running `mint dev` for the first time, you'll need to **disable your VPN** to allow the framework to download. After the initial download completes, you can re-enable your VPN for subsequent runs.

### Running the Dev Server

1. Navigate to the documentation folder you want to work with (where the `docs.json` file is located):
   ```bash
   cd main  # or cd auth4genai
   ```

2. Start the development server:
   ```bash
   mint dev
   ```

3. Open your browser to `http://localhost:3000` to view the local docs

### Useful Commands

- **Update the CLI**: `mint update` or `npm i -g mint@latest`
- **Find broken links**: `mint broken-links`
- **Check accessibility**: `mint a11y`
- **Custom port**: `mint dev --port 3333`

For more details, see the [Mintlify CLI documentation](https://mintlify.com/docs/installation).

## Link Checking

We use [Lychee](https://lychee.cli.rs/) to check for broken non-local links. Our Lychee config is in [`lychee.toml`](lychee.toml), which is used in our CI checks and which you can use locally.

### Check links locally

For local link checking, run `lychee` from the root of the repo. Specify the config file and the path(s) you want to check. For example, to check everything in the main docs site:

```
lychee -c lychee.toml 'main/docs/**/*.mdx' 
```

### Check links in PRs

The `.github/workflows/link-check.yml` GitHub Action runs against PRs that change content files and leaves a comment with a summary of the results which lists any broken links.


Status

Draft for review

Author

@Nick Gagliardi

Date

Oct 6, 2026

Related

DOCS-5634 (Architect new image folder file structure — Done), DOCS-5635 (Team meeting on DB redesign details — Backlog), https://docs.google.com/spreadsheets/d/14I2p_-uZwBr6rY5gJjuGqJowua5itMNMkTcqx0nXV1I/edit?gid=901017615#gid=901017615

Summary

The Auth0 Dashboard is being redesigned, which breaks two things across auth0.com/docs: every link into manage.auth0.com (old routes) and every screenshot of the Dashboard UI (old visuals). The audit spreadsheet tracks 2,833 link references and 819 screenshot references, entirely unassigned-by-status and currently a fully manual find-and-fix job for one or two writers. This PRD proposes two independent automation tracks — a mechanical link-rewrite script (near-zero risk, ships first) and a Playwright-based screenshot regeneration pipeline (higher complexity, gated on tenant/timing dependencies) — so the team can produce the cascading PRs the Definition of Done requires before the redesign ships in mid-to-late October.

Context: what the audit actually contains

I read the full "Auth0 Dashboard Links Audit" spreadsheet (not just the linked tab) via the Sheets API, since the Drive export kept truncating. It has three tabs:

Tab

Rows

Shape

Status

Dashboard Links

2,833 data rows

old manage.auth0.com URL → MDX file path → docs page URL → assignee

Needs updated? and Status columns are empty on every sampled row

New Dashboard links

151 data rows

old URL → new URL lookup table, with a Change Type column (http → https + new path, path changed, add /dashboard/ prefix, malformed)

Already built — this is most of the migration logic, pre-written

Dashboard Screenshots

819 data rows

docs page URL → MDX file + line → current alt text → current image repo path → assignee (Amanda VanScoy / Hazel Virdo)

Japanese, French-Canadian, Light Mode, Status columns are all empty

Two findings change the shape of this project from what the brief implies:

1. The "new file structure" mostly already exists

DOCS-5634 ("architect new image folder file structure") is marked Done, and main/docs/images/ confirms why: it already has ~25 top-level feature-named folders following exactly the featurename/ pattern the brief asks for — ciba/, sender-constraining/, token-vault/, custom-token-exchange/, third-party-applications/, dashboard/, settings/, sessions/, tenants/, universal-login/, user-management/. The real gap isn't architecture — it's migration. Of 2,778 total image files, 860 (31%) still sit in images/cdy7uua7fh8z/, the leftover Contentful migration dump, under opaque hash-named subfolders (e.g. 4l47Xknr2LpuMSOX0T3yCP/800877a39f474faa0c6d83551d56c337/b2b-business-case.png). That's the "further clean-up" task the brief calls out, and this project is a natural forcing function to finish it for every Dashboard screenshot we touch.

2. No CI safety net exists for dashboard links today

lychee.toml explicitly excludes ^https://manage\.auth0\.com/ from link checking (it requires interactive login, so Lychee can't follow it). That means stale or malformed dashboard links — including the ones already in the sheet, like https://manage.auth0.com/?/authentication-profiles (wrong delimiter) or %7D-encoded malformed URLs — ship silently today and will keep doing so after this migration unless we add a guardrail.

Goals

Mechanically rewrite the large majority of the 2,833 dashboard link references using the already-built 151-row lookup table, with a clear report of what couldn't be auto-resolved

Stand up a repeatable screenshot capture pipeline against a seeded demo tenant, producing light-mode screenshots in the new naming convention with correct alt text

Finish migrating any touched images out of the cdy7uua7fh8z dump into the feature-folder structure as a side effect of normal work, not a separate pass

Add a CI guardrail so old-format dashboard links can't silently reappear after this project closes

Keep every change independently revertable so the team's cascading-PR release strategy works without one bad PR blocking the rest

Non-goals

Dark mode screenshots. Per the 09/30 scope update, light mode only for this pass. The naming convention and manifest still carry a -light/-dark suffix slot so dark mode is additive later, not a rename.

Auto-approving screenshot content. The pipeline produces candidate images; a human reviews each before merge, per the existing Technical Correctness Checklist.

Rewriting prose around screenshots. Only the image path, alt text, and <Frame> wrapping are touched — not surrounding instructional text — except where a link rewrite happens to fall inside that text.

Solving the full Contentful dump migration. We migrate what we touch; the remaining non-Dashboard files in cdy7uua7fh8z stay a backlog item.

Track 1: Link rewrite automation

This is almost pure text substitution — no browser automation needed. The 151-row lookup table already encodes the mapping logic; the work is applying it safely at scale and handling what it doesn't cover.

Data source

Export the New Dashboard links tab once to a static file checked into the repo — scripts/data/dashboard-link-map.json — rather than having the script call the Sheets API at runtime. The sheet is a working doc the team will keep editing; CI shouldn't depend on live Google credentials, and a committed snapshot makes every run reproducible and diffable in review.

Script: scripts/update-dashboard-links.js

Follows the existing scripts/localize-links.js convention already in this repo: dry-run by default, --fix to write, --verbose, and a stats summary.

node scripts/update-dashboard-links.js [--fix] [--verbose] [--report=path.csv]

Logic:

Load dashboard-link-map.json, normalize both sides (strip trailing slashes, unescape markdown-escaped braces like \{yourClientId} → {yourClientId}, treat http:// and https:// as equivalent for matching).

Scan every .mdx file under main/docs for manage.auth0.com occurrences, in markdown links, href attributes, and bare URLs in prose/backticks.

Exact match in the map → rewrite in place (or report the diff in dry-run).

Match flagged malformed in the Change Type column → flag for removal, don't silently delete.

No match at all → emit to the review report with file, line, old URL, and nearest fuzzy candidate (if any) — never guess. The map only covers 151 of what are likely several hundred distinct old URL shapes across 2,833 rows (I confirmed instances like #/mfa, #/connections/passwordless, #/phone/templates/phone/provider, and the typo'd ?/authentication-profiles that aren't in the current 151-row table), so a sizeable unmatched tail is expected on the first run.

Output: a CSV report (file, line, old URL, action taken, new URL or reason-not-fixed) that doubles as the per-writer triage list the "Needs updated? / Status" columns in the sheet were meant to track.

Guardrail: scripts/check-dashboard-links.js --ci

Reuses the same lookup table in reverse: fails if any committed .mdx file contains an old-pattern dashboard URL (hash-route without /dashboard/ prefix, or anything matching a malformed entry). Wire into a new .github/workflows/dashboard-link-check.yml, triggered the same way link-check.yml is (PR, path-filtered on **/*.mdx). This is what prevents regression once this project closes and closes the gap Lychee's exclusion rule leaves open.

Track 2: Screenshot regeneration pipeline

This is where Playwright earns its keep, but the bottleneck is tenant state and timing, not the capture scripting itself.

2a. Demo tenant seeding — scripts/seed-demo-tenant.js

A Management-API-driven setup script that provisions a dedicated, synthetic-data-only tenant so screenshots are reproducible and don't depend on whatever state someone's personal test tenant happens to be in. Seed data should reuse names already established in the docs' existing sample narratives (Acme Bot, Travel0 API, Big Holdings, Co. all already appear in current screenshots/alt text per the audit) rather than inventing new fictional companies:

Sample Organizations (for the ~16 org-related screenshot rows)

Sample Applications: SPA, Regular Web, M2M, Native (for application-settings, M2M-access, and login-experience screenshots)

Enterprise connections: at least one SAML, one OIDC, one Okta connection (several rows reference enterprise connection screens)

At least one custom Action in the library, one API/resource server, one organization-scoped M2M grant

This is reusable infrastructure, not a one-off — the same seeded tenant is what makes future Dashboard screenshot refreshes (not just this redesign) fast instead of starting from zero.

2b. Capture script — scripts/capture-dashboard-screenshots.js

Direct Playwright, not shot-scraper — the capture isn't single-URL shots, it's scripted multi-step navigation (open a specific settings tab, trigger a specific modal) per screenshot, which shot-scraper's model doesn't fit well.

Auth once, reuse everywhere. Log in interactively one time against the seeded tenant, persist Playwright's storageState.json, and every subsequent capture run reuses that session — no repeated interactive login, no MFA friction baked into automation.

Manifest-driven. A JSON manifest derived from the Dashboard Screenshots tab drives capture: { pageUrl, mdxFile, line, dashboardDeepLink, newImagePath, altText, cropSelector }. Generating this manifest from the sheet (one-time export script, same pattern as the link map) is itself a deliverable — it's what turns 819 manually-tracked rows into a machine-runnable job list.

Crop, don't full-page. The screenshot use policy caps Dashboard UI elements at 600px wide — this means capturing a defined selector/region per screenshot, not a full-page shot, matching what the policy already asks writers to do by hand.

--locale=ja-jp|fr-ca flag. Sets the Dashboard's UI locale before capture, for the translated-screenshot requirement. Open question below — need to confirm the Dashboard actually exposes a UI-locale switch before assuming this flag can work as designed.

--diff mode. Capture new, perceptual-diff against the existing committed image, and only surface screenshots that substantively changed for human review — most of the 819 rows won't need a human look twice if the diff is clean.

2c. New naming & path convention

Formalizes the brief's two patterns using the already-existing feature-folder precedent:

main/docs/images/dashboard/{screen}/{subscreen}/dashboard-{screen}-{subscreen}-light.png
  e.g. dashboard/tenant-settings/custom-domains/dashboard-tenant-settings-custom-domains-light.png

main/docs/images/{feature}/{job-to-be-done}/{feature}-{job-to-be-done}-light.png
  e.g. ciba/assign-roles/ciba-assign-roles-light.png

2d. MDX rewrite step — scripts/update-screenshot-refs.js

Once a captured screenshot is reviewed and approved, this script updates the corresponding MDX line: new image path, <Frame> wrapping, and alt text in the convention Auth0 Dashboard [section] [subsection] [view].

The brief's alt-text format (Auth0 Dashboard Branding Universal Login Settings tab) differs slightly from the house style guide's current baseline (Auth0 Branding Universal Login Settings tab — no literal "Dashboard"). Pick one and update the https://oktainc.atlassian.net/wiki/spaces/DOCS/pages/642812560 page so the convention is unambiguous before the script hardcodes it.

What's genuinely new vs. reused

Component

Status

Link lookup table (151 rows)

Already built in the sheet — just needs exporting to JSON

Image folder structure

Already built (DOCS-5634) — extend, don't redesign

scripts/update-dashboard-links.js

New, follows localize-links.js convention

scripts/check-dashboard-links.js + CI workflow

New

Demo tenant + seed script

New

Playwright capture script + manifest

New

MDX screenshot-ref rewrite script

New, shares file-editing logic with Track 1

Milestones

Phase

Scope

Exit criteria

0

Export link map to JSON; build update-dashboard-links.js dry-run

Dry-run report categorizes all 2,833 rows into fixed / malformed / unmatched, with real counts

1

Run --fix for exact matches; land guardrail CI check

First cascading PR(s) merged for pure link fixes; CI blocks old-pattern regressions

2

Resolve unmatched tail — extend the 151-row map or hand-fix remaining rows

Dashboard Links tab reaches 100% "Needs updated? = No"

3

Seed demo tenant; build capture script against current (pre-redesign) Dashboard as a dry run of the pipeline

Pipeline produces correctly-named, correctly-cropped screenshots end to end, before the real redesign ships

4

Re-run capture against the live redesigned Dashboard; diff, review, rewrite MDX refs in waves by section

All 819 screenshot rows updated; PRs ready to merge at redesign launch

5

Translated screenshots (ja-jp, fr-ca) — contingent on the locale-switch open question below

Translated image + alt text parity, or documented fallback plan

Risks & open questions

Timing dependency. Phase 4 can't produce accurate screenshots until the redesigned Dashboard is live somewhere capturable (staging/canary tenant, or production at launch). Definition of Done wants PRs ready at mid-to-late-October launch, which means Phase 4 is a tight window unless we get pre-launch staging access. This is the single biggest schedule risk — worth raising at the DOCS-5635 team meeting.

Does the Dashboard support a UI-locale switch? The translated-screenshot requirement assumes we can render the Dashboard in ja-jp/fr-ca. Unconfirmed — if not, translated screenshots may need coordination with the Dashboard team or a documented fallback (e.g., English screenshot + translated alt text/caption only).

Auth for the capture script. Need a demo-tenant account that doesn't require interactive MFA on every run, so storageState reuse actually holds across CI/scheduled runs.

Cross-team review. CODEOWNERS routes anything under /scripts to @auth0/project-docs-management-codeowner, not the writers team — both automation scripts need that team's review even though the content edits they produce are writers' turf.

Lookup table completeness. The 151-row map won't cover every distinct old URL shape in the 2,833-row audit (confirmed gaps above). Someone needs to own extending it as Phase 0's dry-run surfaces misses.

Dependencies

DOCS-5635 outcomes (team alignment on remaining file-structure details) should land before Phase 2 locks in the final naming convention

Access to a pre-launch staging/canary tenant running the redesigned Dashboard, if one exists — determines whether Phase 3's dry run can target real redesigned UI or only rehearses the pipeline mechanics against the current Dashboard

A provisioned demo tenant (owner: TBD) for seeding and capture
