---
name: telegram-bot-production
description: Навык создания продакшн-готовых Telegram-ботов на aiogram с интеграцией внешних сервисов, Docker-деплоем, мониторингом и автоматизацией. Использовать при разработке ботов для коммерческих проектов.
license: MIT
metadata:
  author: koda
  version: "1.0.0"
---

# telegram-bot-production — Продакшн Telegram-боты

## Описание

Навык для создания Telegram-ботов, готовых к продакшну: с интеграцией внешних API, Docker-деплоем, логированием, мониторингом и автоматизацией рутинных задач.

## Когда активировать

- Разработка Telegram-бота для коммерческого проекта
- Интеграция бота с внешними сервисами (API, SSH, базы данных)
- Настройка деплоя, мониторинга или обновления бота
- Нужна автоматизация SSL, бэкапов, статистики

## Философия

> Бот в продакшне — это не код, а система: надёжность, отказоустойчивость, автоматизация.

### Принципы продакшн-бота

1. **Монолит для простоты.** Один `main.py` — проще деплоить, отлаживать, понимать.
2. **Асинхронность везде.** Все внешние вызовы — async/await, никаких блокировок.
3. **Обработка ошибок на каждом уровне.** Каждый внешний вызов обёрнут в try/except с логированием.
4. **Конфигурация через .env.** Никаких хардкодов, все секреты в переменных окружения.
5. **Логирование = мониторинг.** Без логов нет продакшна.

## Обязательный цикл

### Фаза 1: Архитектура проекта

```
ProjectBot/
├── main.py                  # Весь код бота
├── requirements.txt         # Зависимости
├── Dockerfile               # Docker-образ
├── deploy_bot.sh            # Скрипт деплоя
├── .env.example             # Пример конфигурации (НИКОГДА .env)
├── .gitignore               # Игнорировать .env, __pycache__, data/, logs/
├── LICENSE                  # Лицензия
├── README.md                # Документация
├── langs/                   # Локализация (JSON)
│   ├── ru.json
│   └── en.json
├── data/                    # Данные (БД, конфиги)
├── logs/                    # Логи (по дням)
└── sessions/                # Сессии (если telethon)
```

### Фаза 2: Интеграция с внешними сервисами

#### HTTP-клиент (aiohttp)

```python
import aiohttp

class APIClient:
    def __init__(self, base_url: str, token: str):
        self.base_url = str(base_url).rstrip("/")
        self.headers = {"Authorization": f"Bearer {token}"}
        self.session: Optional[aiohttp.ClientSession] = None

    async def connect(self) -> None:
        self.session = aiohttp.ClientSession(
            base_url=self.base_url,
            headers=self.headers,
            timeout=aiohttp.ClientTimeout(total=30),
        )

    async def close(self) -> None:
        if self.session:
            await self.session.close()

    @log_error
    async def get(self, path: str) -> Optional[Dict[str, Any]]:
        if not self.session:
            return None
        async with self.session.get(path) as resp:
            if resp.status == 200:
                return await resp.json()
            logger.warning(f"API GET {path} → {resp.status}")
            return None

    @log_error
    async def post(self, path: str, data: Optional[Dict[str, Any]] = None) -> Optional[Dict[str, Any]]:
        if not self.session:
            return None
        async with self.session.post(path, json=data) as resp:
            if resp.status in (200, 201):
                return await resp.json()
            logger.warning(f"API POST {path} → {resp.status}")
            return None
```

#### SSH-клиент (paramiko) для серверных операций

```python
import paramiko

class SSHClient:
    def __init__(self, host: str, port: int, username: str, key_path: str):
        self.host = host
        self.port = port
        self.username = username
        self.key_path = key_path

    @log_error
    async def execute(self, command: str) -> Optional[str]:
        def _exec() -> str:
            client = paramiko.SSHClient()
            client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            client.connect(
                hostname=self.host,
                port=self.port,
                username=self.username,
                key_filename=self.key_path,
                timeout=10,
            )
            stdin, stdout, stderr = client.exec_command(command)
            result = stdout.read().decode().strip()
            client.close()
            return result
        return await asyncio.get_event_loop().run_in_executor(None, _exec)
```

#### Автоматизация SSL через SSH

