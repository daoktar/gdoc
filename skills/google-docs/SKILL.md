---
name: google-docs
description: Read, edit, and comment on Google Docs and read/write Google Sheets via the local gdoc CLI — markdown export, surgical text-anchored edits, comments, A1-range reads/writes for spreadsheets. Use when the user shares a docs.google.com link (document or spreadsheet), asks to read/summarize/fix/update a Google Doc or Google Sheet, works with гуглдок/гугл-таблица, or mentions comments in a Google Doc. Not for local Word/.docx or Excel/.xlsx files, and not for Confluence.
---

# Google Docs via gdoc

`gdoc` — self-contained uv script, on PATH already (the plugin's `bin/` is added automatically). Auth lives in `~/.config/gdoc/`. `<doc>` = doc ID or any docs.google.com URL.

## Core workflow

1. **Pull** the doc to a local markdown file (in the scratchpad, not the user's project):
   ```bash
   gdoc pull <doc> doc.md
   ```
2. **Work locally** — Read/Grep/analyze `doc.md` like any file. Never feed a whole long doc into context; grep for the relevant section.
3. **Edit surgically** — apply changes as unique old/new text pairs, like the Edit tool:
   ```bash
   gdoc edit <doc> -e "старый текст" -e "новый текст" -e "ещё старый" -e "ещё новый"
   ```
   Many edits → JSON file: `gdoc edit <doc> --json edits.json` with `[{"old":"...","new":"..."}]`.
4. **Verify** — re-pull and grep to confirm the change landed.

## Critical: edit matches PLAIN text, not markdown

The doc's text has no markdown syntax. `**жирный**` in the pulled .md is `жирный` in the doc; `# Заголовок` is `Заголовок`. Strip md syntax from old/new strings before passing to `edit`. Tables: cell text matches, `|` separators don't exist in the doc.

## Guards (errors are protection, not failures)

- `N matches for:` — old string not unique. Extend context, don't force it.
- `match spans a non-text element` — inline image/footnote inside the match. Split the edit to avoid crossing it.
- `edits overlap` — two pairs touch the same text; merge them into one pair.
- HTTP 400 with `requiredRevisionId` — someone edited the doc between read and write. Just retry: `edit` re-reads fresh state each run.
- Deleting text: pass empty new — `-e "текст на удаление" -e ""`.

## Comments

```bash
gdoc comments <doc>                          # list: [id] open/resolved, author, replies
gdoc comment <doc> "текст" --quote "цитата"  # add (unanchored — API can't bind to text; --quote displays context)
gdoc reply <doc> <id> "текст"                # reply in thread
gdoc reply <doc> <id> --resolve              # resolve (with or without text)
```

Review flow: `comments` → address each in the doc via `edit` → `reply <id> "исправлено" --resolve`.

## Search / list

```bash
gdoc ls "название"      # find docs by name, newest first
gdoc ls --sheets        # spreadsheets instead of docs
gdoc ls -n 50           # more results
```

## Google Sheets

Cells have stable A1 addresses — no pull-the-whole-file cycle needed. Quote sheet names with spaces/cyrillic: `"'Лист 1'!A1:D50"`.

```bash
gdoc tabs <sheet>                          # list tabs + rows x cols
gdoc newtab <sheet> "Название"             # add an empty tab
gdoc rmtab <sheet> "Название"              # delete tab (destructive — only when user asks)
gdoc get <sheet> "'Лист1'!A1:D50"          # range -> TSV on stdout
gdoc get <sheet> "'Лист1'!A:D" data.tsv --formulas   # to file, formulas not values
gdoc set <sheet> "'Лист1'!B2" "=SUM(C2:C9)"          # one cell, parsed as if typed
gdoc put <sheet> "'Лист1'!A2" block.csv    # write block (.csv comma / .tsv tab)
gdoc append <sheet> "Лист1" rows.csv       # append after existing data
```

Read ranges, never whole big sheets into context. Writes overwrite cells silently — `get` the target range first to see what's there. Values go through USER_ENTERED: numbers, dates, and `=formulas` parse as if typed by hand.

## Full replace — last resort

```bash
gdoc push <doc> file.md --force
```

Replaces the ENTIRE doc from markdown. Keeps file ID/URL/sharing, but orphans all comments and flattens unsupported formatting. Only when the user explicitly wants the doc rebuilt; prefer `edit` otherwise. Recovery: Google Docs version history.

## Troubleshooting

- `gdoc: command not found` — installed by hand instead of as a plugin. Call the script by its path, or symlink it onto PATH.
- `uv: command not found` — the script runs under [uv](https://docs.astral.sh/uv/); install it first.
- `missing ~/.config/gdoc/client_secret.json` — OAuth not set up; ask the user (needs browser + Google Cloud Console: Desktop-app OAuth client, Docs API + Drive API enabled).
- `invalid_grant` / auth errors after ~7 days — OAuth app is in Testing mode: clear the stored token (`security delete-generic-password -s gdoc -a token`; also delete `~/.config/gdoc/token.json` if present), ask the user to run `gdoc ls` interactively (opens browser). Permanent fix: publish the OAuth app to Production.
- Export limit is 10 MB of markdown — hundreds of pages; not a practical constraint.
- File not found on a URL the user pasted — the ID is truncated; real IDs are ~44 chars. Ask for the full link.
