import asyncio
import logging
import sqlite3
from aiogram import Bot, Dispatcher, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton
from apscheduler.schedulers.asyncio import AsyncIOScheduler
import aiohttp
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.contrib.fsm_storage.memory import MemoryStorage

API_TOKEN = '7801919256:AAEDCCG1NW8fSInpQyTu9tigzHKj9Tx0x-g'

logging.basicConfig(level=logging.INFO)

bot = Bot(token=API_TOKEN)
dp = Dispatcher(bot, storage=MemoryStorage())
scheduler = AsyncIOScheduler()

conn = sqlite3.connect('apps.db')
cursor = conn.cursor()
cursor.execute('''CREATE TABLE IF NOT EXISTS apps (
                    user_id INTEGER,
                    app_name TEXT,
                    app_url TEXT,
                    status TEXT
                )''')
conn.commit()

async def check_app_status(url):
    async with aiohttp.ClientSession() as session:
        try:
            async with session.get(url) as response:
                if response.status == 200:
                    text = await response.text()
                    if '<meta property="og:title"' in text:
                        return 'Active'
                return 'No Active'
        except:
            return 'No Active'

async def periodic_check():
    cursor.execute("SELECT rowid, user_id, app_name, app_url, status FROM apps ORDER BY app_name")
    apps = cursor.fetchall()
    for rowid, user_id, app_name, app_url, old_status in apps:
        new_status = await check_app_status(app_url)
        if new_status != old_status:
            cursor.execute("UPDATE apps SET status = ? WHERE rowid = ?", (new_status, rowid))
            conn.commit()
            if new_status == 'No Active':
                await bot.send_message(user_id, f"🔴 Приложение {app_name} стало недоступно.")
            else:
                await bot.send_message(user_id, f"✅ Приложение {app_name} стало доступно.")



def main_reply_keyboard():
    return ReplyKeyboardMarkup(resize_keyboard=True).add(
        KeyboardButton('Мои приложения'),
        KeyboardButton('Добавить приложение'),
        KeyboardButton('Удалить приложение'),
        KeyboardButton('Изменить название')
    )

class AddAppStates(StatesGroup):
    waiting_for_name = State()
    waiting_for_url = State()

@dp.message_handler(commands=['start'])
async def start_cmd(message: types.Message):
    await message.answer("Добро пожаловать!", reply_markup=main_reply_keyboard())

@dp.message_handler(lambda message: message.text == 'Мои приложения')
async def my_apps(message: types.Message):
    cursor.execute("SELECT app_name, app_url, status FROM apps WHERE user_id = ? ORDER BY app_name", (message.from_user.id,))
    apps = cursor.fetchall()
    updated_apps = []
    for app_name, app_url, old_status in apps:
        new_status = await check_app_status(app_url)
        if new_status != old_status:
            cursor.execute("UPDATE apps SET status = ? WHERE user_id = ? AND app_name = ?", (new_status, message.from_user.id, app_name))
            conn.commit()
        updated_apps.append((app_name, app_url, new_status))
    if not apps:
        await message.answer("У вас нет приложений.")
    else:
        text = "Ваши приложения:\n"

        for app_name, app_url, status in updated_apps:
            status_icon = '🟢' if status == 'Active' else '🔴'
            text += f"[{app_name}]({app_url}) - {status} {status_icon}\n"
        await message.answer(text, parse_mode='Markdown', disable_web_page_preview=True)

@dp.message_handler(lambda message: message.text == 'Добавить приложение')
async def add_app(message: types.Message):
    await message.answer("Введите название приложения:")
    await AddAppStates.waiting_for_name.set()

@dp.message_handler(state=AddAppStates.waiting_for_name)
async def process_app_name(message: types.Message, state):
    data = await state.get_data()
    if 'old_app_name' in data:
        old_name = data['old_app_name']
        new_name = message.text
        cursor.execute("UPDATE apps SET app_name = ? WHERE user_id = ? AND app_name = ?", (new_name, message.from_user.id, old_name))
        conn.commit()
        await message.answer(f"Название приложения изменено с {old_name} на {new_name}.", reply_markup=main_reply_keyboard())
        await state.finish()
        return
    await state.update_data(app_name=message.text)
    await message.answer("Пришлите ссылку на приложение:")
    await AddAppStates.waiting_for_url.set()

@dp.message_handler(state=AddAppStates.waiting_for_url)
async def process_app_url(message: types.Message, state):
    data = await state.get_data()
    app_name = data['app_name']
    app_url = message.text
    status = await check_app_status(app_url)
    cursor.execute("INSERT INTO apps (user_id, app_name, app_url, status) VALUES (?, ?, ?, ?)",
                   (message.from_user.id, app_name, app_url, status))
    conn.commit()
    await message.answer(f"Приложение {app_name} добавлено\nСтатус - {status} {'🟢' if status == 'Active' else '🔴'}", reply_markup=main_reply_keyboard())
    await state.finish()

@dp.message_handler(lambda message: message.text == 'Удалить приложение')
async def delete_app(message: types.Message):
    cursor.execute("SELECT app_name FROM apps WHERE user_id = ?", (message.from_user.id,))
    apps = cursor.fetchall()
    if not apps:
        await message.answer("У вас нет приложений для удаления.", reply_markup=main_reply_keyboard())
    else:
        keyboard = ReplyKeyboardMarkup(resize_keyboard=True)
        for (app_name,) in apps:
            keyboard.add(KeyboardButton(f'Удалить {app_name}'))
        keyboard.add(KeyboardButton('Отмена'))
        await message.answer("Выберите приложение для удаления:", reply_markup=keyboard)

@dp.message_handler(lambda message: message.text.startswith('Удалить '))
async def confirm_delete(message: types.Message):
    app_name = message.text[8:]
    cursor.execute("DELETE FROM apps WHERE user_id = ? AND app_name = ?", (message.from_user.id, app_name))
    conn.commit()
    await message.answer(f"Приложение {app_name} удалено.", reply_markup=main_reply_keyboard())

@dp.message_handler(lambda message: message.text == 'Изменить название')
async def rename_app_prompt(message: types.Message):
    cursor.execute("SELECT app_name FROM apps WHERE user_id = ? ORDER BY app_name", (message.from_user.id,))
    apps = cursor.fetchall()
    if not apps:
        await message.answer("У вас нет приложений для изменения.", reply_markup=main_reply_keyboard())
    else:
        keyboard = ReplyKeyboardMarkup(resize_keyboard=True)
        for (app_name,) in apps:
            keyboard.add(KeyboardButton(f'Изменить {app_name}'))
        keyboard.add(KeyboardButton('Отмена'))
        await message.answer("Какое приложение вы хотите изменить?", reply_markup=keyboard)

@dp.message_handler(lambda message: message.text.startswith('Изменить '))
async def rename_app_input(message: types.Message, state):
    app_name = message.text[9:]
    await state.update_data(old_app_name=app_name)
    await message.answer(f"Введите новое название для {app_name}:")
    await AddAppStates.waiting_for_name.set()

@dp.message_handler(lambda message: message.text == 'Отмена')
async def cancel_action(message: types.Message):
    await message.answer("Действие отменено.", reply_markup=main_reply_keyboard())

if __name__ == '__main__':
    async def main():
        scheduler.start()
        scheduler.add_job(periodic_check, 'interval', minutes=15)
        await dp.start_polling()

    asyncio.run(main())
