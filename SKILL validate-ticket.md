---
name: validate-ticket
description: Generates a staging validation walkthrough for a TDIMQ ticket — fetches the ticket and its comments from Jira via the Atlassian MCP connector (falling back to the jira-acli skill if the connector is unavailable), cross-references the local story file if one exists, and returns the ACs rewritten as objective, reproducible-by-anyone steps, plus a separate list of ACs that aren't testable through the UI with the reason why. Use when the user invokes `/validate-ticket`, says "I want to validate ticket TDIMQ-XXX", "generate test steps for ticket X", "I need to test this ticket on staging", or asks to turn a ticket's ACs into a test walkthrough.
---

# Validate Ticket → Staging QA Walkthrough

This skill standardizes manual validation of sprint tickets: it turns a ticket's ACs into an
objective, step-by-step walkthrough that anyone on the team — technical or not — can reproduce
against the staging environment. The staging UI is the only surface available for this
validation (no database, no infra, no codebase access), so the output leads with every AC that
can be exercised through the UI, rewritten as steps, and only at the very end lists the ACs that
genuinely can't be validated through the UI (ready to paste as a ticket comment).

## How to call it

Run `/validate-ticket` with no argument. If no ticket link or key is already present in the
request, **ask for the Jira ticket** (link or key, e.g. `TDIMQ-123`) before doing anything else —
don't run Step 1 yet, no point pulling `develop` before knowing which ticket to validate. Once the
ticket is given, extract the key (e.g. `https://telus-cio.atlassian.net/browse/TDIMQ-123` →
`TDIMQ-123`) and continue to Step 1.

If the request already includes a ticket link or key (e.g. "I want to validate ticket TDIMQ-123"
or a pasted Jira URL), skip the question and extract the key directly from what was given.

## The contract

- **Read-only on the codebase.** This skill never edits project files — it only reads, and at most
  updates the local `develop` branch via `git pull`.
- **Never silently switches branches or pulls** if the current branch isn't `develop` or there are
  uncommitted changes — in that case, stop and ask before continuing (avoids discarding
  in-progress work).
- **Ticket data comes from the Atlassian MCP connector** (`mcp__claude_ai_Atlassian__getAccessibleAtlassianResources`
  + `mcp__claude_ai_Atlassian__getJiraIssue`) — falls back to the `jira-acli` skill's scripts
  (`view_ticket.py` / `list_comments.py`) only if the MCP connector is unavailable, unauthenticated,
  or errors. Don't reimplement Jira REST calls directly in either case.
- **Never posts a comment to Jira automatically.** The "not testable" list is only meant to be
  copied and pasted manually wherever it's needed.
- **The test environment is always staging** (`https://staging.internship.wtahosting.net`) —
  never localhost, never production. Every step in the walkthrough assumes this environment.
- **No `cd`.** Use `git -C <repo-root> ...` (per the harness, `cd` in a compound command triggers
  a permission prompt).
- **Output goes to the chat only** — this skill never writes a file to disk.
- **The final walkthrough (Step 4) is always written in English**, regardless of the language the
  request was made in — the output is meant to be pasted into Jira / shared with the team as-is.
  This applies only to the Step 4 output; conversational replies around it (confirmations,
  clarifying questions, the Report back summary) can stay in whatever language the conversation is
  already in.

---

## Step 1 — Update the `develop` branch

Before anything else, run:

```bash
REPO=$(git rev-parse --show-toplevel)
git -C "$REPO" status --porcelain
git -C "$REPO" rev-parse --abbrev-ref HEAD
```

- **If the current branch is `develop` and the working tree is clean** → proceed directly with:
  ```bash
  git -C "$REPO" pull origin develop
  ```
- **Otherwise** (different branch and/or uncommitted changes) → **stop and ask** whether whoever
  is running the skill wants it to:
  - switch to `develop` (stashing any uncommitted changes first, if there are any), or
  - continue without updating `develop`, using the local code as-is.

  Don't assume either option — this could discard or hide in-progress work.

Note at the end of the run whether the pull happened, was skipped, or was declined, and why.

---

## Step 2 — Fetch the ticket from Jira via the Atlassian MCP connector

1. Validate the extracted ticket key against the project's format: `TDIMQ-\d+`, rejecting
   all-zero suffixes (same rule enforced in CI, `pr-jira-link.yml`:
   `/TDIMQ-(?!0+\b)\d{2,}\b/i`). If it doesn't match, ask for a correct ticket link or key.
2. Resolve the `cloudId` fresh every run — call `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources`
   (no args) and take the `id` of the resource whose `url` is `https://willowtree.atlassian.net`.
   Don't hardcode the cloudId in this file — it belongs to the connector session, not the codebase,
   and may change if the connector is reauthorized.
3. Fetch the issue with `mcp__claude_ai_Atlassian__getJiraIssue`:
   - `cloudId`: from step 2
   - `issueIdOrKey`: the ticket key
   - `fields`: `["summary","description","status","issuetype","priority","assignee","reporter","created","updated","labels","parent","comment"]`
   - `responseContentFormat`: `"markdown"` (keeps description/comments as plain text instead of ADF JSON)
