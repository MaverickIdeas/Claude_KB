# Cluade_KB — Claude Knowledge Base (Maverick Ideas)

Version-controlled home for the knowledge Claude has accumulated working on Maverick Ideas
projects — hard-won gotchas, playbooks, integration details, and project context.

These are the **canonical, reviewable copies**. Claude's live per-machine memory lives at
`~/.claude/projects/<project>/memory/`; it's synced here so the knowledge is version-controlled,
shareable, and travels to other machines / Claude accounts.

## Contents

### `optix/` — OPTIX Android app + its Jira ↔ GitHub ↔ Bitrise ↔ Firebase CI/CD
- **`MEMORY.md`** — index of the OPTIX memory files
- **`project_overview.md`** — stack, architecture, CI/CD, version, known gaps
- **`jira_integration.md`** — Jira workspace + CI automation: project/board/field IDs, `jira-sync.js`
  bridge, secrets, and the Jira CLI activation requirement
- **`optix_learnings.md`** — resolved gotchas (token type, env/activation, team-managed UI-only
  limits, sprints≠versions, model migration, functions deploy), shipped wins, and the
  `/watch-inprogress` watcher + portable CI/CD onboarding kit

## Using it on another machine / account
Clone this repo, then copy the relevant files into that machine's
`~/.claude/projects/<project>/memory/` so Claude loads them as memory — or just read them for
reference.

## Keeping it in sync
Treat the local `.claude` memory as the working copy; export changes here (re-copy the files)
to version them. The local copies must remain in `.claude` for Claude's auto-memory to work.
