import os
import json
import sqlite3
# ---------- CREATE FOLDERS ----------
os.makedirs(FOLDER_NAME, exist_ok=True)
os.makedirs(os.path.join(FOLDER_NAME, "database"), exist_ok=True)
os.makedirs(os.path.join(FOLDER_NAME, "data"), exist_ok=True)

# ---------- INITIALIZE JSON FILES ----------
json_files = {
    "chat_id.json": {"chat_id": None},
    "data/casban_list.json": {},
    "data/protected_users.json": {},
    "data/shadowban_list.json": {}
}

for path, content in json_files.items():
    full_path = os.path.join(FOLDER_NAME, path)
    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    with open(full_path, "w") as f:
        json.dump(content, f)

# ---------- CREATE SQLITE DATABASE ----------
db_path = os.path.join(FOLDER_NAME, "database", "cirrus_bot.db")
conn = sqlite3.connect(db_path)
c = conn.cursor()

# Users table (history tracking)
c.execute("""CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER,
    chat_id INTEGER,
    username TEXT,
    first_name TEXT,
    last_name TEXT,
    status TEXT,
    timestamp TEXT
)""")

# Chat configuration table
c.execute("""CREATE TABLE IF NOT EXISTS chat_config (
    chat_id INTEGER PRIMARY KEY,
    private_mode INTEGER DEFAULT 0,
    silent_mode INTEGER DEFAULT 0
)""")

# Group history table
c.execute("""CREATE TABLE IF NOT EXISTS group_history (
    chat_id INTEGER,
    event TEXT,
    user_id INTEGER,
    timestamp TEXT
)""")

conn.commit()
conn.close()

