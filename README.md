import asyncio
import json
import sqlite3
import logging
import os
import re
from datetime import datetime
from functools import wraps
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.errors import FloodWaitError, SessionPasswordNeededError, PhoneNumberInvalidError
from aiogram import Bot, Dispatcher, types, F
from aiogram.types import Message, InlineKeyboardMarkup, InlineKeyboardButton, CallbackQuery
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
import httpx

# ===================== КОНФІГ =====================
BOT_TOKEN = "8311931201:AAF54yb1vbB35a-lQP-RYDfQGRhjcWY9zcU"
CRYPTO_TOKEN = "546545:AAg4ERVBQrmvzdkDMslmYFCLGBuebAaNjWh"
ADMIN_ID = 5810055220
DEFAULT_LANG = "ru"

STORAGE_FILE = "products_data.json"
PROMOCODES_FILE = "promocodes.json"
SESSIONS_FILE = "sessions.json"
APIS_FILE = "apis.json"
USER_PURCHASES_FILE = "user_purchases.json"

CRYPTOPAY_BASE = "https://pay.crypt.bot/api"
PRODUCT_ASSET = "USDT"
INVOICE_TTL = 3600

# ==================================================

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
log = logging.getLogger(__name__)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# Данные для авторега
user_sessions = {}
temp_product_data = {}
saved_sessions = {}
saved_apis = {}
user_purchases = {}

# -------------------- ФУНКЦІЇ ЗАВАНТАЖЕННЯ --------------------
def load_sessions():
    global saved_sessions
    if os.path.exists(SESSIONS_FILE):
        with open(SESSIONS_FILE, 'r') as f:
            saved_sessions = json.load(f)
    else:
        saved_sessions = {}

def save_sessions():
    with open(SESSIONS_FILE, 'w') as f:
        json.dump(saved_sessions, f, indent=2)

def load_apis():
    global saved_apis
    if os.path.exists(APIS_FILE):
        with open(APIS_FILE, 'r') as f:
            saved_apis = json.load(f)
    else:
        saved_apis = {}

def save_apis():
    with open(APIS_FILE, 'w') as f:
        json.dump(saved_apis, f, indent=2)

def load_user_purchases():
    global user_purchases
    if os.path.exists(USER_PURCHASES_FILE):
        with open(USER_PURCHASES_FILE, 'r') as f:
            user_purchases = json.load(f)
    else:
        user_purchases = {}

def save_user_purchases():
    with open(USER_PURCHASES_FILE, 'w') as f:
        json.dump(user_purchases, f, indent=2)

load_sessions()
load_apis()
load_user_purchases()

# -------------------- БД --------------------
conn = sqlite3.connect("bot.db", check_same_thread=False)
cur = conn.cursor()
cur.executescript("""
CREATE TABLE IF NOT EXISTS users (user_id INTEGER PRIMARY KEY, lang TEXT);
CREATE TABLE IF NOT EXISTS sales_history (id INTEGER PRIMARY KEY AUTOINCREMENT, product_id INTEGER, buyer_id INTEGER, amount TEXT, paid_at TEXT);
CREATE TABLE IF NOT EXISTS used_invoices (invoice_id TEXT PRIMARY KEY, product_id INTEGER, user_id INTEGER);
""")
conn.commit()

