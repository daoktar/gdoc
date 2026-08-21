# gdoc

Read/edit Google Docs and Sheets from the shell. Single self-contained [uv](https://docs.astral.sh/uv/) script — dependencies install themselves on first run.

Built for AI-agent workflows (Claude Code): docs are pulled as markdown and edited with unique old/new text pairs — no character-index arithmetic, so edits stay reliable on documents dozens of pages long.

## Install

```sh
cp gdoc ~/bin/gdoc && chmod +x ~/bin/gdoc   # or anywhere on PATH
```

With the Claude Code skill (teaches Claude the safe pull -> edit -> verify workflow):

```sh
cp gdoc ~/bin/gdoc && chmod +x ~/bin/gdoc && mkdir -p ~/.claude/skills/google-docs && cp skill/SKILL.md ~/.claude/skills/google-docs/
```

Requires `uv`.

## Auth setup (once)

1. [Google Cloud Console](https://console.cloud.google.com) → new project
2. Enable **Google Docs API**, **Google Drive API**, **Google Sheets API**
3. OAuth consent screen → External → add yourself as a test user
   (publish to Production later to stop weekly re-login)
4. Credentials → Create OAuth client ID → **Desktop app** → download JSON
5. Save it as `~/.config/gdoc/client_secret.json`
6. Run `gdoc ls` — browser opens once; token lands in `~/.config/gdoc/token.json`

Credentials live only in `~/.config/gdoc/`, never in this repo.

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
