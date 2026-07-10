---
name: dynamics-release-watch
description: Check Microsoft Learn pages for Dynamics 365 Customer Service / Copilot Service workspace release notes and email Rachel Hodgson a plain-English summary of every change, however small, with an assessment of whether it could affect her workflow. Use this skill whenever running the scheduled Dynamics release-note check, or whenever asked to check for Dynamics/Copilot Service workspace updates, changes, or release notes.
---

# Dynamics Release Watch

Monitors Microsoft's official Dynamics 365 Customer Service documentation for edits and
notifies Rachel Hodgson (rachel.hodgson@tiltinsurance.co.uk) by email — every change, not
just the ones that look significant. A change that looked cosmetic previously broke a live
workflow, so nothing gets filtered out here on size or apparent importance.

## Primary signal: Microsoft's own change history page

Microsoft maintains a structured, dated changelog for each release wave — a proper table
of "feature added / release date changed / feature removed," per product, updated as
changes happen (not just twice a year). This is the primary detection source, in
preference to diffing rendered page text, because:

- It's already structured (a table with a date column), not something to be diffed by hand
- It logs small schedule tweaks too, not just headline features — including exactly the
  category of "looks cosmetic" change that broke a workflow previously
- Detection becomes a date comparison, not a content hash: any row with a date later than
  the last run's `last_checked_date` is new, full stop

Current wave: `https://learn.microsoft.com/en-us/dynamics365/release-plan/2025wave2/change-history`

This single page covers every Dynamics product in one table, so pull out just the
**Dynamics 365 Customer Service** and **Dynamics 365 Contact Center** sections (Copilot
Service workspace spans both). Each section has three tables: Features added, Release
date changed, Features removed — each with its own date column.

When the next wave's change-history page is published (2026wave1), add its URL here
alongside the current one; don't replace it until 2025wave2 has fully rolled out.

## Secondary signal: direct page fetch (for pages with no change-history table)

Some pages aren't covered by a change history table and still need a content diff. Fetch
each of these using the `microsoft_docs_fetch` tool (Microsoft Learn Docs MCP server,
`https://learn.microsoft.com/api/mcp`). If that MCP isn't connected in this session, fall
back to `web_fetch`.

1. `https://learn.microsoft.com/en-us/dynamics365/customer-service/implement/whats-new-customer-service`
2. `https://learn.microsoft.com/en-us/dynamics365/customer-service/implement/csw-overview`
3. `https://learn.microsoft.com/en-us/dynamics365/customer-service/administer/configure-copilot-features`

**Precision option for page 1 specifically:** this page's source is confirmed public on
GitHub (`MicrosoftDocs/dynamics-365-customer-engagement`, path
`ce/customer-service/implement/whats-new-customer-service.md`), which means its per-file
commit history is available as a lightweight Atom feed:
`https://github.com/MicrosoftDocs/dynamics-365-customer-engagement/commits/main/ce/customer-service/implement/whats-new-customer-service.md.atom`
Checking that feed is cheaper and more precise than fetching and diffing the full rendered
page. The GitHub source for pages 2 and 3 above hasn't been confirmed yet — check via
`microsoft_docs_fetch` (the page metadata includes a `meta-github_feedback_content_git_url`
field) before assuming the same pattern holds.

(Add more page URLs here as Gaelen identifies other pages worth tracking.)

## Known workflow dependencies (fill in before first real run)

> TODO — Gaelen: this section needs the actual rendered layout of Copilot Service
> workspace, not the app-shell HTML (that file only contains the pre-render loading
> skeleton, no sitemap/session/productivity-pane content). Capture it properly —
> Ctrl+P → Save as PDF on the fully-loaded page, or a full-page screenshot, or
> select-all + copy of the visible text — and paste it in so this list can be filled in
> with real specifics.
>
> Once populated, this becomes the checklist the skill weighs every page diff against,
> e.g.:
> - Productivity pane / Smart Assist configuration in use
> - Which experience profile(s) and session templates are active
> - Any custom scripts or macros in play
> - The specific incident where a "minor" edit broke a workflow — what changed and what broke

## Steps

1. **Fetch the change history page** for the current wave. Extract every row from the
   Customer Service and Contact Center "Features added," "Release date changed," and
   "Features removed" tables.
2. **Compare each row's date column against the stored `last_checked_date`.** Any row
   dated after it is a change to report — no filtering by whether it looks significant.
   A one-line "public preview date moved" entry gets reported exactly the same as a major
   feature addition.
3. **Fetch the secondary pages** (whats-new-customer-service, csw-overview,
   configure-copilot-features) and compare against their stored snapshots at the
   paragraph level, same no-filtering rule.
4. **For every change found, from either source:**
   - Write a plain-English summary (1–3 sentences).
   - Cross-reference against the Known Workflow Dependencies section above. If a change
     plausibly touches something listed there, say so explicitly and specifically. If it
     doesn't obviously touch anything listed, say that too — don't invent a risk that
     isn't there, but don't stay silent either.
5. **If nothing changed anywhere:** update `last_checked_date` and the page snapshots
   silently. Do not send anything.