# -------------------- ФАЙЛИ ТОВАРІВ --------------------
def load_products():
    if os.path.exists(STORAGE_FILE):
        with open(STORAGE_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return []

def save_products(products):
    with open(STORAGE_FILE, 'w', encoding='utf-8') as f:
        json.dump(products, f, indent=2, ensure_ascii=False)

def load_promocodes():
    if os.path.exists(PROMOCODES_FILE):
        with open(PROMOCODES_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {}

def save_promocodes(promocodes):
    with open(PROMOCODES_FILE, 'w', encoding='utf-8') as f:
        json.dump(promocodes, f, indent=2, ensure_ascii=False)

products = load_products()
promocodes = load_promocodes()

# -------------------- МОВА --------------------
def get_lang(uid: int) -> str:
    cur.execute("SELECT lang FROM users WHERE user_id=?", (uid,))
    r = cur.fetchone()
    return r[0] if r else DEFAULT_LANG

def set_lang(uid: int, lang: str) -> None:
    cur.execute("INSERT OR REPLACE INTO users VALUES (?,?)", (uid, lang))
    conn.commit()

# -------------------- СТАНІ FSM --------------------
class AddProductState(StatesGroup):
    waiting_name = State()
    waiting_description = State()
    waiting_price = State()
    waiting_country = State()

class EditProductState(StatesGroup):
    waiting_field = State()
    waiting_value = State()

class PromoState(StatesGroup):
    waiting_create_code = State()
    waiting_create_product = State()
    waiting_use_code_for_product = State()
    waiting_delete_promo = State()

class RegState(StatesGroup):
    waiting_phone = State()
    waiting_code = State()
    waiting_password = State()
    waiting_api_id = State()
    waiting_api_hash = State()

class CryptoPayState(StatesGroup):
    waiting_payment = State()

# ==================== КРИПТОПЛАТЕЖІ ====================
async def create_invoice(amount: float, description: str) -> tuple[str, str]:
    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.post(
            f"{CRYPTOPAY_BASE}/createInvoice",
            headers={"Crypto-Pay-API-Token": CRYPTO_TOKEN},
            json={
                "asset": PRODUCT_ASSET,
                "amount": str(amount),
                "description": description[:128],
                "expires_in": INVOICE_TTL,
            },
        )
        result = resp.json()
        log.info("createInvoice response: %s", result)
        if not result.get("ok"):
            raise RuntimeError(f"CryptoPay error: {result}")
        inv = result["result"]
        return inv["bot_invoice_url"], str(inv["invoice_id"])

async def check_invoice(invoice_id: str) -> bool:
    try:
        async with httpx.AsyncClient(timeout=10) as client:
            resp = await client.get(
                f"{CRYPTOPAY_BASE}/getInvoices",
                headers={"Crypto-Pay-API-Token": CRYPTO_TOKEN},
                params={"invoice_ids": invoice_id},
            )
            result = resp.json()
            log.info("getInvoices response: %s", result)
            if not result.get("ok"):
                return False
            items = result.get("result", {}).get("items", [])
            if not items:
                return False
            return items[0].get("status") == "paid"
    except Exception as e:
        log.error("check_invoice exception: %s", e)
        return False

# -------------------- ТЕКСТИ (ПОВНІ ВЕРСІЇ) --------------------
T = {
    "start": {
        "ru": "Добро пожаловать! Выберите действие:",
        "ua": "Вітаємо! Оберіть дію:",
        "en": "Welcome! Choose an action:",
    },
    "catalog_btn": {"ru": "📦 Каталог", "ua": "📦 Каталог", "en": "📦 Catalog"},
    "my_purchases_btn": {"ru": "📱 Мои аккаунты", "ua": "📱 Мої акаунти", "en": "📱 My accounts"},
    "support_btn": {"ru": "🆘 Поддержка", "ua": "🆘 Підтримка", "en": "🆘 Support"},
    "policy_btn": {"ru": "📜 Политика", "ua": "📜 Політика", "en": "📜 Policy"},
    "lang_btn": {"ru": "🌐 Язык", "ua": "🌐 Мова", "en": "🌐 Language"},
    "buy_crypto_btn": {"ru": "💰 Купить за USDT", "ua": "💰 Купити за USDT", "en": "💰 Buy with USDT"},
    "use_promo_btn": {"ru": "🎫 Получить по промокоду", "ua": "🎫 Отримати за промокодом", "en": "🎫 Use promo code"},
    "get_codes_btn": {"ru": "📲 Получить коды входа", "ua": "📲 Отримати коди входу", "en": "📲 Get login codes"},
    "delete_account_btn": {"ru": "🗑️ Удалить аккаунт", "ua": "🗑️ Видалити акаунт", "en": "🗑️ Delete account"},
    "back_to_catalog": {"ru": "◀️ В каталог", "ua": "◀️ В каталог", "en": "◀️ Back to catalog"},
    "back_to_menu": {"ru": "◀️ В меню", "ua": "◀️ В меню", "en": "◀️ Back to menu"},
    "check_btn": {"ru": "✅ Проверить оплату", "ua": "✅ Перевірити оплату", "en": "✅ Check payment"},
    "pay_btn": {"ru": "💳 Оплатить", "ua": "💳 Оплатити", "en": "💳 Pay"},
    "paid": {
        "ru": "✅ ОПЛАЧЕНО!\n\nВаш аккаунт:\n",
        "ua": "✅ ОПЛАЧЕНО!\n\nВаш акаунт:\n",
        "en": "✅ PAID!\n\nYour account:\n",
    },
    "paid_already": {
        "ru": "Аккаунт уже был выдан:\n",
        "ua": "Акаунт вже було видано:\n",
        "en": "Account already issued:\n",
    },
    "not_paid": {
        "ru": "❌ Оплата не найдена. Пожалуйста, оплатите счёт и нажмите 'Проверить оплату' снова.",
        "ua": "❌ Оплату не знайдено. Будь ласка, сплатіть рахунок і натисніть 'Перевірити оплату' знову.",
        "en": "❌ Payment not found. Please pay the invoice and click 'Check payment' again.",
    },
    "lang_saved": {
        "ru": "Язык сохранён",
        "ua": "Мову збережено",
        "en": "Language saved",
    },
    "report": {
        "ru": "Обнаружили ошибку или есть вопрос?\nНапишите нам: @hoxxp",
        "ua": "Зустріли помилку або маєте питання?\nНапишіть нам: @hoxxp",
        "en": "Found a bug or have a question?\nContact us: @hoxxp",
    },
    "pay_text": {
        "ru": (
            "Оплата {amount} USDT\n\n"
            "Нажмите для оплаты:\n\n"
            "Совершая покупку, вы подтверждаете согласие с Политикой "
            "конфиденциальности и условиями использования.\n/policy"
        ),
        "ua": (
            "Оплата {amount} USDT\n\n"
            "Натисніть для оплати:\n\n"
            "Купуючи товар у нашому боті, ви підтверджуєте згоду з Політикою "
            "конфіденційності та умовами користування.\n/policy"
        ),
        "en": (
            "Payment {amount} USDT\n\n"
            "Click to pay:\n\n"
            "By purchasing, you agree to the Privacy Policy "
            "and Terms of Service.\n/policy"
        ),
    },
    "policy": {
        "ru": (
            "📜 Политика конфиденциальности и условия использования\n\n"
            "• Сервис предоставляет цифровые товары.\n"
            "• Физическая доставка отсутствует.\n"
            "• Оплата одноразовая, без подписок.\n"
            "• Возврат средств невозможен после выдачи товара.\n"
            "• Исключение — техническая ошибка сервиса.\n"
            "• Пользователь несёт ответственность за использование.\n"
            "• Сервис предоставляется «как есть».\n"
            "• Telegram ID и username используются для работы бота.\n"
            "• Данные не передаются третьим лицам.\n"
            "• Попытки взлома ведут к блокировке без возврата средств.\n"
            "• Разработчик может менять функционал.\n"
            "• Использование бота означает согласие с правилами."
        ),
        "ua": (
            "📜 Політика конфіденційності та умови користування\n\n"
            "• Сервіс надає цифрові товари.\n"
            "• Фізична доставка відсутня.\n"
            "• Оплата одноразова, без підписок.\n"
            "• Повернення коштів неможливе після видачі товару.\n"
            "• Виняток — технічна помилка сервісу.\n"
            "• Користувач несе відповідальність за використання.\n"
            "• Сервіс надається «як є».\n"
            "• Telegram ID та username використовуються для роботи бота.\n"
            "• Дані не передаються третім особам.\n"
            "• Спроби злому ведуть до блокування без повернення коштів.\n"
            "• Розробник може змінювати функціонал.\n"
            "• Використання бота означає згоду з правилами."
        ),
        "en": (
            "📜 Privacy Policy and Terms of Service\n\n"
            "• The service provides digital goods.\n"
            "• No physical delivery.\n"
            "• One-time payments only.\n"
            "• No refunds after delivery.\n"
            "• Exception — technical service failure.\n"
            "• Users are responsible for usage.\n"
            "• Service provided 'as is'.\n"
            "• Telegram ID and username are used for bot operation.\n"
            "• Data is not shared with third parties.\n"
            "• Violations lead to blocking without refunds.\n"
            "• Developer may change functionality.\n"
            "• Using the bot means agreement with rules."
        ),
    },
    "invoice_error": {
        "ru": "❌ Ошибка создания счёта. Попробуйте позже.\n\nЕсли ошибка повторяется, обратитесь к @hoxxp",
        "ua": "❌ Помилка створення рахунку. Спробуйте пізніше.\n\nЯкщо помилка повторюється, зверніться до @hoxxp",
        "en": "❌ Invoice creation error. Please try again later.\n\nIf the error persists, contact @hoxxp",
    },
    "no_products": {"ru": "❌ Нет товаров.", "ua": "❌ Немає товарів.", "en": "❌ No products."},
    "product_sold_already": {"ru": "❌ Товар продан.", "ua": "❌ Товар продано.", "en": "❌ Sold."},
    "promo_created": {"ru": "✅ Промокод {code} для {product}", "ua": "✅ Промокод {code} для {product}", "en": "✅ Promocode {code} for {product}"},
    "promo_already_used": {"ru": "❌ Промокод уже использован.", "ua": "❌ Вже використано.", "en": "❌ Already used."},
    "promo_invalid": {"ru": "❌ Неверный промокод.", "ua": "❌ Невірний код.", "en": "❌ Invalid."},
    "promo_wrong_product": {"ru": "❌ Не подходит для этого товара.", "ua": "❌ Не підходить.", "en": "❌ Wrong product."},
    "free_product_received": {"ru": "🎉 Товар получен!\n\n", "ua": "🎉 Товар отримано!\n\n", "en": "🎉 Product received!\n\n"},
    "no_codes_found": {"ru": "📭 Кодов не найдено.", "ua": "📭 Кодів не знайдено.", "en": "📭 No codes found."},
    "getting_codes": {"ru": "🔍 Получаю коды...", "ua": "🔍 Отримую коди...", "en": "🔍 Getting codes..."},
    "enter_phone_for_reg": {"ru": "📱 Введите номер телефона для регистрации аккаунта:\nФормат: +380991234567", "ua": "📱 Введіть номер телефону:", "en": "📱 Enter phone number:"},
    "enter_code": {"ru": "✅ Код отправлен!\nВведите код из Telegram:", "ua": "✅ Код надіслано!\nВведіть код:", "en": "✅ Code sent!\nEnter code:"},
    "enter_2fa": {"ru": "🔐 Введите пароль 2FA:", "ua": "🔐 Введіть пароль 2FA:", "en": "🔐 Enter 2FA password:"},
    "no_purchases": {"ru": "📭 У вас нет купленных аккаунтов.", "ua": "📭 У вас немає куплених акаунтів.", "en": "📭 You have no purchased accounts."},
    "my_accounts": {"ru": "📱 МОИ АККАУНТЫ\n\nВыберите аккаунт для получения кодов:", "ua": "📱 МОЇ АКАУНТИ\n\nВиберіть акаунт:", "en": "📱 MY ACCOUNTS\n\nSelect account:"},
    "account_deleted": {"ru": "✅ Аккаунт удален из вашего списка.", "ua": "✅ Акаунт видалено.", "en": "✅ Account deleted."},
    "confirm_delete": {"ru": "⚠️ Вы уверены, что хотите удалить аккаунт?\n\n{name}\n🌍 {country}\n\nДанные будут удалены безвозвратно.", "ua": "⚠️ Ви впевнені? {name}", "en": "⚠️ Are you sure? {name}"},
    "admin_only": {"ru": "⛔ Только для админа.", "ua": "⛔ Тільки для адміна.", "en": "⛔ Admin only."},
    "support": {"ru": "По вопросам: @hoxxp", "ua": "З питань: @hoxxp", "en": "Questions: @hoxxp"},
}

# -------------------- КЛАВІАТУРИ --------------------
def main_menu(user_id: int) -> InlineKeyboardMarkup:
    lang = get_lang(user_id)
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text=T["catalog_btn"][lang], callback_data="catalog")],
        [InlineKeyboardButton(text=T["my_purchases_btn"][lang], callback_data="my_purchases")],
        [InlineKeyboardButton(text=T["support_btn"][lang], callback_data="support")],
        [InlineKeyboardButton(text=T["policy_btn"][lang], callback_data="policy")],
        [InlineKeyboardButton(text=T["lang_btn"][lang], callback_data="change_lang")],
    ])

def admin_menu() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="➕ Добавить товар", callback_data="admin_add_product")],
        [InlineKeyboardButton(text="📋 Список товаров", callback_data="admin_list_products")],
        [InlineKeyboardButton(text="✏️ Редактировать товар", callback_data="admin_edit_product")],
        [InlineKeyboardButton(text="🗑️ Удалить товар", callback_data="admin_delete_product")],
        [InlineKeyboardButton(text="🎫 Промокоды", callback_data="admin_promo_menu")],
        [InlineKeyboardButton(text="➕ API ключи", callback_data="admin_add_api")],
        [InlineKeyboardButton(text="◀️ Выйти в меню", callback_data="back_to_user_menu")],
    ])

