# SkipShare skills

> **Dev branch.** These skills drive the dev build of the CLI, which talks to the dev server and is
> installed from `https://te-fsharing-dev-cli.s3.ap-northeast-1.amazonaws.com/skipshare-cli-latest.tgz`, not from npm. The release skills are on `main`.

Agent skills that let coding agents (Claude Code, Cursor, Codex and others) set up
[SkipShare](https://www.npmjs.com/package/@skipshare/cli) and send files by email for you. You ask
in plain words ("send report.pdf to a@example.com"); the agent drives the `@skipshare/cli`
command-line client, shows you what it is about to send, and sends only after you confirm.

| Skill             | What it does                                                                                      |
| ----------------- | ------------------------------------------------------------------------------------------------- |
| `skipshare-setup` | Guided setup: installs Node.js and the CLI, logs you in, then sets your language and default team |
| `skipshare-send`  | Sends local files to email recipients, personally or as one of your SkipShare teams, via the CLI  |

## Quick start

1. Install the skills (see [Install](#install)).
2. Set up first: tell your agent **skipshare init** (or call the skill directly, see
   [Using the skills](#using-the-skills)).
3. Reply **ok** to the one confirm, create a token in the web app when asked, copy it, reply **ok**.
4. Pick your language and default team.
5. Send: "Send report.pdf to a@example.com".

## Install

With the [skills](https://www.npmjs.com/package/skills) installer, which works with most agents:

```bash
npx skills add https://github.com/ExecutionLab/skipshare-skills/tree/dev
```

The installer asks which agents and which scope. To install for Claude Code and Codex in every
project without the questions:

```bash
npx skills add https://github.com/ExecutionLab/skipshare-skills/tree/dev -g -a claude-code -a codex
```

Update to the latest version later (only these two skills):

```bash
npx skills update skipshare-send skipshare-setup -g
```

To pin a released version, add the git tag: `ExecutionLab/skipshare-skills#v1.0.0`.

Or copy by hand: each folder under `skills/` is one skill (`SKILL.md` plus its `evals/`). For
Claude Code, copy them into `~/.claude/skills/` (all projects) or `.claude/skills/` (one project):

```bash
git clone git@github.com:ExecutionLab/skipshare-skills.git
cp -R skipshare-skills/skills/* ~/.claude/skills/
```

For other agents, copy the folders into that agent's skills directory.

## Using the skills

Run **skipshare-setup** once, then use **skipshare-send** for every share. Start a new session
after installing or updating, so the agent loads the skills.

| Agent       | Set up             | Send                                          |
| ----------- | ------------------ | --------------------------------------------- |
| Claude Code | `/skipshare-setup` | `/skipshare-send report.pdf to a@example.com` |
| Codex       | `$skipshare-setup` | `$skipshare-send report.pdf to a@example.com` |
| Any agent   | "skipshare init"   | "Send report.pdf to a@example.com"            |

Calling the skill by name is the surest way. Plain requests work too: the agent picks the skill
from what you ask. If you send before setting up, the send skill lists what is missing and offers
the setup; nothing is installed until you reply **ok**.

## Setup: `skipshare init`

Say **skipshare init** (or "set up SkipShare"). The agent:

1. **Checks** Node.js (20 or newer), the `skipshare` command and your login. Nothing changes yet.
2. **Shows one confirm** listing only what is missing, for example:

   ```text
   Set up SkipShare:
   • Node.js: install v22 (you have v18, the CLI needs 20 or newer)
   • SkipShare CLI: install @skipshare/cli
   • Login: you create a token in the SkipShare web app, I connect it
   Then pick your language and default team.
   ```

   It starts only after you reply **ok**.

3. **Installs Node.js** with the tool you already have: nvm, Homebrew or winget. It never runs
   `sudo` or asks for a password. Without one of those tools it links you to
   [nodejs.org](https://nodejs.org) and continues when you reply **retry**.
4. **Installs the CLI** dev build with `npm install -g https://te-fsharing-dev-cli.s3.ap-northeast-1.amazonaws.com/skipshare-cli-latest.tgz`, or updates an
   installed one with `skipshare update`. If npm has no permission for global installs, it asks
   you to fix the npm prefix first.
5. **Logs you in.** It gives you the link to the token page of your SkipShare web app. You create a
   personal access token, copy it, and reply **ok**. The agent pipes your clipboard straight into
   `skipshare login` (`pbpaste` on macOS, `Get-Clipboard` on Windows, `xclip` or `wl-paste` on
   Linux), so the token never appears in the chat. If the clipboard cannot be read, it asks you to
   run `skipshare login` in your own terminal instead.
6. **Sets your language** (English or 日本語) and **default team**, the team your shares come from
   when you name none. Reply **skip** to keep sending personally.
7. **Shows a few things to try** next.

Already set up? It says so and goes straight to the settings. Run it again any time to change them.

## Sending files

Ask in your own words:

- "Send report.pdf to a@example.com"
- "Send the files in ./export to a@example.com, cc b@example.com, expire in 3 days"
- "Send logo.png to a@example.com from the Design team"
- "Send these files personally, subject: Q3 report"

What the agent does:

- **Checks your setup first.** If Node.js or the login is missing, it lists what is missing and
  offers the setup. Nothing is installed until you reply **ok**; then it sends the files you asked for.
- **Picks who the share comes from:** your default team, or personally when none is set, unless you
  name a team. If the name matches no team or several, it asks.
- **Asks once about optional fields** you did not mention (CC, subject, message, expiry). Reply
  **skip** to leave them empty.
- **Shows one Share card** before anything is sent:

  ```text
  ━━━━━━━━━━━━━━━━━━━━━━━━
  📤 Share files
  ━━━━━━━━━━━━━━━━━━━━━━━━

  Files (1)
    report.pdf

  To
    a@example.com

  CC
    None

  From
    Your personal team

  Expires in
    7 days (default)

  Subject
    None

  Message
    None

  ━━━━━━━━━━━━━━━━━━━━━━━━
  ```

  Type **Y** or **yes** to send. A sent email cannot be recalled, so nothing goes out before that.
  Reply with a change instead ("subject: Report") and it shows a new card.

- **Reports the result** with the share's download link.

### Files attached in the chat

The agent sends files from your disk. Files you drag into the chat as attachments work too, with
one catch: chat apps turn **images** into compressed, renamed copies. The agent tells you so and
asks you to reply **ok** to send the copies, or to give the paths of the originals or of the folder
they are in. Other files (PDF,
video, archives) keep their original path and are sent as they are.

### Errors

Errors say what is wrong and end with what to reply: a value, **skip**, **retry** or a number to
pick. Problems the agent can check before uploading (too many files, file too large, expiry over
your plan's limit, not enough storage) are reported together, so you fix them in one go. If the
share was saved but the notification email failed, it does not send it a second time.

## Language

Replies follow the CLI's language setting (`skipshare config language`, English or Japanese),
whatever language you type in. Ask for another one in the conversation ("reply in Vietnamese") to
switch for the rest of it. Reply keywords (**Y**, **ok**, **skip**, **retry**), file names, emails
and links stay as they are.

## Requirements

- Node.js 20 or newer. `skipshare init` installs or updates it.
- A SkipShare account.
- A personal access token, created in the web app under **Settings → Personal access tokens**.
  `skipshare init` walks you through it.

`skipshare-send` runs the global `skipshare` command, so the dev build must be installed.

## Security

- The agent never asks you to paste a token into the chat and never puts one on a command line.
  Do not paste one even if asked.
- The CLI checks a token with the server before saving it, and stores it in your OS keychain (or a
  file readable only by you when no keychain is available).
- In CI or other non-interactive environments, set the `SKIPSHARE_TOKEN` environment variable
  instead of logging in.
- Nothing is sent without your **Y** / **yes** on the Share card.

## Troubleshooting

| Problem                          | What to do                                                                              |
| -------------------------------- | --------------------------------------------------------------------------------------- |
| "Not logged in" or token expired | Say **skipshare init**, or run `skipshare login` in your terminal                       |
| Node.js could not be installed   | Install the LTS from [nodejs.org](https://nodejs.org), then reply **retry**             |
| `EACCES` on `npm install -g`     | Fix the npm prefix (see the npm EACCES guide), then reply **retry**                     |
| Clipboard login does not work    | Run `skipshare login` in your own terminal and paste the token at the hidden prompt     |
| Wrong team on the card           | Name the team in your request, or change the default: "change my default team"          |
| Replies in the wrong language    | "use Japanese" / "use English" changes the setting; "reply in …" changes this chat only |

## Repository layout

```text
skills/
  skipshare-setup/
    SKILL.md            instructions the agent follows
    evals/evals.json    test prompts with expected behaviour
  skipshare-send/
    SKILL.md
    evals/evals.json
```

The evals describe expected behaviour case by case (a prompt, the expected output, checkable
expectations). Use them to check a change to a skill before releasing it.

## Versioning

Skills are versioned with git tags (`v1.0.0`). A skill release pins one major version of
`@skipshare/cli` (currently `@0`). When the CLI makes a breaking change, the skills get a new major
version too.

## License

Copyright 2026 Intelligent T&E Inc. Licensed under the [Apache License 2.0](LICENSE), the same
license as `@skipshare/cli`.