```python
async def update_ssl_cert(ssh: SSHClient, domain: str) -> bool:
    commands = [
        f"certbot renew --cert-name {domain} --quiet",
        f"systemctl reload nginx",
    ]
    for cmd in commands:
        result = await ssh.execute(cmd)
        if result is None:
            logger.error(f"SSL update failed: {cmd}")
            return False
    logger.info(f"SSL cert updated for {domain}")
    return True
```

### Фаза 3: База данных (aiosqlite)

```python
import aiosqlite

class Database:
    def __init__(self, db_path: str):
        self.db_path: str = db_path
        self.conn: Optional[aiosqlite.Connection] = None
        self.lock: asyncio.Lock = asyncio.Lock()

    async def connect(self) -> None:
        self.conn = await retry_async(
            lambda: aiosqlite.connect(self.db_path, timeout=30.0),
            max_retries=3,
            delay=1.0,
        )
        self.conn.row_factory = aiosqlite.Row
        await self.conn.execute("PRAGMA foreign_keys = ON")
        await self.conn.execute("PRAGMA journal_mode = WAL")
        await self.conn.execute("PRAGMA busy_timeout = 5000")
        await self.init_db()

    async def init_db(self) -> None:
        async with self.lock:
            await self.conn.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    user_id INTEGER PRIMARY KEY,
                    username TEXT,
                    language TEXT DEFAULT 'ru',
                    trust_score INTEGER DEFAULT 0,
                    created_at TEXT DEFAULT CURRENT_TIMESTAMP
                )
            """)
            await self.conn.commit()

    @log_error
    async def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        if not self.conn:
            return None
        async with self.lock:
            cur = await self.conn.execute(
                "SELECT * FROM users WHERE user_id = ?", (user_id,)
            )
            row = await cur.fetchone()
            return dict(row) if row else None

    @log_error
    async def upsert_user(self, user_id: int, **kwargs: Any) -> None:
        if not self.conn:
            return
        async with self.lock:
            existing = await self.get_user(user_id)
            if existing:
                sets = ", ".join(f"{k} = ?" for k in kwargs)
                vals = list(kwargs.values()) + [user_id]
                await self.conn.execute(f"UPDATE users SET {sets} WHERE user_id = ?", vals)
            else:
                kwargs["user_id"] = user_id
                cols = ", ".join(kwargs.keys())
                placeholders = ", ".join("?" * len(kwargs))
                await self.conn.execute(
                    f"INSERT INTO users ({cols}) VALUES ({placeholders})",
                    list(kwargs.values()),
                )
            await self.conn.commit()
```

### Фаза 4: Логирование и мониторинг

```python
import logging
import sys
from datetime import datetime
from pathlib import Path

BASE_DIR: Path = Path(__file__).parent
LOGS_DIR: Path = BASE_DIR / "logs"
LOGS_DIR.mkdir(exist_ok=True)
LOG_FILE: Path = LOGS_DIR / f"bot_{datetime.now().strftime('%Y-%m-%d')}.log"

logger = logging.getLogger("bot")
logger.setLevel(logging.DEBUG)
if not logger.handlers:
    formatter = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )
    console_handler = logging.StreamHandler(sys.stderr)
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)
    file_handler = logging.FileHandler(LOG_FILE, encoding="utf-8", mode="a")
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)
```

#### Декоратор для ошибок

```python
from functools import wraps
from typing import Callable, Any

def log_error(func: Callable) -> Callable:
    @wraps(func)
    async def wrapper(*args: Any, **kwargs: Any) -> Any:
        try:
            return await func(*args, **kwargs)
        except Exception as e:
            logger.error(f"Ошибка в {func.__name__}: {e}", exc_info=True)
            raise
    return wrapper
```

#### Retry-паттерн

```python
async def retry_async(
    func: Callable,
    max_retries: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: Tuple[type, ...] = (Exception,),
) -> Any:
    last_exception: Optional[Exception] = None
    for attempt in range(max_retries):
        try:
            return await func()
        except exceptions as e:
            last_exception = e
            if attempt < max_retries - 1:
                wait_time = delay * (backoff ** attempt)
                logger.warning(
                    f"Повтор {attempt + 1}/{max_retries} "
                    f"через {wait_time:.1f}с: {type(e).__name__}: {e}"
                )
                await asyncio.sleep(wait_time)
            else:
                logger.error(
                    f"Все {max_retries} попыток исчерпаны: {type(e).__name__}: {e}",
                    exc_info=True,
                )
    if last_exception:
        raise last_exception
```

