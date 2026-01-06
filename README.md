# python-bot
from telegram import (
    Update,
    KeyboardButton,
    ReplyKeyboardMarkup,
    ReplyKeyboardRemove
)
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters
)
import nest_asyncio

# Admin ID
ADMIN_ID = 7117192445

# Viloyatlar va shaharlar
viloyatlar = {
    "Toshkent": ["Toshkent shahar", "Chirchiq", "Ohangaron"],
    "Andijon": ["Andijon shahar", "Asaka", "Xo‘jaobod"],
    "Farg‘ona": ["Farg‘ona shahar", "Quva", "Bag‘dod"],
    "Namangan": ["Namangan shahar", "Chust", "Kosonsoy"],
    "Samarqand": ["Samarqand shahar", "Urgut", "Kattaqo‘rg‘on"],
    "Buxoro": ["Buxoro shahar", "G‘ijduvon", "Romitan"],
    "Navoiy": ["Navoiy shahar", "Karmana", "Uchquduq"],
    "Jizzax": ["Jizzax shahar", "Zarbdor", "Forish"],
    "Sirdaryo": ["Guliston", "Boyovut", "Sardoba"],
    "Qashqadaryo": ["Qarshi", "Shahrisabz", "Chiroqchi"],
    "Surxondaryo": ["Termiz", "Boysun", "Angor"],
    "Xorazm": ["Urganch", "Xiva", "Shovot"],
    "Qoraqalpog‘iston": ["Nukus", "Xo‘jayli", "Chimboy"],
    "Toshkent viloyati": ["Olmaliq", "Angren", "Bekobod"]
}

# Bosqichlarni saqlash
user_state = {}
user_data = {}
orders_count = {}
users_count = set()

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    users_count.add(user_id)
    user_state[user_id] = "contact"
    text = "💧 Assalomu alaykum! [Brend nomi] suvlar botiga xush kelibsiz!\n\n" \
           "Bot orqali sifatli suvga buyurtma bera olasiz.\n\n" \
           "📞 Telefon raqamingizni yuborish uchun quyidagi tugmani bosing:"
    keyboard = [[KeyboardButton("📞 Kontaktni yuborish", request_contact=True)]]
    await update.message.reply_text(text, reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True))

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    state = user_state.get(user_id)

    # Kontaktni qabul qilish
    if update.message.contact and state == "contact":
        phone = update.message.contact.phone_number
        user_data[user_id] = {"phone": phone}
        user_state[user_id] = "viloyat"
        await context.bot.send_message(
            chat_id=ADMIN_ID,
            text=f"Yangi mijoz: {update.effective_user.first_name} | {phone}"
        )
        keyboard = [[v] for v in viloyatlar.keys()]
        await update.message.reply_text(
            "📍 Qaysi viloyatdasiz?",
            reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
        )

    # Viloyat tanlash
    elif state == "viloyat":
        vil = update.message.text
        if vil in viloyatlar:
            user_data[user_id]["viloyat"] = vil
            user_state[user_id] = "shahar"
            keyboard = [[sh] for sh in viloyatlar[vil]]
            await update.message.reply_text(
                f"📍 {vil} tanlandi. Shaharni tanlang:",
                reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
            )
        else:
            await update.message.reply_text("Iltimos, viloyatlardan birini tanlang.")

    # Shahar tanlash va lokatsiya so‘rash
    elif state == "shahar":
        shahar = update.message.text
        vil = user_data[user_id]["viloyat"]
        if shahar in viloyatlar[vil]:
            user_data[user_id]["shahar"] = shahar
            user_state[user_id] = "location"
            await update.message.reply_text(
                "📍 Lokatsiyangizni yuboring:",
                reply_markup=ReplyKeyboardMarkup(
                    [[KeyboardButton("📍 Lokatsiyani yuborish", request_location=True)]],
                    resize_keyboard=True
                )
            )
        else:
            await update.message.reply_text("Iltimos, shaharlardan birini tanlang.")

    # Lokatsiya qabul qilish
    elif update.message.location and state == "location":
        loc = update.message.location
        user_data[user_id]["location"] = (loc.latitude, loc.longitude)
        user_state[user_id] = "amount"
        await context.bot.send_message(
            chat_id=ADMIN_ID,
            text=f"Mijoz joylashuvi: {loc.latitude}, {loc.longitude}"
        )
        await update.message.reply_text(
            "💧 Qancha suv buyurtma bermoqchisiz?\n\n"
            "1 dona – 10 000 so‘m\n"
            "2 dona – 19 000 so‘m\n"
            "3 dona – 28 000 so‘m",
            reply_markup=ReplyKeyboardRemove()
        )

    # Buyurtma miqdorini qabul qilish
    elif state == "amount":
        amount_text = update.message.text.strip()
        prices = {"1": 10000, "2": 19000, "3": 28000}
        if amount_text in prices:
            amount = int(amount_text)
            user_data[user_id]["amount"] = amount
            user_data[user_id]["price"] = prices[amount_text]
            user_state[user_id] = "done"

            # Buyurtmani adminga jo‘natish
            orders_count[user_id] = orders_count.get(user_id, 0) + 1
            await context.bot.send_message(
                chat_id=ADMIN_ID,
                text=(
                    f"📦 Yangi buyurtma!\n"
                    f"Mijoz: {update.effective_user.first_name}\n"
                    f"Telefon: {user_data[user_id]['phone']}\n"
                    f"Viloyat: {user_data[user_id]['viloyat']}\n"
                    f"Shahar: {user_data[user_id]['shahar']}\n"
                    f"Miqdor: {amount} dona\n"
                    f"Narx: {user_data[user_id]['price']} so‘m\n"
                    f"Buyurtmalar soni: {orders_count[user_id]}\n"
                    f"Botdan foydalanuvchilar soni: {len(users_count)}"
                )
            )

            await update.message.reply_text(
                f"✅ Buyurtmangiz qabul qilindi!\n"
                f"📍 {user_data[user_id]['viloyat']}, {user_data[user_id]['shahar']}\n"
                f"Miqdor: {amount} dona\n"
                f"Narx: {user_data[user_id]['price']} so‘m"
            )
        else:
            await update.message.reply_text("Iltimos, 1, 2 yoki 3 raqamini tanlang.")

async def main():
    nest_asyncio.apply()  # Async loop xatolarini oldini olish
    app = ApplicationBuilder().token("BOT_TOKEN_HERE").build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.ALL, handle_message))
    await app.run_polling()

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
