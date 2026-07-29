import os
import threading
import logging
import pandas as pd
from flask import Flask
from telegram import Update, ReplyKeyboardMarkup, ReplyKeyboardRemove
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler, 
    ContextTypes, ConversationHandler, filters
)

# --- ផ្នែក Web Server សម្រាប់ Render Free Plan ---
web_app = Flask('')

@web_app.route('/')
def home():
    return "Intern Survey Bot is Running!"

def keep_alive():
    port = int(os.environ.get("PORT", 8080))
    web_app.run(host='0.0.0.0', port=port)

threading.Thread(target=keep_alive).start()
# --------------------------------------------------------

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

HEAD_NAME, DEPT, RATING, LIKE_DISLIKE, FUTURE_WISH = range(5)
EXCEL_FILE = "intern_feedback.xlsx"

# ⚠️ ជំនួស Telegram ID របស់ Admin នៅទីនេះ (មើលពី @userinfobot)
ADMIN_USER_ID = 123456789 

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_first_name = update.message.from_user.first_name
    
    await update.message.reply_text(
        f"ជម្រាបសួរ {user_first_name}! 🌸\n\n"
        f"សូមអរគុណសម្រាប់ការចំណាយពេល និងការខិតខំបំពេញការងារហាត់ការនាពេលកន្លងមក។\n"
        f"ដើម្បីកែលម្អបរិយាកាសធ្វើការឱ្យកាន់តែល្អ សូមប្អូនជួយចែករំលែកអារម្មណ៍ និងមតិយោបល់ដោយស្មោះត្រង់។\n\n"
        f"១. សូមបញ្ចូល **ឈ្មោះពេញរបស់ប្អូន (បុគ្គលិកហាត់ការ)** ៖"
    )
    return HEAD_NAME

async def get_head_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['intern_name'] = update.message.text
    await update.message.reply_text(
        "២. សូមបញ្ចូល **ឈ្មោះនាយកដ្ឋាន** ដែលប្អូនបានហាត់ការ ៖"
    )
    return DEPT

async def get_dept(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['dept'] = update.message.text
    
    # បង្កើតប៊ូតុង Emoji ឱ្យគាត់ជ្រើសរើសកម្រិតចិត្ត/អារម្មណ៍
    reply_keyboard = [['😍 សប្បាយចិត្តខ្លាំង', '🙂 ល្អបង្គួរ'], ['😐 ធម្មតា', '😔 មានសម្ពាធ/ហត់នឿយ']]
    
    await update.message.reply_text(
        "៣. តើអារម្មណ៍ និងបទពិសោធន៍ទូទៅរបស់ប្អូនអំឡុងពេលហាត់ការនៅទីនេះយ៉ាងណាដែរ? (សូមជ្រើសរើសចម្លើយខាងក្រោម) ៖",
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True, resize_keyboard=True)
    )
    return RATING

async def get_rating(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['rating'] = update.message.text
    
    await update.message.reply_text(
        "៤. **តើប្អូនពេញចិត្តនឹងចំណុចណាជាងគេ?** ហើយតើមាន **បញ្ហា ឬសម្ពាធអ្វីខ្លះ** ដែលប្អូនជួបប្រទះក្នុងនាយកដ្ឋាន? (សូមរៀបរាប់ដោយសេរី) ៖",
        reply_markup=ReplyKeyboardRemove()
    )
    return LIKE_DISLIKE

async def get_like_dislike(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['like_dislike'] = update.message.text
    
    reply_keyboard = [['✅ ចង់បន្តធ្វើការ', '❌ មិនចង់ទេ', '🤔 មិនទាន់ប្រាកដ']]
    
    await update.message.reply_text(
        "៥. ប្រសិនបើក្រុមហ៊ុន/អង្គភាពមានឱកាស **តើប្អូនប្រាថ្នាចង់បន្តធ្វើការជាបុគ្គលិកពេញសិទ្ធិនៅទីនេះដែរឬទេ?** ៖",
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True, resize_keyboard=True)
    )
    return FUTURE_WISH

async def save_survey(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['future_wish'] = update.message.text
    user_id = update.message.from_user.id
    user_name = update.message.from_user.full_name

    new_data = {
        'Telegram User ID': [user_id],
        'ឈ្មោះគណនី Telegram': [user_name],
        'ឈ្មោះបុគ្គលិកហាត់ការ': [context.user_data['intern_name']],
        'នាយកដ្ឋាន': [context.user_data['dept']],
        'កម្រិតអារម្មណ៍/ចិត្ត': [context.user_data['rating']],
        'មតិយោបល់/សម្ពាធដែលជួប': [context.user_data['like_dislike']],
        'បំណងបន្តធ្វើការពេញសិទ្ធិ': [context.user_data['future_wish']]
    }
    
    df_new = pd.DataFrame(new_data)

    if os.path.exists(EXCEL_FILE):
        df_existing = pd.read_excel(EXCEL_FILE)
        df_combined = pd.concat([df_existing, df_new], ignore_index=True)
        df_combined.to_excel(EXCEL_FILE, index=False)
    else:
        df_new.to_excel(EXCEL_FILE, index=False)

    await update.message.reply_text(
        "🎉 **សូមអរគុណច្រើនសម្រាប់ការចែករំលែកមតិយោបល់!**\n"
        "ជូនពរប្អូនទទួលបានជោគជ័យ និងសំណាងល្អក្នុងបេសកកម្មការងារទៅថ្ងៃអនាគត! 💐",
        reply_markup=ReplyKeyboardRemove()
    )
    return ConversationHandler.END

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("បានបោះបង់ការស្ទង់មតិ។", reply_markup=ReplyKeyboardRemove())
    return ConversationHandler.END

async def export_excel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    if user_id != ADMIN_USER_ID:
        await update.message.reply_text("❌ មានតែ Admin ប៉ុណ្ណោះដែលមានសិទ្ធិទាញយកទិន្នន័យ។")
        return

    if os.path.exists(EXCEL_FILE):
        await update.message.reply_document(
            document=open(EXCEL_FILE, 'rb'),
            filename=EXCEL_FILE,
            caption="📊 នេះជាលទ្ធផលស្ទង់មតិរបស់បុគ្គលិកហាត់ការ!"
        )
    else:
        await update.message.reply_text("មិនទាន់មានទិន្នន័យស្ទង់មតិនៅឡើយទេ។")

if __name__ == '__main__':
    # ⚠️ ជំនួស API TOKEN របស់ Bot ថ្មីនៅទីនេះ
    TOKEN = 'YOUR_NEW_BOT_TOKEN_HERE'

    app = ApplicationBuilder().token(TOKEN).build()

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start)],
        states={
            HEAD_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_head_name)],
            DEPT: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_dept)],
            RATING: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_rating)],
            LIKE_DISLIKE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_like_dislike)],
            FUTURE_WISH: [MessageHandler(filters.TEXT & ~filters.COMMAND, save_survey)],
        },
        fallbacks=[CommandHandler('cancel', cancel)],
        allow_reentry=True
    )

    app.add_handler(conv_handler)
    app.add_handler(CommandHandler('export', export_excel))

    app.run_polling()