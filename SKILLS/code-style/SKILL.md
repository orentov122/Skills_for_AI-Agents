---
name: code-style
description: Полный стиль кодирования пользователя — Python, Telegram-боты, монолитная архитектура, строгая типизация, логирование, README, Docker.
license: MIT
metadata:
  author: Major-Woolfi
  version: "1.0.0"
---

# Code Style — Стиль кодирования пользователя

## Описание

Этот навык описывает полный стиль кодирования, оформления репозиториев и проектов. Применяй ВСЕГДА при создании или изменении кода — для любых проектов и языков, если не указано иное.

## 📁 Структура проекта

```
ProjectName/
├── main.py                  # Монолитный файл (весь код в одном файле)
├── .env.example             # Пример конфигурации (НИКОГДА .env)
├── requirements.txt         # Зависимости Python
├── Dockerfile               # Docker-образ
├── deploy_bot.sh            # Скрипт деплоя на сервер
├── README.md                # Подробная документация
├── LICENSE                  # MIT License
├── langs/                   # Локализация (JSON)
├── data/                    # Данные (базы, конфиги)
├── logs/                    # Логи (по дням)
└── sessions/                # Сессии (если telethon)
```

**Ключевой принцип:** монолитная архитектура — один `main.py` для простоты деплоя.

## 🐍 Python: Стиль кода

### Импорты

```python
import asyncio
import html
import json
import logging
import os
import secrets
import sys
import time
from datetime import datetime, timedelta
from pathlib import Path
from typing import Any, Dict, List, Optional, Set, Tuple, Union

import aiofiles
import aiosqlite
from aiogram import Bot, Dispatcher, Router, F
from dotenv import load_dotenv
```

Порядок: стандартная библиотека → третьи стороны → локальные. Всегда импортируй конкретные типы из `typing`.

### Типизация — СТРОГАЯ

```python
def format_traffic(traffic_gb: Any, lang: str = DEFAULT_LANGUAGE) -> str:
    ...

async def create_subscription(
    user_id: int,
    plan: Dict[str, Any],
    *,
    extra_days: int = 0,
    days_override: Optional[int] = None,
    earn_trust: bool = True,
) -> Optional[str]:
    ...
```

- `Optional[T]` — НЕТ `T | None`
- `Union[A, B]` — НЕТ `A | B`
- `Dict[str, Any]` для общих словарей
- `List[Dict[str, Any]]` для списков
- `Union[Message, CallbackQuery]` для событий
- `*` для keyword-only аргументов
- `**kwargs: Any` в обработчиках

### Конфигурация

```python
class Config:
    BOT_TOKEN: str = os.getenv("BOT_TOKEN", "").strip()
    PANEL_BASE: str = os.getenv("PANEL_BASE", "").rstrip("/")
    ADMIN_USER_IDS: List[int] = env_int_list("ADMIN_USER_IDS")
    VERIFY_SSL: bool = str_to_bool(os.getenv("VERIFY_SSL", "true"))
    DATA_DIR: str = resolve_local_path(os.getenv("DATA_DIR"), "data")
    REF_BONUS_DAYS: int = env_int("REF_BONUS_DAYS", 7)

    @classmethod
    def validate(cls) -> None:
        errors: List[str] = []
        if not cls.BOT_TOKEN:
            errors.append("BOT_TOKEN не установлен")
        if errors:
            raise ConfigError("Ошибка конфигурации:\n" + "\n".join(f"  • {e}" for e in errors))
```

- `.strip()` для строк, `.rstrip("/")` для URL
- `str_to_bool()`, `env_int()`, `env_float()`, `env_int_list()` для парсинга
- Класс `Config` с методом `validate()`

### Утилиты-конвертеры

```python
def to_int(value: Any, default: int = 0) -> int:
    try:
        if value is None:
            return default
        return int(value)
    except Exception:
        try:
            return int(float(value))
        except Exception:
            return default

def to_float(value: Any, default: float = 0.0) -> float:
    try:
        if value is None:
            return default
        return float(value)
    except Exception:
        return default

def str_to_bool(val: str) -> bool:
    return str(val).strip().lower() in ("1", "true", "yes", "y", "on")

def format_number(value: float) -> str:
    if float(value).is_integer():
        return str(int(value))
    return f"{value:.2f}".rstrip("0").rstrip(".")
```

### Кастомные исключения

```python
class BotError(Exception):
    def __init__(self, message: str, code: int = 500):
        self.message = message
        self.code = code
        super().__init__(self.message)

class ConfigError(BotError):
    def __init__(self, message: str):
        super().__init__(message, code=400)

class DatabaseError(BotError):
    def __init__(self, message: str, original_error: Optional[Exception] = None):
        self.original_error = original_error
        super().__init__(message, code=500)

class ValidationError(BotError):
    def __init__(self, message: str, field: Optional[str] = None):
        self.field = field
        super().__init__(message, code=400)
```

### Логирование — подробное

**Формат:** `%(asctime)s | %(levelname)-8s | %(message)s`

```python
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
logger.info("=== Логгер инициализирован ===")
```

