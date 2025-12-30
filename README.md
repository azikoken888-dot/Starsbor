import telebot
import random
import json
import os
from telebot import types

# ====== SOZLAMALAR ======
TOKEN = "BOT_TOKEN_BU_YERGA"  # Bot tokeningni yozing
ADMIN_ID = 123456789           # Admin Telegram ID
DB_FILE = "db.json"

bot = telebot.TeleBot(TOKEN)

# ====== DATABASE ======
def load_db():
    if os.path.exists(DB_FILE):
        with open(DB_FILE, "r") as f:
            return json.load(f)
    return {"users": {}, "channels": []}

def save_db(db):
    with open(DB_FILE, "w") as f:
        json.dump(db, f, indent=4)

db = load_db()

def check_user(uid):
    uid = str(uid)
    if uid not in db["users"]:
        db["users"][uid] = {
            "stars": 0,
            "cards": 0,
            "referrals": 0,
            "banned": False
        }
        save_db(db)

# ====== MENYU ======
def main_menu():
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("🗄️ Hisobim", "🎁 Sovg‘a qutilari")
    kb.add("💳 Kartochka ishlash", "🏆 Reyting")
    return kb

# ====== MAJBURIY OBUNA TEKSHIRISH ======
def check_subscription(uid):
    for channel in db.get("channels", []):
        try:
            member = bot.get_chat_member(channel, uid)
            if member.status not in ["member", "administrator", "creator"]:
                return False, channel
        except:
            return False, channel
    return True, None

# ====== START ======
@bot.message_handler(commands=["start"])
def start(msg):
    uid = msg.from_user.id
    check_user(uid)

    # Referral
    args = msg.text.split()
    if len(args) > 1 and args[1].isdigit():
        ref = args[1]
        if ref != str(uid) and ref in db["users"] and "ref_by" not in db["users"][str(uid)]:
            db["users"][str(uid)]["ref_by"] = ref
            db["users"][ref]["referrals"] += 1
            db["users"][ref]["cards"] += 1
            save_db(db)
            bot.send_message(ref, "🎉 Sizga yangi taklif keldi!\n+1 💳 kartochka qo‘shildi 🤩")

    # Majburiy obuna tekshirish
    ok, channel = check_subscription(uid)
    if not ok and db["channels"]:
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("✅ Obuna bo‘ldim", callback_data="check_sub"))
        msg_text = "🚨 Iltimos ushbu kanallarga obuna bo‘ling 👇 va ✅ Obuna bo‘ldim tugmasini bosing.\n\n"
        for ch in db["channels"]:
            msg_text += f"{ch}\n"
        bot.send_message(uid, msg_text, reply_markup=markup)
        return

    bot.send_message(uid,
                     "🎉 Xush kelibsiz! Botdan foydalanish uchun menyudan tanlang:",
                     reply_markup=main_menu())

# ====== HISOBIM ======
@bot.message_handler(func=lambda m: m.text == "🗄️ Hisobim")
def hisobim(msg):
    uid = str(msg.from_user.id)
    u = db["users"][uid]
    text = f"🗄️ <b>Hisobingiz</b>\n\n🌟 Stars: {u['stars']}\n💳 Kartochka: {u['cards']}\n👥 Takliflar: {u['referrals']}"
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("💰 Stars yechish", callback_data="withdraw"))
    bot.send_message(uid, text, parse_mode="HTML", reply_markup=kb)

# ====== KARTOCHKA ISHLASH ======
@bot.message_handler(func=lambda m: m.text == "💳 Kartochka ishlash")
def card_work(msg):
    uid = str(msg.from_user.id)
    ref_link = f"https://t.me/{bot.get_me().username}?start={uid}"
    text = (
        "💳 <b>Kartochka ishlash</b>\n\n"
        "🔹 1 ta do‘st taklif qilsangiz — 1 💳 kartochka\n"
        "🔹 5 🌟 Stars = 1 💳 kartochka\n\n"
        f"🔗 Sizning taklif havolangiz:\n{ref_link}"
    )
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("💱 5🌟 → 1💳", callback_data="buy_card"))
    bot.send_message(msg.chat.id, text, parse_mode="HTML", reply_markup=kb)

# ====== SOVG‘A QUTILARI ======
boxes_info = [
    {"name": "Quti 1", "cost": 1, "stars_min": 1, "stars_max": 5},
    {"name": "Quti 2", "cost": 3, "stars_min": 5, "stars_max": 20},
    {"name": "Quti 3", "cost": 5, "stars_min": 15, "stars_max": 40},
    {"name": "Quti 4", "cost": 10, "stars_min": 30, "stars_max": 80},
    {"name": "Quti 5", "cost": 20, "stars_min": 70, "stars_max": 150},
]

@bot.message_handler(func=lambda m: m.text == "🎁 Sovg‘a qutilari")
def show_boxes(msg):
    kb = types.InlineKeyboardMarkup()
    for i, box in enumerate(boxes_info, 1):
        kb.add(types.InlineKeyboardButton(f"🎁 {box['name']} ({box['cost']}💳)", callback_data=f"box_{i}"))
    bot.send_message(msg.chat.id, "🎁 Qaysi sovg‘a qutisini ochamiz? 😍", reply_markup=kb)

