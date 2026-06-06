# Cluade_KB — Claude Knowledge Base (Maverick Ideas)

Version-controlled home for the knowledge Claude has accumulated working on Maverick Ideas
projects — hard-won gotchas, playbooks, integration details, and project context.

This repo is the **single source of truth**. On the primary machine, Claude's live memory folder
(`~/.claude/projects/<project>/memory/`) is a **directory junction** pointing into this repo, so the
knowledge is version-controlled *and* Claude keeps reading/writing it live — edits Claude makes to
memory show up here as changes, ready to commit.

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
Clone this repo, then point that machine's memory folder at it with a junction (PowerShell, no admin,
works across drives):

```powershell
$link   = "$env:USERPROFILE\.claude\projects\D--Maverick-Ideas-LLC-GitHub-Stock-Sentiment-App\memory"
$target = "<clone-path>\Cluade_KB\optix"
if (Test-Path $link) { Remove-Item $link -Recurse -Force }   # only if empty / safe to replace
New-Item -ItemType Junction -Path $link -Target $target
```

(The encoded folder name is the project's absolute path with separators replaced by `-`. Or just read
the files directly for reference.)

## Keeping it in sync
Because the live memory folder *is* this repo (via the junction), anything Claude writes to memory
appears here as an uncommitted change — commit periodically to keep the history. Nothing here is
secret: notes and IDs, not credentials.
