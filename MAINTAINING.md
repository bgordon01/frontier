# Maintaining Frontier

How Frontier is built, how upstream changes come in, and how to change what's in it. Everything here runs only when you run it: there are no hooks or scheduled jobs. The one GitHub Action publishes the landing page on push.

## Layout

```
frontier/
├── install.sh          one-line installer; bootstraps ~/.frontier when piped from curl
├── update.sh           git pull + re-run install.sh, with a summary of changed skills
├── uninstall.sh        removes everything listed in the install manifest
├── sync.sh             pulls upstream skills into vendor/, re-applies overrides/, rebuilds plugin/
├── build-plugin.sh     assembles plugin/ from skills/, vendor/ and rules/
├── .claude-plugin/     marketplace.json: this repo is the "bgordon01" marketplace
├── plugin/             the built "frontier" plugin (generated; don't edit by hand)
├── sources.conf        which upstream repos and skill folders to vendor
├── rules/RULES.md      the house rules, in precise form, for Claude
├── skills/             this repo's own skills (start, build-spec)
├── overrides/          local changes re-applied to vendored skills on every sync
└── vendor/<source>/    vendored skills, one folder per upstream, with LICENSE and SOURCE
```

## What's vendored

| Source | Skills | Licence |
|---|---|---|
| [mattpocock/skills](https://github.com/mattpocock/skills) | grill-me, to-spec, to-tickets, implement, tdd, code-review, diagnosing-bugs, improve-codebase-architecture; support: grilling, setup-matt-pocock-skills, codebase-design, domain-modeling | MIT |
| [pbakaus/impeccable](https://github.com/pbakaus/impeccable) | impeccable (24 commands), plus 4 subagents | Apache-2.0 |
| [hardikpandya/stop-slop](https://github.com/hardikpandya/stop-slop) | stop-slop | MIT |
| [coreyhaines31/marketingskills](https://github.com/coreyhaines31/marketingskills) | copywriting, copy-editing | MIT |

Support skills are called by other skills (`grill-me` calls `grilling`, `tdd` calls `codebase-design`, and so on), so they must stay installed even though you rarely use them directly.

Impeccable's automatic design-check hook is deliberately left out. Its detector script still runs when an `/impeccable` command needs it; on first use it downloads Impeccable's engine binary to `~/.impeccable/bin/`.

## Install script (maintainers and cloud sessions)

Users install the plugin (see the [README](README.md)). The script is for maintainers and cloud sessions. It gives bare command names (`/start` instead of `/frontier:start`) and links skills to this checkout, so edits take effect without a rebuild. Don't use it and the plugin on the same machine, or every skill appears twice.

```bash
curl -fsSL https://raw.githubusercontent.com/bgordon01/frontier/main/install.sh | bash   # install, then restart Claude Code
~/.frontier/update.sh                                                                     # update
~/.frontier/uninstall.sh                                                                  # remove (delete ~/.frontier too if you want the files gone)
```

**Cloud sessions** (Code tab, claude.ai/code): paste this into the environment's **Setup script** once. There's no repo hook; the setup script is the only thing that installs it.

```bash
#!/bin/bash
curl -fsSL https://raw.githubusercontent.com/bgordon01/frontier/main/install.sh | bash || true
```

**From claude-toolkit:** run `~/.claude-toolkit/update.sh` once to move everything to `~/.frontier`. Plugin users remove **toolkit** and add **frontier**. The old cloud setup URL still redirects.

## What install.sh changes

```
~/.frontier/                the downloaded repo (curl install)
~/.claude/
├── CLAUDE.md               one @import line between frontier markers
├── frontier-rules.md       → rules/RULES.md
├── skills/<18 skills>      → skills/ and vendor/*/
├── agents/<4 agents>       → vendor/impeccable/_agents/
└── .frontier-installed     manifest of everything above, for clean updates
```

- Everything is symlinked, so `git pull` takes effect immediately. Use `install.sh --copy` for standalone copies, and re-run after every update.
- An existing skill or agent with the same name is renamed `*.bak-<timestamp>`, never overwritten.
- `settings.json` is never touched.
- On the first run after the rename from `design-discipline`, the old manifest, rules link and CLAUDE.md block are cleaned up automatically.
- Set `FRONTIER_DIR` to install the repo somewhere other than `~/.frontier`.

## The plugin

The repo is also a plugin marketplace named `bgordon01`, with one plugin, `frontier`. The plugin is built from the same files as the script install.

- **Build:** `./build-plugin.sh` writes `plugin/`. It copies every skill, drops `disable-model-invocation` (in Claude app chats, Claude picks skills itself, so invoke-only skills would be unreachable), adds `rules/RULES.md` as a `house-rules` skill (plugins can't add `CLAUDE.md` instructions), copies the agents, and collects the licences into `plugin/licenses/`. `sync.sh` runs it automatically. Run it by hand after editing `skills/` or `rules/`, then commit `plugin/`.
- **Validate:** `claude plugin validate ./plugin` and `claude plugin validate .`
- **Test locally:** `claude plugin marketplace add ./` then `claude plugin install frontier@bgordon01`
- **Naming:** the repo, the plugin and the command prefix are all `frontier` (`/frontier:grill-me`). Plugin names can't start with `claude-`.
- **Old names:** before the rename this was `claude-toolkit`, installed to `~/.claude-toolkit`. `install.sh` moves an old install to `~/.frontier`, re-points its git remote, and swaps the old rules file and CLAUDE.md block for the new ones. `uninstall.sh` cleans up either.
- **Versioning:** `plugin.json` has no `version`, so each commit counts as a new version and every push reaches users on their next update.
- **Script install vs plugin:** don't use both on one machine, or every skill appears twice. Differences: the script gives bare command names and always-on house rules through `CLAUDE.md`, and keeps the invoke-only flags. The plugin works in the Claude app and syncs to Claude Code through the claude.ai account.

## Pulling upstream changes

Run this whenever you choose. Nothing pulls automatically.

```bash
cd ~/.frontier
./sync.sh
```

`sync.sh`:
1. Clones each source in `sources.conf` at its latest commit.
2. Copies the listed skill folders into `vendor/<source>/`, with the upstream LICENSE/NOTICE and a `SOURCE` file recording the commit.
3. Re-applies `overrides/`.
4. Stages the result. It never commits.
5. Lists any changed script files separately, so you can read those first.

Then decide:

```bash
git diff --cached -- vendor                              # review
git commit -m "Sync upstream skills" && git push         # accept (includes the rebuilt plugin/)
git restore --staged --worktree vendor                   # reject
```

After pushing, run `./update.sh` (or say "update my skills") on each Mac.

`sync.sh` stops with an error if two skills share a name, a path in `sources.conf` matches nothing, or an override names a skill that isn't vendored.

## Changing the set

Edit `sources.conf`. Each line is a source name, a GitHub `owner/repo`, then entries:

- a skill folder path, or a glob of them (`skills/engineering/*`); `.` means the repo root
- `agents=<glob>` for subagent `.md` files, installed to `~/.claude/agents/`

Lines can be continued with a trailing `\`. Run `./sync.sh` afterwards.

If you add a skill that calls another skill (look for "Call the Skill tool with …" in its `SKILL.md`), add that one too.

## Overrides: how the house rules beat upstream

Vendored files are replaced on every sync, so local changes live in `overrides/`:

- **`overrides/<skill>.md`** is appended to that skill's `SKILL.md` under a `(Frontier)` heading. `impeccable`, `copywriting`, `copy-editing` and `stop-slop` carry a "Global rules override" that enforces the house copy rules; `implement` carries a "Branch and review lifecycle" section: one ticket per branch cut from the latest default branch (only once its blockers are merged), ending in a review that matches the tracker in `docs/agents/issue-tracker.md`. That means a draft PR on GitHub, a draft MR on GitLab, a pushed branch plus the tracker's own review step on Jira, Linear and others, or a local branch to merge for local markdown.
- **`overrides/frontmatter.conf`** sets frontmatter keys. It makes skills invoke-only (`disable-model-invocation: true`), so they run only when typed and their descriptions stay out of Claude's context.

Auto-triggering stays on for `start`, `impeccable` and the four support skills. `stop-slop` is invoke-only: `start` hands it over at the end of every copy step. The support skills have to stay on because other skills call them through the Skill tool.

`start` runs invoke-only skills by reading their `SKILL.md` directly, so the flag doesn't block it.

Known issue: in some sessions, Claude Code won't load an invoke-only skill even when you type its command, and follows the file by hand instead ([anthropics/claude-code#26251](https://github.com/anthropics/claude-code/issues/26251)). The result is the same. To drop the flag for a skill, delete its line from `frontmatter.conf` and re-sync.

## The /start skill

`skills/start/SKILL.md` is this repo's own skill and isn't synced from anywhere. It routes six intents (build new, improve existing, fix a bug, write copy, design something, update) to the right sequence of skills. It checks for first-time project setup (`docs/agents/issue-tracker.md` and `PRODUCT.md`), runs read-only steps straight away, and asks before any step that changes files. Edit it directly.

## Brand

The mark is a **command prompt**: a chevron (`›`) and cursor inside a rounded square. It stands for running the one command Frontier asks of you — `/start` — and nods to the terminal where it lives. The review gate it used to draw now lives in the UI as the "Changes files · waits" tag on steps that write.

| | |
|---|---|
| Mark | A chevron and cursor in a rounded square (`M10 9L15 14L10 19` plus a short underline, 28×28 and 32×32 boxes). Outline on light grounds; filled crimson for app icons and the favicon. |
| Wordmark | "Frontier" in Inter 600. |
| Accent | Crimson `#AF144B` (hover `#870A3C`). Ink `#1A1A1A`, secondary `#444444`. Light theme only. |
| Type | Inter for the UI and the wordmark, Fira Code for commands, labels and code; see [`DESIGN.md`](DESIGN.md). |

Files in [`site/assets/`](site/assets):

| File | Use |
|---|---|
| `logo.svg` | Mark and wordmark; switches colours with light/dark mode |
| `logo-on-light.svg`, `logo-on-dark.svg` | Fixed-colour versions of the logo |
| `logo.png` | Logo raster (high-res) for places that don't take SVG |
| `mark.svg` | The mark on its own, for avatars and the plugin icon |
| `favicon.svg`, `favicon-32.png` | Browser tab icons |
| `apple-touch-icon.png` (180×180), `icon-512.png` | Home-screen and app icons |
| `og.png` (1200×630) | Social preview card |
| `brent.webp` | Brent's cut-out portrait for the landing-page top bar (background removed with rembg; provenance in `brent.webp.json`) |

## Landing page

The page lives in [`site/index.html`](site/index.html): one static file, no build step, JavaScript only for the copy buttons, no analytics. On every push to `main` that touches `site/`, the [Pages workflow](.github/workflows/pages.yml) publishes it to [bgordon01.github.io/frontier](https://bgordon01.github.io/frontier). Pages has to be switched on once in the repo settings (**Settings → Pages → Source: GitHub Actions**).

The decisions behind the page, name and logo are in [`docs/frontier/SPEC.md`](docs/frontier/SPEC.md).

## Credits and licences

Vendored skills are the work of Matt Pocock, Paul Bakaus, Hardik Pandya and Corey Haines, redistributed under their original licences, which are kept in each `vendor/<source>/` folder. Local changes are applied by `overrides/` and marked in each modified `SKILL.md`. This repo isn't affiliated with any of them.

This repo's own files are MIT-licensed.
