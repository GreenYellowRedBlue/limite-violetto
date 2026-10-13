# Issue tracker: GitHub

Issues and specs for this repo live in GitHub Issues on `GreenYellowRedBlue/limite-violetto`. Use the `gh` CLI for all tracker operations, and always pass `--repo GreenYellowRedBlue/limite-violetto` explicitly. Do not rely on the current checkout's default repository: it may resolve to the upstream repository.

## Conventions

- **Create an issue**: `gh issue create --repo GreenYellowRedBlue/limite-violetto --title "..." --body "..."`. Use a body file for multi-line bodies.
- **Read an issue**: `gh issue view <number> --repo GreenYellowRedBlue/limite-violetto --json number,title,body,labels,comments`.
- **List issues**: `gh issue list --repo GreenYellowRedBlue/limite-violetto --state open --json number,title,body,labels,comments` with appropriate `--label` and `--state` filters.
- **Make an issue a sub-issue of a parent**: `gh issue create --repo GreenYellowRedBlue/limite-violetto --parent <parent> ...`, or link an existing issue with `gh issue edit <parent> --repo GreenYellowRedBlue/limite-violetto --add-sub-issue <child>`. Use GitHub's native sub-issue relationship when available. If the repo/account does not support it, put `Part of #<parent>` at the top of the child body.
- **Add a blocking dependency**: `gh issue edit <child> --repo GreenYellowRedBlue/limite-violetto --add-blocked-by <blocker>`. Use GitHub's native issue dependency relationship when available. If unavailable, put `Blocked by: #<number>` at the top of the child body; remove or resolve that reference when the blocker closes.
- **Comment on an issue**: `gh issue comment <number> --repo GreenYellowRedBlue/limite-violetto --body "..."`.
- **Apply / remove labels**: `gh issue edit <number> --repo GreenYellowRedBlue/limite-violetto --add-label "..."` / `--remove-label "..."`.
- **Close non-implementation issues**: `gh issue close <number> --repo GreenYellowRedBlue/limite-violetto --comment "..."`.

## Implementation issue resolution

Do not close an implementation issue directly. Implement it on a branch, open a pull request targeting `main` in `GreenYellowRedBlue/limite-violetto`, and include `Closes #<issue-number>` in the PR body. Merge the PR into this fork; GitHub then closes the linked issue. Keep the PR base on this fork's `main`, not the upstream repository.

Example: `gh pr create --repo GreenYellowRedBlue/limite-violetto --base main --title "..." --body "Closes #123"`.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` only if this repo starts treating external PRs as feature requests; `/triage` reads this flag.)_

## When a skill says "publish to the issue tracker"

Create a GitHub issue in `GreenYellowRedBlue/limite-violetto`.

## When a skill says "fetch the relevant ticket"

Read it with `gh issue view <number> --repo GreenYellowRedBlue/limite-violetto`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with child issues as tickets, all in `GreenYellowRedBlue/limite-violetto`.

- **Map**: a single issue labelled `wayfinder:map`, holding the Notes / Decisions-so-far / Fog body. Create it with `gh issue create --repo GreenYellowRedBlue/limite-violetto --label wayfinder:map`.
- **Child ticket**: use a GitHub sub-issue link to the map. Where native sub-issues aren't available, add the child to a task list in the map body and put `Part of #<map>` at the top of the child body. Use `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`) labels. Once claimed, assign the ticket to the driving developer.
- **Blocking**: use GitHub's native issue dependencies (`gh issue edit <child> --repo GreenYellowRedBlue/limite-violetto --add-blocked-by <blocker>`). Where unavailable, use a `Blocked by: #<n>` line at the top of the child body. A ticket is unblocked when every blocker is closed.
- **Claim**: `gh issue edit <number> --repo GreenYellowRedBlue/limite-violetto --add-assignee @me`.
- **Resolve**: for implementation tickets, merge a PR into this fork's `main` with `Closes #<number>`; for non-implementation tickets, comment with the answer and close the issue. Then append a context pointer (gist + link) to the map's Decisions-so-far.