def admin_promo_menu() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="➕ Создать промокод", callback_data="admin_create_promo")],
        [InlineKeyboardButton(text="📋 Список промокодов", callback_data="admin_list_promos")],
        [InlineKeyboardButton(text="🗑️ Удалить промокод", callback_data="admin_delete_promo")],
        [InlineKeyboardButton(text="◀️ Назад в админку", callback_data="admin_panel")],
    ])

def back_kb() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="◀️ Назад", callback_data="admin_panel")]])

def catalog_kb(user_id: int, products_list: list, page: int = 0) -> InlineKeyboardMarkup:
    lang = get_lang(user_id)
    items_per_page = 5
    start = page * items_per_page
    end = start + items_per_page
    page_products = products_list[start:end]
    keyboard = []
    for p in page_products:
        keyboard.append([InlineKeyboardButton(text=f"{p.get('name')} - {p.get('price')} USDT [{p.get('country')}]", callback_data=f"view_product_{p.get('id')}")])
    nav = []
    if page > 0: nav.append(InlineKeyboardButton(text="◀️", callback_data=f"catalog_page_{page-1}"))
    if end < len(products_list): nav.append(InlineKeyboardButton(text="▶️", callback_data=f"catalog_page_{page+1}"))
    if nav: keyboard.append(nav)
    keyboard.append([InlineKeyboardButton(text=T["back_to_menu"][lang], callback_data="user_back_to_menu")])
    return InlineKeyboardMarkup(inline_keyboard=keyboard)

def product_kb(user_id: int, product_id: int, has_session: bool = False, for_purchased: bool = False) -> InlineKeyboardMarkup:
    lang = get_lang(user_id)
    kb = []
    if not for_purchased:
        # Каталог: тільки купити і промокод
        kb.append([InlineKeyboardButton(text=T["buy_crypto_btn"][lang], callback_data=f"buy_crypto_{product_id}")])
        kb.append([InlineKeyboardButton(text=T["use_promo_btn"][lang], callback_data=f"use_promo_{product_id}")])
        kb.append([InlineKeyboardButton(text=T["back_to_catalog"][lang], callback_data="catalog")])
    else:
        # Мої акаунти: коди і видалити
        if has_session:
            kb.append([InlineKeyboardButton(text=T["get_codes_btn"][lang], callback_data=f"get_codes_{product_id}")])
        kb.append([InlineKeyboardButton(text=T["delete_account_btn"][lang], callback_data=f"delete_account_{product_id}")])
        kb.append([InlineKeyboardButton(text=T["back_to_menu"][lang], callback_data="user_back_to_menu")])
    return InlineKeyboardMarkup(inline_keyboard=kb)

def purchased_accounts_kb(user_id: int) -> InlineKeyboardMarkup:
    lang = get_lang(user_id)
    purchased_ids = user_purchases.get(str(user_id), [])
    purchased_products = [p for p in products if p.get('id') in purchased_ids and p.get('sold', False)]
    keyboard = []
    for p in purchased_products:
        keyboard.append([InlineKeyboardButton(text=f"📱 {p.get('name')} | {p.get('country')}", callback_data=f"view_purchased_{p.get('id')}")])
    keyboard.append([InlineKeyboardButton(text=T["back_to_menu"][lang], callback_data="user_back_to_menu")])
    return InlineKeyboardMarkup(inline_keyboard=keyboard)

def pay_kb(lang: str, pay_url: str, invoice_id: str) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text=T["pay_btn"][lang], url=pay_url)],
        [InlineKeyboardButton(text=T["check_btn"][lang], callback_data=f"chk:{invoice_id}")],
        [InlineKeyboardButton(text=T["back_to_catalog"][lang], callback_data="catalog")]
    ])

def user_back_kb(user_id: int) -> InlineKeyboardMarkup:
    lang = get_lang(user_id)
    return InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text=T["back_to_menu"][lang], callback_data="user_back_to_menu")]])

def admin_only(func):
    @wraps(func)
    async def wrapper(event, *args, **kwargs):
        user_id = event.from_user.id if hasattr(event, 'from_user') else event.chat.id
        if user_id != ADMIN_ID:
            if hasattr(event, 'answer'):
                await event.answer(T["admin_only"]["ru"], show_alert=True)
            else:
                await event.reply(T["admin_only"]["ru"])
            return
        return await func(event, *args, **kwargs)
    return wrapper

# ==================== АДМІНКА: ТОВАРИ ====================
@admin_only
@dp.message(Command("admin"))
async def admin_panel(message: Message):
    await message.answer("👑 АДМИН ПАНЕЛЬ", reply_markup=admin_menu())

@admin_only
@dp.callback_query(lambda c: c.data == "admin_panel")
async def admin_panel_callback(callback: types.CallbackQuery):
    await callback.answer()
    await callback.message.edit_text("👑 АДМИН ПАНЕЛЬ", reply_markup=admin_menu())

@admin_only
@dp.callback_query(lambda c: c.data == "admin_add_product")
async def admin_add_product(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    await state.set_state(AddProductState.waiting_name)
    await callback.message.edit_text("📝 Введите название товара:", reply_markup=back_kb())

@admin_only
@dp.message(AddProductState.waiting_name)
async def admin_product_name(message: Message, state: FSMContext):
    await state.update_data(name=message.text.strip())
    await state.set_state(AddProductState.waiting_description)
    await message.answer("📝 Введите описание товара:", reply_markup=back_kb())

@admin_only
@dp.message(AddProductState.waiting_description)
async def admin_product_desc(message: Message, state: FSMContext):
    await state.update_data(description=message.text.strip())
    await state.set_state(AddProductState.waiting_price)
    await message.answer("💰 Цена в USDT:", reply_markup=back_kb())

@admin_only
@dp.message(AddProductState.waiting_price)
async def admin_product_price(message: Message, state: FSMContext):
    try:
        price = float(message.text.strip())
        await state.update_data(price=price)
        await state.set_state(AddProductState.waiting_country)
        await message.answer("🌍 Страна (например: Россия, Украина, USA):", reply_markup=back_kb())
    except ValueError:
        await message.answer("❌ Введите число:", reply_markup=back_kb())

@admin_only
@dp.message(AddProductState.waiting_country)
async def admin_product_country(message: Message, state: FSMContext):
    await state.update_data(country=message.text.strip())
    data = await state.get_data()
    temp_product_data[str(message.from_user.id)] = {
        'name': data.get('name'),
        'description': data.get('description'),
        'price': data.get('price'),
        'country': data.get('country')
    }
    user_id_str = str(message.from_user.id)
    if user_id_str not in saved_apis or not saved_apis[user_id_str]:
        await message.answer("❌ Нет API ключей!\n\nСначала добавь API ключи через '➕ API ключи' в админке", reply_markup=admin_menu())
        await state.clear()
        return
    apis = saved_apis[user_id_str]
    keyboard = []
    for i, api in enumerate(apis):
        keyboard.append([InlineKeyboardButton(text=f"API #{i+1} (ID: {api['api_id']})", callback_data=f"reg_api_{i}")])
    keyboard.append([InlineKeyboardButton(text="◀️ Отмена", callback_data="admin_panel")])
    await state.set_state(RegState.waiting_phone)
    await message.answer("🔐 ТЕПЕРЬ НУЖНО ЗАРЕГИСТРИРОВАТЬ АККАУНТ\n\nВыбери API ключи для регистрации:", reply_markup=InlineKeyboardMarkup(inline_keyboard=keyboard))

# -------------------- РЕГІСТРАЦІЯ АККАУНТА ПРИ ДОДАВАННІ ТОВАРУ --------------------
@admin_only
@dp.callback_query(lambda c: c.data.startswith("reg_api_"))
async def reg_select_api(callback: types.CallbackQuery, state: FSMContext):
    current_state = await state.get_state()
    if current_state != RegState.waiting_phone:
        await callback.answer("❌ Дія не дозволена", show_alert=True)
        return
    await callback.answer()
    user_id_str = str(callback.from_user.id)
    index = int(callback.data.split("_")[2])
    api = saved_apis[user_id_str][index]
    await state.update_data(api_id=api['api_id'], api_hash=api['api_hash'])
    await callback.message.edit_text(f"Выбран API #{index+1} (ID: {api['api_id']})\n\n{T['enter_phone_for_reg']['ru']}", reply_markup=back_kb())

@admin_only
@dp.message(RegState.waiting_phone)
async def reg_process_phone(message: Message, state: FSMContext):
    phone = message.text.strip()
    if not phone.startswith('+'):
        phone = '+' + phone
    user_id_str = str(message.from_user.id)
    data = await state.get_data()
    api_id = data.get('api_id')
    api_hash = data.get('api_hash')
    if not api_id or not api_hash:
        await message.answer("❌ Ошибка. Начни заново /admin", reply_markup=admin_menu())
        await state.clear()
        return
    session_name = f"reg_{user_id_str}_{phone.replace('+', '')}_{int(datetime.now().timestamp())}"
    client = TelegramClient(session_name, api_id, api_hash)
    await client.connect()
    try:
        result = await client.send_code_request(phone)
        user_sessions[user_id_str] = {
            'client': client,
            'phone': phone,
            'phone_code_hash': result.phone_code_hash,
            'session_name': session_name,
            'api_id': api_id,
            'api_hash': api_hash,
        }
        await state.set_state(RegState.waiting_code)
        await message.answer(T["enter_code"]["ru"], reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔄 Отправить ещё раз", callback_data="reg_resend_code")],
            [InlineKeyboardButton(text="◀️ Отмена", callback_data="admin_panel")]
        ]))
    except PhoneNumberInvalidError:
        await client.disconnect()
        await message.answer(f"❌ Неверный номер: {phone}\nФормат: +380991234567", reply_markup=back_kb())
    except FloodWaitError as e:
        await client.disconnect()
        await message.answer(f"⏳ Жди {e.seconds} секунд", reply_markup=back_kb())
        await state.clear()
    except Exception as e:
        await client.disconnect()
        await message.answer(f"❌ Ошибка: {str(e)[:100]}", reply_markup=back_kb())
        await state.clear()