6. **If anything changed:** compose the email (format below) and send it by POSTing to
   the notification endpoint (see Notification, below).
7. **Always** update the stored state — `last_checked_date` and the snapshots — whether
   or not an email was sent, so the next run compares against what's actually current.

## State (persistence between runs)

Two things need to survive between runs:

1. `last_checked_date` — the date used to filter change-history table rows.
2. A snapshot of each secondary page's content, for the paragraph-level diff.

Both live in `state.json` in the attached state-persistence repo (see setup below).

**First-run bootstrap guard:** if a page's stored `content` field is empty (this is the
state before any real run has happened), that page has no baseline yet. Fetch it, save
the content, but do NOT report it as a change — there's nothing to compare against yet.
Only start reporting diffs for a page once it has a non-empty stored `content` from a
previous run. The same logic applies to `last_checked_date`: on the very first run, set
it to today's date and don't retroactively report every change-history row that predates
it as new.

**Mechanics for a Claude Code Routine (cloud):** each firing is a fresh session with no
local memory of the last run, so the routine must explicitly:

1. Clone or pull the attached state repo.
2. Read `state.json`.
3. Do the fetch/compare steps above.
4. Write the updated `state.json` (new `last_checked_date`, refreshed page snapshots).
5. Commit and push the change back to the repo, e.g.:
   ```
   git add state.json
   git commit -m "Update state: checked $(date +%F)"
   git push
   ```

This requires the routine to have **write** access to the repo, not just read — confirm
this when attaching the repo during routine setup (see Scheduling, below), otherwise step
5 will silently fail and every run will re-detect the same "changes" from scratch.

## Email format

Send one email per run if any page changed, covering all changed pages — don't send a
separate email per page.

```
Subject: Dynamics 365 release note change(s) — [N] page(s) updated — [date]

Hi Rachel,

Microsoft updated the following Dynamics 365 documentation. Here's what changed and
whether it looks like it could affect your setup.

[PAGE TITLE 1] — [URL]
WHAT CHANGED
- [change 1]
- [change 2]

HOW THIS COULD AFFECT YOU
- [specific plausible impact, referencing the known workflow dependencies]
  -- or --
- No obvious link to your current workflow, but flagging since even small edits
  have caused breakage before.

[Repeat per page, if more than one changed]

— Automated Dynamics release-note watch
```

## Notification (the send step)

This skill cannot send email directly — no send-mail tool is connected. Instead, POST
the composed subject and body as JSON to the Power Automate flow endpoint below, which
has an Office 365 Outlook "Send an email (V2)" action wired to rachel.hodgson@tiltinsurance.co.uk:

```
POST [Power Automate HTTP-trigger URL — TODO: add once flow is built]
Content-Type: application/json

{ "subject": "...", "body": "..." }
```

If that endpoint isn't set up yet, stop before this step and tell Gaelen the summary is
ready but couldn't be sent, showing the drafted email in full so it can be sent manually
in the meantime.

## Scheduling

Run this as a **Claude Code Routine** (cloud-hosted, doesn't need a machine on), not a
Cowork scheduled task (which requires Desktop to be open) and not a Desktop local task
(same limitation).

There's no genuine push/webhook trigger available here — Microsoft doesn't expose one for
Learn docs, and you don't administer the repos to wire a GitHub trigger to them (see the
earlier discussion on why "onEdit" isn't achievable). But since the change-history table
makes each check cheap and precise rather than a fuzzy full-page diff, there's no reason
to only check weekly any more — daily is reasonable.

**Setup, step by step:**

1. Save the skill where Claude Code can find it:
   ```
   mkdir -p ~/.claude/skills/dynamics-release-watch
   cp SKILL.md ~/.claude/skills/dynamics-release-watch/
   ```
2. Create a private repo on your work GitHub account (single file is fine) and push the
   starter `state.json` into it (provided alongside this skill). Worth a quick check with
   whoever governs GitHub access/app installs at Tilt before relying on this — same class
   of gate you hit with the Databricks OAuth app.
3. When creating the routine and attaching this repo, make sure the access granted is
   **write**, not read-only — the routine commits the updated state back after every run
   (see State, above). Read-only access will let the routine read `state.json` fine but
   fail silently on the commit step, so every run will look like the first run.
4. In a Claude Code session (CLI or the web UI at `claude.ai/code/routines`), create the
   routine:
   ```
   /schedule daily at 8am: run the dynamics-release-watch skill
   ```
   Attach the repo from step 2, and confirm the Microsoft Learn Docs MCP connector is
   enabled for the routine.
5. Run it once manually (`Run now` in the web UI, or ask Claude in the session) before
   trusting the schedule. On this first run, confirm: a) it populates the empty `content`
   fields without emailing Rachel about false "changes" (the bootstrap guard above), and
   b) it successfully commits the updated `state.json` back to the repo — check the
   repo's commit history to be sure, don't just take the session's word for it.

If cadence ever needs adjusting (e.g. down to hourly, or up to weekly once you've seen how
noisy daily actually is in practice), edit the routine from `claude.ai/code/routines` or
ask Claude to update it from within a session — no need to recreate it from scratch.