**Правила:**

- `logger.info()` — ключевые операции (создание, удаление, команды)
- `logger.warning()` — ожидаемые нестандартные ситуации
- `logger.error()` — ошибки, исключения
- `logger.critical()` — критические ошибки
- `logger.debug()` — детальная отладка
- Эмодзи в info: `✅` успех, `🗑` удаление, `⚠️` предупреждение
- `exc_info=True` при логировании ошибок

**Примеры:**

```python
logger.info(f"✅ Подписка создана для user {user_id}")
logger.info(f"🗑 Клиент {user_id} удалён (reason={reason})")
logger.warning(f"⚠️ Не удалось удалить клиента {user_id}")
logger.error(f"Ошибка в {func.__name__}: {e}", exc_info=True)
```

### Декораторы логирования

```python
def log_error(func: Callable) -> Callable:
    async def wrapper(*args: Any, **kwargs: Any) -> Any:
        try:
            return await func(*args, **kwargs)
        except Exception as e:
            logger.error(f"Ошибка в {func.__name__}: {e}", exc_info=True)
            raise
    return wrapper

def log_warning(func: Callable) -> Callable:
    async def wrapper(*args: Any, **kwargs: Any) -> Any:
        try:
            return await func(*args, **kwargs)
        except Exception as e:
            logger.warning(f"Предупреждение в {func.__name__}: {e}")
            raise
    return wrapper
```

Применяй к методам классов (Database, PanelAPI, JSONStorage).

### Retry паттерн

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
                    f"Повторная попытка {attempt + 1}/{max_retries} "
                    f"через {wait_time:.1f}с: {type(e).__name__}: {e}"
                )
                await asyncio.sleep(wait_time)
            else:
                logger.error(
                    f"Все {max_retries} попытки исчерпаны: {type(e).__name__}: {e}",
                    exc_info=True,
                )
    if last_exception:
        raise last_exception
    raise BotError("Неизвестная ошибка в retry_async")
```

### База данных (SQLite)

```python
class Database:
    def __init__(self, db_path: str):
        self.db_path: str = db_path
        self.conn: Optional[aiosqlite.Connection] = None
        self.lock: asyncio.Lock = asyncio.Lock()

    async def connect(self) -> None:
        async def _connect() -> aiosqlite.Connection:
            conn = await aiosqlite.connect(self.db_path, timeout=30.0)
            return conn
        self.conn = await retry_async(
            _connect, max_retries=3, delay=1.0, exceptions=(aiosqlite.Error, Exception)
        )
        self.conn.row_factory = aiosqlite.Row
        await self.conn.execute("PRAGMA foreign_keys = ON")
        await self.conn.execute("PRAGMA journal_mode = WAL")
        await self.conn.execute("PRAGMA busy_timeout = 5000")
        await self.init_db()

    @log_error
    async def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        if not self.conn:
            return None
        try:
            async with self.lock:
                cur = await self.conn.execute(
                    "SELECT * FROM users WHERE user_id = ?", (user_id,)
                )
                row = await cur.fetchone()
                return dict(row) if row else None
        except Exception as e:
            logger.error(f"get_user {user_id}: {e}")
            return None
```

- `asyncio.Lock` для потокобезопасности
- `@log_error` на публичных методах
- `PRAGMA journal_mode = WAL`
- Retry для подключения

### Обработчики aiogram

```python
# Middleware
router.message.middleware(ban_middleware)
router.callback_query.middleware(language_middleware)

# Глобальный обработчик ошибок
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

# Команды
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
    ...
```

- `@router.message(Command(...))` + `@router.callback_query(F.data == "...")` на ОДНУ функцию
- `**kwargs: Any` в сигнатуре
- `Union[Message, CallbackQuery]` для универсальных хендлеров

### Клавиатуры и переводы

```python
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

# Переводы
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

### FSM состояния

```python
class CustomTariffState(StatesGroup):
    waiting_for_gb = State()
    waiting_for_ip = State()
    waiting_for_days = State()
    waiting_for_locations = State()
    waiting_for_confirm = State()

@router.message(CustomTariffState.waiting_for_gb)
async def process_custom_gb(event: Message, state: FSMContext, **kwargs):
    val = (event.text or "").strip()
    if val.lower() in ("отмена", "cancel", "/cancel"):
        await state.clear()
        await cmd_start(event, state)
        return
    ...
    await state.update_data(custom_traffic_gb=gb)
    await state.set_state(CustomTariffState.waiting_for_ip)
```

### Организация кода

```python
# === Настройка логирования ===
# === Константы ===
# === Утилиты ===
# === Кастомные исключения ===
# === Config ===
# === Класс Database ===
# === Класс PanelAPI ===
# === FSM Состояния ===
# === Глобальные объекты ===
# === Middleware ===
# === Обработчики ошибок ===
# === Обработчики команд ===
```

Секции: `# === Название секции ===`
Комментарии: на русском, только для неочевидной логики
Имена: на английском, snake_case для функций, PascalCase для классов

## 📝 README.md — Шаблон

````markdown
# ProjectName — Краткое описание