@admin_only
@dp.callback_query(lambda c: c.data == "reg_resend_code")
async def reg_resend_code(callback: types.CallbackQuery, state: FSMContext):
    current_state = await state.get_state()
    if current_state != RegState.waiting_code:
        await callback.answer("❌ Дія не дозволена", show_alert=True)
        return
    await callback.answer()
    user_id_str = str(callback.from_user.id)
    if user_id_str not in user_sessions:
        await callback.message.edit_text("❌ Сессия потеряна.", reply_markup=admin_menu())
        return
    old_client = user_sessions[user_id_str].get('client')
    if old_client:
        try: await old_client.disconnect()
        except: pass
    phone = user_sessions[user_id_str]['phone']
    api_id = user_sessions[user_id_str]['api_id']
    api_hash = user_sessions[user_id_str]['api_hash']
    session_name = f"reg_{user_id_str}_{phone.replace('+', '')}_{int(datetime.now().timestamp())}"
    client = TelegramClient(session_name, api_id, api_hash)
    await client.connect()
    await callback.message.edit_text(f"🔄 Отправляю код на {phone}...")
    try:
        result = await client.send_code_request(phone)
        user_sessions[user_id_str] = {
            'client': client,
            'phone': phone,
            'phone_code_hash': result.phone_code_hash,
            'session_name': session_name,
            'api_id': api_id,
            'api_hash': api_hash,
        }
        await state.set_state(RegState.waiting_code)
        await callback.message.edit_text(T["enter_code"]["ru"], reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔄 Ещё раз", callback_data="reg_resend_code")],
            [InlineKeyboardButton(text="◀️ Отмена", callback_data="admin_panel")]
        ]))
    except FloodWaitError as e:
        await client.disconnect()
        await callback.message.edit_text(f"⏳ Жди {e.seconds} сек", reply_markup=back_kb())
    except Exception as e:
        await client.disconnect()
        await callback.message.edit_text(f"❌ {str(e)[:100]}", reply_markup=back_kb())

@admin_only
@dp.message(RegState.waiting_code)
async def reg_process_code(message: Message, state: FSMContext):
    code = message.text.strip()
    user_id_str = str(message.from_user.id)
    if user_id_str not in user_sessions:
        await message.answer("❌ Сессия потеряна. /admin", reply_markup=admin_menu())
        await state.clear()
        return
    session_data = user_sessions[user_id_str]
    client = session_data.get('client')
    phone = session_data.get('phone')
    phone_code_hash = session_data.get('phone_code_hash')
    if not client:
        await message.answer("❌ Ошибка.", reply_markup=admin_menu())
        await state.clear()
        return
    if not code.isdigit():
        await message.answer("❌ Код только цифры! Введи ещё раз:")
        return
    try:
        await client.sign_in(phone, code, phone_code_hash=phone_code_hash)
        me = await client.get_me()
        session_string = StringSession.save(client.session)
        temp_data = temp_product_data.get(user_id_str, {})
        new_id = max([p.get('id', 0) for p in products] + [0]) + 1
        product = {
            'id': new_id,
            'name': temp_data.get('name'),
            'description': temp_data.get('description'),
            'price': temp_data.get('price'),
            'country': temp_data.get('country'),
            'credentials': session_string,
            'sold': False,
            'created_at': datetime.now().isoformat()
        }
        products.append(product)
        save_products(products)
        if user_id_str in temp_product_data:
            del temp_product_data[user_id_str]
        await client.disconnect()
        del user_sessions[user_id_str]
        await state.clear()
        await message.answer(f"✅ ТОВАР ДОБАВЛЕН!\n\n📦 {product['name']}\n💰 {product['price']} USDT\n🌍 {product['country']}\n\n👤 Аккаунт: {me.first_name}\n📱 {phone}\n\n🆔 ID товара: {new_id}", reply_markup=admin_menu())
    except SessionPasswordNeededError:
        await state.set_state(RegState.waiting_password)
        await state.update_data(client=client, phone=phone)
        await message.answer(T["enter_2fa"]["ru"], reply_markup=back_kb())
    except Exception as e:
        error_str = str(e)
        if "PHONE_CODE_INVALID" in error_str or "expired" in error_str.lower():
            await message.answer("❌ Код неверный или истек.\n\nНажми 'Отправить ещё раз'", reply_markup=admin_menu())
        else:
            await message.answer(f"❌ {error_str[:200]}", reply_markup=admin_menu())

@admin_only
@dp.message(RegState.waiting_password)
async def reg_process_2fa(message: Message, state: FSMContext):
    password = message.text.strip()
    user_id_str = str(message.from_user.id)
    data = await state.get_data()
    client = data.get('client')
    phone = data.get('phone')
    if not client:
        await message.answer("❌ Ошибка. /admin", reply_markup=admin_menu())
        await state.clear()
        return
    try:
        await client.sign_in(password=password)
        me = await client.get_me()
        session_string = StringSession.save(client.session)
        temp_data = temp_product_data.get(user_id_str, {})
        new_id = max([p.get('id', 0) for p in products] + [0]) + 1
        product = {
            'id': new_id,
            'name': temp_data.get('name'),
            'description': temp_data.get('description'),
            'price': temp_data.get('price'),
            'country': temp_data.get('country'),
            'credentials': session_string,
            'sold': False,
            'created_at': datetime.now().isoformat()
        }
        products.append(product)
        save_products(products)
        if user_id_str in temp_product_data:
            del temp_product_data[user_id_str]
        await client.disconnect()
        await state.clear()
        await message.answer(f"✅ ТОВАР ДОБАВЛЕН!\n\n📦 {product['name']}\n💰 {product['price']} USDT\n🌍 {product['country']}\n\n👤 Аккаунт: {me.first_name}\n📱 {phone}\n\n🆔 ID товара: {new_id}", reply_markup=admin_menu())
    except Exception as e:
        await message.answer(f"❌ Неверный пароль: {str(e)[:100]}", reply_markup=back_kb())

# -------------------- АДМІНКА: СПИСОК ТОВАРІВ (скорочено, але працює) --------------------
@admin_only
@dp.callback_query(lambda c: c.data == "admin_list_products")
async def admin_list_products(callback: types.CallbackQuery):
    await callback.answer()
    available = [p for p in products if not p.get('sold', False)]
    if not available:
        await callback.message.edit_text("📭 Нет товаров в продаже.", reply_markup=admin_menu())
        return
    keyboard = []
    for p in available:
        keyboard.append([InlineKeyboardButton(text=f"{p.get('id')} | {p.get('name')} | {p.get('price')} USDT", callback_data=f"admin_view_product_{p.get('id')}")])
    keyboard.append([InlineKeyboardButton(text="◀️ Назад", callback_data="admin_panel")])
    await callback.message.edit_text("📋 ТОВАРЫ В ПРОДАЖЕ:", reply_markup=InlineKeyboardMarkup(inline_keyboard=keyboard))

