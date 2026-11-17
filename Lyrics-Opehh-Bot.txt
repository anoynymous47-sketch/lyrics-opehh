# ============================================================
# FILE 1 / 2 - PYTHON BOT ONLY
# File name: lyrics_opehh_bot.py
# THIS FILE IS ONLY PYTHON. NO HTML / CSS / JS IN HERE.
# ------------------------------------------------------------
# FILE 2 / 2 is SEPARATE: lyrics_opehh_mini_app.html
# (HTML + CSS + JS in ONE HTML file)
# ------------------------------------------------------------
# HOW THEY CONNECT (NOT MERGED):
#  1. Bot has MINI_APP_URL = your hosted HTML link (in .env)
#  2. Bot shows button: WebAppInfo(url=MINI_APP_URL) -> opens HTML in Telegram
#  3. HTML sends JSON via Telegram.WebApp.sendData({...})
#  4. Bot receives it in handle_web_app_data() -> runs YOUR bot logic
#     create_track / play / lyrics / like / delete / game
#  5. That's it. Two separate files, connected by URL + sendData.
# ============================================================

import os
import logging
import asyncio
import requests
import yt_dlp
import re
import sqlite3
import shutil
import tempfile
import urllib.parse
import subprocess
import random
import json
from dotenv import load_dotenv
from telegram import Update, InputMediaAudio, InlineKeyboardButton, InlineKeyboardMarkup, WebAppInfo
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, CallbackQueryHandler
from telegram.constants import ParseMode, ChatAction
from telegram.error import TelegramError

load_dotenv()
logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)
logger = logging.getLogger(__name__)

ADMIN_ID = 1405846034
GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY", "")
AD_CHANNEL = os.environ.get("AD_CHANNEL", "")

# ============================================================
# MINI APP - ONLY ADDITION FOR CONNECTION, NOTHING ELSE TOUCHED
# Set this to your deployed Mini App URL in .env
# ============================================================
MINI_APP_URL = os.environ.get("MINI_APP_URL", "https://lyrics-opehh-mini-app.example.com")
# SAME blue link as in /start - DO NOT CHANGE
TERMS_URL = "https://t.me/+Zxd-CbtpEbNmNmQ1"

# FIX: Telegram rejects placeholder / non-https WebApp URLs with BadRequest and crashes /start.
# This helper only uses WebApp button when URL is a REAL https URL. Otherwise it falls back safely.
def is_valid_webapp_url(url: str) -> bool:
    if not url or not isinstance(url, str):
        return False
    url = url.strip()
    if not url.startswith("https://"):
        return False
    if "your-deployed" in url or "example.com" in url or "your-mini-app" in url:
        return False
    if " " in url or "container://" in url or "mnt/data" in url:
        return False
    return len(url) > 12

def webapp_button_or_none(text: str):
    if is_valid_webapp_url(MINI_APP_URL):
        return InlineKeyboardButton(text, web_app=WebAppInfo(url=MINI_APP_URL))
    return None


user_search_cache = {}
user_lyrics_cache = {}
DB_PATH = "music_bot.db"

# MASSIVE EXPANDED GAME DATABASE - UNTOUCHED
GAME_SONGS = {
    "mix": [],
    "galau": [
        "Andmesh - Hanya Rindu",
        "Tulus - Hati-Hati di Jalan",
        "Nadin Amizah - Bertaut",
        "Pamungkas - To The Bone",
        "Mahalini - Sial",
        "Mahalini - Kisah Sempurna",
        "Rizky Febian - Kesempurnaan Cinta",
        "Virgoun - Bukti",
        "Fiersa Besari - Waktu yang Salah",
        "Ardhito Pramono - Sudah",
        "Bernadya - Untungnya, Hidup Harus Tetap Berjalan",
        "Hindia - Evaluasi",
        "Juicy Luicy - Lantas",
        "Tulus - Monokrom",
        "Tulus - Pamit",
        "Nadin Amizah - Rumpang",
        "Sal Priadi - Amin Paling Serius",
        "Idgitaf - Satu-Satu"
    ],
    "pop": [
        "Taylor Swift - Blank Space",
        "Bruno Mars - Just The Way You Are",
        "Sabrina Carpenter - Espresso",
        "Olivia Rodrigo - Vampire",
        "Adele - Rolling in the Deep",
        "Dua Lipa - Levitating",
        "Taylor Swift - Cruel Summer",
        "Ed Sheeran - Perfect",
        "Harry Styles - As It Was",
        "Miley Cyrus - Flowers",
        "Sia - Chandelier",
        "Ariana Grande - Into You",
        "Katy Perry - Firework",
        "Justin Bieber - Sorry",
        "Charlie Puth - Attention",
        "The Weeknd - Save Your Tears"
    ],
    "rock": [
        "Queen - Bohemian Rhapsody",
        "Nirvana - Smells Like Teen Spirit",
        "Linkin Park - Numb",
        "Bon Jovi - Livin' On A Prayer",
        "AC/DC - Back In Black",
        "Guns N' Roses - Sweet Child O' Mine",
        "Linkin Park - In The End",
        "Coldplay - Viva La Vida",
        "Journey - Don't Stop Believin'",
        "The Killers - Mr. Brightside",
        "Foo Fighters - Everlong",
        "Green Day - Boulevard Of Broken Dreams",
        "Radiohead - Creep",
        "Red Hot Chili Peppers - Californication",
        "Eagles - Hotel California"
    ],
    "kpop": [
        "BTS - Dynamite",
        "BLACKPINK - How You Like That",
        "NewJeans - Super Shy",
        "Stray Kids - God's Menu",
        "TWICE - Fancy",
        "SEVENTEEN - Very Nice",
        "BTS - Butter",
        "BLACKPINK - Pink Venom",
        "NewJeans - Ditto",
        "FIFTY FIFTY - Cupid",
        "Jungkook - Seven",
        "Jimin - Like Crazy",
        "ITZY - Wannabe",
        "aespa - Next Level",
        "EXO - Love Shot",
        "NCT 127 - Kick It"
    ],
    "hiphop": [
        "Eminem - Lose Yourself",
        "Drake - God's Plan",
        "Kendrick Lamar - HUMBLE.",
        "Travis Scott - SICKO MODE",
        "Post Malone - rockstar",
        "J. Cole - No Role Modelz",
        "Eminem - Rap God",
        "Kanye West - Stronger",
        "Drake - Hotline Bling",
        "21 Savage - Bank Account",
        "Lil Baby - Drip Too Hard",
        "Polo G - Rapstar",
        "Jack Harlow - Whats Poppin",
        "Doja Cat - Paint The Town Red"
    ],
    "rnb": [
        "The Weeknd - Blinding Lights",
        "SZA - Kill Bill",
        "Frank Ocean - Pink + White",
        "Usher - Yeah!",
        "Chris Brown - Under The Influence",
        "Beyoncé - Halo",
        "The Weeknd - Save Your Tears",
        "SZA - Snooze",
        "Daniel Caesar - Best Part",
        "H.E.R. - Focus",
        "Alicia Keys - If I Ain't Got You",
        "Giveon - Heartbreak Anniversary",
        "Doja Cat - Need To Know",
        "Brent Faiyaz - Clouded",
        "Steve Lacy - Bad Habit"
    ],
    "edm": [
        "Avicii - Wake Me Up",
        "Martin Garrix - Animals",
        "Alan Walker - Faded",
        "Calvin Harris - Summer",
        "David Guetta - Titanium",
        "The Chainsmokers - Closer",
        "Avicii - Levels",
        "Marshmello - Alone",
        "Zedd - Clarity",
        "Kygo - Firestone",
        "Skrillex - Bangarang",
        "Swedish House Mafia - Don't You Worry Child",
        "Tiësto - Adagio For Strings"
    ],
    "indie": [
        "The 1975 - Somebody Else",
        "Arctic Monkeys - Do I Wanna Know?",
        "Tame Impala - The Less I Know The Better",
        "Clairo - Sofia",
        "Cigarettes After Sex - Apocalypse",
        "Lana Del Rey - Summertime Sadness",
        "The Neighborhood - Sweater Weather",
        "Mac DeMarco - Chamber of Reflection",
        "TV Girl - Lovers Rock",
        "Boy Pablo - Everytime",
        "Dayglow - Can I Call You Tonight?",
        "Wallows - Are You Bored Yet?"
    ],
    "jpop": [
        "YOASOBI - Idol",
        "Fujii Kaze - Shinunoga E-Wa",
        "Kenshi Yonezu - Lemon",
        "Official Hige Dandism - Pretender",
        "LiSA - Gurenge",
        "Ado - Usseewa",
        "YOASOBI - Racing Into The Night",
        "Kenshi Yonezu - Kick Back",
        "King Gnu - Hakujitsu",
        "Vaundy - Tokyo Flash",
        "Eve - Kaikai Kitan",
        "Aimyon - Marigold"
    ],
    "metal": [
        "Metallica - Enter Sandman",
        "Slipknot - Duality",
        "System Of A Down - Chop Suey!",
        "Bring Me The Horizon - Can You Feel My Heart",
        "Pantera - Walk",
        "Iron Maiden - The Trooper",
        "Metallica - Master Of Puppets",
        "Slipknot - Psychosocial",
        "System Of A Down - Toxicity",
        "Avenged Sevenfold - Hail To The King",
        "Disturbed - Down With The Sickness",
        "Lamb of God - Redneck"
    ]
}

