import logging
from queue import Queue
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ContextTypes,
    filters,
    JobQueue,
    Job
)

DEVELOPER = "ကိုအောင်မင်းထက် (Telegram: @wayxian)"

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Queues for users waiting by gender
waiting_users = {
    'male': Queue(),
    'female': Queue()
}

# Active chat pairs: user_id -> partner_id
active_chats = {}

# To keep track of job references by user
user_jobs = {}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    keyboard = [
        [InlineKeyboardButton("👨 ယောက်ျား", callback_data='gender_male')],
        [InlineKeyboardButton("👩 မိန်းမ", callback_data='gender_female')]
    ]
    await update.message.reply_text(
        "နာမည်မဖော် စကားပြောဘော့မှ ကြိုဆိုပါသည်!\n"
        "ကျေးဇူးပြု၍ သင့်ကိုယ်ပိုင်လိင် ရွေးချယ်ပါ:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def gender_selection(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    user_gender = query.data.split('_')[1]  # male or female

    await query.edit_message_text(text=f"🔍 {user_gender} အဖြစ် စကားပြောဖော် ရှာနေသည်...")

    opposite_gender = 'female' if user_gender == 'male' else 'male'

    if not waiting_users[opposite_gender].empty():
        partner_id = waiting_users[opposite_gender].get()
        await connect_users(user_id, partner_id, context)
    else:
        waiting_users[user_gender].put(user_id)
        job = context.job_queue.run_repeating(
            check_match,
            interval=5,
            first=0,
            data={'user_id': user_id, 'gender': user_gender},
            name=str(user_id)
        )
        user_jobs[user_id] = job

async def check_match(context: ContextTypes.DEFAULT_TYPE):
    data = context.job.data
    user_id = data['user_id']
    user_gender = data['gender']
    opposite_gender = 'female' if user_gender == 'male' else 'male'

    if not waiting_users[opposite_gender].empty():
        partner_id = waiting_users[opposite_gender].get()
        if user_id in waiting_users[user_gender].queue:
            waiting_users[user_gender].queue.remove(user_id)
        await connect_users(user_id, partner_id, context)
        job: Job = context.job
        job.schedule_removal()
        user_jobs.pop(user_id, None)

async def connect_users(user1_id: int, user2_id: int, context: ContextTypes.DEFAULT_TYPE):
    active_chats[user1_id] = user2_id
    active_chats[user2_id] = user1_id

    await context.bot.send_message(
        user1_id,
        "✅ စကားပြောဖော်တွေ့ပြီ! စတင်စကားပြောပါ\n"
        "/stop - ရပ်ရန်\n/next - အသစ်ရှာရန်\n\n"
        f"ဖန်တီးသူ: {DEVELOPER}"
    )
    await context.bot.send_message(
        user2_id,
        "✅ စကားပြောဖော်တွေ့ပြီ! စတင်စကားပြောပါ\n"
        "/stop - ရပ်ရန်\n/next - အသစ်ရှာရန်"
    )

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    partner_id = active_chats.get(user_id)

    if partner_id:
        await context.bot.send_message(chat_id=partner_id, text=update.message.text)
    else:
        await update.message.reply_text("❌ စကားပြောနေခြင်းမရှိပါ /start ဖြင့်စတင်ပါ")

async def stop_chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    partner_id = active_chats.pop(user_id, None)

    if partner_id:
        active_chats.pop(partner_id, None)
        await context.bot.send_message(partner_id, "🚫 မိတ်ဖက်က စကားပြောခြင်း ရပ်လိုက်သည်။")
    if user_id in user_jobs:
        user_jobs[user_id].schedule_removal()
        user_jobs.pop(user_id, None)
    await update.message.reply_text("စကားပြောခြင်းရပ်ဆိုင်းပြီး /start ဖြင့် အသစ်ရှာပါ")

async def next_partner(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await stop_chat(update, context)
    await start(update, context)

async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE):
    logger.error(msg="Exception while handling an update:", exc_info=context.error)

def main():
    TOKEN = "7849496695:AAGtVAhzylsyqcJEDLR9LY0x1sAEnJQBr0c"
    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("stop", stop_chat))
    app.add_handler(CommandHandler("next", next_partner))
    app.add_handler(CallbackQueryHandler(gender_selection, pattern='^gender_'))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    app.add_error_handler(error_handler)

    logger.info(f"Bot started by {DEVELOPER}")
    app.run_polling()

if __name__ == '__main__':
    main()
