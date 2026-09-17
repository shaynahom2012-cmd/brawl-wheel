import discord
from discord.ext import commands
from discord import app_commands
import json, os, random
from datetime import datetime, timedelta

TOKEN = "PASTE_YOUR_TOKEN_HERE"
DATA_FILE = "casino_data.json"

intents = discord.Intents.default()
bot = commands.Bot(command_prefix="!", intents=intents)

def load_data():
    if not os.path.exists(DATA_FILE): return {}
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f: return json.load(f)
    except: return {}

def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f: json.dump(data, f, indent=4)

def get_user(user_id):
    data = load_data()
    uid = str(user_id)
    if uid not in data:
        data[uid] = {"balance": 1000, "last_daily": None}
        save_data(data)
    return data[uid]

def update_balance(user_id, amount):
    data = load_data()
    uid = str(user_id)
    if uid not in data: data[uid] = {"balance": 1000, "last_daily": None}
    data[uid]["balance"] += amount
    save_data(data)
    return data[uid]["balance"]

def card_value(card):
    rank = card[:-1]
    if rank in ["J","Q","K"]: return 10
    if rank == "A": return 11
    return int(rank)

def hand_value(hand):
    val = 0; aces = 0
    for c in hand:
        v = card_value(c); val += v
        if c.startswith("A"): aces += 1
    while val > 21 and aces:
        val -= 10; aces -= 1
    return val

class BlackjackView(discord.ui.View):
    def __init__(self, user_id, bet):
        super().__init__(timeout=90)
        self.user_id = user_id; self.bet = bet
        ranks = ["A","2","3","4","5","6","7","8","9","10","J","Q","K"]
        suits = ["\u2660","\u2665","\u2666","\u2663"]
        self.deck = [r+s for r in ranks for s in suits]
        random.shuffle(self.deck)
        self.player = [self.deck.pop(), self.deck.pop()]
        self.dealer = [self.deck.pop(), self.deck.pop()]
        self.finished = False

    @discord.ui.button(label="Hit", style=discord.ButtonStyle.green)
    async def hit(self, interaction: discord.Interaction, button: discord.ui.Button):
        if interaction.user.id != self.user_id:
            await interaction.response.send_message("Not your game!", ephemeral=True); return
        if self.finished: return
        self.player.append(self.deck.pop())
        pv = hand_value(self.player)
        if pv >= 21:
            self.finished = True; self.clear_items()
            dv = hand_value(self.dealer)
            while dv < 17:
                self.dealer.append(self.deck.pop()); dv = hand_value(self.dealer)
            embed = discord.Embed(title="Blackjack Finished", color=0xe74c3c if pv>21 else 0x2ecc71)
            embed.add_field(name=f"Your Hand ({pv})", value=", ".join(self.player), inline=False)
            embed.add_field(name=f"Dealer Hand ({dv})", value=", ".join(self.dealer), inline=False)
            if pv > 21:
                embed.add_field(name="Result", value=f"BUST! Lost {self.bet}$", inline=False)
            elif dv > 21 or pv > dv:
                win = int(self.bet*1.5) if pv==21 and len(self.player)==2 else self.bet
                update_balance(self.user_id, self.bet+win)
                embed.add_field(name="Result", value=f"WIN! +{win}$ profit!", inline=False)
            elif pv == dv:
                update_balance(self.user_id, self.bet)
                embed.add_field(name="Result", value="PUSH! Money back.", inline=False)
            else:
                embed.add_field(name="Result", value=f"Lost {self.bet}$", inline=False)
            await interaction.response.edit_message(embed=embed, view=self)
        else:
            embed = discord.Embed(title="Blackjack", color=0x2ecc71)
            embed.add_field(name=f"Your Hand ({pv})", value=", ".join(self.player), inline=False)
            embed.add_field(name=f"Dealer ({hand_value([self.dealer[0]])}+?)", value=f"{self.dealer[0]}, ?", inline=False)
            await interaction.response.edit_message(embed=embed, view=self)

    @discord.ui.button(label="Stand", style=discord.ButtonStyle.red)
    async def stand(self, interaction: discord.Interaction, button: discord.ui.Button):
        if interaction.user.id != self.user_id:
            await interaction.response.send_message("Not your game!", ephemeral=True); return
        self.finished = True; self.clear_items()
        pv = hand_value(self.player); dv = hand_value(self.dealer)
        while dv < 17:
            self.dealer.append(self.deck.pop()); dv = hand_value(self.dealer)
        embed = discord.Embed(title="Blackjack Finished", color=0x2ecc71)
        embed.add_field(name=f"Your Hand ({pv})", value=", ".join(self.player), inline=False)
        embed.add_field(name=f"Dealer Hand ({dv})", value=", ".join(self.dealer), inline=False)
        if dv > 21 or pv > dv:
            win = int(self.bet*1.5) if pv==21 and len(self.player)==2 else self.bet
            update_balance(self.user_id, self.bet+win)
            embed.add_field(name="Result", value=f"WIN! +{win}$ profit!", inline=False)
        elif pv == dv:
            update_balance(self.user_id, self.bet)
            embed.add_field(name="Result", value="PUSH! Money back.", inline=False)
        else:
            embed.add_field(name="Result", value=f"Lost {self.bet}$", inline=False)
        await interaction.response.edit_message(embed=embed, view=self)

@bot.event
async def on_ready():
    print(f"ONLINE {bot.user}")
    try: await bot.tree.sync()
    except Exception as e: print(e)