@admin_only
@dp.callback_query(lambda c: c.data.startswith("admin_view_product_"))
async def admin_view_product_details(callback: types.CallbackQuery):
    await callback.answer()
    product_id = int(callback.data.split("_")[3])
    product = next((p for p in products if p.get('id') == product_id and not p.get('sold', False)), None)
    if not product:
        await callback.message.edit_text("❌ Товар не найден.", reply_markup=admin_menu())
        return
    text = f"📦 {product.get('name')}\n💰 {product.get('price')} USDT\n🌍 {product.get('country')}\n\n📝 {product.get('description', 'Нет описания')}\n🆔 ID: {product_id}"
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✏️ Редактировать", callback_data=f"admin_edit_this_{product_id}")],
        [InlineKeyboardButton(text="🗑️ Удалить", callback_data=f"admin_delete_this_{product_id}")],
        [InlineKeyboardButton(text="◀️ Назад", callback_data="admin_list_products")]
    ])
    await callback.message.edit_text(text, reply_markup=keyboard)

@admin_only
@dp.callback_query(lambda c: c.data.startswith("admin_edit_this_"))
async def admin_edit_this_product(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    product_id = int(callback.data.split("_")[3])
    product = next((p for p in products if p.get('id') == product_id and not p.get('sold', False)), None)
    if not product:
        await callback.message.edit_text("❌ Товар не найден.", reply_markup=admin_menu())
        return
    await state.update_data(edit_product_id=product_id)
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📝 Название", callback_data=f"edit_field_name_{product_id}")],
        [InlineKeyboardButton(text="💰 Цена", callback_data=f"edit_field_price_{product_id}")],
        [InlineKeyboardButton(text="🌍 Страна", callback_data=f"edit_field_country_{product_id}")],
        [InlineKeyboardButton(text="📄 Описание", callback_data=f"edit_field_description_{product_id}")],
        [InlineKeyboardButton(text="◀️ Назад", callback_data="admin_panel")]
    ])
    await callback.message.edit_text(f"✏️ Редактирование: {product.get('name')}", reply_markup=kb)

@admin_only
@dp.callback_query(lambda c: c.data.startswith("edit_field_"))
async def admin_edit_field_select(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    parts = callback.data.split("_")
    field = parts[2]
    product_id = int(parts[3])
    await state.update_data(edit_field=field, edit_product_id=product_id)
    await state.set_state(EditProductState.waiting_value)
    prompts = {'name': "📝 Новое название:", 'price': "💰 Новая цена USDT:", 'country': "🌍 Новая страна:", 'description': "📄 Новое описание:"}
    await callback.message.edit_text(prompts.get(field, "Введите значение:"), reply_markup=back_kb())

@admin_only
@dp.message(EditProductState.waiting_value)
async def admin_edit_save_value(message: Message, state: FSMContext):
    data = await state.get_data()
    product_id = data.get('edit_product_id')
    field = data.get('edit_field')
    value = message.text.strip()
    for p in products:
        if p.get('id') == product_id and not p.get('sold', False):
            if field == 'price':
                try:
                    value = float(value)
                except ValueError:
                    await message.answer("❌ Введите число:", reply_markup=back_kb())
                    return
            p[field] = value
            break
    save_products(products)
    await state.clear()
    product = next((p for p in products if p.get('id') == product_id), None)
    await message.answer(f"✅ Обновлено!\n\n📦 {product.get('name')}\n💰 {product.get('price')} USDT\n🌍 {product.get('country')}", reply_markup=admin_menu())

@admin_only
@dp.callback_query(lambda c: c.data == "admin_delete_product")
async def admin_delete_products_list(callback: types.CallbackQuery):
    await callback.answer()
    available = [p for p in products if not p.get('sold', False)]
    if not available:
        await callback.message.edit_text("📭 Нет товаров для удаления.", reply_markup=admin_menu())
        return
    keyboard = []
    for p in available:
        keyboard.append([InlineKeyboardButton(text=f"{p.get('id')} | {p.get('name')}", callback_data=f"admin_delete_confirm_{p.get('id')}")])
    keyboard.append([InlineKeyboardButton(text="◀️ Назад", callback_data="admin_panel")])
    await callback.message.edit_text("🗑️ Выберите товар для удаления:", reply_markup=InlineKeyboardMarkup(inline_keyboard=keyboard))

@admin_only
@dp.callback_query(lambda c: c.data.startswith("admin_delete_confirm_"))
async def admin_delete_confirm(callback: types.CallbackQuery):
    await callback.answer()
    product_id = int(callback.data.split("_")[3])
    product = next((p for p in products if p.get('id') == product_id and not p.get('sold', False)), None)
    if not product:
        await callback.message.edit_text("❌ Товар не найден.", reply_markup=admin_menu())
        return
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ ДА, УДАЛИТЬ", callback_data=f"admin_delete_do_{product_id}")],
        [InlineKeyboardButton(text="❌ НЕТ", callback_data="admin_delete_product")]
    ])
    await callback.message.edit_text(f"⚠️ УДАЛИТЬ «{product.get('name')}»?\n\nЭто действие нельзя отменить!", reply_markup=keyboard)

@admin_only
@dp.callback_query(lambda c: c.data.startswith("admin_delete_do_"))
async def admin_delete_do(callback: types.CallbackQuery):
    await callback.answer()
    product_id = int(callback.data.split("_")[3])
    global products
    found = next((p for p in products if p.get('id') == product_id and not p.get('sold', False)), None)
    if found:
        products.remove(found)
        save_products(products)
        await callback.message.edit_text(f"✅ Товар «{found.get('name')}» удалён!", reply_markup=admin_menu())
    else:
        await callback.message.edit_text("❌ Товар не найден.", reply_markup=admin_menu())

@admin_only
@dp.callback_query(lambda c: c.data.startswith("admin_delete_this_"))
async def admin_delete_this_product(callback: types.CallbackQuery):
    await callback.answer()
    product_id = int(callback.data.split("_")[3])
    product = next((p for p in products if p.get('id') == product_id and not p.get('sold', False)), None)
    if not product:
        await callback.message.edit_text("❌ Товар не найден.", reply_markup=admin_menu())
        return
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ ДА, УДАЛИТЬ", callback_data=f"admin_delete_do_{product_id}")],
        [InlineKeyboardButton(text="❌ НЕТ", callback_data="admin_list_products")]
    ])
    await callback.message.edit_text(f"⚠️ УДАЛИТЬ «{product.get('name')}»?", reply_markup=keyboard)

@admin_only
@dp.callback_query(lambda c: c.data == "admin_edit_product")
async def admin_edit_products_list(callback: types.CallbackQuery):
    await callback.answer()
    available = [p for p in products if not p.get('sold', False)]
    if not available:
        await callback.message.edit_text("📭 Нет товаров для редактирования.", reply_markup=admin_menu())
        return
    keyboard = []
    for p in available:
        keyboard.append([InlineKeyboardButton(text=f"{p.get('id')} | {p.get('name')}", callback_data=f"admin_edit_select_{p.get('id')}")])
    keyboard.append([InlineKeyboardButton(text="◀️ Назад", callback_data="admin_panel")])
    await callback.message.edit_text("✏️ Выберите товар:", reply_markup=InlineKeyboardMarkup(inline_keyboard=keyboard))

@admin_only
@dp.callback_query(lambda c: c.data.startswith("admin_edit_select_"))
async def admin_edit_select_product(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    product_id = int(callback.data.split("_")[3])
    product = next((p for p in products if p.get('id') == product_id and not p.get('sold', False)), None)
    if not product:
        await callback.message.edit_text("❌ Товар не найден.", reply_markup=admin_menu())
        return
    await state.update_data(edit_product_id=product_id)
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📝 Название", callback_data=f"edit_field_name_{product_id}")],
        [InlineKeyboardButton(text="💰 Цена", callback_data=f"edit_field_price_{product_id}")],
        [InlineKeyboardButton(text="🌍 Страна", callback_data=f"edit_field_country_{product_id}")],
        [InlineKeyboardButton(text="📄 Описание", callback_data=f"edit_field_description_{product_id}")],
        [InlineKeyboardButton(text="◀️ Назад", callback_data="admin_panel")]
    ])
    await callback.message.edit_text(f"✏️ Редактирование: {product.get('name')}", reply_markup=kb)

# -------------------- АДМІНКА: ПРОМОКОДИ (скорочено, але працює) --------------------
@admin_only
@dp.callback_query(lambda c: c.data == "admin_promo_menu")
async def admin_promo_menu_handler(callback: types.CallbackQuery):
    await callback.answer()
    await callback.message.edit_text("🎫 УПРАВЛЕНИЕ ПРОМОКОДАМИ", reply_markup=admin_promo_menu())

