---
description: Read-only Pull request review against its Jira ticket and the repo's own conventions
argument-hint: Pull request link or number
allowed-tools: Read, Glob, Grep, Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr checks:*), Bash(git diff:*), Bash(git --no-pager diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*), Bash(git branch:*), Bash(curl:*)
---

You are running a **read-only** code review of a Pull request against its Jira ticket and the
conventions of the repository it belongs to. Gather inputs once, then run to completion without
stopping for confirmations, and end with the report.

## Absolute rules (never violate)
- **Never write to git or GitHub.** No commit, push, add, checkout, branch, merge, rebase, `gh pr
  comment/review/merge/edit`, and no file edits. Read-only git/`gh`/`curl` only.
- **Never modify any file** — you only read and report. The ONE exception: on first-time Jira setup you
  may write the user's credential entry to their home `_netrc` / `.netrc` (outside the repo). Never
  write to anything inside the repository.
- **Never print the Jira API token** into the conversation, and never read the `_netrc` / `.netrc` file
  back into the conversation.
- Do **not** run builds/tests yourself — if coverage needs confirming, ask the user to run them.

## Step 0 — Gather inputs (ask ONCE, up front — then do not ask again)

Always write **"Pull request"** in full in anything shown to the user (the ask, the summary, the
findings) — do not use the abbreviation "PR". (`gh pr ...` commands stay as-is.)

The only REQUIRED input is the **Pull request link/number** (from `$ARGUMENTS`). The Jira ticket is
**derived from the Pull request** (Step 1) — do NOT ask the user for a Jira ticket link.

**Do NOT guess the Pull request.** Never infer it from the current local git branch, recent commits, or
`git status` (the only exception: the user explicitly says "current branch").

- **If the Pull request link is missing:** ask one plain line — *"Please paste the Pull request link."*
  — and wait. Do not run any command until you have it.

After this step, run autonomously to the end — no further prompts (except first-time Jira setup).

## Step 1 — Find the Jira key and host, then read the ticket (authoritative scope)

**Key**, in this order:
1. A ticket link passed in `$ARGUMENTS` — use its trailing key.
2. Otherwise `gh pr view {NUMBER} --json title,body,headRefName` — take the first match of
   `[A-Z][A-Z0-9]+-[0-9]+` in the title, body, or head branch name.

**Host**, in this order:
1. The host of a ticket link passed in `$ARGUMENTS`.
2. A `https://<something>.atlassian.net/browse/...` link in the Pull request body.
3. A `Jira host:` line in the repository's `CLAUDE.md`.
4. Otherwise ask the user once for the Jira host (e.g. `yourcompany.atlassian.net`).

If no key is found, **skip this step and the scope-drift check in Step 3** — review purely against the
repo's conventions and note in the report that no Jira ticket was linked. Do not ask the user for one.

With a key and host, fetch the ticket read-only. **The command must contain NO shell variable
(`$...`)** — use a fully literal URL and let `curl` read the credentials from the `_netrc` file, so the
token never appears in the command.

- **Credentials live in the user's home `_netrc`** (`%USERPROFILE%\_netrc` on Windows, `~/.netrc`
  elsewhere), one entry per Jira host:
  ```
  machine <jira-host>
  login <jira-email>
  password <jira-api-token>
  ```
  - **First-time setup (only if the fetch returns 401 / no credentials):** ask the user once for their
    Jira email + API token, append the entry to their home `_netrc`, then continue.
- **Fetch** — literal host and key, no `$`:
```
curl -s -n "https://{HOST}/rest/api/3/issue/{KEY}?fields=summary,description,status,issuetype,comment"
```
Read the description, acceptance criteria, and **all comments** (comments often override the
description). Note the issue type (bug vs feature) — it drives the root-cause and ripple checks.

## Step 2 — Learn the repo, then get the diff

**Repo conventions.** Read the repository's `CLAUDE.md` (root and any in the changed folders), and a
review guide if one exists (`REVIEW.md`, `.claude/CODE_REVIEW_GUIDE.md`, `docs/review*`). These define
the structural checks for this repo — its architecture, layering rules, naming, required registrations,
and its known silent failures. Treat them as the checklist in Step 3.

**Diff.** Run these as **plain single commands — no pipes (`|`), no redirects (`2>&1`), no
`head`/`tail`** — so they match the allowlist and don't trigger permission prompts.

- `gh pr diff {NUMBER}`
- `gh pr view {NUMBER} --json title,body,author,baseRefName,headRefName,state,files`

**Skip** vendored and generated content: third-party libraries (`node_modules`, `vendor`, `lib`
folders holding external packages), lock files, minified files, build output, and generated code —
unless the Pull request deliberately changes them.

