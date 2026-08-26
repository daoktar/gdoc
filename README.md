# gdoc

Read/edit Google Docs and Sheets from the shell. Single self-contained [uv](https://docs.astral.sh/uv/) script — dependencies install themselves on first run.

Built for AI-agent workflows: docs are pulled as markdown and edited with unique old/new text pairs — no character-index arithmetic, so edits stay reliable on documents dozens of pages long. Ships with an Agent Skill that installs into Claude Code, ChatGPT or Codex.

## Install

The repo ships two things: the `gdoc` CLI and a `google-docs` [Agent Skill](https://agentskills.dev)
that teaches an agent the safe pull -> edit -> verify workflow. They live together in
`skills/google-docs/` — one folder, one copy of the script.

### Claude Code

```sh
claude plugin marketplace add daoktar/gdoc && claude plugin install gdoc@gdoc
```

Installs the skill and puts `gdoc` on PATH for every session. Update with
`claude plugin update gdoc`.

### ChatGPT

ChatGPT reads the same Agent Skills format, and its uploader takes a zip. Build one from a
clone:

```sh
git clone https://github.com/daoktar/gdoc && cd gdoc/skills && zip -r ../google-docs-skill.zip google-docs
```

Then in ChatGPT: **Skills -> Create -> Upload from your computer** and pick
`google-docs-skill.zip` (the unpacked `skills/google-docs/` folder is accepted too). The
bundle is ~13 KB against limits of 50 MB and 500 files.

Skills are available on Business, Enterprise, Edu, Teachers and Healthcare plans; on
Enterprise and Edu an admin has to enable **Skills** and **skill uploading** first.

**Read this before you upload.** `gdoc` drives the Google APIs from a shell. In a ChatGPT
workspace with no shell tool the skill still carries the knowledge — the edit workflow,
what each guard means, the Sheets A1 rules — but the commands will not execute. It earns
its keep where the agent has a shell.

### Codex and other agent CLIs

Copy `skills/google-docs/` into a skills directory — `.agents/skills/` in a repo for
project scope, `~/.agents/skills/` for every session.

### Just the CLI

```sh
ln -s "$PWD/skills/google-docs/scripts/gdoc" ~/bin/gdoc   # or anywhere on PATH
```

`bin/gdoc` in this repo is a symlink to the same file, so the plugin loader and your shell
run one script, not two copies.

Requires `uv` — the script declares its own dependencies and installs them on first run.
Without `uv`: `pip install google-api-python-client google-auth-oauthlib keyring` and call
it with `python3`.

## Auth setup (once)

Full walkthrough, token storage and the seven-day expiry: [skills/google-docs/references/setup.md](skills/google-docs/references/setup.md). Short version:

1. [Google Cloud Console](https://console.cloud.google.com) → new project
2. Enable **Google Docs API**, **Google Drive API**, **Google Sheets API**
3. OAuth consent screen → External → add yourself as a test user
   (publish to Production later to stop weekly re-login)
4. Credentials → Create OAuth client ID → **Desktop app** → download JSON
5. Save it as `~/.config/gdoc/client_secret.json`
6. Run `gdoc ls` — browser opens once; the token is stored in the macOS Keychain (service `gdoc`), with `~/.config/gdoc/token.json` as automatic fallback for headless/cron runs

The OAuth token lives in the Keychain (file fallback in `~/.config/gdoc/`), the client secret in `~/.config/gdoc/` — never in this repo.

## Usage

```
gdoc ls "query"                       list/search docs (--sheets for spreadsheets)
gdoc pull <doc> [out.md]              export doc -> markdown
gdoc edit <doc> -e OLD -e NEW ...     surgical text replace (unique-match, atomic batch)
gdoc edit <doc> --json edits.json     [{"old":"...","new":"..."}, ...]
gdoc push <doc> file.md --force       replace WHOLE doc from markdown (orphans comments)
gdoc comments <doc>                   list comments + replies
gdoc comment <doc> "text" [--quote Q]
gdoc reply <doc> <id> ["text"] [--resolve]

gdoc tabs <sheet>                     list sheet tabs + sizes
gdoc newtab <sheet> "Title"           add an empty tab
gdoc rmtab <sheet> "Title"            delete a tab by exact title
gdoc get <sheet> "'Tab'!A1:D50" [out.tsv] [--formulas]
gdoc set <sheet> "'Tab'!B2" "value"   one cell (USER_ENTERED: formulas work)
gdoc put <sheet> "'Tab'!A2" file.csv  write a block
gdoc append <sheet> "Tab" file.csv    append rows
```

`<doc>` is an ID or any docs.google.com URL. `edit` matches the doc's **plain text** (no markdown syntax) and refuses non-unique matches, ranges crossing images/footnotes, overlapping edits, and writes against a stale revision.

## Safety model

- every `edit` re-reads the doc and applies all changes in one `batchUpdate`, bottom-up, guarded by `requiredRevisionId`
- destructive full replace (`push`) is gated behind `--force`
- `gdoc --selftest` runs offline sanity checks

MIT.
