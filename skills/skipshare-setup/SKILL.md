---
name: skipshare-setup
description: Set up SkipShare for the user in one guided run, installing or updating Node.js, installing the `@skipshare/cli` client, logging in with a personal access token, then choosing the reply language and default team. Use when the user says "skipshare init", "set up SkipShare" or "install SkipShare", or when the skipshare-send skill finds the CLI missing or the user not logged in.
---

# Set up SkipShare

This skill takes a user from nothing to their first send: Node.js, the SkipShare CLI, login, then
two settings. Many users have never installed Node.js, or have one too old for the CLI, so the
agent does the work. It checks first, shows **one** confirm, and starts only after the user replies
**ok**.

## Reply language

Until the CLI is installed, reply in the language the user writes in. Once it runs, use the
`effective` value of `config language --json` (`en` English, `ja` Japanese), and after step 5 the
language the user picked. Use another language only when the user asks for one.

## 1. Check what is already there

Run these quietly, before saying anything:

- `node -v`: Node.js is ready when it prints `v20` or newer. Missing or older: it needs an install
  or an update.
- `skipshare --version`: the CLI is installed globally when this prints a version.
- `skipshare login status --json` (or `npx -y @skipshare/cli@0 login status --json` when the CLI is
  not installed but Node.js is ready): exit 0 means logged in, and `email` is the account. Exit 3
  means not logged in; keep `error.data.token_page_url` for step 4.

In the rest of this skill, `skipshare` means the global command when it is installed, and
`npx -y @skipshare/cli@0` otherwise. Always pass `--json`.

When everything is ready and logged in, skip to step 5 (settings) and say
"SkipShare is already set up as `<email>`."

## 2. One confirm

Show one short list of what will happen. Leave out the steps that are already done:

```text
Set up SkipShare:
• Node.js: install v22 (you have v18, the CLI needs 20 or newer)
• SkipShare CLI: install @skipshare/cli
• Login: you create a token in the SkipShare web app, I connect it
Then pick your language and default team.
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

Run `npm install -g @skipshare/cli@0`, then `skipshare --version`.

On a permission error (`EACCES`), do not retry with `sudo`. Use `npx -y @skipshare/cli@0` for the
rest of the setup and tell the user once: "I'll run SkipShare through npx instead of installing it."

## 4. Log in

The token is a password. **Never ask the user to paste it into the chat, and never put it on a
command line.** It goes from the user's clipboard straight to the CLI.

Say:

```text
Create a token here: <token_page_url>
Copy it, then reply **ok**. Don't paste it here.
```

`token_page_url` comes from `error.data.token_page_url` of the exit-3 `login status --json` in
step 1. When it is missing, say "in the SkipShare web app: Settings → Personal access tokens"
instead of the link.

After **ok**, pipe the clipboard into the CLI:

- macOS: `pbpaste | skipshare login --json`
- Windows (PowerShell): `Get-Clipboard | skipshare login --json`
- Linux: `xclip -o -selection clipboard | skipshare login --json`, or `wl-paste | ...` on Wayland

Never print the clipboard, echo it, or store it in a variable or file.

- Exit 0: say "✅ Logged in as `<email>`." and go on.
- Token rejected (exit 3) or empty: "❌ That token isn't valid. Copy the whole token again (or
  create a new one), then reply **ok**."
- No clipboard tool, or the clipboard cannot be read here: ask the user to run `skipshare login`
  (or `npx -y @skipshare/cli@0 login`) in their own terminal, paste the token at the hidden prompt,
  then reply **ok**. Check with `login status --json`.

## 5. Settings

These two settings show the user what the CLI can do. Ask one at a time.

### Language

Ask, as a numbered list:

```text
Which language should SkipShare use?
1. English
2. 日本語 (Japanese)
```

Run `config language en --json` or `config language ja --json`. From now on, reply in that language
(translate this skill's replies when it is `ja`). Keep **ok**, **skip**, **retry**, **N**, file
names, emails and links as they are.

### Default team

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

## 6. Done: show how to send

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
