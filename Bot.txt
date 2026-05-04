import logging
import json
import os
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, ConversationHandler, filters, ContextTypes
from telegram.constants import ParseMode

logging.basicConfig(format=”%(asctime)s - %(levelname)s - %(message)s”, level=logging.INFO)
logger = logging.getLogger(**name**)

BOT_TOKEN = os.environ.get(“BOT_TOKEN”)
ADMIN_CHAT_ID = int(os.environ.get(“ADMIN_CHAT_ID”)
KASPI_QR_URL = “https://pay.kaspi.kz/pay/rrpncedw”
DB_FILE = “applications.json”

PLANS = {
“single”: {
“name”: “Razovaya poyezdka”,
“emoji”: “🚗”,
“price”: “3 000 T”,
“price_int”: 3000,
“desc”: “do 8 km”,
“features”: [“Detskoe kreslo”, “Professional voditel”, “Podacha 7-10 min”, “Chisty salon”]
},
“priority”: {
“name”: “Mama Priority”,
“emoji”: “⭐”,
“price”: “45 000 T”,
“price_int”: 45000,
“desc”: “primerno 16 poyezdok v mesyac”,
“features”: [“Vse iz Razovoy”, “Prioritetnaya podacha”, “Skidka 15%”, “Besplatnaya otmena”]
},
“family”: {
“name”: “Semeynyy”,
“emoji”: “👨‍👩‍👧”,
“price”: “60 000 T”,
“price_int”: 60000,
“desc”: “do 24 poyezdok v mesyac”,
“features”: [“Vse iz Mama Priority”, “2+ detskih kresla”, “Miniven po zaprosu”, “Skidka 25%”, “Personal menedzher”]
}
}

ASK_NAME, ASK_PHONE, ASK_PLAN, CONFIRM_ORDER, PAYMENT_CONFIRM = range(5)

def load_db():
if os.path.exists(DB_FILE):
with open(DB_FILE, “r”, encoding=“utf-8”) as f:
return json.load(f)
return {“applications”: [], “counter”: 0}

def save_db(data):
with open(DB_FILE, “w”, encoding=“utf-8”) as f:
json.dump(data, f, ensure_ascii=False, indent=2)

def add_application(app):
db = load_db()
db[“counter”] += 1
app[“id”] = db[“counter”]
app[“created_at”] = datetime.now().strftime(”%d.%m.%Y %H:%M”)
app[“status”] = “new”
db[“applications”].append(app)
save_db(db)
return app[“id”]

def get_applications(status=None):
db = load_db()
apps = db[“applications”]
if status:
apps = [a for a in apps if a.get(“status”) == status]
return apps

def update_application_status(app_id, status):
db = load_db()
for app in db[“applications”]:
if app[“id”] == app_id:
app[“status”] = status
break
save_db(db)

def delete_application(app_id):
db = load_db()
before = len(db[“applications”])
db[“applications”] = [a for a in db[“applications”] if a[“id”] != app_id]
save_db(db)
return len(db[“applications”]) < before

def is_admin(user_id):
return user_id == ADMIN_CHAT_ID

def main_keyboard():
return ReplyKeyboardMarkup([
[KeyboardButton(“Ostavit zayavku”)],
[KeyboardButton(“Tarify”), KeyboardButton(“O servise”)]
], resize_keyboard=True)

def admin_keyboard():
return ReplyKeyboardMarkup([
[KeyboardButton(“Vse zayavki”), KeyboardButton(“Novye zayavki”)],
[KeyboardButton(“Oplachennye”), KeyboardButton(“Statistika”)]
], resize_keyboard=True)

def plan_keyboard():
buttons = []
for key, plan in PLANS.items():
buttons.append([InlineKeyboardButton(
f”{plan[‘emoji’]} {plan[‘name’]} — {plan[‘price’]}”, callback_data=f”plan_{key}”
)])
return InlineKeyboardMarkup(buttons)

def phone_keyboard():
return ReplyKeyboardMarkup([
[KeyboardButton(“Otpravit nomer telefona”, request_contact=True)],
[KeyboardButton(“Otmena”)]
], resize_keyboard=True, one_time_keyboard=True)

def payment_keyboard():
return InlineKeyboardMarkup([
[InlineKeyboardButton(“Oplatit cherez Kaspi QR”, url=KASPI_QR_URL)],
[InlineKeyboardButton(“Ya oplatil(a)”, callback_data=“paid”)],
[InlineKeyboardButton(“Otmenit zayavku”, callback_data=“cancel_order”)]
])

def app_actions_keyboard(app_id, status):
buttons = []
if status != “paid”:
buttons.append([InlineKeyboardButton(“Otmetit oplachennym”, callback_data=f”mark_paid_{app_id}”)])
buttons.append([InlineKeyboardButton(“Otmenit”, callback_data=f”cancel_app_{app_id}”)])
buttons.append([InlineKeyboardButton(“Udalit zayavku”, callback_data=f”delete_app_{app_id}”)])
return InlineKeyboardMarkup(buttons)

def format_app(app):
plan = PLANS.get(app.get(“plan”, “”), {})
