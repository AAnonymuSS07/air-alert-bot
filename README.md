"""
🚨 Telegram бот для сповіщень про повітряну тривогу в Україні
Використовує API: alerts.in.ua

ВСТАНОВЛЕННЯ:
pip install python-telegram-bot requests

НАЛАШТУВАННЯ:
1. Отримай Telegram токен у @BotFather
2. Отримай API ключ на https://devs.alerts.in.ua
3. Встав токени нижче і запусти: python ukraine_alarm_bot.py
"""

import os
import logging
import asyncio
import requests
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    ContextTypes,
)

# ============================================================
# 🔧 НАЛАШТУВАННЯ — ВСТАВ СВОЇ ТОКЕНИ
# ============================================================
TELEGRAM_TOKEN = "ТВІЙ_TELEGRAM_BOT_TOKEN"   # від @BotFather
ALERTS_API_KEY = "ТВІЙ_ALERTS_API_KEY"        # від devs.alerts.in.ua
ALERTS_API_URL = "https://api.alerts.in.ua/v1"

# Інтервал перевірки тривог (секунди)
CHECK_INTERVAL = 30

# ============================================================
# СПИСОК ОБЛАСТЕЙ УКРАЇНИ
# ============================================================
REGIONS = {
    1:  "Вінницька область",
    2:  "Волинська область",
    3:  "Дніпропетровська область",
    4:  "Донецька область",
    5:  "Житомирська область",
    6:  "Закарпатська область",
    7:  "Запорізька область",
    8:  "Івано-Франківська область",
    9:  "Київська область",
    10: "Кіровоградська область",
    11: "Луганська область",
    12: "Львівська область",
    13: "Миколаївська область",
    14: "Одеська область",
    15: "Полтавська область",
    16: "Рівненська область",
    17: "Сумська область",
    18: "Тернопільська область",
    19: "Харківська область",
    20: "Херсонська область",
    21: "Хмельницька область",
    22: "Черкаська область",
    23: "Чернівецька область",
    24: "Чернігівська область",
    25: "м. Київ",
}

# ============================================================
# ЗБЕРІГАННЯ ПІДПИСНИКІВ (chat_id -> region_id)
# ============================================================
subscribers: dict[int, int] = {}
# Кеш останнього стану тривог {region_id: bool}
last_alert_state: dict[int, bool] = {}

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)


# ============================================================
# API ФУНКЦІЇ
# ============================================================
def get_all_alerts() -> dict:
    """Отримати статус тривог по всіх областях"""
    try:
        response = requests.get(
            f"{ALERTS_API_URL}/alerts/active.json",
            headers={"X-API-Key": ALERTS_API_KEY},
            timeout=10
        )
        response.raise_for_status()
        return response.json()
    except Exception as e:
        logger.error(f"Помилка API: {e}")
        return {}


def get_region_alert(region_id: int) -> bool | None:
    """Перевірити тривогу в конкретній області"""
    try:
        response = requests.get(
            f"{ALERTS_API_URL}/alerts/{region_id}.json",
            headers={"X-API-Key": ALERTS_API_KEY},
            timeout=10
        )
        response.raise_for_status()
        data = response.json()
        return data.get("alert", False)
    except Exception as e:
        logger.error(f"Помилка API для регіону {region_id}: {e}")
        return None


# ============================================================
# КОМАНДИ БОТА
# ============================================================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда /start — вітання"""
    await update.message.reply_text(
        "🇺🇦 *Бот сповіщень про повітряну тривогу*\n\n"
        "Я надсилатиму тобі сповіщення коли в твоєму регіоні оголошують або скасовують тривогу.\n\n"
        "📍 Використай /subscribe щоб обрати свій регіон\n"
        "📊 /status — поточний статус тривог по всій Україні\n"
        "❌ /unsubscribe — відписатися від сповіщень\n"
        "ℹ️ /help — довідка",
        parse_mode="Markdown"
    )


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда /help"""
    await update.message.reply_text(
        "📖 *Довідка*\n\n"
        "/start — запустити бота\n"
        "/subscribe — обрати регіон для сповіщень\n"
        "/unsubscribe — відписатися\n"
        "/status — статус тривог зараз\n"
        "/myregion — мій поточний регіон\n"
        "/help — ця довідка\n\n"
        "🔔 Бот автоматично сповіщає про початок і кінець тривоги у вибраному регіоні.",
        parse_mode="Markdown"
    )


async def subscribe(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда /subscribe — вибір регіону через кнопки"""
    keyboard = []
    row = []
    for region_id, region_name in REGIONS.items():
        row.append(InlineKeyboardButton(
            region_name,
            callback_data=f"region_{region_id}"
        ))
        if len(row) == 2:
            keyboard.append(row)
            row = []
    if row:
        keyboard.append(row)

    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(
        "📍 *Обери свій регіон:*",
        reply_markup=reply_markup,
        parse_mode="Markdown"
    )


