import discord
from discord.ext import commands
import aiosqlite
import random
from PIL import Image, ImageDraw, ImageFont
import io
import os
import asyncio
from datetime import timedelta
import requests
import aiohttp
import json
import datetime
from discord.ext import tasks
import yt_dlp
from discord import app_commands
from flask import Flask
from threading import Thread
from keep_alive import keep_alive
keep_alive()

intents = discord.Intents.all()
bot = commands.Bot(command_prefix="!", intents=intents)


#Command //--------------------------------------------------------------

@bot.event
async def on_ready():
    print(f"✅ Bot ist online als {bot.user}")
    try:
        synced = await bot.tree.sync(guild=discord.Object(id=GUILD_ID))
        print(f"🔁 Slash-Commands synchronisiert: {len(synced)}")
    except Exception as e:
        print(f"❌ Fehler beim Sync: {e}")


#Level System--------------------------------------------------------------

LEVEL_ROLES = {
    1: 1267505131155619845, 2: 1267505131155619846, 3: 1267505131155619847,
    4: 1267505131155619848, 5: 1267505131155619849, 6: 1267505131193106576,
    7: 1267505131193106577, 8: 1267505131193106578, 9: 1267505131193106579,
    10: 1363552335292530959, 11: 1363552468759220355, 12: 1363552486723424560,
    13: 1363552530113495140, 14: 1363552568256630784, 15: 1363552606198431794,
    16: 1363552675186344056, 17: 1363552710472892658, 18: 1363552737723154663,
    19: 1363552788210122812, 20: 1363552843596042410
}

LOG_CHANNEL_ID = 123456789012345678

def get_xp_for_next_level(level):
    return int(7.5 * (level ** 2) + 65 * level + 150)

def get_level(xp):
    level = 0
    while xp >= get_xp_for_next_level(level):
        level += 1
    return level

async def handle_roles(member, old_level, new_level):
    guild = member.guild
    for level, role_id in LEVEL_ROLES.items():
        role = guild.get_role(role_id)
        if role and role in member.roles:
            await member.remove_roles(role)

    new_role_id = LEVEL_ROLES.get(new_level)
    if new_role_id:
        new_role = guild.get_role(new_role_id)
        if new_role:
            await member.add_roles(new_role)

@bot.event
async def on_ready():
    print(f"Eingeloggt als {bot.user}")
    async with aiosqlite.connect("database.db") as db:
        await db.execute("""CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER,
            guild_id INTEGER,
            xp INTEGER,
            level INTEGER,
            PRIMARY KEY (user_id, guild_id))""")
        await db.commit()

@bot.event
async def on_message(message):
    if message.author.bot:
        return

    async with aiosqlite.connect("database.db") as db:
        user_id = message.author.id
        guild_id = message.guild.id
        cursor = await db.execute("SELECT xp, level FROM users WHERE user_id = ? AND guild_id = ?", (user_id, guild_id))
        row = await cursor.fetchone()

        base_xp = random.randint(5, 10)

        if row is None:
            xp = base_xp
            old_level = 0
            new_level = get_level(xp)
            await db.execute("INSERT INTO users (user_id, guild_id, xp, level) VALUES (?, ?, ?, ?)", (user_id, guild_id, xp, new_level))
        else:
            xp, old_level = row
            xp_gain = max(2, int(base_xp * (1 / (old_level + 1))))
            xp += xp_gain
            new_level = get_level(xp)

            if new_level > old_level:
                await message.channel.send(f"{message.author.mention} ist auf Level {new_level} aufgestiegen!")
                await handle_roles(message.author, old_level, new_level)

            await db.execute("UPDATE users SET xp = ?, level = ? WHERE user_id = ? AND guild_id = ?", (xp, new_level, user_id, guild_id))
        await db.commit()

    await bot.process_commands(message)

@bot.command()
async def leaderboard(ctx):
    async with aiosqlite.connect("database.db") as db:
        cursor = await db.execute("SELECT user_id, xp, level FROM users WHERE guild_id = ? ORDER BY level DESC, xp DESC LIMIT 10", (ctx.guild.id,))
        rows = await cursor.fetchall()

    if not rows:
        await ctx.send("Noch niemand hat XP gesammelt.")
        return

    embed = discord.Embed(title=" Leaderboard", color=discord.Color.gold())
    for i, (user_id, xp, level) in enumerate(rows):
        user = ctx.guild.get_member(user_id)
        if user:
            embed.add_field(name=f"{i+1}. {user.name}", value=f"Level {level} | XP: {xp}", inline=False)

    await ctx.send(embed=embed)