# ====== REYTING ======
@bot.message_handler(func=lambda m: m.text == "🏆 Reyting")
def rating(msg):
    top = sorted(db["users"].items(), key=lambda x: x[1]["stars"], reverse=True)[:10]
    text = "🏆 <b>TOP 10 Reyting</b>\n\n"
    for i, (uid, u) in enumerate(top, 1):
        text += f"{i}. {uid} — {u['stars']} 🌟\n"
    bot.send_message(msg.chat.id, text, parse_mode="HTML")

# ====== CALLBACK ======
@bot.callback_query_handler(func=lambda c: True)
def callback(c):
    uid = str(c.from_user.id)

    # Obuna tekshirish
    if c.data == "check_sub":
        ok, channel = check_subscription(uid)
        if ok:
            bot.answer_callback_query(c.id, "✅ Obuna tekshirildi! Endi botdan foydalanishingiz mumkin.")
            bot.send_message(uid, "🎉 Endi botdan foydalanishingiz mumkin!", reply_markup=main_menu())
        else:
            bot.answer_callback_query(c.id, f"🚫 Siz hali {channel} kanaliga obuna bo‘lmadingiz!")

    # Kartochka sotib olish
    elif c.data == "buy_card":
        if db["users"][uid]["stars"] >= 5:
            db["users"][uid]["stars"] -= 5
            db["users"][uid]["cards"] += 1
            save_db(db)
            bot.answer_callback_query(c.id, "✅ 1 💳 kartochka qo‘shildi!")
        else:
            bot.answer_callback_query(c.id, "❌ Stars yetarli emas")

    # Sovg‘a qutilari
    elif c.data.startswith("box_"):
        idx = int(c.data.split("_")[1]) - 1
        box = boxes_info[idx]
        if db["users"][uid]["cards"] < box["cost"]:
            bot.answer_callback_query(c.id, "💳 Kartochka yetarli emas!")
            return
        db["users"][uid]["cards"] -= box["cost"]
        chance = random.randint(1, 100)
        if chance <= 70:
            stars = random.randint(box["stars_min"], box["stars_max"])
            db["users"][uid]["stars"] += stars
            result = f"🎉 Tabriklaymiz!\nSiz {stars} 🌟 Stars yutdingiz!"
        else:
            result = random.choice([
                "✨ Omad siz bilan!",
                "🔥 Katta yutuqlar oldinda!",
                "🌈 Bugun sizning kuningiz!"
            ])
        save_db(db)
        bot.send_message(uid, result)

# ====== STARS YECHISH ======
@bot.callback_query_handler(func=lambda c: c.data == "withdraw")
def withdraw_stars(c):
    uid = str(c.from_user.id)
    if db["users"][uid]["stars"] < 15:
        bot.answer_callback_query(c.id, "❌ Hisobingizda miqdor yetarli emas! (Min: 15)")
        return
    msg = bot.send_message(uid, "Qancha stars yechmoqchisiz? Miqdorni kiriting:")
    bot.register_next_step_handler(msg, process_withdraw)

def process_withdraw(msg):
    uid = str(msg.from_user.id)
    try:
        amt = int(msg.text)
        if amt >= 15 and db["users"][uid]["stars"] >= amt:
            db["users"][uid]["stars"] -= amt
            save_db(db)
            bot.send_message(uid, "✅ So‘rovingiz qabul qilindi")
            bot.send_message(ADMIN_ID, f"💰 Yechish so‘rovi!\nID: {uid}\nMiqdor: {amt} 🌟")
        else:
            bot.send_message(uid, "❌ Hisobingizda miqdor yetarli emas yoki xato miqdor")
    except:
        bot.send_message(uid, "❌ Faqat raqam kiriting!")

# ====== ADMIN PANEL (BUYRUQLAR) ======
@bot.message_handler(commands=["admin", "addchannel", "delchannel", "channels"])
def admin_panel(msg):
    if msg.from_user.id != ADMIN_ID:
        return
    cmd = msg.text.split()
    if msg.text.startswith("/addchannel"):
        if len(cmd) < 2: return
        ch = cmd[1]
        if "channels" not in db: db["channels"] = []
        if ch not in db["channels"]:
            db["channels"].append(ch)
            save_db(db)
            bot.send_message(ADMIN_ID, f"✅ Kanal qo‘shildi: {ch}")
    elif msg.text.startswith("/delchannel"):
        if len(cmd) < 2: return
        ch = cmd[1]
        if ch in db["channels"]:
            db["channels"].remove(ch)
            save_db(db)
            bot.send_message(ADMIN_ID, f"🗑 Kanal o‘chirildi: {ch}")
    elif msg.text.startswith("/channels"):
        if not db["channels"]:
            bot.send_message(ADMIN_ID, "📭 Majburiy kanal yo‘q")
        else:
            bot.send_message(ADMIN_ID, "📢 Majburiy kanallar:\n" + "\n".join(db["channels"]))

# ====== POLLING ======
bot.infinity_polling()