async def region_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обробка вибору регіону"""
    query = update.callback_query
    await query.answer()

    region_id = int(query.data.replace("region_", ""))
    region_name = REGIONS.get(region_id, "Невідомий регіон")
    chat_id = query.message.chat_id

    subscribers[chat_id] = region_id

    # Перевірити поточний стан тривоги
    is_alert = get_region_alert(region_id)
    status = "🚨 *ТРИВОГА ЗАРАЗ АКТИВНА!*" if is_alert else "✅ Тривоги немає"

    await query.edit_message_text(
        f"✅ *Підписка оформлена!*\n\n"
        f"📍 Регіон: *{region_name}*\n"
        f"📊 Статус зараз: {status}\n\n"
        f"🔔 Я надішлю тобі сповіщення при зміні статусу тривоги.",
        parse_mode="Markdown"
    )


async def unsubscribe(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда /unsubscribe"""
    chat_id = update.message.chat_id
    if chat_id in subscribers:
        region_name = REGIONS.get(subscribers[chat_id], "")
        del subscribers[chat_id]
        await update.message.reply_text(
            f"❌ Ти відписався від сповіщень про тривогу в *{region_name}*.",
            parse_mode="Markdown"
        )
    else:
        await update.message.reply_text("Ти ще не підписаний на жоден регіон.")


async def my_region(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда /myregion"""
    chat_id = update.message.chat_id
    if chat_id in subscribers:
        region_id = subscribers[chat_id]
        region_name = REGIONS.get(region_id, "")
        is_alert = get_region_alert(region_id)
        status = "🚨 ТРИВОГА АКТИВНА!" if is_alert else "✅ Тривоги немає"
        await update.message.reply_text(
            f"📍 Твій регіон: *{region_name}*\n"
            f"📊 Статус: {status}",
            parse_mode="Markdown"
        )
    else:
        await update.message.reply_text(
            "Ти ще не підписаний. Використай /subscribe щоб обрати регіон."
        )


async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда /status — статус по всіх областях"""
    await update.message.reply_text("⏳ Отримую дані...")

    alert_regions = []
    clear_regions = []

    for region_id, region_name in REGIONS.items():
        is_alert = get_region_alert(region_id)
        if is_alert:
            alert_regions.append(f"🚨 {region_name}")
        else:
            clear_regions.append(f"✅ {region_name}")

    now = datetime.now().strftime("%H:%M:%S")
    text = f"📊 *Статус тривог в Україні* ({now})\n\n"

    if alert_regions:
        text += "🚨 *ТРИВОГА:*\n" + "\n".join(alert_regions) + "\n\n"
    if clear_regions:
        text += "✅ *Спокійно:*\n" + "\n".join(clear_regions)

    await update.message.reply_text(text, parse_mode="Markdown")


# ============================================================
# ФОНОВИЙ МОНІТОРИНГ ТРИВОГ
# ============================================================
async def check_alerts(context: ContextTypes.DEFAULT_TYPE):
    """Перевіряє тривоги і надсилає сповіщення підписникам"""
    global last_alert_state

    if not subscribers:
        return

    # Отримати унікальні регіони підписників
    regions_to_check = set(subscribers.values())

    for region_id in regions_to_check:
        is_alert = get_region_alert(region_id)
        if is_alert is None:
            continue

        prev_state = last_alert_state.get(region_id)

        # Якщо стан змінився
        if prev_state != is_alert:
            last_alert_state[region_id] = is_alert
            region_name = REGIONS.get(region_id, "")
            now = datetime.now().strftime("%H:%M")

            if is_alert:
                message = (
                    f"🚨 *УВАГА! ПОВІТРЯНА ТРИВОГА!*\n\n"
                    f"📍 Регіон: *{region_name}*\n"
                    f"🕐 Час: {now}\n\n"
                    f"⚠️ Прямуйте до укриття!"
                )
            else:
                message = (
                    f"✅ *Відбій тривоги*\n\n"
                    f"📍 Регіон: *{region_name}*\n"
                    f"🕐 Час: {now}\n\n"
                    f"Тривогу скасовано. Будьте обережні."
                )

            # Надіслати всім підписникам цього регіону
            for chat_id, sub_region_id in subscribers.items():
                if sub_region_id == region_id:
                    try:
                        await context.bot.send_message(
                            chat_id=chat_id,
                            text=message,
                            parse_mode="Markdown"
                        )
                    except Exception as e:
                        logger.error(f"Помилка надсилання до {chat_id}: {e}")


# ============================================================
# ЗАПУСК БОТА
# ============================================================
def main():
    print("🚀 Запуск бота сповіщень про тривоги...")

    app = Application.builder().token(TELEGRAM_TOKEN).build()

    # Команди
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("subscribe", subscribe))
    app.add_handler(CommandHandler("unsubscribe", unsubscribe))
    app.add_handler(CommandHandler("status", status))
    app.add_handler(CommandHandler("myregion", my_region))
    app.add_handler(CallbackQueryHandler(region_callback, pattern="^region_"))

    # Фоновий моніторинг кожні 30 секунд
    app.job_queue.run_repeating(check_alerts, interval=CHECK_INTERVAL, first=10)

    print("✅ Бот запущений! Натисни Ctrl+C щоб зупинити.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
