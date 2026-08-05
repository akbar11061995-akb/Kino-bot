import asyncio
import logging
import sqlite3
import sys
import os
from aiohttp import web

from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import Message

# Environment variables (Render'dagi maxfiy kalitlardan olinadi)
BOT_TOKEN = os.getenv("BOT_TOKEN", "YOUR_BOT_TOKEN_HERE")
ADMIN_ID = int(os.getenv("ADMIN_ID", "123456789"))

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

def init_db():
    conn = sqlite3.connect("movies.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS movies (
            code TEXT PRIMARY KEY,
            file_id TEXT,
            caption TEXT
        )
    """)
    conn.commit()
    conn.close()

def add_movie(code, file_id, caption):
    conn = sqlite3.connect("movies.db")
    cursor = conn.cursor()
    cursor.execute(
        "INSERT OR REPLACE INTO movies (code, file_id, caption) VALUES (?, ?, ?)",
        (code, file_id, caption)
    )
    conn.commit()
    conn.close()

def get_movie(code):
    conn = sqlite3.connect("movies.db")
    cursor = conn.cursor()
    cursor.execute("SELECT file_id, caption FROM movies WHERE code = ?", (code,))
    result = cursor.fetchone()
    conn.close()
    return result

class AdminStates(StatesGroup):
    waiting_for_video = State()
    waiting_for_code = State()

@dp.message(Command("start"))
async def start_handler(message: Message):
    await message.answer(f"Salom {message.from_user.first_name}!\n\nKino kodini yuboring:")

@dp.message(Command("add"))
async def start_add_movie(message: Message, state: FSMContext):
    if int(message.from_user.id) != ADMIN_ID:
        await message.answer("❌ Siz admin emassiz!")
        return
    await state.set_state(AdminStates.waiting_for_video)
    await message.answer("📹 Botingizga kino **videosini** yuboring:")

@dp.message(AdminStates.waiting_for_video, F.video)
async def process_video(message: Message, state: FSMContext):
    await state.update_data(
        file_id=message.video.file_id,
        caption=message.caption or ""
    )
    await state.set_state(AdminStates.waiting_for_code)
    await message.answer("🔢 Endi kino uchun **KOD** kiriting (masalan: 101):")

@dp.message(AdminStates.waiting_for_code, F.text)
async def process_code(message: Message, state: FSMContext):
    code = message.text.strip()
    data = await state.get_data()
    add_movie(code, data['file_id'], data['caption'])
    await state.clear()
    await message.answer(f"✅ Kino saqlandi! Kodi: {code}")

@dp.message(F.text)
async def search_movie(message: Message):
    code = message.text.strip()
    movie = get_movie(code)
    if movie:
        file_id, caption = movie
        text = caption if caption else f"🎬 Kino kodi: {code}"
        await message.answer_video(video=file_id, caption=text)
    else:
        await message.answer("❌ Bu kod bo'yicha kino topilmadi.")

# Render uchun oddiy Web Server (Ping qilib turish uchun)
async def handle_ping(request):
    return web.Response(text="Bot is running!")

async def main():
    init_db()
    logging.basicConfig(level=logging.INFO, stream=sys.stdout)

    # Web server tayyorlash
    app = web.Application()
    app.router.add_get("/", handle_ping)
    runner = web.AppRunner(app)
    await runner.setup()
    port = int(os.getenv("PORT", 8080))
    site = web.TCPSite(runner, "0.0.0.0", port)
    await site.start()

    # Bot Polling
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