@bot.command()
@commands.has_permissions(administrator=True)
async def xpreset(ctx, member: discord.Member):
    async with aiosqlite.connect("database.db") as db:
        await db.execute("UPDATE users SET xp = 0, level = 0 WHERE user_id = ? AND guild_id = ?", (member.id, ctx.guild.id))
        await db.commit()
    await ctx.send(f"XP & Level von {member.mention} wurden zurückgesetzt.")


#Moderation--------------------------------------------------------------

@bot.command()
async def cmds(ctx):
    await ctx.send(f"Geladene Befehle: {', '.join([cmd.name for cmd in bot.commands])}")


@bot.command(name="clear")
@commands.has_permissions(manage_messages=True)
async def clear(ctx, amount: int = 5):
    if amount < 1:
        await ctx.send("Mindestens 1 Nachricht löschen.")
        return
    deleted = await ctx.channel.purge(limit=amount + 1)
    confirm = await ctx.send(f"{len(deleted)-1} Nachrichten gelöscht!")
    await confirm.delete(delay=3)

@clear.error
async def clear_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("Keine Berechtigung zum Löschen.")
    elif isinstance(error, commands.BadArgument):
        await ctx.send("Ungültige Anzahl.")
    else:
        await ctx.send("Ein Fehler ist aufgetreten.")

@bot.command()
async def announce(ctx):
    OWNER_ID = 1215763220875448342
    if ctx.author.id != OWNER_ID:
        await ctx.send("Du darfst diesen Befehl nicht verwenden.")
        return

    embed = discord.Embed(
        title="📜 Serverregeln",
        description="Mit dem Beitritt zum Server bestätigst du, dass du die Regeln gelesen und akzeptiert hast.",
        color=discord.Color.orange()
    )

    embed.add_field(name="[1]", value="Die Administratoren & der Inhaber dürfen die Regeln jederzeit anpassen.", inline=False)
    embed.add_field(name="[2]", value="Unwissenheit schützt vor Strafe nicht!", inline=False)
    embed.add_field(name="[3]", value="Werbung ist verboten, außer der Inhaber erlaubt es ausdrücklich.", inline=False)
    embed.add_field(name="[4]", value="Bleibt freundlich – auch bei Meinungsverschiedenheiten.", inline=False)
    embed.add_field(name="[5]", value="Jeder hat ein Recht auf Privatsphäre.", inline=False)
    embed.add_field(name="[6]", value="Haltet euch an grundsätzliche Gesprächsregeln.", inline=False)
    embed.add_field(name="[7]", value="Missbraucht das Support-Ticket-System nicht.", inline=False)
    embed.add_field(name="[8]", value="Unangemessene Profilbilder, Infos oder Namen sind verboten.", inline=False)
    embed.add_field(name="[9]", value="Keine politischen Diskussionen!", inline=False)
    embed.add_field(name="[10]", value="Streak-Zerstören im Counting-Channel = Verwarnung.", inline=False)
    embed.add_field(name="[11]", value="Ban-Umgehung und Doxxing = Verboten!", inline=False)
    embed.add_field(name="[12]", value="Keine NSFW-Inhalte oder Links dazu.", inline=False)
    embed.add_field(name="[13]", value="Join/Leave-Spam wird bestraft.", inline=False)
    embed.add_field(name="[14]", value="Toxisches Verhalten wird nicht geduldet.", inline=False)

    embed.set_footer(text="Bei Fragen wende dich an das Team. Danke fürs Einhalten der Regeln!")
    await ctx.send("@everyone", embed=embed)

@bot.command()
@commands.has_permissions(ban_members=True)
async def tempban(ctx, member: discord.Member, stunden: int, *, reason="Kein Grund angegeben"):
    if ctx.author != ctx.guild.owner and not ctx.author.guild_permissions.administrator:
        await ctx.send("Nur Admins oder der Server-Inhaber dürfen das!")
        return
    try:
        await member.ban(reason=reason)
        await ctx.send(f"{member} wurde für {stunden} Stunden gebannt. Grund: {reason}")

        await asyncio.sleep(stunden * 3600)
        await ctx.guild.unban(discord.Object(id=member.id))
        log_channel = bot.get_channel(LOG_CHANNEL_ID)
        if log_channel:
            await log_channel.send(f"{member} wurde automatisch entbannt (Ban-Zeit abgelaufen).")
    except Exception as e:
        await ctx.send(f"Fehler beim temporären Bann: {e}")