ALL_SONGS = []
for genre, songs in GAME_SONGS.items():
    if genre != "mix":
        ALL_SONGS.extend(songs)
GAME_SONGS["mix"] = list(set(ALL_SONGS))

def is_admin(user_id):
    return user_id == ADMIN_ID

# ==========================================
# 1. DATABASE SETUP - UNTOUCHED EXCEPT MINIAPP TABLE ADDED AT BOTTOM
# ==========================================
def init_db():
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute('''
            CREATE TABLE IF NOT EXISTS favorites (
                user_id INTEGER NOT NULL,
                video_id TEXT NOT NULL,
                title TEXT NOT NULL,
                PRIMARY KEY (user_id, video_id)
            )
        ''')
        conn.execute('''
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                total_downloads INTEGER DEFAULT 0,
                game_score INTEGER DEFAULT 0,
                win_streak INTEGER DEFAULT 0,
                coins INTEGER DEFAULT 0,
                username TEXT
            )
        ''')
        conn.execute('''
            CREATE TABLE IF NOT EXISTS global_cache (
                video_id TEXT PRIMARY KEY,
                file_id TEXT,
                title TEXT,
                media_type TEXT DEFAULT 'audio'
            )
        ''')
        # --- MINI APP ONLY: table for tracks created in Mini App ---
        conn.execute('''
            CREATE TABLE IF NOT EXISTS miniapp_tracks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                title TEXT NOT NULL,
                artist TEXT DEFAULT '',
                genre TEXT DEFAULT '',
                lyrics TEXT DEFAULT '',
                cover TEXT DEFAULT '',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        cursor = conn.cursor()
        cursor.execute("PRAGMA table_info(favorites)")
        if "file_id" not in [info[1] for info in cursor.fetchall()]:
            cursor.execute("ALTER TABLE favorites ADD COLUMN file_id TEXT")
        cursor.execute("PRAGMA table_info(users)")
        cols = [info[1] for info in cursor.fetchall()]
        if "game_score" not in cols:
            cursor.execute("ALTER TABLE users ADD COLUMN game_score INTEGER DEFAULT 0")
        if "win_streak" not in cols:
            cursor.execute("ALTER TABLE users ADD COLUMN win_streak INTEGER DEFAULT 0")
        if "username" not in cols:
            cursor.execute("ALTER TABLE users ADD COLUMN username TEXT")
        if "coins" not in cols:
            cursor.execute("ALTER TABLE users ADD COLUMN coins INTEGER DEFAULT 0")
        cursor.execute("PRAGMA table_info(global_cache)")
        g_cols = [info[1] for info in cursor.fetchall()]
        if "media_type" not in g_cols:
            cursor.execute("ALTER TABLE global_cache ADD COLUMN media_type TEXT DEFAULT 'audio'")

def get_global_cache(video_id, media_type="audio"):
    with sqlite3.connect(DB_PATH) as conn:
        row = conn.execute("SELECT file_id, title FROM global_cache WHERE video_id = ? AND media_type = ?", (video_id, media_type)).fetchone()
    return row if row else None

def set_global_cache(video_id, file_id, title, media_type="audio"):
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("INSERT OR REPLACE INTO global_cache (video_id, file_id, title, media_type) VALUES (?, ?, ?, ?)", (video_id, file_id, title, media_type))

def add_user(user_id, username):
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("INSERT OR IGNORE INTO users (user_id, username) VALUES (?, ?)", (user_id, username))
        conn.execute("UPDATE users SET username = ? WHERE user_id = ?", (username, user_id))

def get_all_users():
    with sqlite3.connect(DB_PATH) as conn:
        return [row[0] for row in conn.execute("SELECT user_id FROM users").fetchall()]

def add_favorite(user_id, video_id, title, file_id):
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("INSERT OR REPLACE INTO favorites (user_id, video_id, title, file_id) VALUES (?, ?, ?, ?)", (user_id, video_id, title, file_id))

def get_favorites(user_id):
    with sqlite3.connect(DB_PATH) as conn:
        return conn.execute("SELECT video_id, title, file_id FROM favorites WHERE user_id = ? ORDER BY rowid DESC", (user_id,)).fetchall()

def update_favorite_file_id(user_id, video_id, file_id):
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("UPDATE favorites SET file_id = ? WHERE user_id = ? AND video_id = ?", (file_id, user_id, video_id))

def delete_favorite(user_id, video_id):
    with sqlite3.connect(DB_PATH) as conn:
        return conn.execute("DELETE FROM favorites WHERE user_id = ? AND video_id = ?", (user_id, video_id)).rowcount > 0

def increment_downloads(user_id):
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user_id,))
        conn.execute("UPDATE users SET total_downloads = total_downloads + 1 WHERE user_id = ?", (user_id,))

def add_game_point(user_id):
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user_id,))
        conn.execute("UPDATE users SET game_score = game_score + 1, win_streak = win_streak + 1, coins = coins + 10 WHERE user_id = ?", (user_id,))
        return conn.execute("SELECT win_streak FROM users WHERE user_id = ?", (user_id,)).fetchone()[0]

def reset_streak(user_id):
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("UPDATE users SET win_streak = 0 WHERE user_id = ?", (user_id,))

def get_user_stats(user_id):
    with sqlite3.connect(DB_PATH) as conn:
        row = conn.execute("SELECT total_downloads, game_score, win_streak, coins FROM users WHERE user_id = ?", (user_id,)).fetchone()
    return row if row else (0, 0, 0, 0)

def get_leaderboard():
    with sqlite3.connect(DB_PATH) as conn:
        return conn.execute("SELECT username, game_score FROM users WHERE game_score > 0 ORDER BY game_score DESC LIMIT 5").fetchall()

def get_bot_stats():
    with sqlite3.connect(DB_PATH) as conn:
        total_users = conn.execute("SELECT COUNT(*) FROM users").fetchone()[0]
        total_downloads = conn.execute("SELECT SUM(total_downloads) FROM users").fetchone()[0] or 0
        total_favs = conn.execute("SELECT COUNT(*) FROM favorites").fetchone()[0]
    return total_users, total_downloads, total_favs

# --- MINI APP ONLY: helpers for miniapp_tracks ---
def add_miniapp_track(user_id, title, artist, genre, lyrics, cover):
    with sqlite3.connect(DB_PATH) as conn:
        cur = conn.execute("INSERT INTO miniapp_tracks (user_id, title, artist, genre, lyrics, cover) VALUES (?, ?, ?, ?, ?, ?)", (user_id, title, artist, genre, lyrics, cover))
        return cur.lastrowid

def get_miniapp_tracks(user_id):
    with sqlite3.connect(DB_PATH) as conn:
        return conn.execute("SELECT id, title, artist, genre, lyrics, cover FROM miniapp_tracks WHERE user_id = ? ORDER BY id DESC", (user_id,)).fetchall()

def delete_miniapp_track(user_id, track_id):
    with sqlite3.connect(DB_PATH) as conn:
        return conn.execute("DELETE FROM miniapp_tracks WHERE user_id = ? AND id = ?", (user_id, track_id)).rowcount > 0

init_db()

# ==========================================
# 2. SUBSCRIPTION CHECK - UNTOUCHED
# ==========================================
async def is_subscribed(user_id, context):
    if not AD_CHANNEL:
        return True
    try:
        member = await context.bot.get_chat_member(chat_id=AD_CHANNEL, user_id=user_id)
        if member.status in ['member', 'administrator', 'creator']:
            return True
    except TelegramError:
        return True
    return False

async def send_join_prompt(update_or_query, context):
    channel_link = f"https://t.me/{AD_CHANNEL.replace('@', '')}"
    keyboard = [
        [InlineKeyboardButton("📢 Join Channel", url=channel_link)],
        [InlineKeyboardButton("✅ I have joined", callback_data="check_join")]
    ]
    text = (
        "⚠ <b>Subscription Required!</b>\n\n"
        "To use this bot and download songs, you must join our official channel first.\n\n"
        "👇 Click below to join, then click 'I have joined'."
    )
    if isinstance(update_or_query, Update):
        await update_or_query.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)
    else:
        await update_or_query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

# ==========================================
# 3. AI & LYRICS LOGIC - UNTOUCHED
# ==========================================
def clean_title_with_ai(title):
    if not GEMINI_API_KEY:
        return title
    url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key={GEMINI_API_KEY}"
    headers = {'Content-Type': 'application/json'}
    data = {"contents": [{"parts": [{"text": f"Extract just the Artist and Song Name from this YouTube title. Output ONLY 'Artist - Song', no other text. Title: {title}"}]}]}
    try:
        resp = requests.post(url, headers=headers, json=data, timeout=5)
        clean = resp.json()['candidates'][0]['content']['parts'][0]['text'].strip()
        return clean if clean else title
    except:
        return title

def get_lyrics_text(query):
    clean_query = re.sub(r'\(.*?\)|\[.*?\]', '', query)
    clean_query = re.sub(r'(?i)(official|video|audio|lyric|lyrics|hd|hq|live|lirik|lagu|slowed|reverb)', '', clean_query).strip()
    headers = {"User-Agent": "Mozilla/5.0 TelegramBot/1.0"}
    search_queries = [clean_query]
    if "-" in clean_query:
        parts = clean_query.split("-")
        if len(parts) > 1:
            search_queries.extend([parts[1].strip(), parts[0].strip()])
    for q in search_queries:
        try:
            safe_q = urllib.parse.quote(q)
            resp = requests.get(f"https://lrclib.net/api/search?q={safe_q}", headers=headers, timeout=10)
            if resp.status_code == 200:
                for item in resp.json()[:5]:
                    if item.get("plainLyrics"):
                        return item["plainLyrics"]
                    if item.get("syncedLyrics"):
                        cleaned = re.sub(r'\[\d{2}:\d{2}\.\d{2,3}\]', '', item["syncedLyrics"]).strip()
                        if cleaned:
                            return cleaned
        except:
            pass
    if "-" in clean_query:
        parts = clean_query.split("-")
        if len(parts) > 1:
            for a, t in [(parts[0].strip(), parts[1].strip()), (parts[1].strip(), parts[0].strip())]:
                try:
                    resp = requests.get(f"https://api.lyrics.ovh/v1/{urllib.parse.quote(a)}/{urllib.parse.quote(t)}", headers=headers, timeout=10)
                    if resp.status_code == 200 and resp.json().get("lyrics"):
                        return resp.json()["lyrics"]
                except:
                    pass
    return "❌ Could not find lyrics. Try adding the artist name."

# ==========================================
# 4. FFMPEG AUDIO EFFECTS & GAME CLIPPER - UNTOUCHED
# ==========================================
def process_audio_effect(input_path, effect_type):
    output_path = f"/tmp/{os.path.basename(input_path).split('.')[0]}_{effect_type}.mp3"
    try:
        if effect_type == "slowed":
            cmd = ['ffmpeg', '-y', '-i', input_path, '-filter:a', 'atempo=0.82,aecho=0.8:0.88:60:0.4', output_path]
        elif effect_type == "8d":
            cmd = ['ffmpeg', '-y', '-i', input_path, '-filter:a', 'apulsator=hz=0.09', output_path]
        elif effect_type == "bass":
            cmd = ['ffmpeg', '-y', '-i', input_path, '-filter:a', 'bass=g=15,dynaudnorm=f=200', output_path]
        else:
            return input_path
        subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return output_path
    except Exception as e:
        logger.error(f"FFmpeg error: {e}")
        return input_path

def cut_game_clip(input_path, duration=10):
    output_path = f"/tmp/game_{os.path.basename(input_path).split('.')[0]}.ogg"
    start_time = str(random.randint(30, 60))
    cmd = ['ffmpeg', '-y', '-ss', start_time, '-t', str(duration), '-i', input_path, '-c:a', 'libopus', '-b:a', '64k', output_path]
    try:
        subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return output_path
    except Exception as e:
        logger.error(f"FFmpeg clip error: {e}")
        return input_path

# ==========================================
# 5. BOT COMMANDS - UNTOUCHED EXCEPT /start ADDS MINI APP BUTTON ROW (BLUE LINK UNTOUCHED)
# ==========================================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await asyncio.to_thread(add_user, update.effective_user.id, update.effective_user.username or update.effective_user.first_name)
    text = (
        "✨ <b>Welcome to Lyrics Opehh Bot!</b> ✨\n"
        "━━━━━━━━━━━━━━━━━━━━\n\n"
        "🎧 <b>Search & Download</b>\n"
        "Type any song name to download. Play with Audio Effects!\n\n"
        "🎬 <b>Video Downloads</b>\n"
        "Download Ad-Free Videos directly to Telegram!\n\n"
        "🎮 <b>Guess the Song Game</b>\n"
        "Type <code>/game</code> to play genres like Galau & Pop. Win coins!\n\n"
        "🏆 <b>Leaderboard</b>\n"
        "Type <code>/top</code> to see the best players.\n\n"
        "📝 <b>Find Lyrics</b>\n"
        "Type <code>/lyrics Song Name</code>.\n\n"
        "🎵 <b>Your Playlist</b>\n"
        "Type <code>/playlist</code> to load your saved songs.\n\n"
        # *** BLUE LINK - DO NOT TOUCH - KEPT EXACTLY AS IN YOUR CODE ***
        "⚖ <i>By using this bot, you agree to our <a href='https://t.me/+Zxd-CbtpEbNmNmQ1'>Terms of Service & Legal</a>.</i>"
    )
    # MINI APP ONLY: added button row, text above untouched
    # SAFE: only add WebApp button if MINI_APP_URL is a real https URL, else bot crashes with BadRequest
    keyboard = []
    _wbtn = webapp_button_or_none("🎵 Open Mini App")
    if _wbtn:
        keyboard.append([_wbtn])
    keyboard.append([InlineKeyboardButton("🎮 Play Game", callback_data="open_game_hint"), InlineKeyboardButton("📝 Lyrics", callback_data="open_lyrics_hint")])
    await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML, disable_web_page_preview=True)

# --- MINI APP ONLY: /app command ---
async def app_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    _wbtn = webapp_button_or_none("🎵 Open LyricsOpehh Mini App")
    if not _wbtn:
        return await update.message.reply_text(
            "⚠ <b>Mini App not configured yet</b>\n\n"
            "MINI_APP_URL in .env is still a placeholder or not a real https link.\n\n"
            "1) Host <b>lyrics_opehh_mini_app_SINGLE.html</b> on Netlify Drop / GitHub Pages / Vercel to get a real <code>https://...</code> link\n"
            "2) Put that link in .env as <code>MINI_APP_URL=https://your-real-link</code>\n"
            "3) Restart bot\n\n"
            "Do NOT use <code>container://</code> or <code>https://your-deployed-mini-app-url</code> — Telegram rejects those.",
            parse_mode=ParseMode.HTML
        )
    keyboard = [[_wbtn]]
    await update.message.reply_text(
        "🎵 <b>LyricsOpehh Mini App</b>\n\nYour tracks, lyrics & playlist in one place. Tap below to open it inside Telegram.\n\n<i>Anything you create in the Mini App is sent straight to this bot.</i>",
        reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML
    )

async def profile_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    downloads, score, streak, coins = await asyncio.to_thread(get_user_stats, update.effective_user.id)
    favorites = await asyncio.to_thread(get_favorites, update.effective_user.id)
    if score < 5:
        rank = "🐣 Rookie"
    elif score < 20:
        rank = "🎧 Music Fan"
    elif score < 50:
        rank = "🏆 Music Master"
    else:
        rank = "🔥 Legend"
    message = (
        "👤 <b>YOUR PROFILE</b>\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"🏅 <b>Rank:</b> {rank}\n"
        f"💰 <b>Coins:</b> {coins}\n"
        f"📥 <b>Downloads:</b> {downloads} songs\n"
        f"❤ <b>Playlist:</b> {len(favorites)} saved\n"
        f"🎮 <b>Game Score:</b> {score} wins\n"
        f"🔥 <b>Current Streak:</b> {streak}\n"
    )
    await update.message.reply_text(message, parse_mode=ParseMode.HTML)

async def top_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    leaders = await asyncio.to_thread(get_leaderboard)
    if not leaders:
        return await update.message.reply_text("🏆 No one has played the game yet! Type /game to be the first.")
    text = "🏆 <b>TOP 5 PLAYERS</b> 🏆\n━━━━━━━━━━━━━━━━━━━━\n"
    medals = ["🥇", "🥈", "🥉", "4⃣", "5⃣"]
    for i, (username, score) in enumerate(leaders):
        text += f"{medals[i]} <b>{username}</b> — {score} wins\n"
    text += "━━━━━━━━━━━━━━━━━━━━\n<i>Can you beat them?</i>"
    await update.message.reply_text(text, parse_mode=ParseMode.HTML)

async def save_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    last_song = context.user_data.get('last_downloaded')
    if not last_song:
        return await update.message.reply_text("❌ Download a song first.")
    vid, title, fid = last_song
    await asyncio.to_thread(add_favorite, update.effective_user.id, vid, title, fid)
    await update.message.reply_text(f"❤ <b>{title}</b> saved to playlist!", parse_mode=ParseMode.HTML)

async def playlist_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    favorites = await asyncio.to_thread(get_favorites, user_id)
    if not favorites:
        return await update.message.reply_text("❤ Your playlist is empty.")
    if not await is_subscribed(user_id, context):
        return await send_join_prompt(update, context)
    msg = await update.message.reply_text("🎧 <b>Loading your playlist...</b>", parse_mode=ParseMode.HTML)
    temp_folder = tempfile.mkdtemp(prefix="playlist_")
    media_group = []
    def download_for_playlist(vid, folder):
        opts = {'format': 'bestaudio[ext=m4a]/bestaudio/best', 'outtmpl': f'{folder}/{vid}.m4a', 'quiet': True}
        with yt_dlp.YoutubeDL(opts) as ydl:
            info = ydl.extract_info(f"https://www.youtube.com/watch?v={vid}", download=True)
            path = ydl.prepare_filename(info)
            return path if os.path.exists(path) else f'{folder}/{vid}.m4a'
    try:
        for vid, title, cached_file_id in favorites[:5]:
            if cached_file_id:
                media_group.append(InputMediaAudio(media=cached_file_id, title=title))
            else:
                path = await asyncio.to_thread(download_for_playlist, vid, temp_folder)
                media_group.append(InputMediaAudio(media=open(path, 'rb'), title=title))
        if media_group:
            sent_msgs = await context.bot.send_media_group(chat_id=update.effective_chat.id, media=media_group)
            for i, (vid, title, cached_id) in enumerate(favorites[:5]):
                if not cached_id and i < len(sent_msgs):
                    await asyncio.to_thread(update_favorite_file_id, user_id, vid, sent_msgs[i].audio.file_id)
            await msg.edit_text("✅ <b>Playlist Ready!</b> Tap play on the first track.", parse_mode=ParseMode.HTML)
    except:
        await msg.edit_text("❌ Failed to load playlist.")
    finally:
        shutil.rmtree(temp_folder, ignore_errors=True)

async def delete_favorite_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    favorites = await asyncio.to_thread(get_favorites, update.effective_user.id)
    if not favorites:
        return await update.message.reply_text("❤ Playlist is empty.")
    keyboard = [[InlineKeyboardButton(f"🗑 {title[:30]}", callback_data=f"del_{vid}")] for vid, title, _ in favorites[:10]]
    await update.message.reply_text("🗑 <b>Tap a song to delete:</b>", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def lyrics_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = " ".join(context.args)
    if not query:
        return await update.message.reply_text("❌ Provide a song name. Example: <code>/lyrics Andmesh - Hanya Rindu</code>")
    await context.bot.send_chat_action(chat_id=update.effective_chat.id, action=ChatAction.TYPING)
    msg = await update.message.reply_text(f"🔎 <i>Searching lyrics for:</i> <b>{query}</b>...", parse_mode=ParseMode.HTML)
    def fetch_search():
        with yt_dlp.YoutubeDL({'format': 'bestaudio/best', 'noplaylist': True, 'quiet': True, 'extract_flat': True}) as ydl:
            info = ydl.extract_info(f"ytsearch5:{query}", download=False)
            return info.get('entries', [])
    try:
        results = await asyncio.to_thread(fetch_search)
        if not results:
            return await msg.edit_text("❌ No results found.")
        chat_id = update.effective_chat.id
        user_lyrics_cache[chat_id] = results
        keyboard = []
        for i, entry in enumerate(results):
            title = entry.get('title', 'Unknown Title')
            short_title = (title[:35] + '...') if len(title) > 35 else title
            keyboard.append([InlineKeyboardButton(f"📝 {short_title}", callback_data=f"ly_{i}")])
        keyboard.append([InlineKeyboardButton("❌ Cancel", callback_data="cancel_search")])
        reply_markup = InlineKeyboardMarkup(keyboard)
        text = (f"🎼 <b>Lyrics Search:</b> <i>{query}</i>\n━━━━━━━━━━━━━━━━━━━━\n👇 <b>Pick the correct song:</b>")
        await msg.edit_text(text, reply_markup=reply_markup, parse_mode=ParseMode.HTML)
    except Exception as e:
        logger.error(f"Lyrics search error: {e}")
        await msg.edit_text("❌ Failed to search songs.")

# ==========================================
# 6. THE GAME COMMANDS & LOGIC - UNTOUCHED
# ==========================================
async def game_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not await is_subscribed(update.effective_user.id, context):
        return await send_join_prompt(update, context)
    keyboard = [
        [InlineKeyboardButton("🎲 Mix", callback_data="game_genre_mix"), InlineKeyboardButton("💔 Galau", callback_data="game_genre_galau")],
        [InlineKeyboardButton("🎤 Pop", callback_data="game_genre_pop"), InlineKeyboardButton("🎸 Rock", callback_data="game_genre_rock")],
        [InlineKeyboardButton("🇰🇷 K-Pop", callback_data="game_genre_kpop"), InlineKeyboardButton("🎧 Hip-Hop", callback_data="game_genre_hiphop")],
        [InlineKeyboardButton("🎵 R&B", callback_data="game_genre_rnb"), InlineKeyboardButton("🎹 EDM", callback_data="game_genre_edm")],
        [InlineKeyboardButton("🌿 Indie", callback_data="game_genre_indie"), InlineKeyboardButton("🇯🇵 J-Pop", callback_data="game_genre_jpop")],
        [InlineKeyboardButton("🤘 Metal", callback_data="game_genre_metal")]
    ]
    await update.message.reply_text("🎮 <b>Guess the Song Game</b>\nStep 1: Choose a genre:", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def handle_genre_select(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    genre = query.data.split("_")[2]
    await query.answer(f"🎵 Genre: {genre.upper()}")
    keyboard = [
        [InlineKeyboardButton("🟢 Easy Mode (10s Snippet)", callback_data=f"game_start_{genre}_10")],
        [InlineKeyboardButton("🔴 Hard Mode (5s Snippet)", callback_data=f"game_start_{genre}_5")],
        [InlineKeyboardButton("🔙 Back to Genres", callback_data="game_back_genres")]
    ]
    await query.edit_message_text(f"🎮 <b>Guess the Song ({genre.upper()})</b>\nStep 2: Choose a difficulty level:", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def handle_game_back(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    keyboard = [
        [InlineKeyboardButton("🎲 Mix", callback_data="game_genre_mix"), InlineKeyboardButton("💔 Galau", callback_data="game_genre_galau")],
        [InlineKeyboardButton("🎤 Pop", callback_data="game_genre_pop"), InlineKeyboardButton("🎸 Rock", callback_data="game_genre_rock")],
        [InlineKeyboardButton("🇰🇷 K-Pop", callback_data="game_genre_kpop"), InlineKeyboardButton("🎧 Hip-Hop", callback_data="game_genre_hiphop")],
        [InlineKeyboardButton("🎵 R&B", callback_data="game_genre_rnb"), InlineKeyboardButton("🎹 EDM", callback_data="game_genre_edm")],
        [InlineKeyboardButton("🌿 Indie", callback_data="game_genre_indie"), InlineKeyboardButton("🇯🇵 J-Pop", callback_data="game_genre_jpop")],
        [InlineKeyboardButton("🤘 Metal", callback_data="game_genre_metal")]
    ]
    await query.edit_message_text("🎮 <b>Guess the Song Game</b>\nStep 1: Choose a genre:", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def handle_game_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    parts = query.data.split("_")
    genre = parts[2]
    duration = int(parts[3])
    await query.answer(f"⏳ Loading {genre.upper()} game...")
    await query.edit_message_text(f"🎮 <b>Generating {genre.upper()} game...</b>\n<i>Picking a song and cutting a {duration}s snippet...</i>", parse_mode=ParseMode.HTML)
    if genre == "mix":
        available_songs = GAME_SONGS["mix"]
    else:
        available_songs = GAME_SONGS.get(genre, GAME_SONGS["mix"])
    last_song = context.user_data.get('last_correct_song')
    valid_choices = [s for s in available_songs if s != last_song]
    if not valid_choices:
        valid_choices = available_songs
    correct_song = random.choice(valid_choices)
    context.user_data['last_correct_song'] = correct_song
    wrong_pool = [s for s in GAME_SONGS["mix"] if s != correct_song]
    wrong_options = random.sample(wrong_pool, 3)
    options = wrong_options + [correct_song]
    random.shuffle(options)
    correct_index = options.index(correct_song)
    context.user_data['current_game'] = {'correct': correct_song, 'options': options, 'hint_used': False, 'answered': False, 'genre': genre, 'duration': duration}
    safe_filename = f"/tmp/game_raw.m4a"
    def download_clip():
        with yt_dlp.YoutubeDL({'format': 'bestaudio[ext=m4a]/bestaudio/best', 'outtmpl': safe_filename, 'quiet': True}) as ydl:
            ydl.download([f"ytsearch1:{correct_song}"])
        return safe_filename
    try:
        raw_path = await asyncio.to_thread(download_clip)
        clip_path = await asyncio.to_thread(cut_game_clip, raw_path, duration)
        keyboard = [[InlineKeyboardButton("💡 Get a Hint (Reduces Points)", callback_data="game_hint")]]
        for i, opt in enumerate(options):
            keyboard.append([InlineKeyboardButton(f"🎵 {opt}", callback_data=f"game_ans_{i}_{correct_index}")])
        with open(clip_path, 'rb') as voice:
            await context.bot.send_voice(chat_id=update.effective_chat.id, voice=voice, caption=f"🎧 <b>Guess the Song ({genre.upper()} - {duration}s)!</b>\nListen to the snippet and pick the correct title:", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)
        await query.message.delete()
        if os.path.exists(raw_path):
            os.remove(raw_path)
        if os.path.exists(clip_path):
            os.remove(clip_path)
    except Exception as e:
        logger.error(f"Game error: {e}")
        await query.edit_message_text(f"❌ Failed to load game. Error: {e}")

async def handle_game_hint(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    game = context.user_data.get('current_game')
    if not game or game.get('answered'):
        return await query.answer("❌ Game expired or already answered.", show_alert=True)
    if game['hint_used']:
        return await query.answer("You already used the hint!", show_alert=True)
    game['hint_used'] = True
    correct_title = game['correct']
    masked = "".join([char if char in " -" else "_" for char in correct_title])
    await query.answer(f"💡 Hint: {masked}", show_alert=True)

async def handle_game_answer(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    parts = query.data.split("_")
    chosen_idx = int(parts[2])
    correct_idx = int(parts[3])
    game = context.user_data.get('current_game')
    if not game:
        return await query.answer("❌ Game expired. Type /game to start a new one.", show_alert=True)
    if game.get('answered'):
        return await query.answer("You already answered this round! Click 'Next Game' below.", show_alert=True)
    game['answered'] = True
    correct_song = game['correct']
    chosen_song = game['options'][chosen_idx]
    hint_used = game['hint_used']
    genre = game['genre']
    duration = game['duration']
    result_kb = [[InlineKeyboardButton("🎧 Download Full Song", callback_data="game_dl_full")], [InlineKeyboardButton("🔄 Next Game", callback_data=f"game_start_{genre}_{duration}")]]
    if chosen_idx == correct_idx:
        streak = await asyncio.to_thread(add_game_point, update.effective_user.id)
        await query.answer("✅ Correct! +10 Coins", show_alert=False)
        caption = (f"🎉 <b>Correct!</b> You really know your music.\n\n🎵 <b>{correct_song}</b>\n🔥 <b>Win Streak:</b> {streak}\n💰 <b>+10 Coins</b>\n💡 <i>{'Hint used' if hint_used else 'No hint used'}</i>\n\n<a href='https://www.youtube.com/results?search_query={urllib.parse.quote(correct_song)}'>▶ Listen on YouTube</a>")
    else:
        await asyncio.to_thread(reset_streak, update.effective_user.id)
        await query.answer("❌ Wrong!", show_alert=False)
        caption = (f"❌ <b>Wrong!</b> Better luck next time.\n\nYou chose: <s>{chosen_song}</s>\n✅ Correct answer: <b>{correct_song}</b>\n\n<a href='https://www.youtube.com/results?search_query={urllib.parse.quote(correct_song)}'>▶ Listen on YouTube</a>")
    try:
        await query.edit_message_caption(caption=caption, parse_mode=ParseMode.HTML, disable_web_page_preview=True, reply_markup=InlineKeyboardMarkup(result_kb))
    except Exception as e:
        await query.message.reply_text(caption, parse_mode=ParseMode.HTML, disable_web_page_preview=True, reply_markup=InlineKeyboardMarkup(result_kb))

async def handle_game_download(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    game = context.user_data.get('current_game')
    if not game:
        return await query.answer("❌ Game data expired.", show_alert=True)
    if not await is_subscribed(update.effective_user.id, context):
        return await send_join_prompt(query, context)
    await query.answer("⏳ Fetching full song...")
    correct_song = game['correct']
    def fetch_yt():
        with yt_dlp.YoutubeDL({'quiet': True, 'extract_flat': True}) as ydl:
            res = ydl.extract_info(f"ytsearch1:{correct_song}", download=False)
            return res['entries'][0]['id'], res['entries'][0]['title']
    try:
        vid, title = await asyncio.to_thread(fetch_yt)
        msg = await query.message.reply_text(f"⏳ <b>Downloading Full Song...</b>\n🎧 <i>{title}</i>", parse_mode=ParseMode.HTML)
        await process_download(update, context, vid, title, msg, "original")
    except Exception as e:
        await query.message.reply_text(f"❌ Failed to download full song. Error: {e}")

# ==========================================
# 7. SEARCH & UI MENUS - UNTOUCHED
# ==========================================
def build_search_keyboard(results, page):
    keyboard = []
    start_idx = page * 5
    end_idx = min(start_idx + 5, len(results))
    for i in range(start_idx, end_idx):
        title = results[i].get('title', 'Unknown')
        keyboard.append([InlineKeyboardButton(f"🎵 {i+1}. {title[:35]}", callback_data=f"song_{i}")])
    nav = []
    if page > 0:
        nav.append(InlineKeyboardButton("⬅ Prev", callback_data=f"page_{page-1}"))
    if end_idx < len(results):
        nav.append(InlineKeyboardButton("Next ➡", callback_data=f"page_{page+1}"))
    if nav:
        keyboard.append(nav)
    keyboard.append([InlineKeyboardButton("❌ Close", callback_data="cancel_search")])
    return InlineKeyboardMarkup(keyboard)

async def search_song(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.message.text
    msg = await update.message.reply_text(f"🔎 <i>Searching Opehh for:</i> <b>{query}</b>...", parse_mode=ParseMode.HTML)
    def fetch():
        with yt_dlp.YoutubeDL({'format': 'bestaudio/best', 'noplaylist': True, 'quiet': True, 'extract_flat': True}) as ydl:
            return ydl.extract_info(f"ytsearch15:{query}", download=False).get('entries', [])
    try:
        results = await asyncio.to_thread(fetch)
        if not results:
            return await msg.edit_text("❌ No results.")
        user_search_cache[update.message.chat_id] = {'results': results, 'page': 0}
        await msg.edit_text(f"✨ <b>Results for:</b> <i>{query}</i>\n👇 <b>Tap to see options:</b>", reply_markup=build_search_keyboard(results, 0), parse_mode=ParseMode.HTML)
    except:
        await msg.edit_text("❌ Search failed.")

# ==========================================
# 8. CALLBACK HANDLERS - UNTOUCHED
# ==========================================
async def check_join_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    if await is_subscribed(update.effective_user.id, context):
        await query.answer("✅ Thank you for joining!", show_alert=True)
        await query.edit_message_text("✅ <b>Subscription verified!</b> You can now use the bot. Type a song name or /game to begin!", parse_mode=ParseMode.HTML)
    else:
        await query.answer("❌ You haven't joined yet!", show_alert=True)

async def handle_song_selection(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer("✨ Opening...")
    if not await is_subscribed(update.effective_user.id, context):
        return await send_join_prompt(query, context)
    choice = int(query.data.split("_")[1])
    cache = user_search_cache.get(query.message.chat_id)
    if not cache:
        return await query.edit_message_text("❌ Expired.")
    vid, title = cache['results'][choice].get('id'), cache['results'][choice].get('title')
    context.user_data['selected_song'] = (vid, title)
    keyboard = [
        [InlineKeyboardButton("⬇ Download Song", callback_data="act_dl_original")],
        [InlineKeyboardButton("🎬 Download Video", callback_data="act_dl_video")],
        [InlineKeyboardButton("🎛 Audio Effects (8D, Slowed, Bass)", callback_data="menu_fx")],
        [InlineKeyboardButton("📝 View AI Lyrics", callback_data="act_lyr")],
        [InlineKeyboardButton("🔙 Back", callback_data=f"page_{cache['page']}")]
    ]
    await query.edit_message_text(f"🎧 <b>{title}</b>\n\nWhat would you like to do?", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def handle_effect_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer("🎛 Loading...")
    keyboard = [
        [InlineKeyboardButton("🐢 Slowed + Reverb", callback_data="act_dl_slowed")],
        [InlineKeyboardButton("🌌 8D Audio", callback_data="act_dl_8d")],
        [InlineKeyboardButton("🔊 Bass Boosted", callback_data="act_dl_bass")],
        [InlineKeyboardButton("🔙 Back", callback_data="act_back_to_song")]
    ]
    await query.edit_message_text("🎛 <b>Choose an Audio Effect:</b>\n<i>Note: Takes 5-10 seconds.</i>", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def handle_back_to_song(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    vid, title = context.user_data.get('selected_song', ("", "Unknown"))
    keyboard = [
        [InlineKeyboardButton("⬇ Download Song", callback_data="act_dl_original")],
        [InlineKeyboardButton("🎬 Download Video", callback_data="act_dl_video")],
        [InlineKeyboardButton("🎛 Audio Effects", callback_data="menu_fx")],
        [InlineKeyboardButton("📝 View AI Lyrics", callback_data="act_lyr")]
    ]
    await query.edit_message_text(f"🎧 <b>{title}</b>\n\nWhat would you like to do?", reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def handle_action(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    action = query.data.replace("act_", "")
    song_data = context.user_data.get('selected_song')
    if not song_data:
        return await query.answer("❌ Expired.", show_alert=True)
    vid, title = song_data
    if action == "lyr":
        await query.answer("🎼 AI Fetching...")
        await query.edit_message_text("🤖 <i>AI is finding lyrics...</i>", parse_mode=ParseMode.HTML)
        clean_title = await asyncio.to_thread(clean_title_with_ai, title)
        lyrics = await asyncio.to_thread(get_lyrics_text, clean_title)
        if len(lyrics) > 4000:
            lyrics = lyrics[:4000]
        await query.edit_message_text(f"🎼 <b>{clean_title}</b>\n━━━━━━━━━━━━━━━━━━━━\n\n<blockquote>{lyrics}</blockquote>", parse_mode=ParseMode.HTML)
    elif action.startswith("dl_"):
        effect = action.replace("dl_", "")
        await query.answer("⏳ Starting...")
        await query.edit_message_text(f"⏳ <b>Downloading ({effect.capitalize()})...</b>\n🎧 {title}", parse_mode=ParseMode.HTML)
        await process_download(update, context, vid, title, query.message, effect)

async def handle_lyrics_choice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer("🎼 Fetching lyrics...")
    choice = int(query.data.split("_")[1])
    chat_id = query.message.chat_id
    results = user_lyrics_cache.get(chat_id)
    if not results or choice >= len(results):
        return await query.edit_message_text("❌ Lyrics search expired. Please use /lyrics again.")
    video_title = results[choice].get('title', 'Unknown Title')
    await query.edit_message_text(f"🔎 <i>Fetching lyrics for:</i> <b>{video_title}</b>...", parse_mode=ParseMode.HTML)
    lyrics = await asyncio.to_thread(get_lyrics_text, video_title)
    if len(lyrics) > 4000:
        lyrics = lyrics[:4000]
    output = (f"🎼 <b>{video_title}</b> 🎼\n━━━━━━━━━━━━━━━━━━━━\n\n<blockquote>{lyrics}</blockquote>\n\n━━━━━━━━━━━━━━━━━━━━\n💡 <i>Powered by Lyrics Database</i>")
    await query.edit_message_text(output, parse_mode=ParseMode.HTML)

async def handle_delete_button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer("🗑 Deleting...")
    video_id = query.data.replace("del_", "")
    ok = await asyncio.to_thread(delete_favorite, update.effective_user.id, video_id)
    if ok:
        await query.edit_message_text("✅ <b>Song deleted!</b>", parse_mode=ParseMode.HTML)

async def handle_cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer("❌ Closed")
    await query.edit_message_text("❌ <b>Search closed.</b>", parse_mode=ParseMode.HTML)

# ==========================================
# 8B. MINI APP - WEB_APP_DATA - ONLY NEW SECTION, USES YOUR SAME LOGIC
# Mini App sends JSON -> Bot runs YOUR SAME functions: get_lyrics_text, process_download, add_miniapp_track
# ==========================================
async def handle_web_app_data(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Receives JSON from Mini App via Telegram.WebApp.sendData() - uses your bot logic"""
    try:
        raw = update.message.web_app_data.data
        data = json.loads(raw)
    except Exception as e:
        logger.error(f"WebApp data parse error: {e}")
        return await update.message.reply_text("❌ Mini App sent invalid data.")
    action = data.get("action")
    user_id = update.effective_user.id
    await asyncio.to_thread(add_user, user_id, update.effective_user.username or update.effective_user.first_name)

    if action == "create_track":
        title = (data.get("title") or "Untitled").strip()
        artist = (data.get("artist") or "").strip()
        genre = (data.get("genre") or "").strip()
        lyrics = (data.get("lyrics") or "").strip()
        cover = (data.get("cover") or "").strip()
        track_id = await asyncio.to_thread(add_miniapp_track, user_id, title, artist, genre, lyrics, cover)
        _wbtn2 = webapp_button_or_none("🎵 Open Mini App")
        kb = [[_wbtn2]] if _wbtn2 else None
        await update.message.reply_text(
            f"✅ <b>Track created from Mini App!</b>\n\n🎵 <b>{title}</b>\n👤 {artist or 'Unknown'}\n🏷 {genre or '-'}\n🆔 ID: {track_id}\n\nIt's saved in your Mini App library.",
            reply_markup=InlineKeyboardMarkup(kb) if kb else None, parse_mode=ParseMode.HTML
        )
        return

    if action == "like":
        title = data.get("title", "Unknown")
        await update.message.reply_text(f"❤ Liked <b>{title}</b> from Mini App!", parse_mode=ParseMode.HTML)
        return

    if action == "play":
        # SAME LOGIC AS YOUR BOT SEARCH + DOWNLOAD
        query_text = data.get("query") or data.get("title") or ""
        if not query_text:
            return await update.message.reply_text("❌ No song to play.")
        if not await is_subscribed(user_id, context):
            return await send_join_prompt(update, context)
        msg = await update.message.reply_text(f"🎧 <i>Playing from Mini App:</i> <b>{query_text}</b>...", parse_mode=ParseMode.HTML)
        def fetch_yt():
            with yt_dlp.YoutubeDL({'quiet': True, 'extract_flat': True}) as ydl:
                res = ydl.extract_info(f"ytsearch1:{query_text}", download=False)
                return res['entries'][0]['id'], res['entries'][0]['title']
        try:
            vid, title = await asyncio.to_thread(fetch_yt)
            await process_download(update, context, vid, title, msg, "original")
        except Exception as e:
            await msg.edit_text(f"❌ Failed to play: {e}", parse_mode=ParseMode.HTML)
        return

    if action == "search":
        # SAME LOGIC AS search_song - Mini App search asks bot to search
        query_text = data.get("query") or ""
        if not query_text:
            return await update.message.reply_text("❌ No search query.")
        # We reply with instruction to type in bot, Mini App also shows its own list
        # Bot will handle full download if user taps result in chat
        await update.message.reply_text(f"🔎 <i>Searching from Mini App:</i> <b>{query_text}</b>\n\nType the song name in chat to download, or tap Play in Mini App.", parse_mode=ParseMode.HTML)
        return

    if action == "lyrics":
        # SAME LOGIC AS YOUR get_lyrics_text
        query_text = data.get("query") or data.get("title") or ""
        lyrics = await asyncio.to_thread(get_lyrics_text, query_text)
        if len(lyrics) > 4000:
            lyrics = lyrics[:4000]
        await update.message.reply_text(f"🎼 <b>{query_text}</b>\n\n<blockquote>{lyrics}</blockquote>", parse_mode=ParseMode.HTML)
        return

    if action == "game":
        # Mini App game button -> bot starts YOUR game
        await update.message.reply_text("🎮 Opening game from Mini App...\nType /game to play!", parse_mode=ParseMode.HTML)
        return

    if action == "my_tracks":
        tracks = await asyncio.to_thread(get_miniapp_tracks, user_id)
        if not tracks:
            return await update.message.reply_text("🎵 No Mini App tracks yet. Create one in the Mini App!")
        text = "🎵 <b>Your Mini App Tracks</b>\n━━━━━━━━━━━━\n"
        for tid, t, a, g, ly, cov in tracks[:10]:
            text += f"{tid}. <b>{t}</b> - {a} [{g}]\n"
        await update.message.reply_text(text, parse_mode=ParseMode.HTML)
        return

    if action == "delete_track":
        track_id = data.get("id")
        if track_id:
            ok = await asyncio.to_thread(delete_miniapp_track, user_id, int(track_id))
            await update.message.reply_text("✅ Track deleted!" if ok else "❌ Track not found.", parse_mode=ParseMode.HTML)
        return

# ==========================================
# 9. CORE DOWNLOAD LOGIC - UNTOUCHED
# ==========================================
async def process_download(update: Update, context: ContextTypes.DEFAULT_TYPE, vid, title, msg_obj, effect):
    if effect == "original":
        cached = await asyncio.to_thread(get_global_cache, vid, "audio")
        if cached:
            file_id, cached_title = cached
            await msg_obj.edit_text("⚡ <b>Found in cache! Sending instantly...</b>", parse_mode=ParseMode.HTML)
            audio_kb = InlineKeyboardMarkup([[InlineKeyboardButton("❤ Save", callback_data="save_last"), InlineKeyboardButton("📝 Lyrics", callback_data="lyr_last")]])
            sent = await context.bot.send_audio(chat_id=msg_obj.chat_id, audio=file_id, title=cached_title, caption=f"✅ <b>{cached_title}</b>\n━━━━━━━━━━━━━━━━━━━━\n📥 <i>Downloaded via @{context.bot.username}</i>", reply_markup=audio_kb, parse_mode=ParseMode.HTML)
            context.user_data['last_downloaded'] = (vid, cached_title, sent.audio.file_id)
            await asyncio.to_thread(increment_downloads, update.effective_user.id)
            await msg_obj.delete()
            return
    if effect == "video":
        cached = await asyncio.to_thread(get_global_cache, vid, "video")
        if cached:
            file_id, cached_title = cached
            await msg_obj.edit_text("⚡ <b>Video found in cache! Sending instantly...</b>", parse_mode=ParseMode.HTML)
            sent = await context.bot.send_video(chat_id=msg_obj.chat_id, video=file_id, caption=f"🎬 <b>{cached_title}</b>\n━━━━━━━━━━━━━━━━━━━━\n📥 <i>Downloaded via @{context.bot.username}</i>", parse_mode=ParseMode.HTML)
            await asyncio.to_thread(increment_downloads, update.effective_user.id)
            await msg_obj.delete()
            return
    if effect == "video":
        await msg_obj.edit_text("⬇ <b>Downloading Video...</b>", parse_mode=ParseMode.HTML)
        temp_dir = tempfile.mkdtemp(prefix="vid_")
        safe_filename = f"{temp_dir}/{vid}.mp4"
        def download_video():
            ydl_opts = {'format': 'bestvideo[ext=mp4][height<=480]+bestaudio[ext=m4a]/best[ext=mp4][height<=480]/best', 'outtmpl': safe_filename, 'quiet': True, 'merge_output_format': 'mp4'}
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                info = ydl.extract_info(f"https://www.youtube.com/watch?v={vid}", download=True)
                thumb_url = info.get('thumbnail')
                return safe_filename, thumb_url
        try:
            file_path, thumb_url = await asyncio.wait_for(asyncio.to_thread(download_video), timeout=180.0)
            if os.path.getsize(file_path) > 50 * 1024 * 1024:
                shutil.rmtree(temp_dir, ignore_errors=True)
                await msg_obj.edit_text("❌ <b>Video is too large.</b> It exceeds Telegram's 50MB limit.", parse_mode=ParseMode.HTML)
                return
            await msg_obj.edit_text("⬆ <b>Uploading Video to Telegram...</b>", parse_mode=ParseMode.HTML)
            clean_title = await asyncio.to_thread(clean_title_with_ai, title)
            thumb_path = f"{temp_dir}/thumb.jpg"
            if thumb_url:
                try:
                    r = requests.get(thumb_url, timeout=10)
                    with open(thumb_path, 'wb') as f:
                        f.write(r.content)
                except:
                    thumb_path = None
            video_file = open(file_path, 'rb')
            thumb_file = open(thumb_path, 'rb') if thumb_path and os.path.exists(thumb_path) else None
            sent = await context.bot.send_video(chat_id=msg_obj.chat_id, video=video_file, caption=f"🎬 <b>{clean_title}</b>\n━━━━━━━━━━━━━━━━━━━━\n📥 <i>Downloaded via @{context.bot.username}</i>", thumbnail=thumb_file, parse_mode=ParseMode.HTML)
            await asyncio.to_thread(set_global_cache, vid, sent.video.file_id, clean_title, "video")
            video_file.close()
            if thumb_file:
                thumb_file.close()
            shutil.rmtree(temp_dir, ignore_errors=True)
            await msg_obj.delete()
            await asyncio.to_thread(increment_downloads, update.effective_user.id)
            return
        except Exception as e:
            logger.error(f"Video download error: {e}")
            shutil.rmtree(temp_dir, ignore_errors=True)
            await msg_obj.edit_text("❌ <b>Failed to download video.</b> YouTube blocked the server. Try another song.", parse_mode=ParseMode.HTML)
            return
    safe_filename = f"/tmp/{vid}.m4a"
    def download():
        ydl_opts = {'format': 'bestaudio[ext=m4a]/bestaudio/best', 'outtmpl': safe_filename, 'quiet': True}
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(f"https://www.youtube.com/watch?v={vid}", download=True)
            path = ydl.prepare_filename(info)
            thumb_url = info.get('thumbnail')
            return (path if os.path.exists(path) else safe_filename), thumb_url
    try:
        file_path, thumb_url = await asyncio.wait_for(asyncio.to_thread(download), timeout=60.0)
        if effect != "original":
            await msg_obj.edit_text(f"🎛 <b>Applying {effect.upper()} effect...</b>", parse_mode=ParseMode.HTML)
            file_path = await asyncio.to_thread(process_audio_effect, file_path, effect)
        await msg_obj.edit_text("⬆ <b>Uploading to Telegram...</b>", parse_mode=ParseMode.HTML)
        clean_title = await asyncio.to_thread(clean_title_with_ai, title)
        thumb_path = f"/tmp/{vid}.jpg"
        if thumb_url:
            try:
                r = requests.get(thumb_url, timeout=10)
                with open(thumb_path, 'wb') as f:
                    f.write(r.content)
            except:
                thumb_path = None
        audio_kb = InlineKeyboardMarkup([[InlineKeyboardButton("❤ Save", callback_data="save_last"), InlineKeyboardButton("📝 Lyrics", callback_data="lyr_last")]])
        caption = f"✅ <b>{clean_title}</b> ({effect})\n━━━━━━━━━━━━━━━━━━━━\n📥 <i>Downloaded via @{context.bot.username}</i>"
        audio_file = open(file_path, 'rb')
        thumb_file = open(thumb_path, 'rb') if thumb_path and os.path.exists(thumb_path) else None
        sent = await context.bot.send_audio(chat_id=msg_obj.chat_id, audio=audio_file, title=clean_title, caption=caption, thumbnail=thumb_file, reply_markup=audio_kb, parse_mode=ParseMode.HTML)
        if effect == "original":
            await asyncio.to_thread(set_global_cache, vid, sent.audio.file_id, clean_title, "audio")
        context.user_data['last_downloaded'] = (vid, clean_title, sent.audio.file_id)
        await asyncio.to_thread(increment_downloads, update.effective_user.id)
        audio_file.close()
        if thumb_file:
            thumb_file.close()
        if os.path.exists(file_path):
            os.remove(file_path)
        if thumb_path and os.path.exists(thumb_path):
            os.remove(thumb_path)
        await msg_obj.delete()
    except Exception as e:
        logger.error(f"Error: {e}")
        await msg_obj.edit_text("❌ <b>Failed.</b> Check logs.", parse_mode=ParseMode.HTML)

# ==========================================
# 10. OWNER-ONLY COMMANDS & LEGAL
# ==========================================
# *** WRITTEN TERMS DELETED HERE - ONLY THIS PART CHANGED ***
# /legal now works EXACTLY like the blue link in /start:
# It just sends the same blue link, tapping it opens your private channel.
# No written terms in chat anymore.
async def legal_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [[InlineKeyboardButton("⚖ Open Terms of Service & Legal", url=TERMS_URL)]]
    text = (
        "⚖ <b>Terms of Service & Legal</b>\n"
        "━━━━━━━━━━━━━━━━━━━━\n\n"
        # SAME BLUE LINK AS IN /start - DO NOT TOUCH
        "Tap to read our full Terms: <a href='https://t.me/+Zxd-CbtpEbNmNmQ1'>Terms of Service & Legal</a>"
    )
    await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.HTML)

async def dmca_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await legal_command(update, context)

async def admin_help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    await update.message.reply_text("👑 <b>OWNER COMMANDS</b>\n/stats\n/broadcast\n/userinfo", parse_mode=ParseMode.HTML)

async def stats_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    u, d, f = await asyncio.to_thread(get_bot_stats)
    await update.message.reply_text(f"📊 <b>STATS</b>\n👥 Users: {u}\n📥 Downloads: {d}\n❤ Favorites: {f}", parse_mode=ParseMode.HTML)

async def broadcast_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    if not context.args:
        return await update.message.reply_text("Usage: <code>/broadcast message</code>", parse_mode=ParseMode.HTML)
    message = update.message.text.replace("/broadcast ", "", 1)
    users = await asyncio.to_thread(get_all_users)
    status = await update.message.reply_text(f"⏳ Broadcasting to {len(users)}...")
    success = 0
    for uid in users:
        try:
            await context.bot.send_message(chat_id=uid, text=message, parse_mode=ParseMode.HTML)
            success += 1
            await asyncio.sleep(0.05)
        except:
            pass
    await status.edit_text(f"✅ Sent to {success} users.", parse_mode=ParseMode.HTML)

async def userinfo_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    if not context.args:
        return await update.message.reply_text("Usage: <code>/userinfo 123456789</code>", parse_mode=ParseMode.HTML)
    try:
        tid = int(context.args[0].strip())
    except:
        return
    d, score, streak, coins = await asyncio.to_thread(get_user_stats, tid)
    f = await asyncio.to_thread(get_favorites, tid)
    await update.message.reply_text(f"🕵 ID: {tid}\n💰 Coins: {coins}\n📥 Downloads: {d}\n🎮 Game Score: {score}\n🔥 Streak: {streak}\n❤ Favorites: {len(f)}", parse_mode=ParseMode.HTML)

# ==========================================
# 11. MAIN SETUP - UNTOUCHED EXCEPT 2 MINI APP HANDLERS ADDED
# ==========================================
def main():
    token = os.environ.get("BOT_TOKEN")
    if not token:
        return
    app = Application.builder().token(token).read_timeout(60).write_timeout(60).connect_timeout(60).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("app", app_command))
    app.add_handler(CommandHandler("miniapp", app_command))
    app.add_handler(CommandHandler("profile", profile_command))
    app.add_handler(CommandHandler("top", top_command))
    app.add_handler(CommandHandler("save", save_command))
    app.add_handler(CommandHandler("playlist", playlist_command))
    app.add_handler(CommandHandler("favorites", playlist_command))
    app.add_handler(CommandHandler("lyrics", lyrics_command))
    app.add_handler(CommandHandler("delete", delete_favorite_command))
    app.add_handler(CommandHandler("game", game_command))
    app.add_handler(CommandHandler("legal", legal_command))
    app.add_handler(CommandHandler("dmca", dmca_command))
    app.add_handler(CommandHandler("adminhelp", admin_help_command))
    app.add_handler(CommandHandler("stats", stats_command))
    app.add_handler(CommandHandler("broadcast", broadcast_command))
    app.add_handler(CommandHandler("userinfo", userinfo_command))
    app.add_handler(CallbackQueryHandler(handle_genre_select, pattern="^game_genre_"))
    app.add_handler(CallbackQueryHandler(handle_game_back, pattern="^game_back_genres"))
    app.add_handler(CallbackQueryHandler(handle_game_start, pattern="^game_start_"))
    app.add_handler(CallbackQueryHandler(handle_game_answer, pattern="^game_ans_"))
    app.add_handler(CallbackQueryHandler(handle_game_hint, pattern="^game_hint"))
    app.add_handler(CallbackQueryHandler(handle_game_download, pattern="^game_dl_full"))
    app.add_handler(CallbackQueryHandler(check_join_callback, pattern="^check_join$"))
    app.add_handler(CallbackQueryHandler(handle_song_selection, pattern="^song_"))
    app.add_handler(CallbackQueryHandler(handle_action, pattern="^act_"))
    app.add_handler(CallbackQueryHandler(handle_effect_menu, pattern="^menu_fx"))
    app.add_handler(CallbackQueryHandler(handle_back_to_song, pattern="^act_back_to_song"))
    app.add_handler(CallbackQueryHandler(handle_delete_button, pattern="^del_"))
    app.add_handler(CallbackQueryHandler(handle_lyrics_choice, pattern="^ly_"))
    app.add_handler(CallbackQueryHandler(handle_cancel, pattern="^cancel_search"))
    async def save_cb(u, c):
        await u.callback_query.answer("❤ Saved!")
        ls = c.user_data.get('last_downloaded')
        if ls:
            await asyncio.to_thread(add_favorite, u.effective_user.id, ls[0], ls[1], ls[2])
            await u.callback_query.message.reply_text(f"❤ {ls[1]} saved!")
    async def lyr_cb(u, c):
        await u.callback_query.answer("🎼 Fetching...")
        ls = c.user_data.get('last_downloaded')
        if ls:
            m = await u.callback_query.message.reply_text("🤖 <i>Finding lyrics...</i>", parse_mode=ParseMode.HTML)
            lyrics = await asyncio.to_thread(get_lyrics_text, ls[1])
            await m.edit_text(f"🎼 <b>{ls[1]}</b>\n\n<blockquote>{lyrics}</blockquote>", parse_mode=ParseMode.HTML)
    app.add_handler(CallbackQueryHandler(save_cb, pattern="^save_last"))
    app.add_handler(CallbackQueryHandler(lyr_cb, pattern="^lyr_last"))
    async def paginate(u, c):
        q = u.callback_query
        await q.answer("⚡ Loading...")
        page = int(q.data.split("_")[1])
        cache = user_search_cache.get(q.message.chat_id)
        if cache:
            cache['page'] = page
            await q.edit_message_text(f"✨ <b>Page {page+1}</b>\n👇 <b>Tap a song:</b>", reply_markup=build_search_keyboard(cache['results'], page), parse_mode=ParseMode.HTML)
    app.add_handler(CallbackQueryHandler(paginate, pattern="^page_"))
    # MINI APP ONLY: must be before generic TEXT handler
    app.add_handler(MessageHandler(filters.StatusUpdate.WEB_APP_DATA, handle_web_app_data))
    async def hint_cb(u, c):
        q = u.callback_query
        if q.data == "open_game_hint":
            await q.answer()
            await q.message.reply_text("Type /game to start!")
        else:
            await q.answer()
            await q.message.reply_text("Type /lyrics Song Name")
    app.add_handler(CallbackQueryHandler(hint_cb, pattern="^open_(game|lyrics)_hint$"))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, search_song))
    logger.info("LyricsOpehh Bot with Mini App + Legal blue link is running...")
    app.run_polling()

if __name__ == "__main__":
    main()