@bot.tree.command(name="balance", description="Check balance")
async def balance_cmd(interaction: discord.Interaction):
    u = get_user(interaction.user.id)
    await interaction.response.send_message(embed=discord.Embed(title="Balance", description=f"You have {u['balance']}$", color=0xf1c40f))

@bot.tree.command(name="daily", description="Daily bonus")
async def daily_cmd(interaction: discord.Interaction):
    data = load_data(); uid = str(interaction.user.id)
    if uid not in data: data[uid] = {"balance": 1000, "last_daily": None}
    last = data[uid].get("last_daily"); now = datetime.now()
    if last:
        lt = datetime.fromisoformat(last)
        if now - lt < timedelta(hours=24):
            rem = timedelta(hours=24)-(now-lt)
            await interaction.response.send_message(f"Already claimed! Wait {int(rem.total_seconds()//3600)}h", ephemeral=True); return
    data[uid]["balance"]+=500; data[uid]["last_daily"]=now.isoformat(); save_data(data)
    await interaction.response.send_message(f"Daily +500$! Balance: {data[uid]['balance']}$")

@bot.tree.command(name="slots", description="Play slots")
@app_commands.describe(amount="Bet amount")
async def slots_cmd(interaction: discord.Interaction, amount: int):
    u = get_user(interaction.user.id)
    if amount<=0 or u["balance"]<amount:
        await interaction.response.send_message(f"No money! Balance: {u['balance']}$", ephemeral=True); return
    update_balance(interaction.user.id, -amount)
    symbols = ["\U0001f352","\U0001f34b","\U0001f514","\u2b50","\U0001f48e"]
    reels = [random.choice(symbols) for _ in range(3)]
    text = " | ".join(reels)
    if reels[0]==reels[1]==reels[2]:
        win = amount*5 if reels[0]=="\U0001f48e" else amount*3
        update_balance(interaction.user.id, amount+win)
        color=0x2ecc71; title="JACKPOT!"; desc=f"{text}\nWon {win}$!"
    elif reels[0]==reels[1] or reels[1]==reels[2] or reels[0]==reels[2]:
        win=int(amount*0.5); update_balance(interaction.user.id, amount+win)
        color=0xf1c40f; title="Small Win"; desc=f"{text}\nWon {win}$!"
    else:
        color=0xe74c3c; title="Lost"; desc=f"{text}\nLost {amount}$"
    embed=discord.Embed(title=title, description=desc, color=color)
    embed.set_footer(text=f"Balance: {get_user(interaction.user.id)['balance']}$")
    await interaction.response.send_message(embed=embed)

@bot.tree.command(name="coinflip", description="Coinflip")
@app_commands.describe(choice="heads or tails", amount="bet")
@app_commands.choices(choice=[app_commands.Choice(name="Heads", value="heads"), app_commands.Choice(name="Tails", value="tails")])
async def coinflip_cmd(interaction: discord.Interaction, choice: str, amount: int):
    u = get_user(interaction.user.id)
    if amount<=0 or u["balance"]<amount:
        await interaction.response.send_message(f"Balance: {u['balance']}$", ephemeral=True); return
    update_balance(interaction.user.id, -amount)
    result = random.choice(["heads","tails"])
    if result==choice:
        update_balance(interaction.user.id, amount*2)
        msg=f"It was {result}! WON {amount}$"; col=0x2ecc71
    else:
        msg=f"It was {result}! LOST {amount}$"; col=0xe74c3c
    embed=discord.Embed(title="Coinflip", description=msg, color=col)
    embed.set_footer(text=f"Balance: {get_user(interaction.user.id)['balance']}$")
    await interaction.response.send_message(embed=embed)

@bot.tree.command(name="blackjack", description="Play blackjack")
@app_commands.describe(amount="bet amount")
async def blackjack_cmd(interaction: discord.Interaction, amount: int):
    u = get_user(interaction.user.id)
    if amount<=0 or u["balance"]<amount:
        await interaction.response.send_message(f"Balance: {u['balance']}$", ephemeral=True); return
    update_balance(interaction.user.id, -amount)
    view = BlackjackView(interaction.user.id, amount)
    pv = hand_value(view.player)
    if pv==21:
        view.finished=True; view.clear_items()
        win=int(amount*1.5); update_balance(interaction.user.id, amount+win)
        embed=discord.Embed(title="BLACKJACK!", description=f"{', '.join(view.player)} - You win {win}$!", color=0xf1c40f)
        await interaction.response.send_message(embed=embed, view=view)
    else:
        embed=discord.Embed(title="Blackjack", color=0x2ecc71)
        embed.add_field(name=f"Your Hand ({pv})", value=", ".join(view.player), inline=False)
        embed.add_field(name=f"Dealer ({hand_value([view.dealer[0]])}+?)", value=f"{view.dealer[0]}, ?", inline=False)
        await interaction.response.send_message(embed=embed, view=view)

@bot.tree.command(name="leaderboard", description="Top players")
async def leaderboard_cmd(interaction: discord.Interaction):
    data=load_data()
    sorted_u=sorted(data.items(), key=lambda x: x[1]["balance"], reverse=True)[:10]
    desc="\n".join([f"{i}. <@{uid}> - {ud['balance']}$" for i,(uid,ud) in enumerate(sorted_u,1)])
    await interaction.response.send_message(embed=discord.Embed(title="Leaderboard", description=desc or "No data", color=0xf1c40f))

bot.run(TOKEN)