Одно-два предложения о проекте.

## 🎯 Цели проекта

- Цель 1
- Цель 2

## 📁 Архитектура

```plaintext
ProjectName/
├── main.py
├── .env.example
├── requirements.txt
├── Dockerfile
├── deploy_bot.sh
├── README.md
├── LICENSE
├── langs/
├── data/
└── logs/
```
````

## 📋 Быстрый старт

### Локальный запуск

```bash
pip install -r requirements.txt
cp .env.example .env
# Отредактируйте .env
python main.py
```

### Docker

```bash
docker build -t project_name .
docker run -d --name project_name --restart always -v "$(pwd)":/app project_name
```

### Деплой на сервер

```bash
cd ProjectName/
./deploy_bot.sh
```

## ✨ Возможности

### 👤 Для пользователей

- Возможность 1
- Возможность 2

### 👑 Для администраторов

- Возможность 1

### 🤖 Для системы

- Возможность 1

## 🛠️ Стек технологий

| Компонент | Технологии    |
| --------- | ------------- |
| Язык      | Python 3.13.x |
| ...       | ...           |

## ⚙️ Конфигурация

| Переменная | Описание | По умолчанию |
| ---------- | -------- | ------------ |
| BOT_TOKEN  | Токен    | -            |

## ⚠️ Ограничения

- Ограничение 1
- Ограничение 2

## 📞 Контакты

- **GitHub**: [@username](https://github.com/username)

## 📄 Лицензия

```
MIT License
Copyright (c) 2026 username
```

````

**Правила README:**
- Эмодзи-заголовки: 🎯 📁 📋 ✨ 🛠️ ⚙️ ⚠️ 📞 📄
- Таблицы для стека и конфигурации
- plaintext-дерево файлов
- Блоки кода для установки
- Секции для разных аудиторий (пользователи, админы, система)
- Контакты и лицензия в конце

## 🐳 Docker

```dockerfile
FROM python:3.13.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "main.py"]
````

- `python:3.13.x-slim` — конкретная версия
- `COPY requirements.txt` до `COPY . .` для кэширования слоёв
- `--no-cache-dir` для уменьшения образа
- `CMD ["python", "main.py"]`

## 🚀 deploy_bot.sh

```bash
cd /root/bots/ProjectName/ || exit 1

docker build -t project_name .
docker rm -f project_name 2>/dev/null || true
docker run -d --name project_name --restart always -v "$(pwd)":/app project_name

echo "✅ Deployed! Logs: docker logs -f project_name"
```

- `|| exit 1` если каталог не найден
- `2>/dev/null || true` чтобы не ругаться если контейнер не существует
- `--restart always` для автоперезапуска
- `echo "✅ Deployed! ..."` в конце

## 📋 Git и репозиторий

### Коммиты

- На русском языке
- Кратко и по делу
- Пример: `Добавить поддержку кастомных тарифов`
- Пример: `Исправить ошибку при удалении подписки`
- Пример: `Добавить логирование в PanelAPI`

### Файлы репозитория

- `.env.example` — ВСЕГДА пример, НЕ `.env`
- `LICENSE` — MIT License
- `README.md` — подробный, по шаблону выше
- `.gitignore` — `.env`, `__pycache__`, `*.pyc`, `data/`, `logs/`, `sessions/`

### .gitignore

```
.env
__pycache__/
*.pyc
data/
logs/
sessions/
```

## 🇷🇺 Язык

- Комментарии в коде: **русский**
- Имена переменных/функций: **английский** (snake_case)
- Имена классов: **английский** (PascalCase)
- Пользовательский текст: **русский** (через translate)
- Ключи переводов: **английский** (`"texts.welcome"`, `"buttons.buy"`)
- DEFAULT_LANGUAGE: `"ru"`

## ✅ Чек-лист стиля

- [ ] Монолитная архитектура (один main.py)
- [ ] Строгая типизация (Optional[T], Union[A, B], Dict[str, Any])
- [ ] Кастомные исключения (BotError + наследники)
- [ ] Логирование с форматом `%(asctime)s | %(levelname)-8s | %(message)s`
- [ ] Логи в папке `logs/` с daily-файлами
- [ ] INFO-уровень для ключевых операций с эмодзи (✅, 🗑, ⚠️)
- [ ] Decorator `@log_error` на методах классов
- [ ] `asyncio.Lock` для БД
- [ ] Retry паттерн для внешних вызовов
- [ ] FSM StatesGroup для многошаговых процессов
- [ ] Переводы через `translate(lang, "key")` с fallback на ru
- [ ] `kb(rows)` для клавиатур
- [ ] `**kwargs: Any` в обработчиках
- [ ] `Union[Message, CallbackQuery]` для хендлеров
- [ ] Секции кода: `# === Название ===`
- [ ] Комментарии на русском
- [ ] README.md по шаблону с эмодзи
- [ ] Dockerfile с python:3.13.x-slim
- [ ] deploy_bot.sh с || exit 1
- [ ] .env.example (не .env)
- [ ] LICENSE (MIT)
- [ ] Коммиты на русском