# ---------- bot.py CONTENT ----------
bot_py = f"""import os
import json
import sqlite3
from telegram import Update, ChatMember
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes, ChatMemberHandler

BOT_TOKEN = "{BOT_TOKEN}"
OWNER_ID = {OWNER_ID}
CHAT_FILE = "chat_id.json"
DB_FILE = "database/cirrus_bot.db"
CASBAN_FILE = "data/casban_list.json"
PROTECT_FILE = "data/protected_users.json"
SHADOWBAN_FILE = "data/shadowban_list.json"

os.makedirs("database", exist_ok=True)
os.makedirs("data", exist_ok=True)

def load_json(path):
    if os.path.exists(path):
        with open(path, "r") as f:
            return json.load(f)
    return {{}}

def save_json(path, data):
    with open(path, "w") as f:
        json.dump(data, f)

def get_log_channel():
    return load_json(CHAT_FILE).get("chat_id")

def send_log(message):
    chat_id = get_log_channel()
    if chat_id:
        app.bot.send_message(chat_id=chat_id, text=message)

conn = sqlite3.connect(DB_FILE, check_same_thread=False)
c = conn.cursor()

async def track_member(update: Update, context: ContextTypes.DEFAULT_TYPE):
    member: ChatMember = update.chat_member.new_chat_member
    user = member.user
    status = member.status

    # Save user history
    c.execute("INSERT INTO users VALUES (?,?,?,?,?,?,datetime('now'))",
              (user.id, update.chat_member.chat.id, user.username, user.first_name, user.last_name, status))
    c.execute("INSERT INTO group_history VALUES (?,?,?,datetime('now'))",
              (update.chat_member.chat.id, status, user.id))
    conn.commit()

    # Enforce CASBAN
    casban_list = load_json(CASBAN_FILE)
    if str(user.id) in casban_list:
        try:
            # Remove admin if present
            await context.bot.promote_chat_member(
                chat_id=update.chat_member.chat.id,
                user_id=user.id,
                can_change_info=False,
                can_post_messages=False,
                can_edit_messages=False,
                can_delete_messages=False,
                can_invite_users=False,
                can_restrict_members=False,
                can_pin_messages=False,
                can_promote_members=False
            )
        except:
            pass
        await context.bot.ban_chat_member(update.chat_member.chat.id, user.id)
        send_log(f"CASBAN enforced on {{user.username or user.id}} in {{update.chat_member.chat.title}}")

    # Enforce SHADOWBAN
    shadow_list = load_json(SHADOWBAN_FILE)
    if str(user.id) in shadow_list:
        send_log(f"User {{user.username or user.id}} is shadowbanned in {{update.chat_member.chat.title}}")

    send_log(f"Tracked {{user.username or user.id}} → {{status}} in {{update.chat_member.chat.title}}")

# ---------- COMMANDS ----------
async def setlog(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    save_json(CHAT_FILE, {{ "chat_id": chat_id }})
    await update.message.reply_text(f"Log channel set: {{chat_id}}")

async def history(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target_user = update.message.reply_to_message.from_user.id if update.message.reply_to_message else \
                  int(context.args[0]) if context.args else update.effective_user.id
    c.execute("SELECT username,status,timestamp FROM users WHERE user_id=? AND chat_id=?",
              (target_user, update.effective_chat.id))
    rows = c.fetchall()
    msg = "\\n".join([f"{{r[0]}} → {{r[1]}} at {{r[2]}}" for r in rows]) or "No history found"
    await update.message.reply_text(msg)

async def allhistory(update: Update, context: ContextTypes.DEFAULT_TYPE):
    target_user = update.message.reply_to_message.from_user.id if update.message.reply_to_message else \
                  int(context.args[0]) if context.args else update.effective_user.id
    c.execute("SELECT chat_id,username,status,timestamp FROM users WHERE user_id=?", (target_user,))
    rows = c.fetchall()
    msg = "\\n".join([f"Chat {{r[0]}}: {{r[1]}} → {{r[2]}} at {{r[3]}}" for r in rows]) or "No history found"
    await update.message.reply_text(msg)

async def grouphistory(update: Update, context: ContextTypes.DEFAULT_TYPE):
    c.execute("SELECT user_id,event,timestamp FROM group_history WHERE chat_id=?", (update.effective_chat.id,))
    rows = c.fetchall()
    msg = "\\n".join([f"User {{r[0]}} → {{r[1]}} at {{r[2]}}" for r in rows]) or "No group history"
    await update.message.reply_text(msg)

# ---------- CASBAN / PROTECT / SHADOWBAN ----------
async def casban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != OWNER_ID or not context.args: return
    user_id = int(context.args[0])
    casban_list = load_json(CASBAN_FILE)
    casban_list[str(user_id)] = True
    save_json(CASBAN_FILE, casban_list)
    try:
        # Remove admin privileges
        await context.bot.promote_chat_member(
            chat_id=update.effective_chat.id,
            user_id=user_id,
            can_change_info=False,
            can_post_messages=False,
            can_edit_messages=False,
            can_delete_messages=False,
            can_invite_users=False,
            can_restrict_members=False,
            can_pin_messages=False,
            can_promote_members=False
        )
    except:
        pass
    await context.bot.ban_chat_member(update.effective_chat.id, user_id)
    send_log(f"CASBAN applied on {{user_id}}")
    await update.message.reply_text(f"User {{user_id}} CASBANed!")

# ---------- PROTECT / UNPROTECT ----------
async def protect(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != OWNER_ID or not context.args: return
    user_id = int(context.args[0])
    protect_list = load_json(PROTECT_FILE)
    protect_list[str(user_id)] = True
    save_json(PROTECT_FILE, protect_list)
    send_log(f"User {{user_id}} protected")
    await update.message.reply_text(f"User {{user_id}} protected")

async def unprotect(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != OWNER_ID or not context.args: return
    user_id = int(context.args[0])
    protect_list = load_json(PROTECT_FILE)
    protect_list.pop(str(user_id), None)
    save_json(PROTECT_FILE, protect_list)
    send_log(f"User {{user_id}} unprotected")
    await update.message.reply_text(f"User {{user_id}} unprotected")

# ---------- SHADOWBAN / UNSHADOWBAN ----------
async def shadowban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != OWNER_ID or not context.args: return
    user_id = int(context.args[0])
    shadow_list = load_json(SHADOWBAN_FILE)
    shadow_list[str(user_id)] = True
    save_json(SHADOWBAN_FILE
