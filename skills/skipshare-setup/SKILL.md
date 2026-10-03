---
name: skipshare-setup
description: Set up SkipShare for the user in one guided run, installing or updating Node.js, installing the `@skipshare/cli` client, choosing the reply language, logging in with a personal access token, then choosing the default team. Use when the user says "skipshare init", "set up SkipShare" or "install SkipShare", or when the skipshare-send skill finds the CLI missing or the user not logged in.
---

# Set up SkipShare

This skill takes a user from nothing to their first send: Node.js, the SkipShare CLI, login, then
two settings. Many users have never installed Node.js, or have one too old for the CLI, so the
agent does the work. It checks first, shows **one** confirm, and starts only after the user replies
**ok**.

## Reply language

Until the CLI is installed, reply in the language the user writes in. Once it runs, use the
`effective` value of `config language --json` (`en` English, `ja` Japanese), and after step 4 the
language the user picked, for every line including status lines. Use another language only when
the user tells you which language to reply in; writing in a language is not such a request.

## 1. Check what is already there

Run these quietly, before saying anything:

- `node -v`: Node.js is ready when it prints `v20` or newer. Missing or older: it needs an install
  or an update.
- `skipshare --version`: the CLI is installed globally when this prints a version. Then run
  `skipshare update --json` to bring it to the newest release. A CLI older than 0.1.2 has no
  `update` command: run `npm install -g @skipshare/cli@1` instead. If either fails, carry on with
  the installed version.
- `skipshare login status --json` (or `npx -y @skipshare/cli@1 login status --json` when the CLI is
  not installed but Node.js is ready): exit 0 means logged in, and `email` is the account. Exit 3
  means not logged in; keep `error.data.token_page_url` for step 5.

In the rest of this skill, `skipshare` means the global command when it is installed, and
`npx -y @skipshare/cli@1` otherwise. Always pass `--json`.

When everything is ready and logged in, say "SkipShare is already set up as `<email>`.", then
ask step 4 (language) and step 6 (default team), skipping the login.

## 2. One confirm

When skipshare-send started this skill after the user replied **ok** to its setup list, that
reply was the confirm: skip to step 3. Otherwise never install anything before this confirm.

Show one short list of what will happen. Leave out the steps that are already done:

```text
Set up SkipShare:
• Node.js: install v22 (you have v18, the CLI needs 20 or newer)
• SkipShare CLI: install @skipshare/cli
• Language: English or 日本語
• Login: you create a token in the SkipShare web app, I connect it
Then pick your default team.
```

Below the block: "Reply **ok** to start, or **N** to cancel."

Do nothing until the user replies **ok** (or yes). Do not ask again for each step.

## 3. Install

### Node.js (only when missing or older than 20)

Use the first tool that exists, and install the current LTS:

- macOS or Linux with nvm (`command -v nvm`, or `~/.nvm/nvm.sh` exists): source `~/.nvm/nvm.sh`,
  then `nvm install --lts && nvm alias default 'lts/*'`.
- macOS with Homebrew (`command -v brew`): `brew install node` or, when Node.js came from brew,
  `brew upgrade node`.
- Windows with winget: `winget install -e --id OpenJS.NodeJS.LTS`.

Then run `node -v` again. A new install may need a new shell; on nvm, source `~/.nvm/nvm.sh` again
before checking.

No tool found, the install asks for an admin password, or it fails:

```text
❌ I can't install Node.js here.
Install the LTS version from https://nodejs.org, then reply **retry**.
```

Never run `sudo`, never ask for a password, and never change system settings.

### SkipShare CLI

Run `npm install -g @skipshare/cli@1`, then `skipshare --version`.

On a permission error (`EACCES`), do not retry with `sudo`. Use `npx -y @skipshare/cli@1` for the
rest of the setup and tell the user once: "I'll run SkipShare through npx instead of installing it."

## 4. Language

Ask right after the CLI is installed, before the login, so the user reads the login steps and
everything after them in their own language. The setting is stored locally and needs no login.
Ask with exactly this text, word for word (do not rephrase it):

```text
Pick a default language for my responses:
1. English
2. 日本語 (Japanese)
```

Run `config language en --json` or `config language ja --json`. From now on, reply in that language
(translate this skill's replies when it is `ja`). Keep **ok**, **skip**, **retry**, **N**, file
names, emails and links as they are.

## 5. Log in

The token is a password. **Never ask the user to paste it into the chat, and never put it on a
command line.** It goes from the user's clipboard straight to the CLI.

Say this as plain text, not inside a code block or a quote, so the link can be clicked:

Create a token here: <token_page_url>
Copy it, then reply **ok**. Don't paste it here.

`<token_page_url>` is the full URL from `error.data.token_page_url` of the exit-3
`login status --json`, for example `https://share.skipapp.jp/settings/access-tokens`. Never shorten
it to a path or to "Settings → Personal access tokens". When it is missing, use
https://share.skipapp.jp/settings/access-tokens.

After **ok**, pipe the clipboard into the CLI:

- macOS: `pbpaste | skipshare login --json`
- Windows (PowerShell): `Get-Clipboard | skipshare login --json`
- Linux: `xclip -o -selection clipboard | skipshare login --json`, or `wl-paste | ...` on Wayland

Never print the clipboard, echo it, or store it in a variable or file.

- Exit 0: say "✅ Logged in as `<email>`." and go on.
- Token rejected (exit 3) or empty: "❌ That token isn't valid. Copy the whole token again (or
  create a new one), then reply **ok**."
- No clipboard tool, or the clipboard cannot be read here: ask the user to run `skipshare login`
  (or `npx -y @skipshare/cli@1 login`) in their own terminal, paste the token at the hidden prompt,
  then reply **ok**. Check with `login status --json`.

## 6. Default team

Run `team list --json`. With no teams besides the user's own, skip this question and say
"Shares will come from you personally."

Otherwise ask, as a numbered list. `1` is always "Your personal team". Then list each row except
the one where `role` is `owner` (that row is the personal team), as `<team_name> (<owner_email>)`:

```text
Which team should shares come from by default?
1. Your personal team
2. Acme (boss@acme.com)
3. Design (lead@design.com)
Reply a number, or **skip** to keep your personal team.
```

Run `config default-team <owner_email> --json`, or `config default-team personal --json` for `1`.
**skip** changes nothing.

## 7. Done: show how to send

```text
✅ SkipShare is ready.
Account: <email>
Language: <English | 日本語>
Default team: <Your personal team | team_name>
```

Then two or three things the user can say next, for example:

- "Send report.pdf to a@example.com"
- "Send the files in ./export to a@example.com, cc b@example.com, expire in 3 days"
- "Send logo.png to a@example.com from the Design team"

The sending itself is the skipshare-send skill's job. Mention that settings can be changed any time
("change my default team", "use Japanese").

## Errors

- Each error is short: what is wrong, then what to reply (**ok**, **retry**, a number, **skip**).
- No error codes or stack traces. Show the CLI's `message` only when nothing else explains it.
- **N** or "cancel" at any point: stop, say what is already installed, and that "skipshare init"
  continues from there.