@bot.command()
@commands.has_permissions(ban_members=True)
async def permban(ctx, member: discord.Member, *, reason="Kein Grund angegeben"):
    if ctx.author != ctx.guild.owner and not ctx.author.guild_permissions.administrator:
        await ctx.send("Nur Admins oder der Server-Inhaber dürfen das!")
        return
    try:
        await member.ban(reason=reason)
        await ctx.send(f"{member} wurde permanent gebannt. Grund: {reason}")
    except Exception as e:
        await ctx.send(f"Fehler beim Bannen: {e}")

@bot.command()
@commands.has_permissions(manage_messages=True)
async def mute(ctx, member: discord.Member, time: int, *, reason="Kein Grund angegeben"):
    if ctx.author != ctx.guild.owner and not ctx.author.guild_permissions.administrator:
        await ctx.send(" Nur Admins oder der Server-Inhaber dürfen diesen Befehl verwenden!")
        return

    try:
        
        muted_role = discord.utils.get(ctx.guild.roles, name="Muted")
        if not muted_role:
            muted_role = await ctx.guild.create_role(name="Muted", permissions=discord.Permissions(send_messages=False, speak=False))

        
        await member.add_roles(muted_role, reason=reason)

        
        await ctx.send(f"{member} wurde für {time} Minuten stummgeschaltet. Grund: {reason}")

    
        await asyncio.sleep(time * 60)  

    
        await member.remove_roles(muted_role, reason="Mute-Zeitraum abgelaufen")

        
        await ctx.send(f"{member} ist jetzt aus dem Mute-Zustand entlassen!")

    except Exception as e:
        await ctx.send(f"Fehler beim Muten: {e}")

@bot.command()
@commands.has_permissions(manage_messages=True)
async def unmute(ctx, member: discord.Member, *, reason="Kein Grund angegeben"):
    if ctx.author != ctx.guild.owner and not ctx.author.guild_permissions.administrator:
        await ctx.send(" Nur Admins oder der Server-Inhaber dürfen diesen Befehl verwenden!")
        return

    try:
        
        muted_role = discord.utils.get(ctx.guild.roles, name="Muted")
        if not muted_role:
            await ctx.send("Es gibt keine 'Muted'-Rolle auf diesem Server.")
            return

        
        if muted_role not in member.roles:
            await ctx.send(f"{member} ist nicht stummgeschaltet.")
            return

        
        await member.remove_roles(muted_role, reason=reason)

        
        await ctx.send(f"{member} wurde erfolgreich entmuted! Grund: {reason}")

    except Exception as e:
        await ctx.send(f"Fehler beim Entmuten: {e}")


#Auto rolle--------------------------------------------------------------

    @bot.event
    async def on_member_join(member):
        guild = member.guild
        role_name = "⊂⊃Mitglied"

        role = discord.utils.get(guild.roles, name=role_name)

        if role:
            await member.add_roles(role)
            print(f"{member} wurde die Rolle {role.name} zugewiesen.")
        else:
            print(f"Rolle {role_name} nicht gefunden.")

#Birthday--------------------------------------------------------------

BIRTHDAYS_FILE = "birthdays.json"


def load_birthdays():
    try:
        with open(BIRTHDAYS_FILE, "r") as f:
            return json.load(f)
    except FileNotFoundError:
        return {}


def save_birthdays(data):
    with open(BIRTHDAYS_FILE, "w") as f:
        json.dump(data, f, indent=4)


@bot.event
async def on_ready():
    check_birthdays.start()
    print(f"Bot ist online als {bot.user}")


@bot.command()
async def birthday(ctx, date: str):
    try:
        datetime.datetime.strptime(date, "%d.%m")  
    except ValueError:
        await ctx.send(" Bitte gib dein Geburtsdatum im Format `TT.MM` an (z. B. `24.04`).")
        return

    data = load_birthdays()
    data[str(ctx.author.id)] = date
    save_birthdays(data)

    await ctx.send(f" Dein Geburtstag wurde auf den {date} gesetzt!")


@tasks.loop(hours=24)
async def check_birthdays():
    await bot.wait_until_ready()
    today = datetime.datetime.now().strftime("%d.%m")
    data = load_birthdays()

    for user_id, date in data.items():
        if date == today:
            user = await bot.fetch_user(int(user_id))
            try:
                await user.send("🎉 @everyone Alles Gute zum Geburtstag! 🥳🎂")
                print(f"Gratulation an {user.name}")
            except:
                print(f"Konnte {user_id} nicht gratulieren (DM evtl. deaktiviert)")


#Music--------------------------------------------------------------

FFMPEG_OPTIONS = {
    'before_options': '-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5',
    'options': '-vn'
}

