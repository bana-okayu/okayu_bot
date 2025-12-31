# ===============================
# 必要なインポート
# ===============================
import os
import asyncio
import random
import datetime
import discord
from discord.ext import commands
from discord import app_commands

# ===============================
# Bot 基本設定
# ===============================
intents = discord.Intents.default()
intents.members = True
intents.message_content = True
bot = commands.Bot(command_prefix="!", intents=intents)
TOKEN = os.environ["DISCORD_BOT_TOKEN"]

# ===============================
# 共通関数
# ===============================
def embed(title: str, desc: str, color=discord.Color.blue()) -> discord.Embed:
    return discord.Embed(title=title, description=desc, color=color)

async def safe_send(interaction: discord.Interaction, embed_obj: discord.Embed):
    try:
        await interaction.response.send_message(embed=embed_obj)
    except discord.errors.InteractionResponded:
        await interaction.followup.send(embed=embed_obj)

# ===============================
# 起動 & コマンド同期
# ===============================
@bot.event
async def on_ready():
    await bot.change_presence(status=discord.Status.dnd, activity=discord.Game(name="okayu bot"))
    synced = await bot.tree.sync()
    print(f"✅ {len(synced)} コマンド同期完了")

# ===============================
# /ping
# ===============================
@bot.tree.command(description="Botが正常に応答しているか確認します。")
async def ping(interaction: discord.Interaction):
    await safe_send(interaction, embed(embed("🏓 Pong", f"応答しています（{int(bot.latency*1000)}ms）")))

# ===============================
# /dice
# ===============================
@bot.tree.command(description="指定した回数・面数でサイコロを振ります（合計・結果表示）。")
@app_commands.describe(count="振る個数（1〜114514）", sides="面数（1〜810）")
async def dice(interaction: discord.Interaction, count: int, sides: int):
    if not (1 <= count <= 114514 and 1 <= sides <= 810):
        return await safe_send(interaction, embed(embed("❌ エラー", "範囲外です。", discord.Color.red())))
    rolls = [random.randint(1, sides) for _ in range(count)]
    await safe_send(interaction, embed(embed(
        "🎲 ダイス結果",
        f"合計: **{sum(rolls)}**\n結果: {', '.join(map(str, rolls[:50]))}"
    )))

# ===============================
# /timer
# ===============================
TIMER_PRESETS = {
    "10秒": 10, "30秒": 30, "60秒": 60,
    "3分": 180, "5分": 300, "10分": 600,
    "20分": 1200, "30分": 1800,
    "1時間": 3600, "3時間": 10800
}

async def timer_task(interaction: discord.Interaction, sec: int):
    await asyncio.sleep(sec)
    await safe_send(interaction, embed(embed("⏰ タイマー終了", f"{interaction.user.mention} 時間になりました！")))

@bot.tree.command(description="指定時間後に通知するタイマーを設定します。")
@app_commands.describe(preset="プリセット時間", custom="カスタム秒数（秒単位）")
async def timer(interaction: discord.Interaction, preset: str = None, custom: int = None):
    sec = TIMER_PRESETS.get(preset, custom or 180)
    if sec > 10800:
        return await safe_send(interaction, embed(embed("⚠ エラー", "最大3時間までです", discord.Color.red())))
    await safe_send(interaction, embed(embed("⏱ タイマー開始", f"{sec}秒後に通知します。")))
    asyncio.create_task(timer_task(interaction, sec))

# ===============================
# /topic
# ===============================
@bot.tree.command(description="会話用のお題をランダムで表示します。")
async def topic(interaction: discord.Interaction):
    topics = [
        "今までで一番印象に残った出来事は？",
        "最近ハマっていることは？",
        "もし1週間休みがあったら何をする？",
        "好きな映画やアニメは？",
        "将来の夢は？",
        "休日の過ごし方は？"
    ]
    await safe_send(interaction, embed(embed("💬 トークテーマ", random.choice(topics))))

# ===============================
# /translate
# ===============================
@bot.tree.command(description="文章を指定言語に翻訳します（簡易表示）。")
@app_commands.describe(text="翻訳したい文章", target_lang="翻訳先の言語コード")
async def translate(interaction: discord.Interaction, text: str, target_lang: str):
    await safe_send(interaction, embed(embed("🌐 翻訳結果", f"翻訳先: {target_lang}\n{text}")))

# ===============================
# /weather
# ===============================
@bot.tree.command(description="指定地域の天気情報を表示します。")
@app_commands.describe(place="天気を確認したい地域名（例: Tokyo, Osaka）")
async def weather(interaction: discord.Interaction, place: str):
    await safe_send(interaction, embed(embed("🌤 天気情報", f"{place} の天気情報です（現在はダミー表示）")))

# ===============================
# /info
# ===============================
@bot.tree.command(description="おかゆbotの詳細情報を表示します。")
async def info(interaction: discord.Interaction):
    text = (
        "おかゆbotはDiscordサーバーでのモデレーション、便利機能、メモ管理、タイマー機能、"
        "オートモッド設定などを統合した多機能Botです。ユーザーが直感的に操作可能で、"
        "荒らし対策から日常会話サポートまで幅広く対応。Python 3.11 + discord.py 2.3以上対応。"
    )
    await safe_send(interaction, embed(embed("ℹ おかゆbotについて", text)))

# ===============================
# /help
# ===============================
@bot.tree.command(description="おかゆbotの全コマンド一覧を表示します。")
async def help(interaction: discord.Interaction):
    text = (
        "📌 **コマンド一覧**\n"
        "- `/ping` : Bot応答確認\n"
        "- `/dice count sides` : サイコロを振る\n"
        "- `/timer preset/custom` : タイマー設定\n"
        "- `/topic` : 会話用お題表示\n"
        "- `/memo` : メモ作成・通知\n"
        "- `/memo_list` : 自分のメモ一覧をDMで表示\n"
        "- `/memo_history_list` : 過去メモ履歴をDMで表示\n"
        "- `/timeout member minutes` : タイムアウト\n"
        "- `/untimeout member` : タイムアウト解除\n"
        "- `/automod-setting options` : AutoMod荒らし対策設定\n"
        "- `/translate text target_lang` : 翻訳\n"
        "- `/weather place` : 天気情報\n"
        "- `/info` : Bot詳細\n"
        "- `/help` : コマンド一覧表示\n"
    )
    await safe_send(interaction, embed(embed("📖 おかゆbotヘルプ", text)))

# ===============================
# /timeout /untimeout
# ===============================
@bot.tree.command(description="指定ユーザーを一時的にタイムアウトします。")
async def timeout(interaction: discord.Interaction, member: discord.Member, minutes: int):
    until = datetime.datetime.utcnow() + datetime.timedelta(minutes=minutes)
    await member.timeout(until)
    await safe_send(interaction, embed(embed("⏳ Timeout", f"{member} を {minutes} 分タイムアウトしました。")))

@bot.tree.command(description="指定ユーザーのタイムアウトを解除します。")
async def untimeout(interaction: discord.Interaction, member: discord.Member):
    await member.timeout(None)
    await safe_send(interaction, embed(embed("✅ Timeout解除", f"{member} の制限を解除しました。")))

# ===============================
# Bot 起動
# ===============================
bot.run(TOKEN)