import discord
import os
import json
import time

from discord.ext import commands

# Load bot token from environment variable
BOT_TOKEN = os.getenv('DISCORD_BOT_TOKEN')

# Cooldown time in seconds (e.g., 1 hour)
VOUCH_COOLDOWN = 3600

# Data file for vouches and cooldowns
VOUCHES_FILE = "vouches.json"
COOLDOWN_FILE = "cooldowns.json"

intents = discord.Intents.default()
intents.members = True
intents.messages = True
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)

def load_vouches():
    try:
        with open(VOUCHES_FILE, "r") as f:
            return json.load(f)
    except Exception:
        return {}

def save_vouches(data):
    with open(VOUCHES_FILE, "w") as f:
        json.dump(data, f)

def load_cooldowns():
    try:
        with open(COOLDOWN_FILE, "r") as f:
            return json.load(f)
    except Exception:
        return {}

def save_cooldowns(data):
    with open(COOLDOWN_FILE, "w") as f:
        json.dump(data, f)

def user_on_cooldown(guild_id, user_id):
    cooldowns = load_cooldowns()
    guild_id = str(guild_id)
    user_id = str(user_id)
    now = int(time.time())
    if guild_id in cooldowns and user_id in cooldowns[guild_id]:
        last_time = cooldowns[guild_id][user_id]
        wait_time = VOUCH_COOLDOWN - (now - last_time)
        if wait_time > 0:
            return True, wait_time
    return False, 0

def set_cooldown(guild_id, user_id):
    cooldowns = load_cooldowns()
    guild_id = str(guild_id)
    user_id = str(user_id)
    if guild_id not in cooldowns:
        cooldowns[guild_id] = {}
    cooldowns[guild_id][user_id] = int(time.time())
    save_cooldowns(cooldowns)

@bot.event
async def on_ready():
    print(f"Logged in as {bot.user}")

@bot.event
async def on_message(message):
    if message.author.bot:
        return

    content = message.content.strip()
    lower = content.lower()
    try:
        # VOUCH
        if lower.startswith("vouch "):
            if not message.mentions:
                await message.channel.send("Please mention a user to vouch for.")
                return
            member = message.mentions[0]
            if member == message.author:
                await message.channel.send("You cannot vouch for yourself.")
                return

            on_cooldown, wait_time = user_on_cooldown(message.guild.id, message.author.id)
            if on_cooldown:
                mins = wait_time // 60
                secs = wait_time % 60
                await message.channel.send(f"You're on cooldown! Please wait {mins} minute(s) and {secs} second(s) before vouching again.")
                return

            # Find the reason (text after the mention)
            words = content.split()
            if len(words) >= 3:
                reason = " ".join(words[2:])
            else:
                reason = "No reason provided"

            data = load_vouches()
            user_id = str(member.id)
            if user_id not in data:
                data[user_id] = []
            # Prevent duplicate vouches from the same user for the same member
            already_vouched = any(v['vouched_by'] == str(message.author.id) for v in data[user_id])
            if already_vouched:
                await message.channel.send(f"You have already vouched for {member.mention}! Use `unvouch @user` to remove your vouch before vouching again.")
                return

            data[user_id].append({
                "vouched_by": str(message.author.id),
                "reason": reason
            })
            save_vouches(data)
            set_cooldown(message.guild.id, message.author.id)
            await message.channel.send(f"{message.author.mention} vouched for {member.mention}! Reason: {reason}")

        # UNVOUCH
        elif lower.startswith("unvouch "):
            if not message.mentions:
                await message.channel.send("Please mention a user to unvouch.")
                return
            member = message.mentions[0]
            data = load_vouches()
            user_id = str(member.id)
            author_id = str(message.author.id)
            if user_id not in data or not any(v['vouched_by'] == author_id for v in data[user_id]):
                await message.channel.send(f"You haven't vouched for {member.mention}.")
                return
            # Remove vouch by this user
            data[user_id] = [v for v in data[user_id] if v['vouched_by'] != author_id]
            save_vouches(data)
            await message.channel.send(f"{message.author.mention} removed their vouch for {member.mention}.")

        # VOUCHES
        elif lower.startswith("vouches"):
            if message.mentions:
                member = message.mentions[0]
            else:
                member = message.author

            data = load_vouches()
            user_id = str(member.id)
            vouch_list = data.get(user_id, [])
            count = len(vouch_list)
            if count == 0:
                await message.channel.send(f"{member.mention} has no vouches yet.")
                return

            msg = f"{member.mention} has {count} vouch(es):\n"
            for v in vouch_list:
                try:
                    user_obj = message.guild.get_member(int(v['vouched_by']))
                    username = user_obj.name if user_obj else "Unknown"
                except Exception:
                    username = "Unknown"
                msg += f"- Vouched by {username}: {v['reason']}\n"
            await message.channel.send(msg)
    except Exception as e:
        print(f"Error in on_message: {e}")
        try:
            await message.channel.send("An error occurred. Please check the bot logs.")
        except Exception:
            pass

if __name__ == "__main__":
    if not BOT_TOKEN:
        print("Please set the DISCORD_BOT_TOKEN environment variable.")
    else:
        bot.run(BOT_TOKEN)
