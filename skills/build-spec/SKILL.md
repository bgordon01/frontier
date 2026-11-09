---
name: build-spec
description: Build all the sub-tickets of a spec, one clean subagent per ticket, as stacked draft PRs. Use when the user gives a spec (an issue number or a file path) and asks to build its tickets in one go, "run the whole chain", or hand the build off without waiting between tickets. Takes the spec as its argument.
argument-hint: <spec issue number or file path>
disable-model-invocation: true
---

# Build spec

Build every ticket of one spec, one after another, without waiting for the owner between tickets. You are the parent: a thin coordinator. Each ticket goes to a fresh subagent with a written handover, so the parent stays small and each ticket's context stays under about 150k tokens.

The argument is the spec: an issue number (`#131`) or a path to a spec file. If it is missing, ask for it and stop.

## The rules of the chain

1. **Clean subagent per ticket.** The parent reads, plans, hands over, checks reports and ticks the plan. The parent never builds a ticket itself.
2. **Sequential, never parallel.** Tickets usually edit the same files. One subagent runs at a time; start the next only after the previous has reported.
3. **Stacked branches.** Each ticket's branch starts from the previous ticket's branch. Each draft PR is based on the previous branch (the first on the base branch). A fix to an early PR later means rebasing the ones after it; say so in the report.
4. **Nothing irreversible.** Never deploy, never merge, never run a migration or a push-to-live script against real data. The owner does those.
5. **Honest checks.** Never write "passed" for a check that did not run. Write `not run: <reason>`.

## Before the loop

