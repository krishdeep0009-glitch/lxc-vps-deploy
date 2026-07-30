# DracoHost VPS Manager

A Discord.py bot for selling/managing **Incus LXC** VPS instances, branded as
**DracoHost** (Intel® Xeon® CPU, Incus LXC virtualization).

## Requirements

- A Linux host with [Incus](https://linuxcontainers.org/incus/) installed and
  the `incus` CLI available on PATH, running as the same user/service account
  as the bot (so it can talk to the local Incus unix socket without extra auth).
- Python 3.11+
- A Discord bot application/token with the **Server Members** and
  **Message Content** privileged intents enabled.

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# edit .env: DISCORD_TOKEN, GUILD_ID, ADMIN_ROLE_IDS, etc.

python bot.py
```

## Commands

### Admin
| Command | Description |
|---|---|
| `1create <user> <ram_mb> <cpu> <disk_gb> <expire>` | Deploy a new LXC VPS. `expire` accepts `7d`, `4w`, `2m`, `1y`, or `never`. |
| `1edit <user> <vps> [ram] [cpu] [disk] [expire]` | Reassign owner and/or resize. |
| `1delete <vps>` | Destroy the container and remove its DB record. |
| `1rename <vps> <new>` | Rename a container. |
| `1suspend <vps>` / `1unsuspend <vps>` | Pause/resume a container. |
| `1list` | List every VPS on the host. |

### User
| Command | Description |
|---|---|
| `1manage [vps]` | Interactive control panel: Start / Stop / Restart / Reinstall / Regenerate SSH / Refresh. |
| `1myvps` | List VPS you own. |
| `1info <vps>` | Detailed info about a VPS you own. |
| `1help` | Show this list in Discord. |

Note: since command names literally start with `1` (e.g. `1create`), the bot's
prefix is empty by default (`COMMAND_PREFIX=""` in `.env`) — just type the
command name directly, or mention the bot.

### Per-command help

Every command supports a `help` subcommand for detailed usage and examples,
e.g. `1create help`, `1edit help`, `1delete help`, `1rename help`,
`1suspend help`, `1unsuspend help`, `1list help`, `1manage help`,
`1myvps help`, `1info help`. Running any command with missing required
arguments (e.g. just `1create`) also shows the same help embed automatically.

## How it works

- `incus_manager.py` shells out to the `incus` CLI (`launch`, `config set`,
  `exec`, `pause`, `start`, `stop`, `restart`, `delete`, `rename`) using
  `asyncio.create_subprocess_exec`, so nothing blocks the bot's event loop.
- After every deploy/reinstall the bot runs, inside the container:
  ```bash
  apt update -y
  apt install -y nano tmate openssh-server curl wget sudo
  ```
- **Regenerate SSH** resets the root password and opens a `tmate` session
  (installing `tmate` first if missing), then DMs the user the new password
  and the tmate SSH connection string.
- `db.py` stores everything in SQLite (`aiosqlite`): owner, RAM/CPU/disk,
  OS image, IP, root password, status (`active` / `suspended` / `expired`),
  and expiry date.
- A background task (`expiry_check`, runs every `EXPIRY_CHECK_INTERVAL_MINUTES`)
  automatically pauses and marks VPS instances `expired` once their expiry
  date passes, and DMs the owner.

## Running as a service

See `dracohost.service` for a sample systemd unit. Adjust `WorkingDirectory`,
`ExecStart`, and `User` to match your deployment, then:

```bash
sudo cp dracohost.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now dracohost
```

## Security notes

- Run the bot under a user that has permission to run `incus` but is otherwise
  restricted — the bot does not sandbox arbitrary shell input, so keep
  `ADMIN_ROLE_IDS` limited to trusted staff.
- Root passwords are stored in plaintext in SQLite for retrieval, so protect
  `dracohost.db` file permissions and back it up carefully.
