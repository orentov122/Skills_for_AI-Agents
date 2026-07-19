# Code Style (Стиль кодирования)

## Описание
Полный стиль кодирования, оформления репозиториев и проектов. Применяй ВСЕГДА при создании или изменении кода, если не указано иное.

## Структура проекта
```
ProjectName/
├── main.py                  # Монолитный файл
├── .env.example             # Пример конфигурации (НИКОГДА .env)
├── requirements.txt         # Зависимости
├── Dockerfile               # Docker-образ
├── deploy_bot.sh            # Скрипт деплоя
├── README.md                # Документация
├── LICENSE                  # MIT License
├── langs/                   # Локализация (JSON)
├── data/                    # Данные
├── logs/                    # Логи
└── sessions/                # Сессии
```

## Python: Стиль кода

### Импорты
```python
import asyncio
import logging
import os
from typing import Any, Dict, List, Optional, Tuple

import aiofiles
import aiosqlite
from aiogram import Bot, Dispatcher, Router, F
```
Порядок: стандартная библиотека → третьи стороны → локальные.

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
- `*` для keyword-only аргументов

### Конфигурация
```python
class Config:
    BOT_TOKEN: str = os.getenv("BOT_TOKEN", "").strip()
    PANEL_BASE: str = os.getenv("PANEL_BASE", "").rstrip("/")
    ADMIN_USER_IDS: List[int] = env_int_list("ADMIN_USER_IDS")
    VERIFY_SSL: bool = str_to_bool(os.getenv("VERIFY_SSL", "true"))

    @classmethod
    def validate(cls) -> None:
        errors: List[str] = []
        if not cls.BOT_TOKEN:
            errors.append("BOT_TOKEN не установлен")
        if errors:
            raise ConfigError("Ошибка конфигурации:\n" + "\n".join(f"  • {e}" for e in errors))
```

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

def str_to_bool(val: str) -> bool:
    return str(val).strip().lower() in ("1", "true", "yes", "y", "on")
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
```

### Логирование
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
```
- `logger.info()` — ключевые операции с эмодзи (✅, 🗑, ⚠️)
- `logger.error()` — с `exc_info=True`
- `@log_error` декоратор на методах классов

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
                await asyncio.sleep(wait_time)
    if last_exception:
        raise last_exception
```

### База данных (SQLite)
```python
class Database:
    def __init__(self, db_path: str):
        self.db_path: str = db_path
        self.conn: Optional[aiosqlite.Connection] = None
        self.lock: asyncio.Lock = asyncio.Lock()

    async def connect(self) -> None:
        self.conn = await retry_async(
            lambda: aiosqlite.connect(self.db_path, timeout=30.0),
            max_retries=3, delay=1.0,
        )
        self.conn.row_factory = aiosqlite.Row
        await self.conn.execute("PRAGMA foreign_keys = ON")
        await self.conn.execute("PRAGMA journal_mode = WAL")
        await self.conn.execute("PRAGMA busy_timeout = 5000")

    @log_error
    async def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        async with self.lock:
            cur = await self.conn.execute(
                "SELECT * FROM users WHERE user_id = ?", (user_id,)
            )
            row = await cur.fetchone()
            return dict(row) if row else None
```

### Обработчики aiogram
```python
router.message.middleware(ban_middleware)
router.callback_query.middleware(language_middleware)

@router.message(Command("start"))
@router.callback_query(F.data == "start")
@log_error
async def cmd_start(
    event: Union[Message, CallbackQuery], state: FSMContext, **kwargs: Any
) -> None:
    user = getattr(event, "from_user", None)
    if not user:
        return
    user_id: int = user.id
    ...
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

## Docker
```dockerfile
FROM python:3.13.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY .
CMD ["python", "main.py"]
```

## deploy_bot.sh
```bash
cd /root/bots/ProjectName/ || exit 1
docker build -t project_name .
docker rm -f project_name 2>/dev/null || true
docker run -d --name project_name --restart always -v "$(pwd)":/app project_name
echo "✅ Deployed! Logs: docker logs -f project_name"
```

## Git
- Коммиты на русском: `Добавить поддержку кастомных тарифов`
- `.env.example` — ВСЕГДА пример, НЕ `.env`
- `.gitignore`: `.env`, `__pycache__/`, `*.pyc`, `data/`, `logs/`, `sessions/`

## Язык
- Комментарии: русский
- Имена переменных/функций: английский (snake_case)
- Имена классов: английский (PascalCase)
- Пользовательский текст: русский (через translate)
- Ключи переводов: английский (`"texts.welcome"`)
- DEFAULT_LANGUAGE: `"ru"`

## Чек-лист
- [ ] Монолитная архитектура (один main.py)
- [ ] Строгая типизация (Optional[T], Union[A, B])
- [ ] Кастомные исключения (BotError + наследники)
- [ ] Логирование с форматом `%(asctime)s | %(levelname)-8s | %(message)s`
- [ ] `@log_error` на методах классов
- [ ] `asyncio.Lock` для БД
- [ ] Retry паттерн
- [ ] FSM StatesGroup
- [ ] Переводы через `translate(lang, "key")`
- [ ] README.md по шаблону с эмодзи
- [ ] Dockerfile с python:3.13.x-slim
- [ ] .env.example (не .env)
- [ ] LICENSE (MIT)
