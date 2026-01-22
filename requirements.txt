from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, MessageHandler, ContextTypes, filters

TOKEN = "AAH95us4MSYe_ZseGmu9kso7UlA7SO3lFDE
"
ADMIN_ID = 123456789
LTC_ADDRESS = "ltc1xxxxxxxxxxxxxxxx"

PRODUCT = {
    "name": "პროდუქტი X",
    "price_ltc": 0.015
}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 მოგესალმები!\n/shop — პროდუქტი")

async def shop(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [[InlineKeyboardButton(f"{PRODUCT['name']} – {PRODUCT['price_ltc']} LTC", callback_data="buy")]]
    await update.message.reply_text("🛒 პროდუქტი:", reply_markup=InlineKeyboardMarkup(keyboard))

async def buy(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    context.user_data["ordered"] = True
    await q.message.reply_text(
        f"💳 გადაიხადე {PRODUCT['price_ltc']} LTC\nმისამართი:\n{LTC_ADDRESS}\n\nგადახდის შემდეგ გამომიგზავნე TX Hash"
    )

async def receive_tx(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.user_data.get("ordered"):
        return
    context.user_data["tx"] = update.message.text
    await update.message.reply_text("📦 ახლა გამომიგზავნე მიწოდების მისამართი")

async def receive_address(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if "tx" not in context.user_data:
        return

    uid = update.message.chat_id
    tx = context.user_data["tx"]
    addr = update.message.text

    kb = InlineKeyboardMarkup([[InlineKeyboardButton("✅ Confirm Order", callback_data=f"confirm_{uid}")]])

    await context.bot.send_message(
        chat_id=ADMIN_ID,
        text=f"📦 ახალი შეკვეთა\nTX: {tx}\nმისამართი:\n{addr}",
        reply_markup=kb
    )

    await update.message.reply_text("🟡 შეკვეთა მიღებულია, ვამოწმებთ")

async def confirm(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    if q.from_user.id != ADMIN_ID:
        return
    uid = int(q.data.split("_")[1])
    await context.bot.send_message(uid, "🟢 გადახდა დადასტურებულია")
    await q.edit_message_text("✅ შეკვეთა დადასტურებულია")

app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("shop", shop))
app.add_handler(CallbackQueryHandler(buy, pattern="buy"))
app.add_handler(CallbackQueryHandler(confirm, pattern="confirm_"))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, receive_tx))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, receive_address))
app.run_polling()
