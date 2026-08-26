# Google OAuth setup for gdoc (once, ~5 minutes)

The agent cannot do this - it needs a browser. Do it under the Google account whose
documents you will be editing.

1. Open [console.cloud.google.com](https://console.cloud.google.com) and create a new
   project (any name, e.g. `gdoc-cli`).
2. **APIs & Services -> Library**: enable **Google Docs API**, **Google Drive API** and
   **Google Sheets API**.
3. **OAuth consent screen**: choose **External** for a personal account (**Internal** for a
   Workspace account, which skips the test-user step). App name `gdoc`, your email in both
   support and developer fields. Scopes can stay empty - the script asks for its own.
   For External, add your own email under **Audience -> Test users**, otherwise login hits
   "access blocked".
4. **Credentials -> Create credentials -> OAuth client ID -> Desktop app** -> Download JSON.
5. Put the file in place:

   ```sh
   mkdir -p ~/.config/gdoc
   mv ~/Downloads/client_secret*.json ~/.config/gdoc/client_secret.json
   chmod 600 ~/.config/gdoc/client_secret.json
   ```

6. First run - a browser opens once:

   ```sh
   scripts/gdoc ls
   ```

   On the "Google hasn't verified this app" screen press **Continue** - it is your own app
   in testing mode. A list of your documents means it works.

## Where the token lives

The OAuth token goes into the macOS Keychain (service `gdoc`, account `token`), with
`~/.config/gdoc/token.json` (mode 600) as an automatic fallback when the Keychain is
unavailable - headless boxes, cron, SSH. The client secret always stays a file. Neither
ever belongs in a repository.

## The seven-day expiry

An External app left in **Testing** gets a refresh token that dies after ~7 days, and
`gdoc` then fails with `invalid_grant`. Fix it permanently by pressing **Publish app** on
the consent screen (status *In production*). Google verification is not required; the
"unverified" warning simply stays. Internal/Workspace apps never expire this way.

Re-auth after an expiry:

```sh
security delete-generic-password -s gdoc -a token   # macOS Keychain
rm -f ~/.config/gdoc/token.json
scripts/gdoc ls                                     # opens the browser again
```

## Corporate Workspace

A Workspace admin can block third-party OAuth clients outright. If that is the case here,
step 4 fails - and the same block would stop any Google Docs MCP server too. Ask the admin
to allow the client, there is no way around it from this side.

## Scope

The script requests a single scope, `https://www.googleapis.com/auth/drive` - full Drive
access. Narrower `drive.file` would only reach files the app itself created, which defeats
the purpose (editing documents that already exist). Deliberate trade-off; the blast radius
of a leaked token is the whole Drive.
