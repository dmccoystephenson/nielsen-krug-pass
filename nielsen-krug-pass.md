# nielsen-krug-pass

Run a usability-pass round on a UI app, evaluated against Nielsen's 10 Heuristics and Krug's *Don't Make Me Think* principles. Each round ships 2–5 small, focused commits on the current branch and pushes them; when the user asks for several rounds at once, they execute back-to-back without re-prompting.

Useful any time the user wants to keep grinding UX improvements on a project without designing a roadmap first — most often invoked as `/nielsen-krug-pass` or by saying "do N more rounds" after the first one.

---

## Operating principles

These apply to every round, every commit. Re-read this section any time before starting a round.

- **One round = one push.** Multiple commits per round are fine and often preferable; one PR can grow to dozens. Always push at the end of the round so the PR reflects what you just did.
- **Skip surfaces already fixed in this PR.** Read `git log --oneline <base>..HEAD` before picking targets. Don't propose fixes that overlap commits already on the branch.
- **Real impact beats nitpicks.** A fix is worth shipping when (a) a real player will notice it and (b) it cites a specific heuristic or principle. "The button could be 2px wider" is a nitpick. "The button reuses the destructive color of a Delete button" is a finding.
- **Prefer rebindable / dynamic values over hard-coded strings.** If the codebase has a keybindings system, status manager, or theme provider, read from it; don't repeat literals.
- **Stay on the current branch.** Don't open new PRs or branches unless the user asked. The whole point is to keep stacking commits on the one PR.
- **Autonomous mode is the default for multi-round requests.** When the user says "do N more rounds", do not ask follow-up questions between rounds; make the reasonable judgment call and move on.

## Heuristic cheatsheet

Cite by number / name when writing commit messages — it gives the rationale anchor.

**Nielsen:**
1. Visibility of system status
2. Match between system and real world
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
# Default-branch ref (usually main)
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)

# Commits on this PR
git log --oneline origin/$DEFAULT_BRANCH..HEAD

# Open PR for this branch (so you know what to update at the end)
gh pr view --json number,title,url
```

Note:
- The PR number (you'll edit its title/body at the end).
- The set of surfaces already touched (skip them when picking targets).

### 2 — Pick a theme and surfaces

A "round" has a coherent theme — *discoverability*, *feedback*, *error prevention*, *state visibility*, *consistency*, *information scent*, etc. Pick one theme per round so the commits feel like a chapter.

Survey candidates for that theme by reading the relevant screen / view files. The goal is to find 2–5 issues that share the theme. Prefer:

- Surfaces with no feedback for failure cases ("nothing happens" is the strongest UX smell).
- Hard-coded labels that ignore a configuration / binding the app already supports.
- States that exist in code but are invisible to the player (debug-only labels, silent toggles).
- Affordances that look the same whether or not they're actionable.

When in doubt, run a small `Explore` agent against the codebase asking it to flag candidates for the theme — but verify each candidate by reading the actual file, because Explore reads excerpts and sometimes confidently misreports.

### 3 — Implement one fix at a time

For each fix:

1. Make the edit. Keep diffs small — a fix should typically touch one file and 1–15 lines. If a fix sprawls, split it.
2. Read the result back if you're unsure it's right.
3. Commit *before* moving to the next fix. Don't batch unrelated edits into one commit.

Use the [commit message format](#commit-message-format) below.

### 4 — Push and update the PR

After the round's commits are in:

```bash
git push
```

Re-read the PR description (`gh pr view --json body -q .body`) and decide whether to update now or wait until the user asks for the next round.

- **Single-round invocation** (e.g. `/nielsen-krug-pass`): update the PR title to reflect the new fix count and append the new round under an `### Round N — <theme>` heading.
- **Multi-round invocation** (e.g. "do 5 more rounds"): defer the PR-body update until after the *last* round of the batch. Push between every round, but only edit the PR description once at the end. This avoids spam-editing the PR.

### 5 — Multi-round handling

When the user says "do N more rounds":

1. Decide N themes ahead of time so each round is coherent. Themes should not repeat what's already on the PR.
2. Run rounds 1 through N back-to-back.
3. After each round: commit, push, mark the round's task complete.
4. After the *last* round: rewrite the PR title and append all new round sections to the body in one PR edit.
5. Do not stop between rounds to ask for direction. If you genuinely run out of high-signal candidates, say so and stop — don't pad with nitpicks.

If a round produces zero solid candidates after honest searching, ship that observation. "I looked at X, Y, Z; the remaining surfaces are already well-handled" is a valid round result and signals it's time to stop.

---

## Commit message format

Each commit names the surface, cites the heuristic, explains why the previous state was wrong, and describes the fix. Short rationale-driven prose, not bullet lists.

```
ux: <surface> — <one-line summary of the fix>

<one paragraph: what the previous state looked like to the player and
why that's a problem. Cite the specific Nielsen heuristic or Krug
principle in parentheses.>

<one paragraph: what the fix does. Use plain language; this is the
"why I made the change" for someone reading the PR a year from now.>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

Examples that follow this shape live on PRs like Preponderous-Software/roam#358 — read a few before writing your own if you want the cadence.

What to avoid:
- Bullet lists in the body (use prose).
- Vague "improve X" subject lines (name the surface and the actual change).
- Citing the heuristic without explaining *why* it applies.
- Writing "fixed bug" without saying what the player would have observed.

---

## PR title and body update

After the last round of a batch:

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

Use `gh pr edit <number> --title "<title>" --body "$(cat <<'EOF' ... EOF)"` with a heredoc to preserve formatting.

---

## When to stop

Stop the multi-round loop early — and tell the user — when any of these happen:

- You searched honestly and a round produced fewer than 2 candidates that cite a specific heuristic with a concrete player-observable problem.
- You've been re-touching the same files round after round; the project is approaching saturation for this skill.
- A fix would require behavior or design decisions you can't reasonably infer from the codebase (e.g. "should items have rarity tiers?"). Those belong in a design conversation, not a polish PR.

Don't pad. A PR with 25 grounded fixes is better than 40 fixes where the last 15 are wallpaper.

---

## Self-audit

Run this section when the skill may have drifted from reality — e.g. after the target environment changed, commands started failing, or results are consistently wrong.

1. Read this skill file from top to bottom.
2. For each command, path, or assumption, verify it is still correct:
   - Commands and flags still exist (`gh repo view`, `gh pr view`, `gh pr edit`).
   - The "operating principles" still match how the user invokes the skill.
   - The heuristic cheatsheet is still accurate (Nielsen's 10 heuristics are stable, but rewording happens).
   - The commit message format matches the cadence on recent PRs.
3. For each problem found, open a GitHub issue:
   ```bash
   gh issue create --repo dmccoystephenson/nielsen-krug-pass \
     --title "<problem summary>" \
     --body "$(cat <<'EOF'
   **Section:** <which step or section is wrong>
   **Problem:** <what is incorrect>
   **Expected behavior:** <what it should do instead>
   EOF
   )"
   ```
4. Report a summary: how many issues were filed, or confirm the skill is up to date.