YDL_OPTIONS = {
    'format': 'bestaudio/best',
    'noplaylist': True
}

class MusicCommands(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.queues = {}
        self.current_song = {}
        
    def check_queue(self, ctx):
        if len(self.queues.get(ctx.guild.id, [])) > 0:
            voice = ctx.guild.voice_client
            source = self.queues[ctx.guild.id].pop(0)
            voice.play(source, after=lambda x: self.check_queue(ctx))

    async def join_voice_channel(self, ctx):
        if ctx.author.voice is None:
            await ctx.send("Du musst in einem Sprachkanal sein!")
            return False
        voice_channel = ctx.author.voice.channel
        if ctx.voice_client is None:
            await voice_channel.connect()
        else:
            await ctx.voice_client.move_to(voice_channel)
        return True

    @commands.command()
    async def join(self, ctx):
        """Tritt dem Sprachkanal bei"""
        await self.join_voice_channel(ctx)

    @commands.command()
    async def leave(self, ctx):
        """Verlässt den Sprachkanal"""
        if ctx.voice_client:
            await ctx.voice_client.disconnect()
            self.queues[ctx.guild.id] = []
            self.current_song[ctx.guild.id] = None

    @commands.command()
    async def play(self, ctx, *, url):
        """Spielt Musik von YouTube ab"""
        if not await self.join_voice_channel(ctx):
            return

        async with ctx.typing():
            try:
                with yt_dlp.YoutubeDL(YDL_OPTIONS) as ydl:
                    info = ydl.extract_info(url, download=False)
                    url2 = info['url']
                    source = await discord.FFmpegOpusAudio.from_probe(url2, **FFMPEG_OPTIONS)
                    
                    if ctx.voice_client.is_playing():
                        if ctx.guild.id not in self.queues:
                            self.queues[ctx.guild.id] = []
                        self.queues[ctx.guild.id].append(source)
                        await ctx.send(f" **{info['title']}** wurde zur Warteschlange hinzugefügt!")
                    else:
                        ctx.voice_client.play(source, after=lambda x: self.check_queue(ctx))
                        self.current_song[ctx.guild.id] = info['title']
                        await ctx.send(f" Spiele jetzt: **{info['title']}**")
            except Exception as e:
                await ctx.send(f"Ein Fehler ist aufgetreten: {str(e)}")

    @commands.command()
    async def pause(self, ctx):
        """Pausiert die aktuelle Wiedergabe"""
        if ctx.voice_client and ctx.voice_client.is_playing():
            ctx.voice_client.pause()
            await ctx.send("Musik pausiert")

    @commands.command()
    async def resume(self, ctx):
        """Setzt die Wiedergabe fort"""
        if ctx.voice_client and ctx.voice_client.is_paused():
            ctx.voice_client.resume()
            await ctx.send("Wiedergabe fortgesetzt")

    @commands.command()
    async def skip(self, ctx):
        """Überspringt den aktuellen Song"""
        if ctx.voice_client and ctx.voice_client.is_playing():
            ctx.voice_client.stop()
            await ctx.send("Song übersprungen")

    @commands.command()
    async def queue(self, ctx):
        """Zeigt die aktuelle Warteschlange"""
        if not self.queues.get(ctx.guild.id):
            await ctx.send("Die Warteschlange ist leer!")
            return
        
        current = self.current_song.get(ctx.guild.id, "Keine Wiedergabe")
        queue_list = f"**Aktuell:** {current}\n\n**Warteschlange:**"
        queue_length = len(self.queues[ctx.guild.id])
        
        await ctx.send(f"{queue_list}\n{queue_length} Songs in der Warteschlange")


async def setup(bot):
    await bot.add_cog(MusicCommands(bot))

@bot.event
async def on_ready():
    await setup(bot)
    print(f"Bot ist online als {bot.user}")
    check_birthdays.start()






#spass befehle--------------------------------------------------------------




@bot.command(name='roll')
async def roll(ctx, min_value: int = 1, max_value: int = 100):
    if min_value >= max_value:
        await ctx.send("Der Mindestwert muss kleiner als der Maximalwert sein.")
        return

    number = random.randint(min_value, max_value)

    embed = discord.Embed(
        title="🎲 Zufallswurf",
        description=f"Du hast eine **{number}** gewürfelt (zwischen {min_value} und {max_value})!",
        color=discord.Color.purple()
    )
    embed.set_footer(text=f"Angefordert von {ctx.author.display_name}", icon_url=ctx.author.avatar.url)

    await ctx.send(embed=embed)


#setup--------------------------------------------------------------




CONFIG_FILE = "server_config.json"


if os.path.exists(CONFIG_FILE):
    with open(CONFIG_FILE, "r") as f:
        server_config = json.load(f)
else:
    server_config = {}


@bot.command(name='setup')
@commands.has_permissions(administrator=True)
async def setup(ctx, log_channel: discord.TextChannel = None, level_channel: discord.TextChannel = None):
    if not log_channel or not level_channel:
        await ctx.send("❌ Bitte gib zwei Textkanäle an: `!setup #log-kanal #level-kanal`")
        return

    guild_id = str(ctx.guild.id)  
    server_config[guild_id] = {
        "log_channel_id": log_channel.id,
        "level_channel_id": level_channel.id
    }

    
    with open(CONFIG_FILE, "w") as f:
        json.dump(server_config, f, indent=4)

    embed = discord.Embed(
        title="✅ Setup abgeschlossen",
        description="Deine Einstellungen wurden gespeichert.",
        color=discord.Color.green()
    )
    embed.add_field(name="📝 Log-Channel", value=log_channel.mention, inline=False)
    embed.add_field(name="📊 Level-Channel", value=level_channel.mention, inline=False)
    embed.set_footer(text=f"Eingestellt von {ctx.author.display_name}", icon_url=ctx.author.avatar.url)

    await ctx.send(embed=embed)


@bot.event
async def on_ready():
    print(f"✅ Bot ist online als {bot.user.name} ({bot.user.id})")

















#!invite bot--------------------------------------------------------------

@bot.command(name='invite')
async def invite(ctx):
    client_id = "1267112916562350090"  
    invite_url = f"https://discord.com/oauth2/authorize?client_id={client_id}&permissions=8&scope=bot+applications.commands"

    embed = discord.Embed(
        title="🤖 Bot einladen",
        description=f"[Klicke hier, um mich zu deinem Server hinzuzufügen!]({invite_url})",
        color=discord.Color.green()
    )
    embed.set_footer(text="(Powerd by 𝒮𝑒𝓅𝒾𝒶_𝒷𝓉𝓌) Danke, dass du mich nutzen willst! ❤️")

    await ctx.send(embed=embed)



#Ticket system--------------------------------------------------------------(test)


GUILD_ID = 1267505131079860284  
SUPPORT_CHANNEL_ID = 1267505133382533240  
TICKET_CATEGORY_ID = 1267505133382533239  

class TicketView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="🎫 Ticket erstellen", style=discord.ButtonStyle.green, custom_id="create_ticket")
    async def create_ticket(self, interaction: discord.Interaction, button: discord.ui.Button):
        guild = interaction.guild
        existing_channel = discord.utils.get(guild.text_channels, name=f"ticket-{interaction.user.name.lower()}")
        if existing_channel:
            await interaction.response.send_message(f"Du hast bereits ein offenes Ticket: {existing_channel.mention}", ephemeral=True)
            return

        overwrites = {
            guild.default_role: discord.PermissionOverwrite(view_channel=False),
            interaction.user: discord.PermissionOverwrite(view_channel=True, send_messages=True),
            guild.me: discord.PermissionOverwrite(view_channel=True)
        }

        category = guild.get_channel(TICKET_CATEGORY_ID)
        ticket_channel = await guild.create_text_channel(
            name=f"ticket-{interaction.user.name}",
            overwrites=overwrites,
            category=category,
            reason="Neues Support-Ticket"
        )

        await ticket_channel.send(f"{interaction.user.mention} Willkommen im Ticket! Bitte schildere dein Anliegen.")
        await interaction.response.send_message(f"Ticket erstellt: {ticket_channel.mention}", ephemeral=True)

@bot.event
async def on_ready():
    print(f" Bot ist online als {bot.user}")
    try:
        synced = await bot.tree.sync(guild=discord.Object(id=GUILD_ID))
        print(f"🔁 Slash-Commands synchronisiert: {len(synced)}")
    except Exception as e:
        print(f" Fehler beim Sync: {e}")

    
    channel = bot.get_channel(SUPPORT_CHANNEL_ID)
    if channel:
        await channel.send(" Klicke auf den Button, um ein Ticket zu erstellen:", view=TicketView())



























#bot start--------------------------------------------------------------



app = Flask('')
@app.route('/')
def home():
    return "Keep alive!"
def run():
    app.run(host='0.0.0.0', port=8080)
Thread(target=run).start()


intents = discord.Intents.default()
intents.message_content = True  
bot = commands.Bot(command_prefix="!", intents=intents)

@bot.event
async def on_ready():
    print(f"Bot ist online als {bot.user}")

bot.run(os.environ["TOKEN"])
