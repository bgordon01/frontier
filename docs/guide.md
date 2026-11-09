# Frontier guide

Everything the [README](../README.md) leaves out. You don't need any of it to get started: say **start here** and Frontier hands you each step.

Commands below use the plugin form (`/frontier:…`). In the Claude app you can also just name the step ("run grill-me").

## The six routes

| Route | What happens |
|---|---|
| **Build something new** | Questions → plan → tickets, then per ticket: build test first on its own branch, polish, review, hand over for your review. |
| **Improve a project** | Reads the code or screens and lists fixes, changing nothing. The fixes you pick become tickets. |
| **Fix a bug** | Finds the cause, adds a test that shows the bug, fixes it, reviews it, hands it over for your review. |
| **Write copy** | Writes or tightens copy, then removes machine-sounding phrasing. |
| **Design something** (optional) | A design brief first, then critique and steer, then a `DESIGN.md` that keeps screens on brand. |
| **Update my skills** | Gets the latest Frontier. |

## Words you'll see

- **Spec**: the plan written down, so you and Claude agree on what "done" means.
- **Ticket**: one small piece of the spec that can be built and checked on its own.
- **Test**: a few lines of code that check a feature still works, so it can't quietly break later.
- **Branch**: a separate copy of your project where a ticket is built, so your working version is untouched until you keep the change.
- **Review**: the moment you look at a finished ticket and decide to keep it. On GitHub this is a pull request; on GitLab, a merge request.

## Where tickets live

The first time a step needs somewhere to save tickets, Frontier asks you to choose, then records it in `docs/agents/issue-tracker.md`:

| You pick | Tickets live in | A finished ticket ends as |
|---|---|---|
| GitHub | GitHub Issues | A draft pull request |
| GitLab | GitLab Issues | A draft merge request |
| Jira, Linear or other | Your tracker, as you describe it | A pushed branch plus your tracker's review step |
| Plain files | `.scratch/` in your project | A local branch and the one command that merges it |

Building needs git, because a branch per ticket is how a change gets undone. If your project doesn't use git yet, Frontier offers to set it up.

## Every command

| Command | What it does |
|---|---|
| `/frontier:start` | Asks what you want and walks you through the route. Same as saying "start here". |
| `/frontier:grill-me` | Questions your idea until every decision is made. |
| `/frontier:to-spec` | Writes the agreed plan as a spec in your tracker. |
| `/frontier:to-tickets` | Splits the spec into small tickets. |
| `/frontier:implement` | Builds one ticket on its own branch, test first where it can, then hands it over for your review. |
| `/frontier:build-spec <issue or file>` | Builds every ticket of a spec, one clean helper per ticket, as stacked draft PRs. Stops at anything only you can decide. |
| `/frontier:tdd` | The test-first loop on its own. `implement` uses it for you. |
| `/frontier:code-review` | Reviews a change against the spec and your standards. |
| `/frontier:diagnosing-bugs` | Finds the real cause of a bug before fixing it. |
| `/frontier:improve-codebase-architecture` | Reports what's worth restructuring. Changes nothing. |
| `/frontier:setup-matt-pocock-skills` | Records where your tickets live. `start` runs it for you when needed. |
| `/frontier:impeccable critique` / `polish` / `clarify` | Reviews a screen, fixes it, or rewrites its labels and errors. |
| `/frontier:impeccable shape` | Interviews you and writes a design brief before anything is drawn. |
| `/frontier:impeccable bolder` / `quieter` / `typeset` / `colorize` / `layout` | Pushes a design in one direction. One at a time. |
| `/frontier:impeccable document` / `extract` | Writes `DESIGN.md` from built screens; turns repeated UI into components. |
| `/frontier:copywriting`, `/frontier:copy-editing` | Writes new copy or tightens existing copy. |
| `/frontier:stop-slop` | Removes phrasing that sounds machine-written. |

## Design flow (optional)

1. `/frontier:impeccable shape`, with screenshots of 3–5 products whose look you like.
2. Build from the brief, starting at `/frontier:to-spec`.
3. `/frontier:impeccable critique`, then one steer (`bolder`, `quieter`, `typeset`, `colorize` or `layout`).
4. `/frontier:impeccable document` once real screens exist.

Using Claude Design? Save the look as a Design System artifact in Claude. Where Claude Code's design sync is available, use it to keep that system matched to your code.

## Optional: check it in a browser

After a ticket changes a screen, Frontier offers to open your app in a real browser and click through it, but only if Anthropic's **webapp-testing** skill is turned on. It's Anthropic's own skill, not part of Frontier, so Frontier stays small and you get Anthropic's updates directly. It uses Playwright, so it needs Python on your machine and installs a browser the first time it runs.

- **Claude app:** it's one of Anthropic's example skills. Turn on **webapp-testing** only in your skills settings.
- **Claude Code:** copy the [`skills/webapp-testing`](https://github.com/anthropics/skills/tree/main/skills/webapp-testing) folder from `anthropics/skills` into `~/.claude/skills/`. Don't install the whole `example-skills` plugin: it also brings `frontend-design`, which gives advice that conflicts with Impeccable.

If it isn't on, Frontier skips the offer and nothing else changes.

## House rules

Applied to every skill. Your project's own `CLAUDE.md` wins on conflict.

- **Design:** no purple gradients, glows or floating shadows · no rows of three equal cards · a 4px spacing grid · one bold element per screen.
- **Copy:** no hype words ("seamless", "supercharge"…) · specifics over adjectives ("Exports in 2 seconds", not "Blazing fast").

Full text: [`rules/RULES.md`](../rules/RULES.md).
