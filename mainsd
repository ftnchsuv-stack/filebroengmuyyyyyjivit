import csv
import secrets
from datetime import datetime, timezone
from pathlib import Path
from threading import Lock
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# =================== CONFIG ===================
TOKEN = "8532412255:AAErqUAlFsMansssdBxKo7jpiT42adw6J38"  # replace with your bot token
# ==============================================

csv_lock = Lock()
machine_status = {}
user_files = {}  # store random filenames for each user


# ------------------ File Helpers ------------------
def get_user_csv(user_id: int) -> Path:
    """Get or create a random file for this user"""
    if user_id not in user_files:
        rand_tag = secrets.token_hex(4)
        user_files[user_id] = f"report_{user_id}_{rand_tag}.csv"
    return Path(user_files[user_id])


def ensure_csv(user_id: int):
    """Ensure file exists with headers"""
    csv_path = get_user_csv(user_id)
    if not csv_path.exists():
        with csv_path.open("w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow(["date", "id_number", "amount", "category", "username"])


def clear_csv(user_id: int):
    """Delete or reset file"""
    csv_path = get_user_csv(user_id)
    if csv_path.exists():
        csv_path.unlink()
    ensure_csv(user_id)


# ------------------ Parse message ------------------
def parse_message(text: str):
    """Parse message like /ID 136947097 ..."""
    lines = [l.strip() for l in text.strip().splitlines() if l.strip()]
    lines = lines[:6]  # ✅ keep only first 6 lines

    parsed = {"id_number": "", "amount": "", "category": "", "username": ""}

    try:
        if lines and lines[0].startswith("/ID"):
            parsed["id_number"] = lines[0].replace("/ID", "").strip()
        if len(lines) > 1:
            parsed["amount"] = lines[1]
        if len(lines) > 2:
            parsed["category"] = lines[2]
        if len(lines) > 3:
            parsed["username"] = lines[-1]  # last line usually username
    except Exception:
        pass

    return parsed


# ------------------ Write to CSV ------------------
def append_row(user_id: int, row: list):
    csv_path = get_user_csv(user_id)
    with csv_lock:
        with csv_path.open("a", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow(row)


# ------------------ Reaction Helper ------------------
async def react_to_message(context: ContextTypes.DEFAULT_TYPE, chat_id: int, message_id: int, emoji: str = "👍"):
    """Add emoji reaction using Telegram Bot API"""
    try:
        # This uses Telegram raw API endpoint
        await context.bot._post(
            endpoint="sendReaction",
            data={"chat_id": chat_id, "message_id": message_id, "emoji": emoji},
        )
    except Exception:
        pass  # silently ignore errors


# ------------------ Commands ------------------
async def start_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    machine_status[chat_id] = True
    ensure_csv(update.effective_user.id)
    await update.message.reply_text("🟢 Machine started — ready to record /ID messages.")


async def stop_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    machine_status[chat_id] = False
    await update.message.reply_text("🔴 Machine stopped.")


async def clear_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    clear_csv(user_id)
    await update.message.reply_text("🧹 All saved data cleared. New file created.")


async def file_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    csv_path = get_user_csv(user_id)
    print("your code have been send file")
    if not csv_path.exists():
        await update.message.reply_text("please /start sin jam bot jab pderm save")
        return

    with open(csv_path, "rb") as f:
        await update.message.reply_document(document=f, filename=csv_path.name)


async def id_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    if not machine_status.get(chat_id, False):
        return  # ignore if machine is off

    user_id = update.effective_user.id
    text = update.message.text
    parsed = parse_message(text)

    if not parsed["id_number"]:
        return

    date_str = datetime.now(timezone.utc).strftime("%m/%d/%Y")

    row = [
        date_str,
        parsed["id_number"],
        parsed["amount"],
        parsed["category"],
        parsed["username"],
    ]
    append_row(user_id, row)

    # ✅ React with ❤️ instead of replying
    await react_to_message(context, chat_id, update.message.message_id, "👍")


# ------------------ Main ------------------
def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start_handler))
    app.add_handler(CommandHandler("stop", stop_handler))
    app.add_handler(CommandHandler("file", file_handler))
    app.add_handler(CommandHandler("clear", clear_handler))
    app.add_handler(CommandHandler("ID", id_handler))

    print("Bot started ✅ Press Ctrl+C to stop.")
    app.run_polling()


if __name__ == "__main__":
    main()
