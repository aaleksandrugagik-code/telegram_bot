import telebot
from telebot import types

BOT_TOKEN = "8224242377:AAFWlxblSYycLC1K9fXGXKOQkjWU5w9QKec"
bot = telebot.TeleBot(BOT_TOKEN)

tasks = {}
notes = {}
expenses = {}
shopping = {}

# ---------- Главное меню ----------
@bot.message_handler(commands=['start'])
def start(message):
    chat_id = message.chat.id
    markup = types.ReplyKeyboardMarkup(row_width=2, resize_keyboard=True)
    markup.add("📝 Заметки", "✅ Задачи", "💰 Расходы", "🛒 Покупки")
    bot.send_message(chat_id, "Привет! Выбери раздел:", reply_markup=markup)


# ---------- Выбор раздела ----------
@bot.message_handler(func=lambda m: m.text in ["📝 Заметки", "✅ Задачи", "💰 Расходы", "🛒 Покупки"] )
def section_menu(message):
    section = message.text
    chat_id = message.chat.id

    markup = types.ReplyKeyboardMarkup(row_width=2, resize_keyboard=True)
    markup.add(f"➕ Добавить {section}", f"📜 Показать {section}", f"❌ Удалить {section}")
    markup.add("⬅ Назад")
    bot.send_message(chat_id, f"Вы выбрали раздел: {section}", reply_markup=markup)


# ---------- Возврат назад ----------
@bot.message_handler(func=lambda m: m.text == "⬅ Назад")
def go_back(message):
    start(message)


# ---------- Добавление ----------
@bot.message_handler(func=lambda m: m.text.startswith("➕ Добавить"))
def add_item(message):
    section = message.text.replace("➕ Добавить ", "")
    bot.send_message(message.chat.id, f"Напиши, что добавить в {section.lower()}:")
    bot.register_next_step_handler(message, lambda msg: save_item(msg, section))


def save_item(message, section):
    chat_id = message.chat.id
    text = message.text

    if section.endswith("Заметки"):
        notes.setdefault(chat_id, []).append(text)
    elif section.endswith("Задачи"):
        tasks.setdefault(chat_id, []).append(text)
    elif section.endswith("Покупки"):
        shopping.setdefault(chat_id, []).append(text)
    elif section.endswith("Расходы"):
        try:
            parts = text.split(maxsplit=1)
            amount = float(parts[0])
            desc = parts[1] if len(parts) > 1 else "Без описания"
            expenses.setdefault(chat_id, []).append((amount, desc))
        except:
            bot.send_message(chat_id, "Ошибка! Формат: сумма описание")
            return

    bot.send_message(chat_id, "Добавлено ✅")


# ---------- Показ списка ----------
@bot.message_handler(func=lambda m: m.text.startswith("📜 Показать"))
def show_list(message):
    section = message.text.replace("📜 Показать ", "")
    chat_id = message.chat.id

    if section.endswith("Заметки"):
        items = notes.get(chat_id, [])
    elif section.endswith("Задачи"):
        items = tasks.get(chat_id, [])
    elif section.endswith("Покупки"):
        items = shopping.get(chat_id, [])
    elif section.endswith("Расходы"):
        items = expenses.get(chat_id, [])

    if not items:
        bot.send_message(chat_id, f"{section} пока нет.")
        return

    if section.endswith("Расходы"):
        total = sum([x[0] for x in items])
        text = "\n".join([f"{i+1}. {amt} — {desc}" for i, (amt, desc) in enumerate(items)])
        text += f"\n\n💵 Всего потрачено: {total}"
    else:
        text = "\n".join([f"{i+1}. {x}" for i, x in enumerate(items)])

    bot.send_message(chat_id, f"{section}:\n{text}")


# ---------- Удаление ----------
@bot.message_handler(func=lambda m: m.text.startswith("❌ Удалить"))
def delete_item(message):
    section = message.text.replace("❌ Удалить ", "")
    chat_id = message.chat.id

    if section.endswith("Заметки"):
        items = notes.get(chat_id, [])
    elif section.endswith("Задачи"):
        items = tasks.get(chat_id, [])
    elif section.endswith("Покупки"):
        items = shopping.get(chat_id, [])
    elif section.endswith("Расходы"):
        items = expenses.get(chat_id, [])

    if not items:
        bot.send_message(chat_id, f"{section} пока нет.")
        return

    markup = types.InlineKeyboardMarkup()
    for i, item in enumerate(items):
        if section.endswith("Расходы"):
            markup.add(types.InlineKeyboardButton(f"{i+1}. {item[0]} — {item[1]}", callback_data=f"del_{section}_{i}"))
        else:
            markup.add(types.InlineKeyboardButton(f"{i+1}. {item}", callback_data=f"del_{section}_{i}"))

    bot.send_message(chat_id, "Выбери, что удалить:", reply_markup=markup)


# ---------- Обработка нажатия на Inline ----------
@bot.callback_query_handler(func=lambda call: call.data.startswith("del_"))
def handle_delete(call):
    _, section, index = call.data.split("_")
    chat_id = call.message.chat.id
    index = int(index)

    if section.endswith("Заметки"):
        del notes[chat_id][index]
    elif section.endswith("Задачи"):
        del tasks[chat_id][index]
    elif section.endswith("Покупки"):
        del shopping[chat_id][index]
    elif section.endswith("Расходы"):
        del expenses[chat_id][index]

    bot.answer_callback_query(call.id, "Удалено ✅")
    bot.edit_message_text("Элемент удалён ✅", chat_id, call.message.message_id)


bot.polling(none_stop=True)


# === Запуск бота ===
print("✅ Бот запущен и ждёт сообщений...")
bot.polling(none_stop=True)

