# nielsen-krug-pass

Run a usability-pass round on a UI app, evaluated against Nielsen's 10 Heuristics and Krug's *Don't Make Me Think* principles. Each round usually ships 2–5 small, focused commits on the current branch and pushes them (a final round may ship fewer when solid candidates run out); when the user asks for several rounds at once, they execute back-to-back without re-prompting.

Useful any time the user wants to keep grinding UX improvements on a project without designing a roadmap first — most often invoked as `/nielsen-krug-pass` or by saying "do N more rounds" after the first one.

---

## Operating principles

These apply to every round, every commit. Re-read this section any time before starting a round.

- **One round = one push.** Multiple commits per round are fine and often preferable; one PR can grow to dozens. Always push at the end of the round so the PR reflects what you just did.
- **Skip surfaces already fixed in this PR.** Read `git log --oneline origin/<base>..HEAD` (Step 1, with `<base>` the PR's `baseRefName`) before picking targets. Don't propose fixes that overlap commits already on the branch.
- **Real impact beats nitpicks.** A fix is worth shipping when (a) a real end user will notice it and (b) it cites a specific heuristic or principle. "The button could be 2px wider" is a nitpick. "The button reuses the destructive color of a Delete button" is a finding.
- **Prefer rebindable / dynamic values over hard-coded strings.** If the codebase has a keybindings system, status manager, or theme provider, read from it; don't repeat literals.
- **Stay on the current branch.** Don't open new PRs or branches unless the user asked. The whole point is to keep stacking commits on the one PR.
- **Autonomous mode is the default for multi-round requests.** When the user says "do N more rounds", do not ask follow-up questions between rounds; make the reasonable judgment call and move on.

## Heuristic cheatsheet

Cite by number / name when writing commit messages — it gives the rationale anchor.

**Nielsen:**
1. Visibility of system status
2. Match between the system and the real world
3. User control and freedom
4. Consistency and standards
5. Error prevention
6. Recognition rather than recall
7. Flexibility and efficiency of use
8. Aesthetic and minimalist design
9. Help users recognize, diagnose, and recover from errors
10. Help and documentation

**Krug (Don't Make Me Think):**
- Self-evident labels — the actionable thing should look obviously actionable
- Don't make me think — answer the obvious question before it's asked
- Visible options — features that exist but aren't visible don't exist
- Conventions over invention — match the platform unless there's a reason not to

---

## Steps

### 1 — Orient

Read what's on this branch already so you don't re-fix something.

```bash
# Open PR for this branch (so you know what to update at the end) and the branch it targets
gh pr view --json number,title,url,state,baseRefName

# Current branch and default-branch name (usually main) — compare them for the check below
git branch --show-current
gh repo view --json defaultBranchRef -q .defaultBranchRef.name

# Uncommitted changes to tracked files — no output means there are none
git status --short --untracked-files=no
```

If `git status` prints anything, stop before making any edits and tell the user. Step 3 stages whole files, so those changes could land in a fix's commit. The user decides whether to commit, stash, or discard them.

If `gh pr view` reports `no pull requests found for branch`, its `state` is anything other than `OPEN` (it also returns merged and closed PRs), or the current branch is the default branch, stop before making any edits. Don't commit to the default branch, and don't create a branch or PR on your own. Tell the user and ask whether to create a branch and open a PR for the round. This applies to multi-round requests too. The check runs before the first round, so asking here doesn't break the no-questions-between-rounds rule.

If `gh pr view` fails with any other error, such as `could not determine current branch` on a detached HEAD or an authentication failure, stop before making any edits and tell the user what it printed. Without the PR's number and base, the sync below can't run. Don't create a branch or PR to get past the error.

Otherwise, find the remote that hosts the PR. `gh` may look the PR up in a different repository than `origin`. In a fork clone with an `upstream` remote, it can find the PR on `upstream`. Fetching that PR number from `origin` would then get a different PR, or none.

```bash
# Each remote's name and URL
git remote -v
```

The PR's `url` from `gh pr view` has the form `https://github.com/<owner>/<repo>/pull/<number>`. Use the remote whose URL points at the same `<owner>/<repo>`, ignoring the protocol and any trailing `.git`. That remote is `<remote>` below; it is usually `origin`. If no remote matches, stop before making any edits and tell the user. Don't add a remote on your own.

Sync with that remote so you see commits pushed to the PR from elsewhere (replace `<remote>` with the remote you found, `<base>` with the `baseRefName`, and `<number>` with the PR number printed above):

```bash
# Update <remote>/<base>; the explicit refspec also works in a clone that tracks only the default branch
git fetch <remote> +refs/heads/<base>:refs/remotes/<remote>/<base>

# Fetch the PR head into FETCH_HEAD
git fetch <remote> refs/pull/<number>/head

# Print two counts: commits only on the local branch, then commits only on the PR
git rev-list --left-right --count HEAD...FETCH_HEAD
```

Act on the two counts:
- **Second count is 0:** the local branch has everything on the PR. Continue.
- **First count is 0, second is not:** the PR is ahead. Run `git merge --ff-only FETCH_HEAD`, then continue. If it refuses because of uncommitted changes, stop and tell the user.
- **Neither count is 0:** the branches have diverged. Stop before making any edits and tell the user. Don't rebase, merge, or force-push on your own.

Then list the commits on this PR:

```bash
git log --oneline <remote>/<base>..HEAD
```

Use the PR's `baseRefName` for the log range, not the default branch. A PR stacked on another branch targets that branch, and a range against the default branch would also list the parent branch's commits. The `origin/<base>` range in *Operating principles* means this one: when the PR is hosted on another remote, use `<remote>/<base>` there too.

Note:
- The PR number (you'll edit its title/body at the end).
- The set of surfaces already touched (skip them when picking targets).

### 2 — Pick a theme and surfaces

A "round" has a coherent theme — *discoverability*, *feedback*, *error prevention*, *state visibility*, *consistency*, *information scent*, etc. Pick one theme per round so the commits feel like a chapter.

Survey candidates for that theme by reading the relevant screen / view files. The goal is to find 2–5 issues that share the theme. Prefer:

- Surfaces with no feedback for failure cases ("nothing happens" is the strongest UX smell).
- Hard-coded labels that ignore a configuration / binding the app already supports.
- States that exist in code but are invisible to the end user (debug-only labels, silent toggles).
- Affordances that look the same whether or not they're actionable.

When in doubt, run a small `Explore` agent against the codebase asking it to flag candidates for the theme — but verify each candidate by reading the actual file, because Explore reads excerpts and sometimes confidently misreports.

If your session has no agent-spawning tool, survey candidates yourself instead: search for the theme's telltale patterns (e.g. hard-coded labels, empty error handlers, debug-only flags) with your search tools, then read the matching files. The same rule applies — every candidate must be confirmed in the actual file before you fix it.

### 3 — Implement one fix at a time

For each fix:

1. Make the edit. Keep diffs small — a fix should typically touch one file and 1–15 lines. If a fix sprawls, split it.
2. Read the result back if you're unsure it's right.
3. Run the project's fastest check that covers the change: its build, type-check, lint, or the tests for the file you edited. If it fails, fix the change or drop it; don't commit a broken fix. If the project has no such check, say so when you report the round.
4. Commit *before* moving to the next fix. Stage the fix's files by name (`git add <paths>`), not with `git add -A` or `git add .`, so nothing else lands in the commit. Don't batch unrelated edits into one commit.

Use the [commit message format](#commit-message-format) below.

### 4 — Push and update the PR

After the round's commits are in:

```bash
git push
```

If the push is rejected because the PR gained commits during the round, don't force-push. Keep the round's commits local, stop, and tell the user the branches have diverged. In a multi-round batch this ends the batch; the rounds already pushed still get their PR update.

Re-read the PR description (`gh pr view --json body -q .body`). When to edit it depends on how the skill was invoked:

- **Single-round invocation** (e.g. `/nielsen-krug-pass`): update the PR title to reflect the new fix count and add the new round under an `### Round N — <theme>` heading, as [PR title and body update](#pr-title-and-body-update) describes.
- **Multi-round invocation** (e.g. "do 5 more rounds"): defer the PR-body update until after the *last* round of the batch. Push between every round, but only edit the PR description once at the end. This avoids spam-editing the PR.

A round that ends with no commits adds no `### Round N` section and leaves the title's fix count alone. Report what you searched to the user instead. In a multi-round batch, the rounds before it still get their PR update; if no round in the batch has commits, skip the edit.

### 5 — Multi-round handling

When the user says "do N more rounds":

1. Decide N themes ahead of time so each round is coherent. Themes should not repeat what's already on the PR.
2. Run rounds 1 through N back-to-back.
3. After each round: commit and push. If you're tracking the rounds in a task list, mark the round's task complete; a session without a task tool can skip this.
4. After the *last* round: rewrite the PR title and add all new round sections to the body in one PR edit, as [PR title and body update](#pr-title-and-body-update) describes.
5. Do not stop between rounds to ask for direction. If you genuinely run out of high-signal candidates, say so and stop — don't pad with nitpicks.

If a round produces fewer than 2 solid candidates after honest searching, stop after that round (see [When to stop](#when-to-stop)); it counts as the last round of the batch for the PR update. Commit and push a single solid candidate if there is one, then report what you searched. "I looked at X, Y, Z; the remaining surfaces are already well-handled" is a valid round result.

---

## Commit message format

Each commit names the surface, cites the heuristic, explains why the previous state was wrong, and describes the fix. Short rationale-driven prose, not bullet lists.

```
ux: <surface> — <one-line summary of the fix>

<one paragraph: what the previous state looked like to the end user and
why that's a problem. Cite the specific Nielsen heuristic or Krug
principle in parentheses.>

<one paragraph: what the fix does. Use plain language; this is the
"why I made the change" for someone reading the PR a year from now.>

<the harness's attribution trailer, if it specifies one>
```

Don't hardcode a model name in the trailer. Use the attribution line your harness gives you, so the commit credits the model that actually wrote it. If the harness gives none, leave the line out.

Write the message to a file with your file-writing tool, then commit with `git commit -F <path-to-message-file>`. Don't pass a multi-paragraph message with shell command substitution or a heredoc; some harnesses reject that before it reaches the shell.

Write the file outside the repository, such as in your system's temp directory, so it can't end up in the PR. If your harness only allows writes inside the checkout, use a path the repo ignores, or delete the file right after the command that uses it. `git status` should show no leftover file.

PRs like Preponderous-Software/roam#358 show the subject-line and rationale cadence — read a few before writing your own if you want it. Many of those commits don't match this format: they use bullet lists in the body and a hardcoded model trailer. Where they differ from this section, follow this section.

What to avoid:
- Bullet lists in the body (use prose).
- Vague "improve X" subject lines (name the surface and the actual change).
- Citing the heuristic without explaining *why* it applies.
- Writing "fixed bug" without saying what the end user would have observed.

---

## PR title and body update

After a single round, or after the last round of a multi-round batch (see Step 4):

**Title:** `ux: usability pass — N fixes from Nielsen heuristics + Don't Make Me Think`

**Body structure:**

```markdown
## Summary
Long-running usability-pass batch. Every commit is one self-contained Nielsen / Krug fix, scoped small for easy review.

### Round 1 — <theme>
1. **<Surface>** — <one-line description> (<heuristic citation>)
2. ...

### Round 2 — <theme>
...

## Test plan
- [ ] <Observable check 1>
- [ ] <Observable check 2>
...
```

Put each new round after the last existing round and before `## Test plan`, not at the end of the body, and add the round's observable checks to the test plan. Keep everything else already in the body.

If a landmark is missing, add it without moving the text that's already there:
- **No `### Round` section yet:** put the new round at the end of the `## Summary` section.
- **No `## Summary`:** add the template's `## Summary` heading and its opening line, then the new round under it. Place them just before `## Test plan` if the body has one, otherwise after the existing text.
- **No `## Test plan`:** add the heading at the end of the body, followed by the round's checks.

Write the body to a file with your file-writing tool, placed as [Commit message format](#commit-message-format) describes, then pass it by file to preserve formatting:

```bash
gh pr edit <number> --title "<title>" --body-file <path-to-body-file>
```

Don't inline the body with shell command substitution or a heredoc; some harnesses reject that before it reaches the shell.

---

## When to stop

Stop the multi-round loop early — and tell the user — when any of these happen:

- You searched honestly and a round produced fewer than 2 candidates that cite a specific heuristic with a concrete end-user-observable problem.
- You've been re-touching the same files round after round; the project is approaching saturation for this skill.
- A fix would require behavior or design decisions you can't reasonably infer from the codebase (e.g. "should deleted items go to a trash folder?"). Those belong in a design conversation, not a polish PR.
- The local branch and the PR have diverged (Step 1 or Step 4). Resolving that is the user's call.

Don't pad. A PR with 25 grounded fixes is better than 40 fixes where the last 15 are wallpaper.

---

## Self-audit

Run this section when the skill may have drifted from reality — e.g. after the target environment changed, commands started failing, or results are consistently wrong.

1. Read this skill file from top to bottom.
2. For each command, path, or assumption, verify it is still correct:
   - Commands and flags still exist (`git branch --show-current`, `git status --short --untracked-files=no`, `git remote -v`, `git fetch`, `git rev-list --left-right --count`, `git merge --ff-only`, `git log --oneline`, `git add`, `git commit -F`, `git push`, `gh repo view`, `gh pr view`, `gh pr edit`, `gh issue create`).
   - The "operating principles" still match how the user invokes the skill.
   - The heuristic cheatsheet is still accurate (Nielsen's 10 heuristics are stable, but rewording happens).
   - The Preponderous-Software/roam#358 reference still resolves, and *Commit message format* still describes how its commits differ from the format.
3. For each problem found, write the issue body to a file with your file-writing tool, placed as [Commit message format](#commit-message-format) describes:
   ```markdown
   **Section:** <which step or section is wrong>
   **Problem:** <what is incorrect>
   **Expected behavior:** <what it should do instead>
   ```
   Then open a GitHub issue from it:
   ```bash
   gh issue create --repo dmccoystephenson/nielsen-krug-pass \
     --title "<problem summary>" \
     --body-file <path-to-body-file>
   ```
4. Report a summary: how many issues were filed, or confirm the skill is up to date.
