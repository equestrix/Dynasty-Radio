import discord
from discord.ext import commands, tasks
import random
import json
import os
from pathlib import Path
from datetime import datetime
from zoneinfo import ZoneInfo

TOKEN = os.getenv("DISCORD_TOKEN")

CHANNEL_ID = 1569773854610563213

TIMEZONE = ZoneInfo("America/Denver")

RADIOS = [
    "KTR482",
    "VNX731",
    "DLM204",
    "RQP915",
    "HFS368",
    "ZCW527",
    "BGT641",
    "MJK803",
    "XPL176",
    "NVR459",
    "QTD682",
    "FKA317",
    "WMS940",
    "CGH251",
    "LZX784",
    "PRB536",
    "TYF109",
    "SDN628",
    "JQV473",
    "EHK852",
]

STATE_FILE = Path("dynasty_radio_state.json")

POST_HOUR = 11
POST_MINUTE = 0

intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(
    command_prefix="!",
    intents=intents,
    case_insensitive=True
)

state = {
    "current_radio": None,
    "message_id": None,
    "last_scheduled_post_date": None
}


def load_state():
    global state

    if STATE_FILE.exists():
        try:
            loaded = json.loads(
                STATE_FILE.read_text(encoding="utf-8")
            )

            if isinstance(loaded, dict):
                state.update(loaded)

        except Exception as exc:
            print(f"Could not load state file: {exc}")


def save_state():
    try:
        STATE_FILE.write_text(
            json.dumps(state, indent=2),
            encoding="utf-8"
        )

    except Exception as exc:
        print(f"Could not save state file: {exc}")


def pick_new_radio():
    current = state.get("current_radio")

    choices = [
        radio
        for radio in RADIOS
        if radio != current
    ]

    if choices:
        return random.choice(choices)

    return random.choice(RADIOS)


async def get_radio_channel():
    channel = bot.get_channel(CHANNEL_ID)

    if channel is not None:
        return channel

    try:
        return await bot.fetch_channel(CHANNEL_ID)

    except Exception as exc:
        print(f"Could not find channel: {exc}")
        return None


async def delete_old_radio_message(channel):
    old_message_id = state.get("message_id")

    if not old_message_id:
        return

    try:
        old_message = await channel.fetch_message(
            int(old_message_id)
        )

        await old_message.delete()

    except discord.NotFound:
        pass

    except discord.Forbidden:
        print(
            "Bot does not have permission "
            "to delete the previous radio message."
        )

    except Exception as exc:
        print(
            f"Could not delete old radio message: {exc}"
        )


async def post_radio(
    ping_everyone=False,
    forced_radio=None,
    scheduled=False
):
    channel = await get_radio_channel()

    if channel is None:
        raise RuntimeError(
            "Radio channel not found. "
            "Check CHANNEL_ID and bot permissions."
        )

    await delete_old_radio_message(channel)

    if forced_radio is not None:
        new_radio = forced_radio
    else:
        new_radio = pick_new_radio()

    radio_display = f"```{new_radio}```"

    if ping_everyone:
        message = await channel.send(
            f"@everyone\n"
            f"**DYNASTY ACTIVE RADIO**\n"
            f"{radio_display}",
            allowed_mentions=discord.AllowedMentions(
                everyone=True
            )
        )

    else:
        message = await channel.send(
            f"**DYNASTY ACTIVE RADIO**\n"
            f"{radio_display}"
        )

    state["current_radio"] = new_radio
    state["message_id"] = message.id

    if scheduled:
        now = datetime.now(TIMEZONE)

        state["last_scheduled_post_date"] = str(
            now.date()
        )

    save_state()

    return new_radio


@bot.event
async def on_ready():
    load_state()

    now = datetime.now(TIMEZONE)

    print("------------------------------")
    print(f"Logged in as: {bot.user}")
    print(f"Bot ID: {bot.user.id}")
    print(f"Channel ID: {CHANNEL_ID}")
    print(f"Mountain Time: {now}")
    print("------------------------------")

    if not daily_radio_check.is_running():
        daily_radio_check.start()


@bot.event
async def on_message(message):
    if message.author.bot:
        return

    await bot.process_commands(message)


@tasks.loop(seconds=20)
async def daily_radio_check():
    now = datetime.now(TIMEZONE)
    today = str(now.date())

    if (
        now.hour == POST_HOUR
        and now.minute == POST_MINUTE
        and state.get("last_scheduled_post_date") != today
    ):
        try:
            new_radio = await post_radio(
                ping_everyone=False,
                scheduled=True
            )

            print(
                f"Scheduled Dynasty radio posted: "
                f"{new_radio}"
            )

        except Exception as exc:
            print(
                f"Daily Dynasty radio post failed: "
                f"{exc}"
            )


@daily_radio_check.before_loop
async def before_daily_radio_check():
    await bot.wait_until_ready()


@bot.command()
async def ping(ctx):
    await ctx.send(
        "pong",
        delete_after=5
    )


@bot.command()
async def rotate(ctx):
    try:
        try:
            await ctx.message.delete()
        except Exception:
            pass

        new_radio = await post_radio(
            ping_everyone=True,
            scheduled=False
        )

        print(
            f"{ctx.author} rotated Dynasty radio "
            f"to {new_radio}"
        )

    except Exception as exc:
        print(f"Rotate error: {exc}")

        await ctx.send(
            f"Error: {exc}",
            delete_after=8
        )


@bot.command()
async def setradio(ctx, *, radio_name: str):
    radio_name = (
        radio_name
        .replace("```", "")
        .strip()
        .upper()
    )

    if radio_name not in RADIOS:
        await ctx.send(
            "That radio is not in the preset list.",
            delete_after=8
        )
        return

    try:
        try:
            await ctx.message.delete()
        except Exception:
            pass

        new_radio = await post_radio(
            ping_everyone=True,
            forced_radio=radio_name,
            scheduled=False
        )

        print(
            f"{ctx.author} set Dynasty radio "
            f"to {new_radio}"
        )

    except Exception as exc:
        print(f"Set radio error: {exc}")

        await ctx.send(
            f"Error: {exc}",
            delete_after=8
        )


@bot.command()
async def currentradio(ctx):
    current = state.get("current_radio")

    if current:
        await ctx.send(
            f"Current Dynasty radio:\n"
            f"```{current}```",
            delete_after=15
        )

    else:
        await ctx.send(
            "No Dynasty radio has been posted yet.",
            delete_after=8
        )


@bot.event
async def on_command_error(ctx, error):
    print(
        f"Command error: "
        f"{type(error).__name__}: {error}"
    )


if not TOKEN:
    raise RuntimeError(
        "DISCORD_TOKEN environment variable is not set."
    )


print("Starting Dynasty Radio...")

bot.run(TOKEN)