1. **Read the spec.** For an issue: its body and its sub-issues (the tracker in `docs/agents/issue-tracker.md` says how; on GitHub use the sub-issues list). For a file: the file and the ticket files or links it names.
2. **Derive the ticket list and order from the spec's sub-issues and their "Blocked by" sections.** Topological order: a ticket comes after everything that blocks it. Where two orders are valid, pick the one that keeps tickets touching the same files next to each other. If "Blocked by" has a cycle, or a blocker is not one of the spec's tickets and not merged, stop (see *Stop conditions*).
3. **Read the project rules.** `CLAUDE.md` gives the project's checks, test seams and layout; the house rules (`~/.claude/frontier-rules.md` or the `house-rules` skill) give the design and copy laws. Project checks come from `CLAUDE.md`; this skill names none.
4. **Check the setup once**, so no subagent loses time on it. Run what the project needs and record what worked:
   - dependencies installed from the lockfile (`npm ci` or the project's equivalent);
   - any CLI the checks need, such as `firebase-tools` for emulators (and Java if the emulators need it);
   - the headless Chrome path for browser checks (look for an installed Chromium, then note the path and any `--no-sandbox` flag);
   - which checks cannot run here (blocked hosts, missing keys, no display) and the exact `not run` wording for each.
5. **Pick the base branch** and confirm it is pushed. Open a draft PR for it into the default branch if the spec's handoff asks for one, so the chain has a base the owner can merge first.
6. **Write or update the plan file** (`tasks/plan.md`, or the one the project uses): a section for this spec with one box per ticket, in order, each with its branch name. The parent ticks a box only after the subagent's report is checked.
7. **Write the handover** for the subagents from the template below, filled in once with the setup facts. Save it in the scratchpad, or in `tasks/` if the project keeps handoffs there.

## The loop

For each ticket, in order:

1. Fill the handover template for this ticket: its number, title, branch name, the branch to start from, the PR to base on, and the PRs already open.
2. Spawn one general-purpose subagent with the filled handover as its whole brief. It cannot ask the owner anything; it returns a report or stops.
3. Read the report. Check it against the rules: every check is `pass`, `fail` or `not run: <reason>`; a build with no checks run is not done.
4. Spot-check the claims that are cheap to check: the PR exists and is a draft, the branch is pushed, the working tree is clean (`git status`), the branch is based on the previous one.
5. **Status done:** tick the box in the plan file with the PR link and the checks, commit and push that on the ticket's branch, then start the next ticket from that branch.
6. **Status blocked or needs-decision:** stop the chain (see *Stop conditions*). Do not start the next ticket.

Commit and push after every step, in the parent and in the subagents. Nothing is left uncommitted or unpushed when a turn ends.

Do not leave background commands (emulators, dev servers, watchers) running past a turn. A subagent starts what it needs, stops it before it reports, and says so. The parent checks with `ps` before moving on and stops strays.

## The per-ticket handover (exact template)

Fill every `<...>`. Keep the rest word for word. This is the whole brief the subagent gets.

````markdown
# Handover for one ticket (from the parent session)

You build ONE ticket of a stacked chain in <repo path>, then stop and report. The parent session coordinates; it keeps its own context small by giving each ticket to a fresh agent like you. Keep your own context under about 150k tokens: read files in parts, do not dump big outputs.

## Your ticket
- Ticket: <#number and title>, a sub-issue of the spec <spec reference>.
- Branch to create: <branch name>, cut from <previous branch>.
- Open the draft PR into: <previous branch> (first ticket: <base branch>).
- Already open in the chain: <list of PR links, in order>.

## Read first
1. <handoff file if any>, `CLAUDE.md` and the house rules (`~/.claude/frontier-rules.md` or the `house-rules` skill). Project checks, test seams and layout come from `CLAUDE.md`.
2. The plan file, section for this spec: what is done, PR numbers.
3. The issue for your ticket and the spec. Read the whole ticket: body, acceptance criteria, any Design section, "Blocked by".
4. <Design sources the ticket names, with how to read them. If a read is refused, stop and report. Never build a screen from memory.>

## Environment (already set up)
<The setup facts from the parent: install state, CLI versions, emulator notes, the headless Chrome path and flags, the exact commands for each check in order.>
- Checks that cannot run here: <check: "not run: <reason>"> (one line each). Write exactly that wording; never write "passed" for a check you did not run.

## Rules
- Build test-first through the test seams the spec agreed, with the `implement` skill (invoke it if you can, otherwise follow it by hand). Add no other seams.
- Logic and user-visible copy go in the pure layer the project names in `CLAUDE.md`, with unit tests. Components only render. Do not leave copy, icon or label data inside components.
- Assert the exact thing the acceptance criteria name. A passing test that checks the wrong thing is worse than none.
- After a rename, search the docs and comments for the old name and fix every hit.
- Never skip, disable or weaken a test to get green.
- Never deploy, merge, or run a migration or push script against live data. Migrations are built and tested on the emulator only; the PR body says "Before the rules deploy: run <command> on live data." where the ticket has one.
- Do not make product or design decisions the spec, ticket or design source does not answer.
- Do not leave background processes running when you report.

## Steps
1. Create the branch from <previous branch>.
2. Build test-first. Commit as you go.
3. Run every check in the order above. Record each as `pass`, `fail` or `not run: <reason>`.
4. Check the screens against the design source (if the ticket has UI) and the copy against the house rules: banned words, no em dashes, `stop-slop` over new copy.
5. Review your diff against its base branch with two reviewers, spawned by you: one for **standards** (the project's documented rules in `CLAUDE.md`) and one for **spec** (the ticket's acceptance criteria). Use the `code-review` skill if you can spawn them. If you cannot spawn reviewers, review the diff yourself against both and say so in the report. Fix every real finding. List each finding you leave, with the reason.
6. Commit and push the branch. Open a **draft** PR into <previous branch>. Title: `#<n>: <ticket title>`. Body: what it does; `Closes #<n>`; the design screens it builds; each check as pass, fail or `not run: <reason>`; review findings left; "Stacked on #<previous PR>"; the migration note if any. Follow the repo's PR template if there is one.
7. Tick your ticket in the plan file with the PR link and the check results. Commit and push on the same branch. Leave the working tree clean and pushed.

## Commit and PR text
- Every commit message ends with these two lines:
  `<commit trailer 1>`
  `<commit trailer 2>`
- Every PR body ends with:
  `<PR footer>`
  `<session URL>`
  (Copy both from the session's attribution reminder. If there is none, add nothing.)

## Stop instead of carrying on
- A check fails and you cannot fix it inside the ticket.
- A decision only the owner can make: the spec, ticket or design source does not answer it.
- A design source cannot be read.
- A key, secret or account the ticket needs is missing. Build and test against fakes only. Do not create, request or store a key.
- You would have to deploy, merge, or touch live data.
Say plainly which one you hit.

## Report back (under 250 words)
```
ticket:     <#n and title>
status:     done | blocked | needs-decision
pr:         <URL>
branch:     <exact branch name the next ticket starts from>
checks:     <each as "command: pass|fail|not run: reason"; never claim a check that was not run>
review:     <findings fixed, findings left with reasons; and whether reviewers were spawned or self-review>
decisions:  <questions only the owner can answer, or "none">
followups:  <deferred work, migrations to run, or "none">
```
````

## Stop conditions

Stop the chain, keep everything pushed, and report, when:

- a check fails and neither the subagent nor the parent can fix it inside the ticket;
- a decision only the owner can make comes up (something the spec, the ticket or the design source does not answer);
- a design source cannot be read, for example a private canvas that refuses the read;
- a key, secret or account is missing (an API key, a signing credential). Fakes are fine for tests; creating or requesting a real key is not;
- the next step would deploy, merge, or change live data;
- the ticket order cannot be derived (a "Blocked by" cycle, or a blocker outside the spec that is not merged).

Never skip a ticket to keep going: later tickets are stacked on it. If one ticket is blocked, the tickets before it still get reported as finished.

## The final report

Send it when the chain ends or stops. Plain text, in this shape:

```
Chain: <spec> (<n of m tickets built>)

Per ticket
- #<n> <title>: <PR link>
  checks: <each as pass | fail | not run: reason>
  review findings left: <list with reasons, or none>
- ...

Waiting on you
- Merge order: <base PR, then each ticket PR in chain order>
- Before each rules deploy: <migration command and the PR it belongs to>
- Keys or accounts to set up: <what and where, or none>
- Decisions: <questions only the owner can answer, or none>
- Stopped at: <ticket and which stop condition, or "ran to the end">

Next
- <the next ticket to build, or the owner's first step after merging>
```

If the report will be pasted somewhere, keep it under one screen. Run `stop-slop` over any prose in it.

## Pitfalls seen in earlier chains

- **Stale names after a rename.** Docs, comments and test titles kept the old name. Search for it before the PR.
- **Copy and icon data left in components.** It belongs in the pure view layer with a unit test; components render what they are given.
- **Tests that assert the wrong thing.** A test named for the acceptance criterion but checking something nearby. Read each assertion against the criterion.
- **Background commands outliving the turn.** Emulators and servers kept running and held ports for the next ticket. Stop them before reporting.
- **Rules and view layer drifting apart.** Where two files name the same thing (a permission list, a field name), add a test that reads both.
- **Unrun checks reported as passed.** The report says `not run: <reason>` or it says nothing.
- **Work left uncommitted.** Commit and push after every step, so a stopped chain loses nothing.

## House rules

The house rules apply to the PR text, plan file and report this skill produces: no banned words, no em dashes, concrete over vague.
