# ticketlens

A Claude Code plugin that reviews a GitHub Pull request against its Jira ticket and the repository's own conventions. The review is read-only: it never edits files, never pushes, and never posts to GitHub or Jira. The report appears only in your Claude Code session.

## What it does

- Pulls the Jira key from the Pull request title, body or branch, then reads the ticket and all its comments.
- Reads the repo's `CLAUDE.md` and any review guide (`REVIEW.md`, `.claude/CODE_REVIEW_GUIDE.md`) and uses them as the checklist.
- Reviews the diff through seven lenses: correctness, edge cases, style, performance, architecture, security and code smells.
- Requires evidence: every finding must cite the line that proves it, and findings it can't confirm are dropped.
- On bug tickets, checks whether the fix addresses the root cause or only the symptom.
- Reports each pattern once, listing every place it appears.
- Keeps pre-existing bugs separate from problems the Pull request introduced.
- Always reports scope drift against the ticket, unit test coverage, and a ripple check (whether the same issue exists elsewhere in the codebase).

## Install

```
/plugin marketplace add oii-nasif/ticketLens
/plugin install ticketlens@ticketlens
```

## Requirements

- The [GitHub CLI](https://cli.github.com/) (`gh`), logged in to an account that can read the repository.
- `curl` on your PATH.
- Optional: a Jira Cloud account. Without one, the review uses only the repo's conventions.

## Jira setup

Credentials are read from your home `_netrc` file (`%USERPROFILE%\_netrc` on Windows, `~/.netrc` on macOS and Linux), so the token never shows up in commands or chat:

```
machine yourcompany.atlassian.net
login you@yourcompany.com
password <jira-api-token>
```

If the entry is missing, the first run asks for your email and token and writes the entry for you.

The command finds the Jira host from a ticket link you pass in, from an `atlassian.net` link in the Pull request body, or from a line like this in the repo's `CLAUDE.md`:

```
Jira host: yourcompany.atlassian.net
```

## Usage

```
/ticketlens:code-review https://github.com/<owner>/<repo>/pull/123
```

## Getting better reviews in your repo

The command has no project-specific rules built in. It learns them from the repo, so the more your `CLAUDE.md` or `REVIEW.md` spells out, the sharper the review:

- Layering rules and where each kind of code belongs.
- Registrations that must stay in sync, and that break silently when they don't.
- Naming and error-handling patterns.
- Paths the review should skip.
