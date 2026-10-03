# Async Alembic: noldan oxirigacha

**Stek:** FastAPI + SQLAlchemy (async) + asyncpg + PostgreSQL + Alembic

## Mundarija

1. [Kutubxonalarni o'rnatish](#1-qadam-kutubxonalarni-ornatish)
2. [Loyiha tuzilmasi](#2-qadam-loyiha-tuzilmasi)
3. [database.py](#3-qadam-appdatabasepy)
4. [Alembic'ni async shablon bilan yaratish](#4-qadam-alembicni-async-shablon-bilan-yaratish)
5. [env.py](#5-qadam-alembicenvpy)
6. [Birinchi migratsiya](#6-qadam-birinchi-migratsiya)
7. [Kundalik ish tartibi](#7-qadam-kundalik-ish-tartibi)
8. [Foydali buyruqlar](#foydali-buyruqlar)
9. [Tez-tez uchraydigan xatolar](#tez-tez-uchraydigan-xatolar)

---

## 1-qadam. Kutubxonalarni o'rnatish

```bash
pip install "sqlalchemy[asyncio]" asyncpg alembic
```

## 2-qadam. Loyiha tuzilmasi

```
FastAPI_plus_SQLAlchemy_backend/
├── alembic/              # alembic init'dan keyin paydo bo'ladi
├── alembic.ini
├── app/
│   ├── config.py         # DATABASE_URL shu yerda
│   ├── database.py       # engine, Base, session
│   └── models.py         # barcha modellar
└── main.py
```

## 3-qadam. `app/database.py`

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/dbname"

engine = create_async_engine(DATABASE_URL, echo=True)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass
```

> **Muhim:** URL `postgresql+asyncpg://` bilan boshlanishi shart.

> **Muhim:** `Base.metadata.create_all()` ni **chaqirmang**. Jadvallarni faqat Alembic yaratadi.

## 4-qadam. Alembic'ni async shablon bilan yaratish

```bash
alembic init -t async alembic
```

Asosiy joyi `-t async`: shunda `env.py` avtomatik async uchun tayyor bo'ladi.

## 5-qadam. `alembic/env.py`

Faylni quyidagicha qiling (import yo'llarini o'zingizniki bilan almashtiring):

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

from app.database import Base, DATABASE_URL
import app.models  # noqa: F401  (modellar import qilinishi SHART)

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

config.set_main_option("sqlalchemy.url", DATABASE_URL)
target_metadata = Base.metadata


def run_migrations_offline() -> None:
    context.configure(
        url=config.get_main_option("sqlalchemy.url"),
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,  # ustun turi o'zgarishini ham aniqlaydi
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

> Parolingizda `%` belgisi bo'lsa, `DATABASE_URL.replace("%", "%%")` ishlating, aks holda `configparser` xato beradi.

## 6-qadam. Birinchi migratsiya

```bash
alembic revision --autogenerate -m "initial"
alembic upgrade head
```

- Birinchi buyruq migratsiya faylini yaratadi.
- Ikkinchisi uni bazaga qo'llaydi va `alembic_version` jadvalini o'zi yaratadi.

## 7-qadam. Kundalik ish tartibi

Har safar model o'zgarganda (masalan, `phone` ustuni qo'shdingiz):

1. `models.py` ni o'zgartiring
2. `alembic revision --autogenerate -m "add phone field"`
3. `alembic/versions/` ichidagi yangi faylni ko'zdan kechiring
4. `alembic upgrade head`

> 3-qadamni o'tkazib yubormang. Autogenerate ba'zan noto'g'ri narsa yozadi (masalan, ustun nomini o'zgartirishni "o'chirish + qo'shish" deb tushunadi).

---

## Foydali buyruqlar

| Buyruq | Vazifasi |
|---|---|
| `alembic current` | Bazadagi hozirgi versiya |
| `alembic heads` | Fayllardagi oxirgi versiya |
| `alembic history` | Barcha migratsiyalar ro'yxati |
| `alembic downgrade -1` | Bitta qadam orqaga qaytarish |
| `alembic stamp head` | Migratsiyani bajarmasdan bazani head deb belgilash |

## Tez-tez uchraydigan xatolar

| Xato | Sababi va yechimi |
|---|---|
| `MissingGreenlet` | `env.py` sinxron, URL esa asyncpg. Yechim: 4-5-qadamlar (`-t async`). |
| `Target database is not up to date` | Avval `alembic upgrade head` (jadvallar allaqachon bo'lsa `alembic stamp head`), keyin `revision`. |
| Bo'sh migratsiya (`pass`) | `env.py`'da `import app.models` yo'q yoki `target_metadata` noto'g'ri. |
| Yangi jadval ko'rinmayapti | Model `app.models`'da import qilinmagan. Modellar bir nechta faylda bo'lsa, hammasini import qiling. |