### Фаза 5: Конфигурация

```python
import os
from typing import List

def env_int_list(key: str) -> List[int]:
    val = os.getenv(key, "")
    return [int(x.strip()) for x in val.split(",") if x.strip().isdigit()]

def str_to_bool(val: str) -> bool:
    return str(val).strip().lower() in ("1", "true", "yes", "y", "on")

class Config:
    BOT_TOKEN: str = os.getenv("BOT_TOKEN", "").strip()
    API_BASE: str = os.getenv("API_BASE", "").rstrip("/")
    API_TOKEN: str = os.getenv("API_TOKEN", "").strip()
    ADMIN_USER_IDS: List[int] = env_int_list("ADMIN_USER_IDS")
    VERIFY_SSL: bool = str_to_bool(os.getenv("VERIFY_SSL", "true"))
    SSH_HOST: str = os.getenv("SSH_HOST", "").strip()
    SSH_PORT: int = int(os.getenv("SSH_PORT", "22"))
    SSH_USER: str = os.getenv("SSH_USER", "").strip()
    SSH_KEY: str = os.getenv("SSH_KEY", "").strip()

    @classmethod
    def validate(cls) -> None:
        errors: List[str] = []
        if not cls.BOT_TOKEN:
            errors.append("BOT_TOKEN не установлен")
        if not cls.API_BASE:
            errors.append("API_BASE не установлен")
        if errors:
            raise ConfigError("Ошибка конфигурации:\n" + "\n".join(f"  • {e}" for e in errors))
```

### Фаза 6: Обработчики aiogram

```python
from aiogram import Bot, Dispatcher, Router, F
from aiogram.types import Message, CallbackQuery
from aiogram.fsm.context import FSMContext
from aiogram.filters import Command

router = Router()

# Обработчик ошибок Telegram
@router.errors()
async def error_handler(event: Any, *args: Any, **kwargs: Any) -> bool:
    error = kwargs.get("error")
    if not error and args:
        error = args[0]
    if not error:
        return False
    if isinstance(error, TelegramBadRequest):
        error_msg = str(error).lower()
        if "blocked" in error_msg or "message not modified" in error_msg:
            return True
        logger.warning(f"TelegramBadRequest: {error}")
        return True
    logger.error(f"Необработанная ошибка: {type(error).__name__}: {error}", exc_info=True)
    return False

# Универсальный обработчик команды /start
@router.message(Command("start"))
@router.callback_query(F.data == "start")
@log_error
async def cmd_start(
    event: Union[Message, CallbackQuery], state: FSMContext, **kwargs: Any
) -> None:
    user = getattr(event, "from_user", None)
    if not user:
        logger.warning("cmd_start: событие без from_user")
        return
    user_id: int = user.id
    lang = (user.language_code or DEFAULT_LANGUAGE).strip().lower()

    # Регистрация/обновление пользователя
    await db.upsert_user(
        user_id,
        username=user.username or "",
        language=lang,
    )

    text = translate(lang, "welcome", name=user.first_name)
    markup = kb([
        [{"text": translate(lang, "btn_menu"), "callback_data": "menu"}],
        [{"text": translate(lang, "btn_help"), "callback_data": "help"}],
    ])
    await smart_answer(event, text, reply_markup=markup)
```

### Фаза 7: Утилиты для клавиатур и переводов

