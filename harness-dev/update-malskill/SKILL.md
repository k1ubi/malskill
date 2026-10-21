---
name: update-malskill
description: >
  Sync the k1ubi/malskill GitHub fork with AeonDave/malskill upstream, pull the latest
  changes to the local clone at ~/malskill, and reinstall all 336 skills into
  ~/.claude/skills using claude-sonnet-4-6. Trigger this skill whenever the user says
  "update malskill", "update malskills", "sync malskill", "check malskill updates",
  "refresh my skills", or asks whether their local skills are up to date with the repo.
---

# update-malskill

Keeps the `k1ubi/malskill` fork and the locally installed Claude skills in sync with
the `AeonDave/malskill` upstream.

## Constants

```
UPSTREAM_REPO  = AeonDave/malskill
FORK_REPO      = k1ubi/malskill
LOCAL_CLONE    = ~/malskill
SKILLS_DEST    = ~/.claude/skills
INSTALL_MODEL  = claude-sonnet-4-6
```

## Workflow

### Step 1 — Compare upstream vs fork

Use the GitHub API (no auth needed for public repos) to fetch the latest commit SHA
on `main` for both repos:

```bash
UPSTREAM_SHA=$(curl -s "https://api.github.com/repos/AeonDave/malskill/commits/main" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['sha'][:12])")

FORK_SHA=$(curl -s "https://api.github.com/repos/k1ubi/malskill/commits/main" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['sha'][:12])")

echo "Upstream: $UPSTREAM_SHA  Fork: $FORK_SHA"
```

If `UPSTREAM_SHA == FORK_SHA`, the fork is already in sync — report this and stop.

### Step 2 — Sync the fork on GitHub

If behind, trigger GitHub's upstream sync endpoint (requires auth):

```bash
curl -s -X POST \
  -H "Authorization: token $(git credential fill <<< 'protocol=https\nhost=github.com' \
      2>/dev/null | grep password | cut -d= -f2)" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/k1ubi/malskill/merge-upstream" \
  -d '{"branch":"main"}' | python3 -c "
import json,sys
d=json.load(sys.stdin)
print(d.get('message','done'), '—', d.get('merge_type',''))
"
```

If the API call fails (no stored credential), fall back to the git approach:

```bash
cd ~/malskill
git remote add upstream https://github.com/AeonDave/malskill.git 2>/dev/null || true
git fetch upstream main
git merge upstream/main --ff-only
git push origin main
```

### Step 3 — Update the local clone

If `~/malskill` exists, pull:

```bash
git -C ~/malskill pull --ff-only origin main
```

If it doesn't exist yet (first run), clone from the fork:

```bash
git clone https://github.com/k1ubi/malskill.git ~/malskill
```

### Step 4 — Reinstall skills

Spawn a subagent with model **claude-sonnet-4-6** for the install step (it's a
mechanical file-copy operation — no need for a heavier model):

**Subagent prompt:**
```
Run this command and report any errors:
  bash ~/malskill/install.sh --all --format folder --layout group \
       --destination ~/.claude/skills
The install script validates each skill before copying it. Report the final
line ("Install complete" or any error). Do not print the full output — just
the summary and any skills that failed validation.
```

### Step 5 — Report

After the subagent completes, summarise:

- Whether the fork was already in sync or needed a sync
- New commit SHA (upstream) vs old (fork)
- Number of skills installed
- Any validation failures
- Elapsed time

## Error handling

| Situation | Action |
|---|---|
| Fork is already in sync | Report "already up to date", stop — do not reinstall |
| GitHub API returns 404 | Ask user to confirm fork URL |
| `git push` fails (auth) | Show the exact error; suggest `git credential approve` or checking `~/.git-credentials` |
| `install.sh` validation error | Report the failing skill name; continue with the rest |
| `~/malskill` is dirty (uncommitted changes) | Stash before pulling, restore after |

## Notes

- The `~/malskill` local clone is the persistent reference copy; `/tmp/malskill` from
  the original install session should be considered stale and is not used here.
- Skills installed into `~/.claude/skills/synced/` are account-managed — do not
  overwrite them. Only touch `~/.claude/skills/{ai,behaviours,coding,...}`.
- Re-installing over existing skills is safe: `install.sh` removes and re-copies.
