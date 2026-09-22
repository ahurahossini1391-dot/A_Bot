import os
import random
import asyncio
from threading import Thread
from flask import Flask
from rubka.asynco import Robot, Message

# ================== تنظیمات ==================
BOT_TOKEN = os.environ.get("RUBIKA_TOKEN", "TOKEN_HERE")
bot = Robot(token=BOT_TOKEN)

# ================== داده‌ها ==================
games = {}                # امتیاز
pending_duels = {}        # {challenge_key: {...}} - درخواست‌های در انتظار
user_in_challenge = {}    # کاربر → key
ai_mode = {}              # کاربر → True (بازی با AI)

choice_map = {"rock": "🪨 سنگ", "paper": "📄 کاغذ", "scissors": "✂️ قیچی"}
beats = {"rock": "scissors", "paper": "rock", "scissors": "paper"}

rock_words = ["سنگ", "rock", "r", "1", "۱"]
paper_words = ["کاغذ", "paper", "p", "2", "۲"]
scissors_words = ["قیچی", "scissors", "s", "3", "۳"]


def get_score(chat_id, user_id):
    key = f"{chat_id}_{user_id}"
    if key not in games:
        games[key] = 0
    return games[key]


async def get_name(user_id):
    try:
        name = await bot.get_name(user_id)
        return name or f"کاربر {user_id}"
    except Exception:
        return f"کاربر {user_id}"


def judge(a, b):
    if a == b:
        return "draw"
    return "user" if beats[a] == b else "bot"


def normalize_choice(text):
    if text in rock_words:
        return "rock"
    if text in paper_words:
        return "paper"
    if text in scissors_words:
        return "scissors"
    return None


def get_leaderboard(chat_id):
    prefix = f"{chat_id}_"
    scores = []
    for key, score in games.items():
        if key.startswith(prefix):
            uid = key.replace(prefix, "")
            scores.append((uid, score))
    scores.sort(key=lambda x: x[1], reverse=True)
    return scores[:10]


def make_duel_key(chat_id, user1, user2):
    """ساخت کلید یکتا برای دوئل (ترتیب مهم نیست)"""
    ids = sorted([str(user1), str(user2)])
    return f"{chat_id}_{ids[0]}_{ids[1]}"


# ================== وب‌سرور برای Render ==================
app = Flask(__name__)

@app.route('/')
def home():
    return "Bot is alive!"

def run_flask():
    port = int(os.environ.get("PORT", 10000))
    app.run(host="0.0.0.0", port=port)


