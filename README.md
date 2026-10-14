## `/validate-ticket` — Staging validation walkthrough

A Claude Code skill that turns a Jira ticket (`TDIMQ-XXX`) into a step-by-step test walkthrough for the
**staging** environment (https://staging.internship.wtahosting.net). It's built for manual validation,
so anyone on the team can follow the steps, technical or not.

**What it does**
- Fetches the ticket and its comments from Jira.
- Checks the local story file (`_bmad-output/implementation-artifacts/`) for extra context, if one exists.
- Rewrites each AC as a table of steps you can reproduce in the staging UI (`# | Action | Expected result`).
- Lists the ACs that can't be validated through the UI at the end, with the reason for each. The list is ready to paste into the ticket.

It is read-only. It never edits project files and never posts to Jira. The output appears only in the chat.

**Prerequisites**
- Claude Code opened at the root of this repo.
- The **Atlassian** connector authorized in claude.ai (connector settings) with access to `willowtree.atlassian.net`.
  If the connector isn't available, the skill falls back to the `jira-acli` skill.
- Ideally, be on the `develop` branch with no uncommitted changes. The skill runs `git pull origin develop` before it starts.
  If you're on another branch or have pending changes, it asks what to do before going on.

**How to run**
In Claude Code, type:

    /validate-ticket

and paste the ticket link or key when asked. You can also pass the ticket directly:

    I want to validate ticket TDIMQ-123

**Output**
A summary of the ticket and its ACs, then the validation tables per AC, then the list of ACs that can't be tested through the UI.
The walkthrough is always in English, so it can be shared as-is.
