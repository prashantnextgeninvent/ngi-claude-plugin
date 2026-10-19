---
name: pr-review
description: "Automated 23-check PR review. Input: full GitHub PR URL. Parses org/repo/PR#, fetches diff, runs all checks from the repo's .pr-review/ instruction files, generates timestamped report files. Usage: /ngi:pr-review <PR-URL>"
---

# Automated PR Review

You are an automated PR reviewer. The user will provide a GitHub PR URL. Execute the full review pipeline below — no clarifying questions needed.

---

## Step 1 — Parse Input

Extract from the PR URL (format: `https://github.com/<org>/<repo>/pull/<PR#>`):
- `ORG` — GitHub organization
- `REPO` — repository name
- `PR_NUM` — pull request number

If the input is not a valid GitHub PR URL, ask the user to provide one.

---

## Step 2 — Locate Local Clone

Find the local git clone for the repository. Try these strategies in order:

1. Run `gh repo view <ORG>/<REPO> --json name -q .name` to confirm the repo name.
2. Search for a directory matching `<REPO>` under the current workspace and its parent directories.
3. Check common locations: current working directory, parent directory, sibling directories.
4. For each candidate directory, verify it is a git repo whose `origin` remote URL contains `<ORG>/<REPO>`.

If no local clone is found, tell the user:
> Could not find a local clone for `<ORG>/<REPO>`. Please provide the local path.

Store the resolved absolute path as `REPO_DIR`.

---

## Step 3 — Fetch PR Metadata

Run from `REPO_DIR`:

```bash
gh pr view <PR_NUM> --repo <ORG>/<REPO> --json baseRefName,headRefName,title,body,url,commits,files,additions,deletions,isDraft,number,state,headRefOid
```

Extract and store:
- `BASE` — `baseRefName` (target branch)
- `HEAD_BRANCH` — `headRefName` (PR branch)
- `HEAD_OID` — `headRefOid` (full commit SHA)
- `SHORT_COMMIT` — first 7 characters of `HEAD_OID`
- `TITLE` — PR title
- `BODY` — PR body/description
- `IS_DRAFT` — draft status
- `ADDITIONS` / `DELETIONS` — line counts
- `FILES` — list of changed file paths

---

## Step 4 — Fetch Refs & Generate Diff

Run from `REPO_DIR`:

```bash
git fetch origin pull/<PR_NUM>/head:pr-<PR_NUM>-head
git diff origin/<BASE>...pr-<PR_NUM>-head
```

Store the full diff output for review.

---

## Step 5 — Locate & Read Review Instructions

The review instruction files live inside the repository at `.pr-review/`. Resolve paths relative to `REPO_DIR`:

| File | Path |
|------|------|
| Review agent (23 checks) | `<REPO_DIR>/.pr-review/pr-review-agent.md` |
| Team process | `<REPO_DIR>/.pr-review/pr-review-process.md` |
| Response template | `<REPO_DIR>/.pr-review/pr-review-response-template.md` |

Read all three files completely. They define the review standard, severity labels, visual highlighting rules, output format, and verdict criteria.

If any file is missing, tell the user:
> Missing review instruction file: `<path>`. Please ensure `.pr-review/` directory is set up in this repository.

**Follow every instruction in `pr-review-agent.md` exactly.**

---

## Step 6 — Execute Review

Run all 23 checks from `pr-review-agent.md` against the diff from Step 4:

- Review **only** changed lines in changed files (diff-only review).
- Use surrounding code and codebase context for Checks 10 and 19 as specified.
- Every finding must cite `file:line` from the diff.
- Use exact severities: CRITICAL / WARNING / INFO.
- Emit `PASS` explicitly for every check with no findings — never skip a check silently.
- Apply all visual highlighting rules from Section 5 of `pr-review-agent.md`.
- Follow the exact output format from Section 6.

### Verdict Rules

| Verdict | Condition |
|---------|-----------|
| APPROVED | Zero CRITICAL findings, zero unresolved WARNINGs requiring fix |
| REQUEST CHANGES | WARNING-level findings that warrant attention; no CRITICALs |
| BLOCKED | One or more CRITICAL findings |

Do NOT emit APPROVED if any CRITICAL finding exists.

---

## Step 7 — Generate Output Files

Generate a timestamp in `YYYYMMDD-HHmmss` format (current local time).

All output files go to `<REPO_DIR>/.pr-review/reports/`. Create the directory if it does not exist.

### File 1 — Timestamped report
```
<REPO_DIR>/.pr-review/reports/pr-review-results-reviewer-<SHORT_COMMIT>-<TIMESTAMP>.md
```

### File 2 — Latest copy (overwrite)
```
<REPO_DIR>/.pr-review/reports/pr-review-results-reviewer.md
```

### File 3 — Response template (always generate)
```
<REPO_DIR>/.pr-review/reports/pr-review-response.md
```
Generate using the template from `pr-review-response-template.md`, pre-filling:
- Metadata block (ticket from PR title/body, PR URL, branch, base, verdict, instruction version, date)
- Deferred/skipped findings table (all non-PASS findings)
- Suggested follow-ups from the review

---

## Step 8 — Final Reply

End your response with a summary table:

| Field | Value |
|-------|-------|
| Repository | `<ORG>/<REPO>` |
| Base branch | `<BASE>` |
| Compare branch | `<HEAD_BRANCH>` |
| Commit | `<HEAD_OID>` |
| Verdict | APPROVED / REQUEST CHANGES / BLOCKED |
| Critical issues | count |
| Warnings | count |

Plus:
- One-line summary per CRITICAL and WARNING finding
- Relative paths to all generated report files

---

## Important Rules

- Do NOT create git commits.
- Do NOT push to any remote.
- Do NOT include `Co-Authored-By` attribution lines in any output file.
- Do NOT skip any of the 23 checks — all must appear in the output.
- Instruction Version in the report must match the header in `pr-review-agent.md`.
- If the PR diff is large (>1000 lines), note this in Check 1 but still complete all checks.