# ================== هندلر اصلی ==================
@bot.on_message()
async def handle_all(bot: Robot, message: Message):
    raw = (message.text or "").strip()
    text = raw.replace("/", "").lower()
    user_id = message.sender_id
    chat_id = message.chat_id

    # ---- راهنما ----
    if text in ["راهنما", "start", "شروع"]:
        await message.reply(
            "🎮 **راهنمای بازی سنگ، کاغذ، قیچی**\n"
            "➖➖➖➖➖\n"
            "🎯 **بازی با بات:**\n"
            "بنویس `بازی با بات`\n\n"
            "⚔️ **بازی با حریف:**\n"
            "بنویس `شروع بازی با @آیدی‌حریف`\n"
            "مثال: `شروع بازی با @ali`\n\n"
            "🏆 **امتیاز:** `امتیاز`\n"
            "📊 **لیدربورد:** `لیدربورد`\n\n"
            "✍️ **نحوه بازی:**\n"
            "توی بازی بنویس: سنگ / کاغذ / قیچی"
        )
        return

    # ---- بازی با بات ----
    if text in ["بازی با بات", "بازی با ربات", "play bot"]:
        ai_mode[user_id] = True
        name = await get_name(user_id)
        score = get_score(chat_id, user_id)
        await message.reply(
            f"🤖 **{name}** عزیز، با ربات بازی می‌کنی!\n"
            f"🏆 امتیاز فعلی: **{score}**\n"
            f"➖➖➖➖➖\n"
            f"✍️ بنویس: سنگ / کاغذ / قیچی"
        )
        return

    # ---- امتیاز ----
    if text in ["امتیاز", "score"]:
        s = get_score(chat_id, user_id)
        name = await get_name(user_id)
        await message.reply(f"🏆 **{name}**، امتیاز تو: **{s}**")
        return

    # ---- لیدربورد ----
    if text in ["لیدربورد", "leaderboard", "جدول"]:
        scores = get_leaderboard(chat_id)
        if not scores:
            await message.reply("📊 هنوز کسی امتیازی نگرفته!")
            return

        text_lb = "📊 **لیدربورد (۱۰ نفر برتر)**\n➖➖➖➖➖\n"
        for i, (uid, score) in enumerate(scores, 1):
            name = await get_name(uid)
            medal = "🥇" if i == 1 else "🥈" if i == 2 else "🥉" if i == 3 else f"{i}."
            text_lb += f"{medal} **{name}**: {score}\n"

        await message.reply(text_lb)
        return

    # ---- شروع بازی با حریف ----
    if text.startswith("شروع بازی با "):
        opponent_raw = raw[len("شروع بازی با "):].strip()
        if not opponent_raw:
            await message.reply("❌ درست بنویس: `شروع بازی با @ali`")
            return

        # آیدی حریف رو نرمال کن (بدون @)
        opponent_id = opponent_raw.lstrip("@").strip().lower()

        n1 = await get_name(user_id)
        my_id = str(user_id).lower()

        # چک: کاربر با خودش بازی نکنه
        if opponent_id == my_id:
            await message.reply("😅 نمی‌تونی با خودت بازی کنی!")
            return

        # چک: کاربر توی چالش دیگه‌ای نباشه
        if user_id in user_in_challenge:
            await message.reply("⏳ تو یه بازی فعال داری، صبر کن!")
            return

        # کلید یکتای این دوئل
        duel_key = make_duel_key(chat_id, my_id, opponent_id)

        # ---- حالت ۱: این درخواست جدیده (نفر اول داره دعوت می‌کنه) ----
        if duel_key not in pending_duels:
            # ذخیره درخواست
            pending_duels[duel_key] = {
                "challenger": user_id,
                "challenger_username": my_id,
                "opponent_username": opponent_id,
                "chat_id": chat_id,
                "state": "waiting_accept",
                "challenger_choice": None,
                "opponent": None,
            }
            user_in_challenge[user_id] = duel_key
            ai_mode.pop(user_id, None)

            await message.reply(
                f"⚔️ **{n1}** می‌خواد با **@{opponent_id}** بازی کنه!\n"
                f"➖➖➖➖➖\n"
                f"👉 **@{opponent_id}** اگه قبولی، دقیقاً این رو بفرست:\n"
                f"`شروع بازی با @{my_id}`\n"
                f"➖➖➖➖➖\n"
                f"⏳ منتظر جواب..."
            )
            return

        # ---- حالت ۲: درخواست قبلاً ثبت شده (نفر دوم داره قبول می‌کنه) ----
        duel = pending_duels[duel_key]

        # اگه نفر دوم همون کسیه که دعوت شده
        if duel["state"] == "waiting_accept" and opponent_id == str(duel["challenger_username"]).lower():
            # تأیید قبول
            duel["opponent"] = user_id
            duel["state"] = "waiting_challenger_choice"
            user_in_challenge[user_id] = duel_key

            n2 = await get_name(user_id)
            n1_name = await get_name(duel["challenger"])

            await message.reply(
                f"🎮 **بازی شروع شد!**\n"
                f"⚔️ {n1_name}  vs  {n2}\n"
                f"➖➖➖➖➖\n"
                f"✉️ **هر دو نفر، انتخابتون رو تو پیوی ربات بفرستید!**\n"
                f"👉 برید تو پیوی ربات و بنویسید: سنگ / کاغذ / قیچی\n"
                f"➖➖➖➖➖\n"
                f"**{n1_name}** اول انتخاب کن"
            )

            # به هر دو تو پیوی پیام بده
            try:
                await bot.send_message(
                    duel["challenger"],
                    f"🎯 **{n1_name}** نوبت تو هست!\n"
                    f"➖➖➖➖➖\n"
                    f"✍️ بنویس: سنگ / کاغذ / قیچی"
                )
            except Exception as e:
                print(f"خطا در پیام به چلنجر: {e}")

            try:
                await bot.send_message(
                    user_id,
                    f"🎯 **{n2}** نوبت تو هست!\n"
                    f"➖➖➖➖➖\n"
                    f"✍️ فعلاً صبر کن، نفر اول باید اول انتخاب کنه"
                )
            except Exception as e:
                print(f"خطا در پیام به حریف: {e}")
            return

        else:
            await message.reply("❌ این درخواست برای تو نیست یا منقضی شده!")
            return

    # ---- اگه کاربر توی دوئل هست ----
    if user_id in user_in_challenge:
        duel_key = user_in_challenge[user_id]
        if duel_key not in pending_duels:
            user_in_challenge.pop(user_id, None)
            return
        ch = pending_duels[duel_key]

        choice = normalize_choice(text)
        if not choice:
            return

        # ---- نفر اول انتخاب می‌کنه ----
        if ch["state"] == "waiting_challenger_choice" and user_id == ch["challenger"]:
            ch["challenger_choice"] = choice
            ch["state"] = "waiting_opponent_choice"

            await message.reply(
                f"✅ انتخابت ثبت شد: {choice_map[choice]}\n"
                f"⏳ منتظر حریف بمان..."
            )

            # به نفر دوم تو پیوی خبر بده
            try:
                await bot.send_message(
                    ch["opponent"],
                    f"🎯 نوبت تو هست!\n"
                    f"➖➖➖➖➖\n"
                    f"✍️ بنویس: سنگ / کاغذ / قیچی"
                )
            except:
                pass
            return

        # ---- نفر دوم انتخاب می‌کنه ----
        if ch["state"] == "waiting_opponent_choice" and user_id == ch["opponent"]:
            c1 = ch["challenger_choice"]
            result = judge(c1, choice)
            n1 = await get_name(ch["challenger"])
            n2 = await get_name(ch["opponent"])
            g_chat = ch["chat_id"]

            k1 = f"{g_chat}_{ch['challenger']}"
            k2 = f"{g_chat}_{ch['opponent']}"
            if k1 not in games: games[k1] = 0
            if k2 not in games: games[k2] = 0

            if result == "draw":
                res = "🤝 مساوی!"
            elif result == "user":
                games[k1] += 1
                games[k2] -= 1
                res = f"🎉 **{n1}** برنده شد!"
            else:
                games[k2] += 1
                games[k1] -= 1
                res = f"🎉 **{n2}** برنده شد!"

            await message.reply(
                f"✅ انتخابت ثبت شد: {choice_map[choice]}\n"
                f"⏳ نتیجه تو گروه اعلام میشه..."
            )

            # اعلام نتیجه تو گروه
            try:
                await bot.send_message(
                    g_chat,
                    f"🏁 **نتیجه دوئل**\n"
                    f"➖➖➖➖➖\n"
                    f"{n1}: {choice_map[c1]}\n"
                    f"{n2}: {choice_map[choice]}\n"
                    f"➖➖➖➖➖\n"
                    f"{res}\n"
                    f"🏆 امتیاز {n1}: **{games[k1]}**\n"
                    f"🏆 امتیاز {n2}: **{games[k2]}**\n"
                    f"➖➖➖➖➖\n"
                    f"برای بازی جدید: `شروع بازی با @آیدی`"
                )
            except Exception as e:
                print(f"خطا در ارسال نتیجه به گروه: {e}")

            # پاکسازی
            user_in_challenge.pop(ch["challenger"], None)
            user_in_challenge.pop(ch["opponent"], None)
            del pending_duels[duel_key]
            return

    # ---- بازی با AI ----
    choice = normalize_choice(text)
    if choice and ai_mode.get(user_id):
        bot_choice = random.choice(["rock", "paper", "scissors"])
        name = await get_name(user_id)
        result = judge(choice, bot_choice)
        key = f"{chat_id}_{user_id}"
        if key not in games:
            games[key] = 0

        if result == "user":
            games[key] += 1
            res = "🎉 بردی!"
        elif result == "bot":
            games[key] -= 1
            res = "😔 باختی!"
        else:
            res = "🤝 مساوی!"

        await message.reply(
            f"🤖 **{name}**\n"
            f"انتخاب تو: {choice_map[choice]}\n"
            f"انتخاب ربات: {choice_map[bot_choice]}\n"
            f"➖➖➖➖➖\n"
            f"{res}\n"
            f"🏆 امتیاز تو: **{games[key]}**\n"
            f"➖➖➖➖➖\n"
            f"دوباره انتخاب کن (سنگ/کاغذ/قیچی):"
        )
        return


# ================== اجرای همزمان ==================
def start_bot():
    asyncio.run(bot.run())


def main():
    print("🤖 ربات روشن شد...")
    bot_thread = Thread(target=start_bot, daemon=True)
    bot_thread.start()
    run_flask()


if __name__ == "__main__":
    main()
