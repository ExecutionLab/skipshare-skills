---
name: skipshare-send
description: Send files to people by email through SkipShare using the `@skipshare/cli` command-line client. Use when the user asks to send, share or deliver local files to one or more email addresses via SkipShare, personally or on behalf of a SkipShare team.
---

# Send files with SkipShare

SkipShare uploads files and emails the recipients a download link. This skill drives the
`@skipshare/cli` client. It never talks to the SkipShare API directly.

## How to run the CLI

Always run the CLI through npx, pinned to the major version this skill was written for, and always
pass `--json`:

```bash
npx -y @skipshare/cli@0 <command> ... --json
```

- CLI requirement: Node.js 20 or newer, `@skipshare/cli` 0.x.
- With `--json`, stdout holds exactly one JSON object. On success it is the result; a real send
  returns `folder_key` and `share_url`. On failure it is
  `{"error": {"code": "...", "message": "...", "status": 409, "data": {...}}}`; `status` is `null`
  when the CLI rejected the input before calling the server, and `data` (the numbers behind the
  error) is left out when there are none. Progress lines go to stderr; ignore them.
- With `--json` the CLI never asks a question, so it cannot hang waiting for input. It fails on the
  first problem instead.
- Decide what to do from the **exit code** and `error.code`, never from the wording of `message`.

## Before sending: check the login

Run `npx -y @skipshare/cli@0 login status --json`.

- Exit 0: logged in. `email` is the user's own SkipShare account. `default_team` is the owner email
  of the team used when `--team` is left out, or `null` when sends go out personally.
- `node` or `npx` not found, Node.js older than 20, or exit 3 (not logged in, or the token expired
  or was revoked): SkipShare is not ready on this computer. **Suggest** the setup and stop; never
  start it, install anything or open a login on your own. List only what is missing, then wait:

  ```text
  SkipShare isn't set up on this computer yet:
  • Node.js: install v22 (you have v18, the CLI needs 20 or newer)
  • Login: you create a token in the SkipShare web app, I connect it
  Then pick your language and default team.
  ```

  > Reply **ok** to set it up, then I'll send your files. Or **N** to cancel.

  Only the login missing (exit 3): show just the Login line, and add the token page
  (`error.data.token_page_url`) below the list.
  - **ok** (also yes/continue): run the skipshare-setup skill from its step 3; this reply was its
    confirm, so do not ask again. When it ends with "SkipShare is ready", go straight back to this
    share: keep every file, recipient and field the user already gave, run the dry run and show the
    card. Do not ask again for what you already have.
  - **N**: `Share cancelled. No files were sent.` Mention that **skipshare init** sets it up later.
  - The user would rather log in themselves: create a token at `error.data.token_page_url` (or in
    SkipShare under Settings → Personal access tokens), run `npx -y @skipshare/cli@0 login` in
    their own terminal and paste it at the hidden prompt, then reply **done**; check
    `login status` again and carry on with the share. In CI, set `SKIPSHARE_TOKEN` instead.

Never ask the user to paste a token into the chat, and never put a token on a command line.

## Reply language

Run `npx -y @skipshare/cli@0 config language --json` before you write anything, even a one-line
"I'll prepare this share". Then write every line in its `effective` language (`en` English, `ja`
Japanese): status and progress lines, questions, the card labels, errors and the result. The
replies in this skill are written in English; translate them when the language is `ja`.
When the CLI cannot run yet (no Node.js), reply in English.

The language the user writes in does not change this. A user who writes in Vietnamese with
`effective` `en` gets English replies. Switch only when the user tells you which language to reply
in ("reply in Vietnamese", "trả lời bằng tiếng Việt"), and keep it for the rest of the conversation.
Writing a message in a language is not such a request.

Dates and times are in this computer's local time, 24-hour: `MM/DD/YYYY HH:mm` in English
(`10/05/2026 14:30`), `YYYY/MM/DD HH:mm` in Japanese (`2026/10/05 14:30`). Drop the time when only
a day is meant, such as `Resets on 10/01/2026.` Never show a raw ISO date such as
`2026-10-01T05:00:00Z`.