@admin_only
@dp.callback_query(lambda c: c.data == "admin_create_promo")
async def admin_create_promo_start(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    available = [p for p in products if not p.get('sold', False)]
    if not available:
        await callback.message.edit_text("❌ Нет товаров.", reply_markup=admin_menu())
        return
    keyboard = []
    for p in available:
        keyboard.append([InlineKeyboardButton(text=f"{p.get('name')} - {p.get('price')} USDT", callback_data=f"promo_product_{p.get('id')}")])
    keyboard.append([InlineKeyboardButton(text="◀️ Назад", callback_data="admin_promo_menu")])
    await state.set_state(PromoState.waiting_create_product)
    await callback.message.edit_text("Выберите товар:", reply_markup=InlineKeyboardMarkup(inline_keyboard=keyboard))

@admin_only
@dp.callback_query(lambda c: c.data.startswith("promo_product_"))
async def admin_create_promo_get_code(callback: types.CallbackQuery, state: FSMContext):
    current_state = await state.get_state()
    if current_state != PromoState.waiting_create_product:
        await callback.answer("❌ Дія не дозволена", show_alert=True)
        return
    await callback.answer()
    product_id = int(callback.data.split("_")[2])
    product = next((p for p in products if p.get('id') == product_id), None)
    if not product or product.get('sold', False):
        await callback.answer("❌ Товар не найден", show_alert=True)
        return
    await state.update_data(promo_product_id=product_id, promo_product_name=product.get('name'))
    await state.set_state(PromoState.waiting_create_code)
    await callback.message.edit_text("Введите код промокода (латиница, цифры):", reply_markup=back_kb())

@admin_only
@dp.message(PromoState.waiting_create_code)
async def admin_create_promo_save(message: Message, state: FSMContext):
    code = message.text.strip().upper()
    if code in promocodes:
        await message.answer("❌ Такой уже есть. Введите другой:")
        return
    data = await state.get_data()
    product_id = data.get('promo_product_id')
    product_name = data.get('promo_product_name')
    promocodes[code] = {
        'product_id': product_id,
        'product_name': product_name,
        'used': False,
        'used_by': None,
        'used_at': None,
        'created_at': datetime.now().isoformat()
    }
    save_promocodes(promocodes)
    await state.clear()
    await message.answer(T["promo_created"]["ru"].format(code=code, product=product_name), reply_markup=admin_promo_menu())

@admin_only
@dp.callback_query(lambda c: c.data == "admin_list_promos")
async def admin_list_promos(callback: types.CallbackQuery):
    await callback.answer()
    if not promocodes:
        await callback.message.edit_text("🎫 Нет промокодов.", reply_markup=admin_promo_menu())
        return
    text = "🎫 ПРОМОКОДЫ:\n\n"
    for code, data in promocodes.items():
        status = "✅" if data.get('used') else "❌"
        used_by = f" (юзер: {data.get('used_by')})" if data.get('used_by') else ""
        text += f"{status} {code} → {data.get('product_name')}{used_by}\n"
    await callback.message.edit_text(text, reply_markup=admin_promo_menu())

@admin_only
@dp.callback_query(lambda c: c.data == "admin_delete_promo")
async def admin_delete_promo_prompt(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    if not promocodes:
        await callback.message.edit_text("🎫 Нет промокодов для удаления.", reply_markup=admin_promo_menu())
        return
    keyboard = []
    for code, data in promocodes.items():
        status = "✅" if data.get('used') else "❌"
        keyboard.append([InlineKeyboardButton(text=f"{status} {code} ({data.get('product_name')})", callback_data=f"delete_promo_{code}")])
    keyboard.append([InlineKeyboardButton(text="◀️ Назад", callback_data="admin_promo_menu")])
    await state.set_state(PromoState.waiting_delete_promo)
    await callback.message.edit_text("🗑️ Выбери промокод для удаления:", reply_markup=InlineKeyboardMarkup(inline_keyboard=keyboard))

@admin_only
@dp.callback_query(lambda c: c.data.startswith("delete_promo_"))
async def admin_delete_promo_execute(callback: types.CallbackQuery, state: FSMContext):
    current_state = await state.get_state()
    if current_state != PromoState.waiting_delete_promo:
        await callback.answer("❌ Дія не дозволена", show_alert=True)
        return
    code = callback.data.split("_")[2]
    if code not in promocodes:
        await callback.answer("❌ Не найден", show_alert=True)
        return
    del promocodes[code]
    save_promocodes(promocodes)
    await state.clear()
    await callback.message.edit_text(f"✅ Промокод {code} удалён!", reply_markup=admin_promo_menu())

# -------------------- АДМІНКА: API КЛЮЧІ --------------------
@admin_only
@dp.callback_query(lambda c: c.data == "admin_add_api")
async def admin_add_api(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    await state.set_state(RegState.waiting_api_id)
    await callback.message.edit_text("⚙️ ДОБАВЛЕНИЕ API\n\nВведите API_ID с my.telegram.org (число):", reply_markup=back_kb())

@admin_only
@dp.message(RegState.waiting_api_id)
async def admin_api_id(message: Message, state: FSMContext):
    try:
        api_id = int(message.text.strip())
        await state.update_data(api_id=api_id)
        await state.set_state(RegState.waiting_api_hash)
        await message.answer("Введите API_HASH (32 символа):", reply_markup=back_kb())
    except ValueError:
        await message.answer("❌ Число! Введите API_ID:", reply_markup=back_kb())

@admin_only
@dp.message(RegState.waiting_api_hash)
async def admin_api_hash(message: Message, state: FSMContext):
    api_hash = message.text.strip()
    data = await state.get_data()
    api_id = data.get('api_id')
    user_id = str(message.from_user.id)
    if user_id not in saved_apis:
        saved_apis[user_id] = []
    saved_apis[user_id].append({'api_id': api_id, 'api_hash': api_hash, 'created_at': datetime.now().isoformat()})
    save_apis()
    await state.clear()
    await message.answer(f"✅ API добавлен! ID: {api_id}", reply_markup=admin_menu())

# ==================== ПОКУПЕЦЬ ====================
@dp.callback_query(lambda c: c.data == "catalog")
async def show_catalog(callback: types.CallbackQuery):
    await callback.answer()
    available = [p for p in products if not p.get('sold', False)]
    if not available:
        await callback.message.edit_text(T["no_products"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))
        return
    await callback.message.edit_text("📦 КАТАЛОГ:\n\nНажми на товар для просмотра", reply_markup=catalog_kb(callback.from_user.id, available, 0))

@dp.callback_query(lambda c: c.data.startswith("catalog_page_"))
async def catalog_page(callback: types.CallbackQuery):
    await callback.answer()
    page = int(callback.data.split("_")[2])
    available = [p for p in products if not p.get('sold', False)]
    await callback.message.edit_text("📦 КАТАЛОГ:", reply_markup=catalog_kb(callback.from_user.id, available, page))

@dp.callback_query(lambda c: c.data.startswith("view_product_"))
async def view_product(callback: types.CallbackQuery):
    await callback.answer()
    product_id = int(callback.data.split("_")[2])
    product = None
    for p in products:
        if p.get('id') == product_id and not p.get('sold', False):
            product = p
            break
    if not product:
        await callback.message.edit_text(T["product_sold_already"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))
        return
    has_session = len(product.get('credentials', '')) > 50
    text = f"📦 {product.get('name')}\n\n💰 Цена: {product.get('price')} USDT\n🌍 Страна: {product.get('country')}\n\n📝 Описание:\n{product.get('description', 'Нет описания')}"
    await callback.message.edit_text(text, reply_markup=product_kb(callback.from_user.id, product_id, has_session, for_purchased=False))

# ==================== ОПЛАТА ====================
@dp.callback_query(lambda c: c.data.startswith("buy_crypto_"))
async def buy_with_crypto(callback: CallbackQuery, state: FSMContext):
    await callback.answer()
    product_id = int(callback.data.split("_")[2])
    product = None
    for p in products:
        if p.get('id') == product_id and not p.get('sold', False):
            product = p
            break
    if not product:
        await callback.message.edit_text(T["product_sold_already"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))
        return
    price = product.get('price', 0)
    lang = get_lang(callback.from_user.id)
    try:
        pay_url, invoice_id = await create_invoice(price, f"Покупка: {product.get('name')}")
    except Exception as e:
        log.error("Invoice creation failed: %s", e)
        await callback.message.edit_text(T["invoice_error"][lang], reply_markup=product_kb(callback.from_user.id, product_id, False))
        return
    await state.update_data(pending_product_id=product_id, pending_invoice_id=invoice_id, pending_amount=price)
    await state.set_state(CryptoPayState.waiting_payment)
    text = T["pay_text"][lang].format(amount=price)
    await callback.message.edit_text(text, reply_markup=pay_kb(lang, pay_url, invoice_id))

@dp.callback_query(lambda c: c.data.startswith("chk:"))
async def check_payment(callback: CallbackQuery, state: FSMContext):
    invoice_id = callback.data.split(":", 1)[1]
    user_id = callback.from_user.id
    lang = get_lang(user_id)

    try:
        await callback.answer()
    except Exception:
        pass

    async def safe_send(text):
        try:
            await bot.send_message(user_id, text)
        except Exception as e:
            log.error(f"safe_send failed: {e}")

    # Перевірка чи рахунок вже використаний
    try:
        cur.execute("SELECT product_id FROM used_invoices WHERE invoice_id=?", (invoice_id,))
        row = cur.fetchone()
    except Exception as e:
        log.error(f"DB error: {e}")
        row = None

    if row:
        product_id = row[0]
        product = next((p for p in products if p.get('id') == product_id), None)
        creds = product.get('credentials', '') if product else "Товар видалено"
        await safe_send(T["paid_already"][lang] + creds)
        return

    await safe_send("🔄 Перевіряю оплату...")

    # Перевірка оплати через CryptoPay
    is_paid = False
    error_text = None
    try:
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.get(
                f"{CRYPTOPAY_BASE}/getInvoices",
                headers={"Crypto-Pay-API-Token": CRYPTO_TOKEN},
                params={"invoice_ids": invoice_id},
            )
            result = resp.json()
            log.info(f"getInvoices response: {result}")
            if result.get("ok"):
                items = result.get("result", {}).get("items", [])
                if items:
                    is_paid = items[0].get("status") == "paid"
            else:
                error_text = f"CryptoPay error: {result}"
    except Exception as e:
        log.error(f"check_invoice exception: {e}")
        error_text = str(e)

    if error_text:
        await safe_send(f"❌ Помилка перевірки оплати. Спробуйте ще раз.\n\nДеталі: {error_text[:100]}")
        try:
            await bot.send_message(ADMIN_ID, f"⚠️ Помилка перевірки!\nІнвойс: {invoice_id}\nЮзер: {user_id}\n{error_text[:200]}")
        except Exception:
            pass
        return

    if not is_paid:
        await safe_send(T["not_paid"][lang])
        try:
            await bot.send_message(ADMIN_ID, f"❌ Оплата не знайдена!\nІнвойс: {invoice_id}\nЮзер: @{callback.from_user.username or user_id} (ID: {user_id})")
        except Exception:
            pass
        return

    # Оплата підтверджена
    await safe_send("✅ Оплата підтверджена! Видаю товар...")

    data = await state.get_data()
    product_id = data.get('pending_product_id')
    amount = data.get('pending_amount')
    if not product_id:
        await callback.message.answer("❌ Помилка: не знайдено товар для цього платежу. Зверніться до адміна.")
        return

    product = None
    for p in products:
        if p.get('id') == product_id and not p.get('sold', False):
            product = p
            break
    if not product:
        await callback.message.answer(T["product_sold_already"][lang])
        return

    # Позначаємо проданим
    product['sold'] = True
    product['sold_to'] = user_id
    product['sold_at'] = datetime.now().isoformat()
    save_products(products)

    # Додаємо в покупки
    user_id_str = str(user_id)
    if user_id_str not in user_purchases:
        user_purchases[user_id_str] = []
    if product_id not in user_purchases[user_id_str]:
        user_purchases[user_id_str].append(product_id)
    save_user_purchases()

    # Записуємо в БД
    cur.execute("INSERT INTO sales_history (product_id, buyer_id, amount, paid_at) VALUES (?,?,?,?)",
                (product_id, user_id, amount or product.get('price'), datetime.now().isoformat()))
    cur.execute("INSERT INTO used_invoices (invoice_id, product_id, user_id) VALUES (?,?,?)",
                (invoice_id, product_id, user_id))
    conn.commit()

    # Сповіщення адміну
    try:
        await bot.send_message(ADMIN_ID, f"💰 ПРОДАЖА!\nТовар: {product.get('name')}\nСума: {amount or product.get('price')} USDT\nПокупець: @{callback.from_user.username or user_id}")
    except:
        pass

    # Видача товару
    creds = product.get('credentials', '')
    response_text = T["paid"][lang] + f"📦 {product.get('name')}\n🌍 {product.get('country')}\n\n🔐 Session string:\n<code>{creds}</code>\n\n📲 Аккаунт додано в 'Мої аккаунти'"
    await callback.message.answer(response_text, parse_mode="HTML", reply_markup=main_menu(user_id))
    await state.clear()

# ==================== МОЇ АККАУНТИ ====================
@dp.callback_query(lambda c: c.data == "my_purchases")
async def my_purchases(callback: CallbackQuery):
    await callback.answer()
    user_id_str = str(callback.from_user.id)
    purchased_ids = user_purchases.get(user_id_str, [])
    if not purchased_ids:
        await callback.message.edit_text(T["no_purchases"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))
        return
    await callback.message.edit_text(T["my_accounts"][get_lang(callback.from_user.id)], reply_markup=purchased_accounts_kb(callback.from_user.id))

@dp.callback_query(lambda c: c.data.startswith("view_purchased_"))
async def view_purchased_account(callback: CallbackQuery):
    await callback.answer()
    product_id = int(callback.data.split("_")[2])
    user_id_str = str(callback.from_user.id)
    if product_id not in user_purchases.get(user_id_str, []):
        await callback.answer("❌ Не ваш аккаунт", show_alert=True)
        return
    product = None
    for p in products:
        if p.get('id') == product_id and p.get('sold', False) and p.get('sold_to') == callback.from_user.id:
            product = p
            break
    if not product:
        await callback.message.edit_text("❌ Аккаунт не найден.", reply_markup=main_menu(callback.from_user.id))
        return
    has_session = len(product.get('credentials', '')) > 50
    text = f"📱 {product.get('name')}\n\n🌍 Страна: {product.get('country')}\n📅 Куплен: {product.get('sold_at', 'Неизвестно')[:19]}\n\n📝 Описание:\n{product.get('description', 'Нет описания')}"
    await callback.message.edit_text(text, reply_markup=product_kb(callback.from_user.id, product_id, has_session, for_purchased=True))

@dp.callback_query(lambda c: c.data.startswith("delete_account_"))
async def delete_purchased_account(callback: CallbackQuery, state: FSMContext):
    await callback.answer()
    product_id = int(callback.data.split("_")[2])
    user_id_str = str(callback.from_user.id)
    product = None
    for p in products:
        if p.get('id') == product_id:
            product = p
            break
    if not product or product_id not in user_purchases.get(user_id_str, []):
        await callback.answer("❌ Не ваш аккаунт", show_alert=True)
        return
    await state.update_data(delete_account_id=product_id)
    text = T["confirm_delete"][get_lang(callback.from_user.id)].format(name=product.get('name'), country=product.get('country'))
    await callback.message.edit_text(text, reply_markup=InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Да, удалить", callback_data="confirm_delete_account")],
        [InlineKeyboardButton(text="❌ Нет", callback_data="my_purchases")]
    ]))

@dp.callback_query(lambda c: c.data == "confirm_delete_account")
async def confirm_delete_account(callback: CallbackQuery, state: FSMContext):
    await callback.answer()
    data = await state.get_data()
    product_id = data.get('delete_account_id')
    user_id_str = str(callback.from_user.id)
    if product_id and product_id in user_purchases.get(user_id_str, []):
        user_purchases[user_id_str].remove(product_id)
        save_user_purchases()
        await callback.message.edit_text(T["account_deleted"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))
    else:
        await callback.message.edit_text("❌ Аккаунт не найден.", reply_markup=main_menu(callback.from_user.id))
    await state.clear()

# ==================== ОТРИМАННЯ КОДІВ ====================
def format_time_ago(date):
    now = datetime.now().replace(tzinfo=date.tzinfo) if date.tzinfo else datetime.now()
    diff = now - date
    seconds = int(diff.total_seconds())
    if seconds < 60:
        return f"{seconds} сек"
    elif seconds < 3600:
        minutes = seconds // 60
        secs = seconds % 60
        return f"{minutes} мин {secs} сек"
    else:
        hours = seconds // 3600
        minutes = (seconds % 3600) // 60
        return f"{hours} ч {minutes} мин"

@dp.callback_query(lambda c: c.data.startswith("get_codes_"))
async def get_codes_for_buyer(callback: types.CallbackQuery):
    await callback.answer()
    product_id = int(callback.data.split("_")[2])
    user_id_str = str(callback.from_user.id)
    if product_id not in user_purchases.get(user_id_str, []):
        lang = get_lang(callback.from_user.id)
        await bot.send_message(callback.from_user.id, "❌ Ви не купили цей акаунт. Спочатку придбайте його в каталозі.")
        return
    product = None
    for p in products:
        if p.get('id') == product_id and p.get('sold', False) and p.get('sold_to') == callback.from_user.id:
            product = p
            break
    if not product:
        await callback.answer("❌ Аккаунт не найден", show_alert=True)
        return
    session_string = product.get('credentials', '').strip()
    if not session_string or len(session_string) < 50:
        await callback.answer("❌ Нет session string", show_alert=True)
        return
    await callback.message.edit_text(T["getting_codes"][get_lang(callback.from_user.id)])
    try:
        api_id = 39002566
        api_hash = 'f1147f20f25c147b6bc51cd743e6791a'
        client = TelegramClient(StringSession(session_string), api_id, api_hash)
        await client.connect()
        if not await client.is_user_authorized():
            await callback.message.edit_text("❌ Аккаунт невалиден. Session string устарел.", reply_markup=product_kb(callback.from_user.id, product_id, False, True))
            return
        me = await client.get_me()
        dialogs = await client.get_dialogs()
        telegram_bot = None
        for dialog in dialogs:
            if dialog.name == "Telegram":
                telegram_bot = dialog
                break
        if not telegram_bot:
            await callback.message.edit_text("⚠️ Диалог с Telegram не найден.", reply_markup=product_kb(callback.from_user.id, product_id, False, True))
            await client.disconnect()
            return
        history = await client.get_messages(telegram_bot.entity, limit=100)
        codes_found = []
        for msg in history:
            text = msg.text or ""
            code_match = re.search(r'\b(\d{5,6})\b', text)
            if code_match and msg.date:
                codes_found.append({'code': code_match.group(1), 'date': msg.date})
        await client.disconnect()
        if not codes_found:
            await callback.message.edit_text(T["no_codes_found"][get_lang(callback.from_user.id)], reply_markup=product_kb(callback.from_user.id, product_id, True, True))
            return
        codes_found.sort(key=lambda x: x['date'], reverse=True)
        response = f"<b>📋 Последние коды для {me.first_name}</b>\n\n📱 {me.phone}\n\n<b>Найденные коды ({len(codes_found)}):</b>\n━━━━━━━━━━━━━━━━━━━━\n"
        for i, c in enumerate(codes_found[:15], 1):
            time_str = format_time_ago(c['date'])
            response += f"{i}. <code>{c['code']}</code> — {time_str}\n"
        response += f"\n💡 <b>Инструкция:</b>\n1. Установи Telegram\n2. Введи номер <code>{me.phone}</code>\n3. Когда спросит код — используй самый свежий код из списка выше\n4. Коды действуют ~2 минуты с момента отправки"
        await callback.message.edit_text(response, parse_mode="HTML", reply_markup=product_kb(callback.from_user.id, product_id, True, True))
    except Exception as e:
        await callback.message.edit_text(f"❌ Ошибка: {str(e)[:200]}", reply_markup=product_kb(callback.from_user.id, product_id, False, True))

# ==================== ОТРИМАННЯ ПО ПРОМОКОДУ ====================
@dp.callback_query(lambda c: c.data.startswith("use_promo_"))
async def use_promo_for_product(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    product_id = int(callback.data.split("_")[2])
    product = None
    for p in products:
        if p.get('id') == product_id and not p.get('sold', False):
            product = p
            break
    if not product:
        await callback.message.edit_text(T["product_sold_already"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))
        return
    await state.update_data(promo_product_id=product_id)
    await state.set_state(PromoState.waiting_use_code_for_product)
    await callback.message.edit_text(f"🎫 Введите промокод для {product.get('name')}:", reply_markup=user_back_kb(callback.from_user.id))

@dp.message(PromoState.waiting_use_code_for_product)
async def apply_promo_for_product(message: Message, state: FSMContext):
    code = message.text.strip().upper()
    data = await state.get_data()
    product_id = data.get('promo_product_id')
    lang = get_lang(message.from_user.id)
    if code not in promocodes:
        await message.answer(T["promo_invalid"][lang])
        await state.clear()
        return
    promo = promocodes[code]
    if promo.get('used', False):
        await message.answer(T["promo_already_used"][lang])
        await state.clear()
        return
    if promo.get('product_id') != product_id:
        await message.answer(T["promo_wrong_product"][lang])
        await state.clear()
        return
    product = None
    for p in products:
        if p.get('id') == product_id and not p.get('sold', False):
            product = p
            break
    if not product:
        await message.answer(T["product_sold_already"][lang])
        await state.clear()
        return
    product['sold'] = True
    product['sold_to'] = message.from_user.id
    product['sold_at'] = datetime.now().isoformat()
    promocodes[code]['used'] = True
    promocodes[code]['used_by'] = message.from_user.id
    promocodes[code]['used_at'] = datetime.now().isoformat()
    save_products(products)
    save_promocodes(promocodes)
    user_id_str = str(message.from_user.id)
    if user_id_str not in user_purchases:
        user_purchases[user_id_str] = []
    if product_id not in user_purchases[user_id_str]:
        user_purchases[user_id_str].append(product_id)
    save_user_purchases()
    cur.execute("INSERT INTO sales_history (product_id, buyer_id, amount, paid_at) VALUES (?,?,?,?)",
                (product_id, message.from_user.id, "0 (promo)", datetime.now().isoformat()))
    conn.commit()
    creds = product.get('credentials', '')
    text = T["free_product_received"][lang]
    text += f"📦 {product.get('name')}\n🌍 {product.get('country')}\n🔐 Session string:\n<code>{creds}</code>\n\n📲 Аккаунт добавлен в 'Мои аккаунты'"
    await state.clear()
    await message.answer(text, parse_mode="HTML", reply_markup=main_menu(message.from_user.id))
    try:
        await bot.send_message(ADMIN_ID, f"🎫 ПРОМОКОД АКТИВИРОВАН!\nКод: {code}\nТовар: {product.get('name')}\nЮзер: @{message.from_user.username or message.from_user.id}")
    except:
        pass

# ==================== ЗАГАЛЬНІ ХЕНДЛЕРИ ====================
@dp.callback_query(F.data == "support")
async def support(callback: types.CallbackQuery):
    lang = get_lang(callback.from_user.id)
    await callback.message.edit_text(T["report"][lang], reply_markup=user_back_kb(callback.from_user.id))
    await callback.answer()

@dp.callback_query(F.data == "policy")
async def show_policy(callback: types.CallbackQuery):
    lang = get_lang(callback.from_user.id)
    await callback.message.edit_text(T["policy"][lang], reply_markup=user_back_kb(callback.from_user.id))
    await callback.answer()

@dp.message(Command("policy"))
async def cmd_policy(message: Message):
    lang = get_lang(message.from_user.id)
    await message.answer(T["policy"][lang])

@dp.callback_query(F.data == "change_lang")
async def change_lang(callback: types.CallbackQuery):
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🇷🇺 Русский", callback_data="lang:ru")],
        [InlineKeyboardButton(text="🇺🇦 Українська", callback_data="lang:ua")],
        [InlineKeyboardButton(text="🇬🇧 English", callback_data="lang:en")],
        [InlineKeyboardButton(text="◀️ Назад", callback_data="user_back_to_menu")],
    ])
    await callback.message.edit_text("Выберите язык:", reply_markup=keyboard)
    await callback.answer()

