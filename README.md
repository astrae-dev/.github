# Astrae

Commands in Python. Gateway in Rust.

Astrae is a Discord framework for bots that have to stay up. You write commands, events, cache policy, and SQL in Python. The websocket, HTTP client, and rate limiter run in a Rust extension in the same process. One process is enough to start. Redis is only needed if you split the gateway from the workers, or share cache across clusters.

Version `0.1.0`. Python 3.11 or newer. Proprietary. The Python package does not connect to Discord without the compiled extension.

[Site](https://astrae.sh) · [Source](https://github.com/astrae-dev/astrae) · [Quickstart](https://astrae.sh/getting-started/quickstart/)

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

On a slash or component interaction, `ctx.approve`, `ctx.warn`, and `ctx.neutral` defer if needed, send a followup embed, mention the user, and delete after a delay. The embed variants (`approve_embed`, `warn_embed`, `neutral_embed`) stay public and do not mention. Prefix commands keep the older styled reply.

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

Defaults are mixed on purpose: a bounded user and member cache, short-lived messages, invites that expire, presences off. Full member warmup is opt-in. It needs the Guild Members intent, Redis, and a memory ceiling if you do not want the process to grow without a limit.

```python
from astrae import CacheConfig, CachePolicy, Client, Intents

bot = Client(
    cache_redis="redis://127.0.0.1:6379/2",
    intents=Intents.DEFAULT | Intents.GUILD_MEMBERS,
    cache=CacheConfig(
        users=CachePolicy(size=None, ttl=None),
        members=CachePolicy(size=None, ttl=None),
    ),
    cache_all_members_after_boot=True,
    member_warmup_memory_limit_mb=1024,
)
```

## Cluster

A small bot holds every shard in one process. Split when one process is the wrong failure domain. Gateway processes keep the sockets. Workers take shard-affine events over Redis. The supervisor assigns contiguous ranges, restarts a child that dies, and does not restart exit 78 (`InvalidToken`).

```bash
astrae cluster mybot --clusters 2 --shards 8
```

`--shards auto` calls `/gateway/bot`. Children receive `ASTRAE_CLUSTER_ID`, `ASTRAE_SHARD_IDS`, and the rest. `ClusterContext.from_env()` reads them. The console can restart a child, reshard, and reload extensions. On a small terminal it drops to logs only.

## Dashboard

After `bot.run()`, cluster 0 serves the metrics page on port `9100`. It listens on all interfaces (`0.0.0.0`) by default. Set `metrics_host` or `ASTRAE_METRICS_HOST` to `127.0.0.1` to keep it on loopback. Set `metrics_token` or `ASTRAE_METRICS_TOKEN` before leaving the port open.

Other clusters publish telemetry. They do not open their own dashboard unless `ASTRAE_METRICS_PER_CLUSTER` is set.

| Path | What it shows |
| --- | --- |
| `/` | Metrics, system CPU and memory, bot process usage |
| `/dashboard/commands` | Command runs, without payloads |
| `/dashboard/operations` | Commands, REST, components, loop stalls, errors |
| `/dashboard/guilds` | Guilds, shards, fleet memory |
| `/api/health` | Fleet JSON |
| `/metrics` | Prometheus text |

The page refreshes in place. It does not reload the document. Host stats need `psutil`.

## SQL

Gateway events and the Discord cache do not go here. `client.db` is for your rows. URLs: `sqlite:///bot.db`, `postgres://...`, `mysql://...`. Pass `dsn=` or set `DATABASE_URL`. `Migration` and `Migrator` handle schema.

## Observe

`LoopMonitor` is on by default. `setup_sentry()` and `setup_tracelet()` read `SENTRY_DSN` and `TRACELET_DSN`. `setup_posthog()` sends `command_used`. Call these from `setup_hook()`.

Load Jishaku only where owners are people you trust. Eval runs as the bot. Non-owners are dropped. With Redis telemetry, `jsk` can report the fleet.

Voice names exist. They are not a working music stack.

## Further reading

The site has the long guides: [commands](https://astrae.sh/guides/commands/), [interactions](https://astrae.sh/guides/interactions/), [events](https://astrae.sh/guides/events/), [cache](https://astrae.sh/guides/caching/), [sharding](https://astrae.sh/guides/sharding/), [metrics](https://astrae.sh/operations/metrics/), and the [public API](https://astrae.sh/reference/).

Astrae is proprietary software. Discord is a trademark of Discord Inc. Astrae is not affiliated with it.
