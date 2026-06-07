---
name: optix-learnings
description: "OPTIX session playbook — hard-won learnings/gotchas (resolved uncertainties) + shipped wins, so future sessions don't re-derive them"
metadata: 
  node_type: memory
  type: project
  originSessionId: 8f0f0737-d25e-4bc0-9ca8-a19b3fab1e11
---

# OPTIX — Hard-won learnings & wins (May 2026)

## Gotchas we hit and SOLVED (don't re-derive)

1. **Jira API token must be CLASSIC/unscoped.** Scoped tokens 401 everything with `WWW-Authenticate: OAuth realm=...`. Diagnosis: that header on a 401 = wrong token type. Token lives in Windows User env `JIRA_API_TOKEN` + `~/.config/.jira/credentials.env`.

2. **User env vars don't reach running processes; tool shells don't load profiles.** The Bash/PowerShell tools run non-interactively (no `~/.bashrc` / `$PROFILE`), and a stale harness never inherits newly-set User env. Fix = activation scripts: `source ~/.config/.jira/activate.sh` (or `. activate.ps1`) before any `jira`/Jira-API call. Fresh sessions inherit it; stale ones need the source. Documented in global `~/.claude/CLAUDE.md`.

3. **`jira init` is destructive — never re-run it.** Config (`~/.config/.jira/.config.yml`) is complete + backed up `.config.yml.bak` + set read-only as a guard. Restore: `Copy-Item .config.yml.bak .config.yml -Force`.

4. **Team-managed Jira = board columns & workflow statuses are UI-ONLY.** No public REST API to add a workflow status/column or edit the simplified workflow (greenhopper endpoints 404). "In Review" required a UI add (Project settings → Board → Columns); it reused the pre-created status id 10011 (no duplicate). Once added, transitions appear (id 2 → In Review) and jira-sync auto-uses it.

5. **Sprints ≠ Versions.** Sprints = neutral time-boxes (names MUST be <30 chars; "Sprint 1/2/3"). Versions = Releases (v1.1/v1.2/v2.0) with `fixVersion` per ticket. Do NOT name sprints by version — a release spans multiple sprints. (We initially mis-named sprints by version and corrected it.)

6. **Workflow-tool agents must use a UNIQUE temp file each.** A shared `$TEMP\x.json` across parallel agents races → clobbered payloads (caused 4 duplicate + 4 missing tickets on the first bulk-create). Use `optix-story-$idx.json` per agent.

7. **Claude model migration (use the `claude-api` skill).** Opus 4.7/4.8 REMOVE `temperature`/`top_p`/`top_k`/`budget_tokens` (400). The `aiSignal` callable sends `temperature`, so Opus stays on **4.6**; Sonnet went to **claude-sonnet-4-6** (keeps temperature). Model IDs are bare aliases — NO date suffixes. Always validate the exact model string live before deploy.

8. **Cloud Functions deploy is SEPARATE from the Android/Play release.** No functions-deploy in CI (Bitrise ships only the app). Backend goes live via manual `firebase deploy --only functions --project optix-2ddbc`. firebase CLI is authed locally; ANTHROPIC_API_KEY readable via `firebase functions:secrets:access`.

9. **`/rest/api/3/search/jql` doesn't return `total`** like the old search. Use per-issue GETs or agile endpoints for counts; use `urllib.parse.quote` for JQL.

10. **Bitrise secrets via API:** auth `Authorization: <PAT>` (PAT in User env `BITRISE_API_TOKEN`); app "OPTIX Android" slug `3b0f60bb-17ae-446d-94cc-38de203f2db9`; `POST /v0.1/apps/{slug}/secrets {name,value,is_protected,...}`.

11. **Remote scheduling (claude.ai routines)** had a transient outage ("try again in a few minutes"); the `mcp__scheduled-tasks__*` backend recovered after. Probe with `list_scheduled_tasks` before assuming it's down.

