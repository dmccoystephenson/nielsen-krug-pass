# nielsen-krug-pass

A Claude skill that runs usability-pass rounds on a UI app. It evaluates the app against Nielsen's 10 Heuristics and Krug's *Don't Make Me Think* principles. The whole skill lives in [`nielsen-krug-pass.md`](./nielsen-krug-pass.md).

Use it to keep making small UX improvements to a project without planning a roadmap first.

## Installation

Copy `nielsen-krug-pass.md` into a Claude Code commands directory, either `.claude/commands/` in the target project or `~/.claude/commands/` for all projects. The file name becomes the command name, `/nielsen-krug-pass`.

## Usage

Run it from a checkout of the UI app, on the branch whose open PR should collect the fixes:

- `/nielsen-krug-pass` runs a single round.
- "do N more rounds" runs N rounds back-to-back with no questions in between. The PR description is updated once, after the last round.

## What a round produces

- **One theme per round**, such as *discoverability*, *feedback*, *error prevention*, or *consistency*.
- **2–5 small, focused commits** on the current branch. Each commit names the surface it changes and cites the Nielsen heuristic or Krug principle behind the fix.
- **One push at the end of the round.** The round is added to the PR body as an `### Round N — <theme>` section, and the PR title's fix count is updated. In a multi-round batch, that edit happens once, after the last round.

The skill stops early when a round finds fewer than 2 well-grounded candidates, instead of padding the PR with nitpicks. See the *When to stop* section of the skill file.

## License

This project is licensed under the **Stephenson Software Non-Commercial License (Stephenson-NC)**.

**License:** Stephenson-NC © 2025 Daniel McCoy Stephenson  
See [LICENSE.md](./LICENSE.md) for the full legal text, or the canonical repository at <https://github.com/Stephenson-Software/stephenson-nc-license> for details and commercial-licensing inquiries.
