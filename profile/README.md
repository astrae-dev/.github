# Astrae

Commands in Python. Gateway in Rust.

Astrae is a Discord framework for bots that have to stay up. You write commands, events, cache policy, and SQL in Python. The websocket, HTTP client, and rate limiter run in a Rust extension in the same process. One process is enough to start. Redis is only needed if you split the gateway from the workers, or share cache across clusters.

Version `0.1.0`. Python 3.11 or newer. Proprietary. The Python package does not connect to Discord without the compiled extension.

[Site](https://astrae.sh) · [Source](https://github.com/astrae-dev/astrae) · [Quickstart](https://astrae.sh/getting-started/quickstart/) · [Docs in this repo](https://github.com/astrae-dev/.github)

## Install

Build from source. A Rust toolchain and Maturin are required.

```bash
python -m pip install maturin
git clone https://github.com/astrae-dev/astrae.git
cd astrae
maturin develop
```

Redis, database drivers, and the cluster console ship with the default install. Redis itself is a separate service. Host CPU and memory metrics need `psutil`:

```bash
pip install '.[metrics]'
```

## A small bot

Prefix commands that read message text need the Message Content intent in the Developer Portal and on the client. Slash commands do not.

```python
from typing import Annotated

from astrae import Client, CommandContext, Intents, StringOption

bot = Client(
    command_prefix="!",
    intents=Intents.DEFAULT | Intents.MESSAGE_CONTENT,
)

@bot.command(name="echo", description="Repeat a message")
async def echo(
    ctx: CommandContext,
    message: Annotated[str, StringOption(description="Text to repeat")],
) -> None:
    await ctx.reply(message)

bot.run()  # reads DISCORD_TOKEN when no token is passed
```

Put async setup in `setup_hook()`, not `on_ready`. Shards can ready more than once. Global slash sync is on. Turn it off with `sync_commands=False` while you iterate.

Coming from discord.py, the names will look familiar. Astrae is not a fork. Do not rename the import and expect the bot to boot.

## What you write

`from astrae import …` is the public contract. `UNSET` means omit the key. `None` means JSON null.

### Commands

Prefix, slash, hybrid, and user or message menus. One callback can serve more than one surface. A group does not have to be all slash or all prefix: `@group.command` stays prefix-only unless you pass `slash=True`, and `@group.app_command` publishes that command without a second function.

`Annotated` options are what Discord shows. `Author`, `CurrentUser`, `CurrentMember`, `CurrentGuild`, `CurrentChannel`, and `CurrentRole` are injected from the invoke. They are not options. Guild ones are `None` in a DM.

Raise a `CommandError` subclass for an expected rejection. Astrae maps those to a styled reply.

### Context

`reply` is the first response. `defer` if the work is slow. `followup` after that. Component handlers use `update` or `defer_update`.

Styled helpers mention the invoker and use the client theme: `success`, `error`, `deny`, `neutral`, `approve`, `warn`. Colors can come from `ASTRAE_COLOR_APPROVED`, `ASTRAE_COLOR_WARNED`, `ASTRAE_COLOR_DENIED`, and `ASTRAE_COLOR_NEUTRAL`.

`prompt` and `confirm` wait on Approve or Decline. `paginate` walks strings, embeds, fields, or files. Only the invoker can press the buttons.

On a slash or component interaction, `ctx.approve`, `ctx.warn`, and `ctx.neutral` defer if needed, send a followup embed, mention the user, and delete after a delay. The embed variants stay public and do not mention. Prefix commands keep the older styled reply.

### Components

`View`, `button`, `ActionRow`, `SelectMenu`, `Modal`, `TextInput`. Components V2: `Container`, `Section`, `TextDisplay`, `Thumbnail`, `MediaGallery`, `Separator`.

Persistent views need `timeout=None`, stable `custom_id`s, and `bot.add_view(...)` on every start. Do not put secrets in `custom_id`. Callbacks receive `CommandContext`, same as a command.

### Cogs and extensions

A cog groups commands and listeners. Override `cog_check` and `cog_command_error` when the defaults are wrong. An extension is a module with `setup(client)`, sync or async. Reload tears the old module down first. Keep setup safe to run twice. The cluster console can `load`, `unload`, and `reload` by name.

### Events and loops

`@bot.listen("message_create")`. Names are lowercase. `EventContext` has the shard, delivery id, and the previous snapshot on updates. That snapshot is this process's last look, not a database.

`@loop(minutes=1)` starts and cancels with the client. `Scheduler.cron` is wall-clock, five fields, no ranges. Redis locks are best-effort. Jobs still need to be idempotent.

Listeners sit on the dispatch path. Do not `time.sleep` there.

## Cache

Memory, or Redis if you share it. Users are stored once. A member record is nick, roles, joined_at, guild avatar, guild banner, and a pointer. Opcode 8 still chunks. A cache miss is not "Discord does not have this."

Full member warmup is opt-in. It needs the Guild Members intent, Redis, and a memory ceiling if you do not want the process to grow without a limit.

## Cluster

```bash
astrae cluster mybot --clusters 2 --shards 8
```

`--shards auto` calls `/gateway/bot`. Gateway processes keep the sockets. Workers take events over Redis. A dead child is restarted. Exit 78 (`InvalidToken`) is not. The console can restart a child, reshard, and reload extensions.

## Dashboard

Cluster 0 serves the metrics page on port `9100`, on all interfaces by default. Set `ASTRAE_METRICS_HOST=127.0.0.1` to keep it on loopback, and set `ASTRAE_METRICS_TOKEN` before leaving the port open.

| Path | What it shows |
| --- | --- |
| `/` | Metrics, system CPU and memory, bot process usage |
| `/dashboard/commands` | Command runs, without payloads |
| `/dashboard/operations` | Commands, REST, components, loop stalls, errors |
| `/dashboard/guilds` | Guilds, shards, fleet memory |
| `/metrics` | Prometheus text |

## SQL and observe

`client.db` is for your rows, not Discord's cache. SQLite, Postgres, or MySQL. Pass `dsn=` or set `DATABASE_URL`.

`LoopMonitor` is on. `setup_sentry()`, `setup_tracelet()`, and `setup_posthog()` belong in `setup_hook()`. Load Jishaku only where owners are people you trust.

## Further reading

[Commands](https://astrae.sh/guides/commands/) · [Interactions](https://astrae.sh/guides/interactions/) · [Events](https://astrae.sh/guides/events/) · [Cache](https://astrae.sh/guides/caching/) · [Sharding](https://astrae.sh/guides/sharding/) · [Metrics](https://astrae.sh/operations/metrics/) · [API](https://astrae.sh/reference/)

Astrae is proprietary software. Discord is a trademark of Discord Inc. Astrae is not affiliated with it.