4. The ticket link is always the `webUrl` field in the response — don't reconstruct it manually.
5. Read `fields.comment.comments` for the full comment thread — scope decisions, already-reported
   bugs, AC adjustments made after the ticket was created — all of this should inform the steps
   generated in Step 4.
6. **Fallback:** if the Atlassian MCP tools aren't present in this session, or a call errors with an
   auth/permission failure, fall back to the `jira-acli` skill's scripts instead of stopping:
   ```bash
   python3 ~/.claude/skills/jira-acli/scripts/view_ticket.py TDIMQ-XXX
   python3 ~/.claude/skills/jira-acli/scripts/list_comments.py TDIMQ-XXX
   ```
   In this case the ticket link is whatever `view_ticket.py` prints. Note in the final report
   (Step 5) which source was actually used.

---

## Step 3 — Cross-reference the local story (if it exists)

1. Look in `_bmad-output/implementation-artifacts/` for a story file whose name/slug matches the
   ticket (same matching pattern used in `code-review-pr`'s Step 2b: the ticket's slug usually
   appears in the filename; `_bmad-output/jira-mapping.yaml` also maps story numbers to Jira keys
   if a direct filename match isn't obvious).
2. If one exists, read its Acceptance Criteria section (Given/When/Then) as extra context — useful
   when the Jira ticket has drifted from the story.
3. If no matching file exists, proceed with only the Jira content — that's the normal path for
   tickets without a formal story, not an error.

---

## Step 4 — Assemble the response

Always in the chat, always in this order and format:

```markdown
## <TICKET> — <Ticket title>
**Link:** <the `webUrl` field from the MCP response, or `view_ticket.py`'s link if the fallback was used>

**Description:** <1-3 sentence summary, in plain language>

**ACs requested:**
- AC1: <description>
- AC2: <description>

**Relevant points from comments:** <include only if something changes how the test should be
run — a scope decision, a known bug, an AC adjustment made after creation. Omit this section if
there's nothing relevant.>

---

### Staging validation steps (https://staging.internship.wtahosting.net)

**AC1 — <short description>**

| # | Action | Expected result |
|---|--------|------------------|
| 1 | Go to ... | ... |
| 2 | Click ... | ... |

**AC2 — <short description>**

| # | Action | Expected result |
|---|--------|------------------|
| 1 | ... | ... |

---

### ACs not testable via UI/staging (paste into the ticket)
- **AC3:** <reason> — e.g. requires a direct database check / is an infrastructure requirement
  with no visible UI surface / requires access QA doesn't have on staging.
```

### Rules for writing the steps

- **Table, not prose.** Every AC's steps are a markdown table (`# | Action | Expected result`) —
  this is the format devs already use in their own manual-test comments on tickets, and it's far
  easier to scan than numbered lists with paragraph explanations.
- **One row = one atomic check.** Each row's Action and Expected cells are short phrases (a button
  name, a field name, a tab name) — not full sentences. This is the main lever for keeping a table
  short: terse cells, not verbose ones, even if that means more rows.
- **Plain language.** Every phrase names what appears on screen exactly as it appears — never
  component names, API routes, or code terms.
- **Paraphrase, don't quote.** Don't copy an error/warning message verbatim into the Expected
  column — summarize the gist in a few words (e.g. "error: value is ambiguous, needs a Parent
  Area"). Only quote the exact wording when the AC itself is specifically about the message's
  wording (rare — check before assuming).
- **Skip incidental comparisons.** Don't add a row that compares one view against another (e.g. a
  self-service view vs. the admin view of the same record) unless the AC's own wording explicitly
  requires that comparison to prove it passed. A developer's own manual-test notes on the ticket
  may include such a comparison as a nice-to-have sanity check — it's fine to leave out of the
  table if the AC doesn't call for it.
- **Collapse repeated flows instead of duplicating them.** When the exact same row sequence
  applies again for a second role, tab, or entity (e.g. the same import flow on the Mentors tab and
  the Interns tab), don't write a second full table — add one row: "Same as rows 1–4 above, on the
  <other tab/role>" — and only break that out into real rows if its expected result differs.

### Decision rule: testable vs. not testable

Whoever runs this walkthrough only has the staging UI to validate against — no database access,
no server/infra access, no codebase. That means the default bias is toward the testable bucket:
every AC that has any observable footprint on screen belongs in the main walkthrough, in AC order,
before the not-testable list is written at all. An AC only moves to the separate "not testable"
list if there's genuinely no way to observe the result through the UI on staging — for example:
- It requires a direct database or internal-log check.
- It's an infrastructure setting (e.g. a CloudWatch alarm, IAM, an env var) with no visible
  reflection on screen.
- It only happens in production (e.g. behavior tied to a production domain/certificate).

When there's reasonable doubt about whether something is testable, **don't drop it** — include it
in the normal walkthrough with a `⚠️ validate with caution: <note>` flag instead of moving it to
the separate list. The not-testable list always comes last, after every testable AC's steps, so it
reads as the leftover exception list, not the main content.

---

## Report back

Close with a short chat summary:

- Whether the `git pull` happened, was skipped, or was declined (and why).
- Which Jira source was used: the Atlassian MCP connector (default) or the `jira-acli` fallback
  (and if the fallback fired, why).
- How many ACs ended up in the testable walkthrough vs. the not-testable list.
- Whether a local story file was found and used as extra context, or only Jira was used.