@dp.callback_query(F.data.startswith("lang:"))
async def set_user_lang(callback: types.CallbackQuery):
    lang = callback.data.split(":", 1)[1]
    set_lang(callback.from_user.id, lang)
    await callback.message.edit_text(T["lang_saved"][lang], reply_markup=main_menu(callback.from_user.id))
    await callback.answer()

@dp.callback_query(F.data == "user_back_to_menu")
async def user_back_to_menu(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    await state.clear()
    await callback.message.edit_text(T["start"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))

@dp.callback_query(F.data == "back_to_user_menu")
async def back_to_user_menu_from_admin(callback: types.CallbackQuery, state: FSMContext):
    await callback.answer()
    await state.clear()
    await callback.message.edit_text(T["start"][get_lang(callback.from_user.id)], reply_markup=main_menu(callback.from_user.id))

@dp.message(Command("start"))
async def cmd_start(message: Message, state: FSMContext):
    user_id = message.from_user.id
    if not cur.execute("SELECT 1 FROM users WHERE user_id=?", (user_id,)).fetchone():
        set_lang(user_id, DEFAULT_LANG)
    await state.clear()
    if user_id == ADMIN_ID:
        await message.answer("👑 АДМИН ПАНЕЛЬ", reply_markup=admin_menu())
    else:
        await message.answer(T["start"][get_lang(user_id)], reply_markup=main_menu(user_id))

# -------------------- ЗАПУСК --------------------
async def main():
    log.info("🚀 Бот запущен")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())