## Diagnosing Bitrise CI failures (learned 2026-05-31, OPTIX-24 / PR #37)
- **Read the log via API:** `GET https://api.bitrise.io/v0.1/apps/{slug}/builds/{buildSlug}/log` → follow `expiring_raw_log_url`, strip ANSI (`\x1b\[[0-9;]*m`). The summary-table row `| x | <Step> (Failed) |` names the failing step. Build slug is in the GitHub check-run URL (`gh pr checks <PR>`).
- **Stale process-token gotcha (instance of #2):** in an already-running session, process `$env:BITRISE_API_TOKEN` can be the OLD PAT (401 on `/me`) after a mid-session refresh (#50). Read the User-scoped value instead: `[Environment]::GetEnvironmentVariable('BITRISE_API_TOKEN','User')`.
- **pr-check vs debug-build asymmetry:** the `pr-check` workflow (PR trigger) does a TEST-MERGE of the PR branch into its target (develop) during git-clone; `debug-build`/push checks out the branch HEAD directly. So **push green + pr red on the SAME commit ⇒ the branch conflicts with develop** ("Automatic merge failed" in git-clone), NOT a code bug. Fix: merge develop into the branch, resolve, push (don't chase phantom lint/test failures).
- **source-checks.js runs in `_setup` BEFORE lint/tests** (emoji guard, committed-secret guard, firestore.rules structure). Local `testDebugUnitTest` + on-device `/verify` do NOT run it, so emoji glyphs (e.g. `✓`/`✗` in Compose status labels) pass locally then fail Bitrise fast (~1–2 min). **Run `node scripts/source-checks.js` before pushing any PR** — especially watcher-generated Compose UI. OPTIX-24 hit exactly this.
  - **Now automated** (PR #41): a `.githooks/pre-push` hook runs source-checks on every push — enable per clone with `pwsh scripts/install-git-hooks.ps1` (sets `core.hooksPath`); bypass with `git push --no-verify`. `/watch-inprogress` also runs source-checks before opening a PR.

## Shipped wins
- Jira workspace: 4 epics + 23 stories, full Agile fields; board (id 3) with 4 columns incl wired **In Review**; Releases v1.1/v1.2/v2.0 with fixVersions on all tickets; Sprint 1 ACTIVE, Sprints 2/3 future, v2.0 in backlog.
- CI automation (PRs #24/#25/#26): `scripts/jira-sync.js` + `.github/workflows/jira-sync.yml` + `bitrise.yml` steps; JIRA_* secrets set on BOTH GitHub and Bitrise; verified live end-to-end (push→PR→merge auto-drove tickets).
- OPTIX-11 model bump: validated live → merged → **deployed** to prod. OPTIX-16 resolved (admin feature). OPTIX-31 (CI) Done.
- Jira CLI usable by any agent on this machine (activation scripts + global CLAUDE.md note).
- Repo `/watch-inprogress` slash command + interim session watcher loop (works In-Progress tickets → PR).

## In-Progress watcher — `/watch-inprogress` (run in any session in this repo)
`.claude/commands/watch-inprogress.md`: implements each In-Progress ticket, then **closed-loop verifies every acceptance criterion on the attached Android device** (build+install+drive via the `/verify` skill; screenshot/log/test evidence; iterate up to 3×), and only opens a PR for verified work — skips+flags any AC it can't prove. Targets a phone via `adb -s <serial>`: prefer **SM-S918U (Galaxy S23)**; avoid Fire TV (10.1.10.77) + tablet (SM-X230); serials change, detect at runtime.
- The session-bound interim loop was **retired** (CronDelete 80940c1c) on 2026-05-31. Always-on routine **deferred** per user. For durable/offline, set up a scheduled routine via the `schedule` skill (needs Jira creds in the routine env).

## CI/CD onboarding + access audit kit (to fix the EMPLOYER's broken pipeline)
- **Context:** the user mirrored their employer's Jira→Android→GitHub→Bitrise CI/CD as OPTIX (works). The employer's is broken; this kit diagnoses + fixes it. Employer systems are large/shared (50+ engineers, many projects) — so READ-ONLY safety is paramount. User has GitHub + Bitrise admin; Jira privilege TBD (the audit reveals it).
- **`scripts/setup-cicd.ps1`** (committed PR #33; also in USB bundle v1.1.0): one-pass access audit + onboarding across Jira/GitHub/Bitrise/Firebase. **`-AuditOnly` = strictly read-only** — writes nothing local OR remote, no create/edit/transition/deploy, queries NO tickets, scoped to the one project/repo/app passed. Reports per-capability access incl. Jira perms (CREATE/EDIT/TRANSITION/ADMINISTER_PROJECTS/MANAGE_SPRINTS). Full mode (no flag) writes LOCAL CLI config only (reuses setup-jira-cli.ps1). Parameterized: -JiraServer/-JiraProject/-JiraEmail/-GitHubRepo/-BitriseApp/-FirebaseProject. Headless: pre-set the 4 env vars + CIBOOT_NONINTERACTIVE=1.
- **4 credentials:** Jira API token (classic), Bitrise PAT, GitHub (gh login or PAT), Firebase (firebase login or CI token). BITRISE_API_TOKEN == the Bitrise PAT (same value, env-var name). Bitrise PATs can EXPIRE (one did mid-session 2026-05-31; refreshed).
- **Portable plugin bundle** at `D:\optix-plugin` (shared D: drive; **v1.2.0**): marketplace `maverick-tools`, plugin `optix` → command `/optix:watch-inprogress` + both setup scripts (`setup-cicd.ps1`, `setup-jira-cli.ps1`). Install: `/plugin marketplace add D:\optix-plugin` → `/plugin install optix@maverick-tools` → `/reload-plugins`. The plugin watcher was **retargeted for the employer**: queries `project = SMT`, works only keys matching `(?i)^SMT-\d{5}$` (SMT- + exactly 5 digits), branches off the repo's default branch, and does NOT assume the OPTIX jira-sync Action exists. (The OPTIX repo keeps the loose `/watch-inprogress`, still OPTIX-targeted.) Other copies: `C:\Users\TCarter\optix-plugin` (edit copy), `G:` USB (stale v1.1.0).
- **OPTIX access baseline (2026-05-31): all green** — Jira full admin, GitHub admin, Bitrise OK, Firebase OK. Use as the reference when auditing the employer tenant.
- **DEFERRED phase-2 (do after the employer audit is green):** set Bitrise to read `bitrise.yml` from the repository (app config source = repo). This is a deliberate WRITE — NOT part of the read-only audit.

## GitHub merge-commit convention (user preference, 2026-05-31)
Repo merge settings set so a PR merge uses **PR title** as the commit title and the **PR body (extended description)** as the commit message: `merge_commit_title=PR_TITLE`, `merge_commit_message=PR_BODY` (+ same for squash). Set via `gh api -X PATCH repos/<owner>/<repo> -f merge_commit_title=PR_TITLE -f merge_commit_message=PR_BODY -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=PR_BODY`. **When merging with `gh pr merge`, do NOT pass `--subject`/`--body`** — let the repo default apply. Same one-liner applies this to any other repo (e.g. the employer's).

## Knowledge base location (final, 2026-06)
This memory IS version-controlled: `~/.claude/projects/D--Maverick-Ideas-LLC-GitHub-Stock-Sentiment-App/memory` is a **directory junction** → `D:\Maverick Ideas LLC\GitHub\Claude_KB\optix` (repo `MaverickIdeas/Claude_KB`, branch `main`). Edits Claude makes to memory appear as git changes in Claude_KB — commit periodically. Earlier KB repos `Cluade_KB` and `OPTIX_Claude_KB` are **superseded/orphaned** (duplicate copies; safe to delete). To wire a fresh machine: clone Claude_KB, then junction its `optix/` into that machine's memory path (see `Claude_KB/README.md`).