## Step 3 — Review

Apply seven lenses — **Correctness, Edge cases, Style, Performance, Architecture & Layering, Security,
Code smells** — plus every structural rule the repo's `CLAUDE.md` / review guide defines. Also check:
null/empty handling, error paths, date/time zones, numeric precision and rounding, transactions for
multi-entity writes, authorization on every new entry point, input validation, and no secrets/PII in
logs. Compare behaviour against a sibling feature already in the codebase that follows the pattern.

**Code smells** to flag (usually 🟡 Minor, tagged `[Smell]`): duplicated/copy-paste logic that should
be shared, over-long or deeply-nested methods, handlers doing too much, magic numbers/strings, long
parameter lists or boolean-flag params, dead/unreachable code, repeated if/else a guard clause would
simplify.

**Evidence, not names.** Every finding must rest on code you actually read — the deciding line, not a
guess from a type, method, or file name. If a finding depends on a library's behaviour, read that
library's source or docs in the repo before claiming it.

**Root cause (bug tickets).** When the Jira issue type is Bug, check whether the change fixes the cause or
only patches the symptom. If it is a symptom patch, raise it as a finding and name the real cause.

**Flag the class, not the instance.** When a finding is a pattern (missing authorization, unchecked null,
missing cancellation token, unvalidated input …), list every occurrence in the diff in that one
finding — not just the first one you hit.

**Pre-existing vs introduced.** Decide for each finding whether this Pull request introduced it. Bugs in
code the diff touches but did not cause go in the **Pre-existing** group, never in Blockers/Minor.

**Verification pass (before writing the report).** Re-open the deciding code for every candidate finding
and confirm it is real. Drop anything you cannot confirm from the code — do not hedge it, do not report
it as "possible" or "might".

Mandatory, always included in the output:
- **Scope drift (strict):** flag Pull request changes not covered by the ticket, and ticket acceptance
  criteria with no matching code/test.
  - **No acceptance criteria?** Show a 🔴 "Acceptance criteria missing" line under the Summary. Still
    review (infer scope from title/description/comments, or conventions if empty). Never invent criteria.
- **Unit test coverage:** state OK or MISSING for new/updated/deleted code; if MISSING, list exactly
  what lacks a test.
- **Ripple check:** whether the same bug/feature/improvement likely exists elsewhere in the codebase.

## Step 4 — Report

Keep the **top** short, but give the findings room to breathe — do NOT produce a dense wall of text.
Use Markdown headings, blank lines between every finding, and short labelled sub-lines. Format exactly
like this:

```
**Summary:** <N blockers, N minor, N pre-existing> — <1–2 sentences: verdict + biggest concern.>
             If there is nothing to fix, say exactly that: "0 findings — nothing to fix."

🔴 **Acceptance criteria missing**   ← only if the ticket has no AC; omit otherwise.

**✅ Checked, no issues:** <terse comma list of areas verified — names only. NO explanation.>

**🧪 Test coverage:** `100%`   ·OR·   MISSING — <exactly what lacks a test>

**🔁 Ripple check:** same issue in <files/areas>   ·OR·   none found.

---

### 🔴 Blockers
(silent failures, wrong authorization, data loss, security — omit the heading if none)

**1. [Lens] `File.ext:123` — <one-line title>**
- What: <what is wrong>
- Why: <impact>
- Fix: <suggested alternative / sibling to mirror>

### 🟡 Minor / Style
(cosmetic, conventions, low-risk — omit the heading if none)

**2. [Lens] `File.ext:88-95` — <title>**
- What / Why / Fix ...

### ⚪ Pre-existing
(bugs in touched code that this Pull request did not introduce — not the author's fault; omit if none)

**3. [Lens] `File.ext:40` — <title>**
- What / Why / Fix ...
```

Rules:
- **Be terse everywhere.** Short lines, no filler. Say the point and stop.
- **Only explain problems.** For anything that is fine, a terse mention is enough (a name in the
  "Checked, no issues" list, or a number).
- **Test coverage:** if adequate, just the number; if not, list exactly what is untested.
- **Always cite the line** — every finding's title must be `` `File.ext:LINE` `` (or `:START-END`), taken
  from the diff's new-file line numbers. For something missing, write `` `File.ext` (missing) `` and say
  where it should go.
- One blank line between every finding and between sections.
- Number findings continuously across groups (1, 2, 3 …).
- **Cap Minor / Style at 5.** Keep the 5 most useful; if more were found, end the group with one line:
  "+N more minor items not listed." Blockers and Pre-existing are never capped.
- Each finding: bold title line, then the `What / Why / Fix` sub-bullets.

Do not post anything to GitHub or Jira. Deliver the report in the conversation only.