Keep as they are in every language: the reply keywords (**Y**/**yes**, **N**, **ok**, **skip**,
**retry**, **details**), file names, email addresses, team names and links.

## Choosing who the share comes from

- **The user names no team:** leave `--team` out. The CLI uses the default team, or sends
  personally when none is set. The dry run reports which one (`team`).
- **"Personally" / "from me, not a team":** pass `--team personal`.
- **The user names a team** ("from the Acme team"): run
  `npx -y @skipshare/cli@0 team list --query "<name>" --json`. Each row has `team_name`,
  `owner_name`, `owner_email` and `role`. The row with `role: "owner"` is the user's own team:
  sending from it is sending personally, so treat it as `Your personal team`, never as a team. One match:
  pass `--team <owner_email>`. None or several: list what you found in one short line each and ask
  which one.
- **"My default team":** pass `--team default`.

Never pick a team the user did not clearly identify, and never guess an owner email.

## The share draft

Keep one share draft for the current request and update it turn by turn. Never ask again for a
field the user already gave, and never drop one after an error.

| Field      | Flag                           | Required | When the user said nothing          |
| ---------- | ------------------------------ | -------- | ----------------------------------- |
| Files      | positional paths               | yes      | ask                                 |
| To         | `--to` (repeat)                | yes      | ask                                 |
| CC         | `--cc` (repeat)                | no       | `None` (no flag)                    |
| Expires in | `--expires-in <n>d`            | no       | the CLI default, marked `(default)` |
| Subject    | `--subject`                    | no       | `None` (no flag)                    |
| Message    | `--message` / `--message-file` | no       | `None` (no flag)                    |
| From       | `--team`                       | no       | what the dry run reports            |

Optional means the user may leave it empty, **not** that you may skip showing it. Every field
appears on the Share card before anything is sent.

- `--to me` / `--cc me` is the logged-in user's own address. Use it for "send it to me".
- To plus CC: at most 50 addresses. Subject: up to 100 characters. Message: up to 2000 characters;
  for a multi-line message write it to a file and pass `--message-file`.
- Files must be regular, non-empty files, each path passed explicitly. No directories.
- Default expiry: run the dry run without `--expires-in`; `would_send.access_expiration` is the
  default in days. Once the user has seen it on the card and confirmed, pass that number as
  `--expires-in <n>d` on the real send so the command sends exactly what the card showed.

## Keep every reply short

- A question is at most two lines: the question, and one line on what to enter.
- Do not explain why something is missing, and do not tell the user what you will do next (dry
  run, card, confirmation). The card shows all of that when it comes; the default expiry
  appears only there and in the step 3 question.
- No examples, no reassurance ("nothing goes out until…"), no restating what the user said.
- No side notes: do not say how you found a file, what you inferred or why a step happened.
- Never point out a mismatch in what the user said ("you attached 4 but want to send 2"). Ask the
  question straight away.

## Flow

1. **Collect.** If files or recipients are missing, ask for only those, in one short message:

   > Who should receive these files?
   > Enter one or more email addresses.

   > Which files?

   A file the user drags or attaches into the chat is a file to send. This holds for images too:
   an attached image means "send this image", even when the message says nothing else. Never read
   an image's content to decide what to send: file names, a list or a previous share visible in a
   screenshot are not the files to send. Read it that way only when the user says so ("send the
   files shown in this screenshot"). "Same as before" / "giống vừa rồi" with new attachments means
   the new attachments go to the recipients, CC, team and expiry of the previous share in this
   conversation.

   Non-image files dragged into the chat arrive as their original path (`@"/Users/…/clip.mov"`):
   use that path as it is. Images attached to the chat are different: the chat keeps only a
   compressed, renamed copy (the `source:` path of the image, e.g. `…/images/30.webp`), not the
   user's original. When the agent shows the image but gives no file path at all, ask for the path
   (`I can see the image but not its file. Reply with its path.`). Before the dry run, ask once for
   all attached images together:

   > Chat images are sent as compressed copies (smaller, renamed), not your originals.
   > Reply **ok** to send the copies, or send the paths of the originals or of the folder they are in.

   **ok** (also yes/continue): use the copy paths, run the dry run and carry on with the Flow; the
   card and any storage list show the copies' names and sizes. Names or paths: find the originals
   as below and use them instead. A folder: the copies are renamed, so their names cannot be
   matched. List the images in that folder (not its subfolders), numbered with name and size, and
   ask which ones to send (`Reply with the numbers, or **all**.`). Do not ask again for the same
   images in this draft.

   Always send the original file, and take its name and size from it (`stat`). When the user gives
   a file name but no path, look for it in `~/Downloads`, `~/Desktop`, `~/Documents` and
   `~/Pictures` (`find <dir> -maxdepth 3 -name '<name>'`; on macOS `mdfind -name '<name>'` also
   works). One match: use it; the card shows the name. None: `` `<name>` not found. Check the name. ``
   Several: list them numbered with folder and size and ask which one.

   When both are missing, ask in one message:

   > Which files, and who should get them?
   > Send the file names and email addresses.

   When it is unclear which of the user's local files are meant (the user says "2 files from
   ~/Photos" and the folder has 4), ask
   directly with a numbered list, each file with its size (and a few words on what it is when the
   names alone do not tell them apart):

   > Which 2 files?
   >
   > 1. `1.jpg`: 344.5 KB
   > 2. `2.jpg`: 210.6 KB
   > 3. `3.jpg`: 309.2 KB
   >
   > Reply with 2 numbers (for example **1,3**).

   Ask only for files and recipients here. CC, subject, message and expiry come in step 3.

2. **Validate.** Run the full command with `--dry-run --json`. It uploads and sends nothing. If it
   fails, show the short error (see "Errors"), keep the draft, and wait. The user's next reply
   ("4 days", "use b@x.com instead") updates that one field; run the dry run again.
3. **Ask once about the optional fields** that are still open. A field is settled when the user
   gave a value, or declined it ("no cc", "no message", "default expiry"). "Just send it" / "that's
   all" settles all four. All settled: skip this step. Otherwise one question naming only the open
   fields, with the expiry from the dry run's `access_expiration`:

   > Add CC, a subject or a message? Expiry is 7 days (default).
   > Reply with what to change, or **skip** to continue.

   Only the expiry open: `Expiry is 7 days (default). Change it?` / `Reply with a number of days,
or **skip** to continue.` Ask this once per draft, never again after an error or a change.
   - `skip` (also `no`, `none`, `continue`) → the open fields stay `None` / default. Go to step 4.
   - Values → update the draft, run the dry run again (step 2), then step 4. Fields the reply did
     not mention stay `None` / default; do not ask about them again.
   - `yes` with no value → `What should I add?` and wait.

4. **Show the Share card** (below) built from the dry run result. Then stop and wait.
5. **Confirm.** Read the user's next reply, trimmed and lower-cased:
   - `y` or `yes` → run the same command without `--dry-run` (step 6).
   - `n`, `no` or `cancel` → reply `Share cancelled. No files were sent.` and drop the draft.
   - A change ("change expiration to 2 days", "add cc: abc@example.com", "subject: Monthly report")
     → update the draft and go back to step 2 (not step 3). The old confirmation no longer counts; show a new
     card and ask again.
   - `show files` / `show full message` → print that field in full, then show the card again.
   - Anything else ("ok", "sure", "do it", "fine") is **not** a confirmation. Reply
     `Type Y or yes to send, or N to cancel.`
6. **Send** and reply with the short success message (see "After sending").

Only send when the card for the **current** draft was shown and the user's reply to it was `y` or
`yes`. A `yes` with no card pending sends nothing: reply `Nothing to send yet.` and ask what to
share. An email that has been sent cannot be recalled.

## The Share card

Print the card inside a ` ```text ` code block so the indentation survives, exactly in this
layout: one label per line, the value indented two spaces below it. A code block shows markdown
literally, so the card contains **no markdown at all**: no `**`, no backticks, no links syntax. No
ANSI colours, no tables, no paragraphs.

```text
━━━━━━━━━━━━━━━━━━━━━━━━
📤 Share files
━━━━━━━━━━━━━━━━━━━━━━━━

Files (2)
  A.pdf
  B.pdf

To
  foo@example.com

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

Right below the code block, as ordinary chat text (so bold renders), two short paragraphs:

> **Ready to send.**
>
> Type **Y** or **yes** to confirm. **N** to cancel, or tell me what to change.

- **Files:** up to 5 names. More than 5: the first 3, then `… +N more` (`show files` lists all).
- **From:** always shown. `Your personal team` when sending personally (also for the user's own team; the CLI's own label), or the team as `<team_name> (<owner_email>)` (name from
  `team list`; the owner email alone when you have not looked it up).
- **Expires in:** `N days`, plus `(default)` when the user did not choose it.
- **Subject / Message:** `None` when empty. A long value shows at most 3 lines (about 200
  characters) followed by `… [N characters]`. The command still sends the full text.
- **Dry-run only** (the user asked for a preview, not a send): title `🧪 Share preview`, and the
  text below the block is `No files will be uploaded in dry-run mode.` instead of the confirm
  lines.

## Errors

An error reply has one job: tell the user what to type next. Write it as ordinary chat text,
**never inside a code block**, so bold and inline code render:

> ❌ [what is wrong, naming the file, address or number]
>
> [the limit, only when there is one]
>
> [what to reply]

(The `>` marks in these examples only set them apart here; do not print them.)

- The last line is always a concrete reply: `Reply **4** to use 4 days.`, `Reply **skip** to send
without it.`, or a numbered list ending `Reply with a number.` Never end on advice with no reply
  ("Suggested: use a smaller file"), and never on a question the user cannot answer by typing.
- Bold only the one number or value the user needs. File names, addresses and commands go in
  inline code.
- No guesses about the cause, no plan or quota explanations, no error code, exit code, `status`,
  raw JSON, stack trace or CLI hint text, unless the user asks for details.
- Offer two or three options only when each one really exists, as a numbered list.
- Keep the draft. The user's reply changes one field; run the dry run again and show a new card.

### Check first, then report everything at once

The CLI stops at the first problem, so a draft with three mistakes costs three round trips. Before
the dry run, check what you can yourself: every path exists, is a regular file and is not empty
(`ls -l`), every address looks like an email, To plus CC is at most 50, the subject is at most 100
characters and the message at most 2000. When several things are wrong, send one reply:

> ❌ 2 things to fix before sending:
>
> 1. `report.pdf` not found.
> 2. `abc@` is not a valid email.
>
> Reply with the correct path and address, or **skip** to leave out `report.pdf`.

### What to say for each error

Take the numbers from `error.data` when it is there, and from `error.message` otherwise. Sizes in
`data` are bytes; show them as KB/MB/GB.

| `error.code`                                         | `error.data`                                                                  |
| ---------------------------------------------------- | ----------------------------------------------------------------------------- |
| `file_size_exceeded`                                 | `file_size_limit`, `files` (each `name`, `size`)                              |
| `quota_exceeded`, `over_quota_blocked`               | `limit`, `used`, `remaining`, and `needed` (what this share needs) when known |
| `monthly_upload_new_files_exceeded`, `..._limit_...` | `remain` (files left), `limit`, `resetAt` when the server sends it            |
| `access_expiration_exceeded`                         | `max_expiration_days`                                                         |
| `recipient_emails_not_allowed`                       | `invalid_recipients`, `invalid_cc_emails` (from the server)                   |

| Error                                        | Line 1 (and 2)                                                                                        | Last line                                                                                      |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `file_not_found` (missing)                   | `report.pdf` not found.                                                                               | Reply with the correct path, or **skip** to send without it.                                   |
| `file_not_found` (a folder)                  | `photos` is a folder. Only files can be sent.                                                         | Reply **zip** to send it as `photos.zip`, or list the files inside to send.                    |
| empty file                                   | `notes.txt` is empty (0 bytes).                                                                       | Reply **skip** to send without it, or with another path.                                       |
| `file_size_exceeded`                         | `video.mp4` is **55 MB**. / The limit is **5 MB** per file. (Several files: list each with its size.) | Reply **skip** to send the other files. When no file would be left: Reply with a smaller file. |
| invalid email                                | `abc@` is not a valid email.                                                                          | Reply with the correct address.                                                                |
| `recipient_emails_not_allowed`               | This team only sends to approved addresses. Not allowed: `x@gmail.com`.                               | 1. Remove `x@gmail.com` 2. Send from your personal team — Reply with a number.                 |
| `receivers_limit_exceeded`                   | **55** recipients. / The limit is **50** (To + CC).                                                   | Reply with the addresses to remove.                                                            |
| subject / message too long                   | The subject is **130** characters. / The limit is **100**.                                            | Reply with a shorter subject.                                                                  |
| `access_expiration_exceeded`                 | Expiration is too long. / Your plan allows up to **4 days**.                                          | Reply **4** to use 4 days, or a smaller number.                                                |
| `access_expiration_invalid`                  | `soon` is not a number of days.                                                                       | Reply with a number of days, from 1 to the plan's maximum.                                     |
| `monthly_upload_new_files_exceeded`          | Only **3** files left this month. You are sending 5.                                                  | Reply with the 3 files to send now.                                                            |
| `monthly_upload_limit_exceeded`              | This month's **10** files are used up. (Add `Resets on [date].` from `resetAt`.)                      | Nothing can be sent until the reset. For more, upgrade in the SkipShare web app.               |
| `quota_exceeded` / `over_quota_blocked`      | Not enough storage: **8 MB** needed, **5 MB** left. (No `needed`: Storage is full.)                   | See "Storage full" below.                                                                      |
| not logged in (exit 3)                       | See "Before sending: check the login": suggest the setup and wait for **ok**.                         |                                                                                                |
| `team_not_found`, `default_team_not_set`     | See "Team errors" below.                                                                              |                                                                                                |
| network (exit 10)                            | Can't reach SkipShare.                                                                                | Check your connection, then reply **retry**.                                                   |
| rate limit (8), server (9), upload twice (6) | SkipShare is busy right now. (`maintenance`: SkipShare is under maintenance.)                         | Reply **retry** in a few minutes.                                                              |
| `node_version_unsupported`                   | SkipShare CLI needs Node.js **20** or newer.                                                          | Install Node.js 20+, then reply **retry**.                                                     |
| `cli_upgrade_required` (12)                  | This SkipShare CLI version is no longer supported.                                                    | Update the `skipshare-send` skill, then reply **retry**.                                       |
| `idempotency_in_progress` (7)                | Do not reply yet: wait a minute and run the same command yourself once.                               |                                                                                                |
| anything else                                | Couldn't send: [first sentence of `error.message`].                                                   | Reply **retry**, or **details** to see the full error.                                         |

Only the Storage-full, monthly-limit and not-logged-in errors leave nothing to fix in the draft.
Keep the draft anyway: after **retry** or **done**, run the dry run again and show the card.

### Replies to an error

- A number picks that option. **skip** removes the named file(s). **zip** zips the folder into one
  file next to it and uses that. The draft changes, so run the dry run and show a new card.
- **retry** / **done** runs the step that failed again: the dry run, or the real send when the
  real send failed after the user confirmed (it is the identical command, so it never sends twice).
- **details** shows `error.code` and the full `error.message`, then the same last line again.
- A new value ("4 days", "use b@x.com") updates that field, as in the Flow.

### Storage full: list the files to drop

On `quota_exceeded` with `needed`, the user can either free space or send fewer files. Show both.
Number every file in the draft with its size (from the local check, in MB/KB like the CLI, draft
order), and say how much to remove (`needed − remaining`):

> ❌ Not enough storage: these files need **11.9 MB**, but only **6.9 MB** is left. Remove at
> least **5 MB**.
>
> 1. `1.webp`: 3.2 MB
> 2. `2.webp`: 2.8 MB
> 3. `css-handbook.pdf`: 1.1 MB
>
> Reply with the numbers to remove (for example **1,2,3,...**), or delete old files in the SkipShare web
> app and reply **retry**.

Numbers (commas or spaces, or file names) drop those files from the draft; run the dry run again and show the card,
or this error again with the new numbers if it still does not fit. Every file removed: reply
`At least one file must stay.` and show the list again. Without `needed` (storage already full) or
on `over_quota_blocked`, fewer files cannot help: show only the web app line.

When you cannot read the draft files' names and sizes, do not list them or say why. Show the error
with one generic line:

> ❌ Not enough storage for these files. Send fewer files, or delete old files in the SkipShare web
> app and reply **retry**.

### Team errors: list the teams to pick from

On `team_not_found` or `default_team_not_set`, do not explain what might have happened. Run
`npx -y @skipshare/cli@0 team list --json` first, then reply with one short line and a numbered
list: `Your personal team` first, then every other team as `team_name (owner_email)`, using the owner email
alone when `team_name` is null. Leave out the row with `role: "owner"`: it is the user's own team,
which is `Your personal team` already.

> ❌ Can't send from team `owner@example.com`: you are no longer a member.
>
> Send from:
>
> 1. Your personal team
> 2. Acme (owner@acme.com)
> 3. Design (lead@design.io)
>
> Reply with a number.

With no other teams, list only `1. Your personal team`. When the user picks, set the draft's From and show a
new card; the pick still needs `y`/`yes`. Pass `--team personal` or `--team <owner_email>`.

**When the broken team is the default team** (the user passed no `--team`, or `--team default`),
every later send without a team fails the same way. So fix the default first: title the list
`Pick a new default team:` and end with `Reply with a number. It becomes your default.`

> ❌ Your default team `owner@example.com` is gone: you are no longer a member.
>
> Pick a new default team:
>
> 1. Your personal team
> 2. Acme (owner@acme.com)
>
> Reply with a number. It becomes your default.

On the pick, run `npx -y @skipshare/cli@0 config default-team <owner_email|personal> --json`,
then rerun the dry run without `--team` and show the new card. Change the default only in this
case or when the user asks.

## After sending

Keep it short. Do not print the JSON. Same rule as the card: inside a ` ```text ` block, no
markdown.

```text
✅ Files shared successfully.

Recipients
  2

Files
  3

Expires in
  4 days (until 10/05/2026 14:30)

Download URL
  https://…/files/abc123
```

When the language is `ja`, use exactly this wording:

```text
✅ ファイルの共有が完了しました。

宛先
  2 名

ファイル
  3 件

有効期限
  4 日間（2026/10/05 14:30 まで）

ダウンロード URL
  https://…/files/abc123
```

The date in brackets is when the link stops working, in this computer's local time. SkipShare sets
it to the send time plus the chosen days, so compute it right after the send succeeds (`N` is the
days sent with `--expires-in`):

```bash
node -e "const d=new Date(Date.now()+N*864e5),p=n=>String(n).padStart(2,'0'),t=p(d.getHours())+':'+p(d.getMinutes());console.log('en: '+p(d.getMonth()+1)+'/'+p(d.getDate())+'/'+d.getFullYear()+' '+t);console.log('ja: '+d.getFullYear()+'/'+p(d.getMonth()+1)+'/'+p(d.getDate())+' '+t)"
```

Use the `en` line (`MM/DD/YYYY HH:mm`) for English and the `ja` line (`YYYY/MM/DD HH:mm`) for
Japanese. The time is 24-hour.

Write `1 day` / `1 日間` for one day.

`Download URL` is `share_url` from the result: the download page. Whoever opens it must still
verify a recipient email by OTP, so it is safe to show. SkipShare also emails the recipients; do
not claim the email has arrived. Exit 11 (saved, email failed) replaces the first line with
`⚠️ Shared, but the email to recipients failed.` (`ja`: `⚠️ 共有は完了しましたが、宛先へのメール送信に失敗しました。`)
and adds, after the block: `Send them the link yourself.` (`ja`: `お手数ですが、ダウンロード URL を宛先に直接お送りください。`)
Never offer **retry** here: the share exists, and sending again makes a second one.

## Handling each failure (for you, not for the user)

Use this table to decide what to do. Tell the user only the short error from "Errors"; the codes
below stay internal.

| Exit | Meaning                     | What to do                                                                                                                                                                                                                                                                   |
| ---- | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0    | Share saved                 | Reply with the success message, including `share_url` as the link.                                                                                                                                                                                                           |
| 2    | Invalid input               | Show the short error; the user's reply fixes the draft. Do not change their values yourself. `recipient_emails_not_allowed`: the team only accepts certain addresses; say which address is not allowed and ask.                                                              |
| 3    | Not authenticated           | Ask the user to log in again, as described above.                                                                                                                                                                                                                            |
| 4    | Team or permission problem  | `team_not_found`: the team does not exist, or the user was removed from it or left it (the server does not say which). `default_team_not_set`: `--team default` was used but none is configured. List the teams as in Team errors. Do not fall back to personal on your own. |
| 5    | Plan limit                  | Quota, file size or monthly limit reached. `error.message` names every file that is too big. Tell the user. Do not retry.                                                                                                                                                    |
| 6    | Upload failed               | Run the same command again once. If it fails again, report it.                                                                                                                                                                                                               |
| 7    | Idempotency conflict        | `idempotency_in_progress`: the same send is still running; wait a minute and run the same command. `idempotency_key_reused`: a custom `--idempotency-key` was reused for different content; use a new key.                                                                   |
| 8    | Rate limited                | The CLI already waited and retried. Wait a few minutes before trying again.                                                                                                                                                                                                  |
| 9    | Server error                | Run the same command again later. `maintenance`: tell the user SkipShare is under maintenance.                                                                                                                                                                               |
| 10   | Network error               | Check the connection, then run the same command again.                                                                                                                                                                                                                       |
| 11   | Saved, but the email failed | The share exists. Tell the user the notification email failed. **Do not send again**, or the recipients get a second share.                                                                                                                                                  |
| 12   | CLI too old                 | The server no longer accepts this CLI version. Ask the user to update the skill, as in Errors. Do not retry until they reply.                                                                                                                                                |
| 130  | Cancelled                   | `aborted`: stopped before anything was sent; do not retry. `aborted_unknown`: stopped while the share was being created, so it may exist; run the identical command once to see the result (it never sends twice).                                                           |

**Running the same command again is safe.** The CLI remembers each send for 24 hours. If the first
attempt actually reached the server, the second one returns the saved result instead of creating a
second share. That only holds for the identical command: same files, recipients, team, subject,
message and expiry.