```python
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton

def kb(rows: List[List[Dict[str, str]]]) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(**btn) for btn in row] for row in rows
        ]
    )

async def smart_answer(
    event: Union[Message, CallbackQuery],
    text: str,
    reply_markup: Optional[InlineKeyboardMarkup] = None,
    delete_origin: bool = False,
) -> bool:
    try:
        if isinstance(event, Message):
            await event.answer(text, reply_markup=reply_markup)
        elif isinstance(event, CallbackQuery):
            if event.message:
                await event.message.answer(text, reply_markup=reply_markup)
                if delete_origin:
                    try:
                        await event.message.delete()
                    except Exception:
                        pass
            try:
                await event.answer()
            except Exception:
                pass
        return True
    except Exception as e:
        logger.error(f"smart_answer error: {e}")
        return False

# Загрузка переводов
_LANG_CACHE: Dict[str, Any] = {}
LANGUAGES: Dict[str, Any] = {}

def load_languages(langs_dir: Path) -> None:
    for lang_file in langs_dir.glob("*.json"):
        lang_code = lang_file.stem
        with open(lang_file, "r", encoding="utf-8") as f:
            LANGUAGES[lang_code] = json.load(f)
    logger.info(f"Загружены языки: {list(LANGUAGES.keys())}")

def _resolve_key(data: Any, key: str) -> Optional[str]:
    parts = key.split(".")
    current = data
    for part in parts:
        if isinstance(current, dict) and part in current:
            current = current[part]
        else:
            return None
    return str(current) if current is not None else None

def translate(language_code: str, key: str, **kwargs: Any) -> str:
    lang = (language_code or DEFAULT_LANGUAGE).strip().lower()
    data = _LANG_CACHE.get(lang) or LANGUAGES.get(DEFAULT_LANGUAGE, {})
    text = _resolve_key(data, key)
    if text is None and lang != DEFAULT_LANGUAGE:
        data = LANGUAGES.get(DEFAULT_LANGUAGE, {})
        text = _resolve_key(data, key)
    if text is None:
        return key
    if kwargs and isinstance(text, str):
        try:
            return text.format(**kwargs)
        except Exception:
            return text
    return str(text)
```

### Фаза 8: Docker и деплой

#### Dockerfile

```dockerfile
FROM python:3.13.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "main.py"]
```

#### deploy_bot.sh

```bash
#!/bin/bash
cd /root/bots/ProjectName/ || exit 1

docker build -t project_name .
docker rm -f project_name 2>/dev/null || true
docker run -d --name project_name --restart always -v "$(pwd)":/app project_name

echo "✅ Deployed! Logs: docker logs -f project_name"
```

### Фаза 9: Lifecycle бота

```python
async def main() -> None:
    # Валидация конфигурации
    Config.validate()

    # Инициализация компонентов
    load_languages(BASE_DIR / "langs")
    await db.connect()
    await api.connect()

    # Создание бота и диспетчера
    bot = Bot(token=Config.BOT_TOKEN)
    dp = Dispatcher()
    dp.include_router(router)

    # Регистраção middleware
    dp.message.middleware(ban_middleware)
    dp.callback_query.middleware(language_middleware)

    # Запуск
    logger.info("=== Бот запущен ===")
    try:
        await dp.start_polling(bot)
    finally:
        await api.close()
        if db.conn:
            await db.conn.close()
        logger.info("=== Бот остановлен ===")

if __name__ == "__main__":
    asyncio.run(main())
```

## Чек-лист

- [ ] Один `main.py` — монолитная архитектура
- [ ] Все внешние вызовы обёрнуты в `@log_error`
- [ ] `asyncio.Lock` для потокобезопасности БД
- [ ] Retry-паттерн для внешних API и БД
- [ ] Конфигурация через `.env` с валидацией при старте
- [ ] Логирование в файл + консоль с форматом `%(asctime)s | %(levelname)-8s | %(message)s`
- [ ] Обработка `TelegramBadRequest` (blocked, message not modified)
- [ ] `smart_answer` для безопасной отправки сообщений
- [ ] `translate(lang, "key")` с fallback на дефолтный язык
- [ ] Dockerfile с `python:3.13.x-slim` и кэшированием слоёв
- [ ] `deploy_bot.sh` с `|| exit 1` и `2>/dev/null || true`
- [ ] `.env.example` (никогда `.env` в коммитах)
- [ ] Graceful shutdown: закрытие сессий API и БД

## Scope

- Применять ко всем Telegram-ботам, которые идут в продакшн
- Для прототипов и экспериментов — использовать здравый смысл
- SSH-интеграция опциональна — только если нужен доступ к серверу
- Мультиязычность опциональна — можно начать с одного языка
