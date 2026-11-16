# hermes-team-skills

Reusable [Hermes Agent](https://github.com/NousResearch/hermes-agent) skills for running a
multi-bot team on one machine: each Hermes profile is an agent with its own canonical
**Bot Chat**, and these skills cover the plumbing around that — rotating a bot's main
conversation, handing work to another bot, and keeping task boards in sync.

## Skills

| Skill | What it does |
|---|---|
| [`bot-chat-rotation`](skills/bot-chat-rotation/) | Archive a bot's main conversation, create a same-titled replacement, and make its first turn load the work state from the old one. |

## bot-chat-rotation

The interesting part is not the archiving — it is **how the new conversation is told to pick
up work**. The handoff does *not* read the old transcript end to end. It walks a fixed short
ladder:

1. Load the discipline/orientation skill.
2. Read the team working-agreement sections that define roles and boundaries.
3. Read the bot's own line-state card — a three-field status file (north star / in progress /
   blockers) that is the single source of truth for that line.
4. Only if the card is missing something, or is older than the conversation's last message,
   page the old conversation by id.

Then it works. No acknowledgement message, no roll-call. An idle bot answers with one line.

Design notes that cost real debugging to learn:

- **Bot Chat identity is `(profile, title == "Bot Chat")`** — there is no id pointer. Create the
  replacement with `session.create {title, hidden:true}` and set the title immediately; rows are
  created lazily and a new row without the title silently opens a second chat.
- **Archive first, then title the new row.** The title transaction nulls the archived row's
  title, handing the registered name to the new one. Consequence: the old conversation is only
  reachable *by id* afterwards — the handoff message must carry it.
- **The first message is the handoff.** Skip it and `prompt.submit` lands, but the new Bot Chat
  starts empty. `session.create` always returns a fresh prompt budget, so a brand-new chat can
  *look* healthy while inheriting nothing.
- **Talk to the desktop backend** (the `hermes serve` process the Electron app starts), over its
  localhost WS JSON-RPC. `hermes -p X chat -q` creates a different session that clicking the bot
  will never land on.
- **Verify from the database, not the acknowledgement.** A successful RPC ack is not a
  successful rotation: re-read `profiles.list.canonical_session`, confirm exactly one row holds
  the title, and check the old row is `archived=1 / title IS NULL`.

## Requirements

- Hermes Agent with the desktop app (or any `hermes serve` backend) running locally.
- Python with `websockets`. The script re-execs into the Hermes venv when it can find it;
  set `HERMES_PYTHON` to point it at one explicitly, or `HERMES_DESKTOP_URL` /
  `HERMES_SESSION_TOKEN` to override endpoint discovery.

## Install

Each profile has its own skills directory and they do **not** inherit from each other. To make a
skill available to a bot, copy its folder into that profile's `skills/`:

```
<HERMES_HOME>/profiles/<profile>/skills/<category>/<skill>/
```

The rotation script defaults to **the profile it lives in** — the bot you install it under is the
bot it rotates. Pass profile names, or `--all`, to rotate more than one.

## Usage

```bash
python scripts/rotate_bot_chats.py --dry-run      # show what would be touched
python scripts/rotate_bot_chats.py                # rotate this profile
python scripts/rotate_bot_chats.py research ops   # named profiles
python scripts/rotate_bot_chats.py --all          # every profile on the machine
```

Run `--dry-run` first. The script takes only `--dry-run`, `--all`, and `--no-prompt`; unknown
arguments are ignored, not rejected — so a stray flag behaves like a bare run and *will* rotate.

## Caveats

- A busy session's in-flight turn is interrupted by `prompt.submit`. Check turn leases first, or
  let the bot go idle.
- On Windows, the system `python` may re-exec and return `rc=0` with no output. That is not
  success — run the script with the Hermes venv interpreter directly and read its verification
  output.
- Prefer targeted patches over rewriting a state card wholesale; a rejected write is not an
  updated card.

## License

MIT
