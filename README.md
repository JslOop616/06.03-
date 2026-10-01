# FakeBank

Учебный проект: симулятор интернет-банка на Flask.
**Не предназначен для реального использования.**

Демонстрирует полный стек веб-разработки на Python: аутентификация с 2FA, денежные операции с лимитами, роли, аудит, REST API.

## Возможности

### Пользователь
- Регистрация и вход (Argon2id для пароля и PIN)
- Двухфакторная аутентификация (TOTP) + резервные коды
- Переводы между счетами, пополнения, снятия
- Лимиты: на одну операцию, дневной, часовой
- Виртуальные карты (выпуск, заморозка, удаление)
- Кредиты с аннуитетным платежом
- Вклады со ставкой, зависящей от срока
- Платежи: мобильная связь, ЖКХ, интернет, штрафы
- История операций с фильтрами
- Уведомления
- Смена пароля, PIN, отключение 2FA, удаление аккаунта

### Администратор
- Панель со статистикой
- Управление пользователями: блокировка, выдача прав, корректировка баланса
- Разблокировка аккаунтов после неудачных попыток входа
- Просмотр транзакций, аудита, попыток входа

### API
- JWT-аутентификация
- `/api/v1/me` — информация о пользователе
- `/api/v1/transactions` — последние операции

## Безопасность

| Угроза | Мера защиты |
|---|---|
| Утечка пароля | Argon2id (`time_cost=3`, `memory_cost=64 MiB`) |
| Брутфорс | Rate-limit + lockout аккаунта после N неудач |
| XSS | Jinja autoescape + CSP с nonce |
| CSRF | Flask-WTF CSRFProtect |
| SQL-инъекции | SQLAlchemy ORM, параметризованные запросы |
| IDOR | Проверка владельца в сервисах |
| Утечка сессии | Fingerprint + инвалидация при смене пароля |
| NaN/Infinity в суммах | `_d()` с `is_finite()` |
| Обход лимитов | Проверка внутри сервисов, а не только в роутах |
| Каскадное удаление | `ON DELETE CASCADE` + `PRAGMA foreign_keys=ON` |
| 2FA-брутфорс | TTL pending-сессии (5 мин) + лимит попыток |
| Honeypot | Скрытые поля в формах регистрации |
| WAF | Упрощённая проверка подозрительных payload'ов |

## Стек

- Python 3.10+
- Flask 3.x
- Flask-SQLAlchemy, Flask-Login, Flask-WTF, Flask-Limiter, Flask-Mail, Flask-Talisman, Flask-Migrate
- SQLite (по умолчанию), PostgreSQL (через `DATABASE_URL`)
- Alembic для миграций
- Argon2id, PyJWT, pyotp, qrcode, bleach
- pytest для тестов

## Быстрый старт

### 1. Клонирование и окружение

```bash
git clone <url>
cd fakebank
python -m venv .venv

# Linux/Mac
source .venv/bin/activate

# Windows (PowerShell)
.\.venv\Scripts\Activate.ps1

# Windows (cmd)
.venv\Scripts\activate.bat
```

### 2. Зависимости

```bash
pip install -r requirements.txt
```

### 3. Переменные окружения

```bash
cp .env.example .env
```

**Обязательно заполни** в `.env` — сгенерируй значения:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

Запусти 4 раза, вставь в:
- `SECRET_KEY`
- `JWT_SECRET`
- `CSRF_SECRET`
- `ENCRYPTION_KEY`

### 4. Миграции БД

```bash
# Linux/Mac
export FLASK_APP=app.py
export FLASK_ENV=development

# Windows PowerShell
$env:FLASK_APP="app.py"
$env:FLASK_ENV="development"

flask db upgrade
```

### 5. Запуск

```bash
python app.py
```

Открой http://127.0.0.1:5018

### 6. Первый админ

Первый зарегистрированный пользователь автоматически становится админом.
Или вручную:

```bash
python make_admin.py your@email.com
```

## Тесты

```bash
pytest
```

Тесты покрывают: аутентификацию, переводы, лимиты, IDOR, PIN, 2FA, инвалидацию сессий.

## Переменные окружения

| Переменная | Обязательна | По умолчанию | Описание |
|---|---|---|---|
| `FLASK_ENV` | да | `production` | `development` — dev-дефолты секретов; `production` — падать при отсутствии |
| `SECRET_KEY` | в prod | — | Ключ подписи сессий |
| `JWT_SECRET` | в prod | — | Ключ подписи JWT |
| `CSRF_SECRET` | в prod | — | Ключ CSRF-токенов |
| `ENCRYPTION_KEY` | в prod | — | Ключ Fernet-шифрования |
| `DATABASE_URL` | нет | `sqlite:///fakebank.db` | Строка подключения |
| `REDIS_URL` | нет | — | Для rate-limit в multi-worker |
| `FORCE_HTTPS` | нет | `0` | `1` — редирект на HTTPS + Secure-cookie |
| `MAIL_*` | нет | — | Настройки SMTP |
| `MAIL_SUPPRESS_SEND` | нет | `1` | `1` — не отправлять письма |

## Структура проекта

```
fakebank/
├── app.py                  # create_app(), регистрация blueprint'ов
├── config.py               # Конфигурация, проверка ENV
├── constants.py            # Бизнес-константы
├── extensions.py           # Singleton-объекты Flask
├── models.py               # SQLAlchemy-модели
├── forms.py                # WTForms
├── validators.py           # Валидация паролей, телефонов, сумм
├── security.py             # JWT, fingerprint, шифрование
├── audit.py                # Журнал действий
├── waf.py                  # Упрощённый WAF
├── honeypot.py             # Honeypot-проверки
├── totp_utils.py           # TOTP + backup-коды
├── email_utils.py          # Отправка писем
├── decorators.py           # @admin_required и др.
├── make_admin.py           # CLI: выдать права админа
├── routes/                 # Blueprint'ы (auth, dashboard, transfer, ...)
├── services/               # Бизнес-логика (banking, cards, loans, ...)
├── templates/              # Jinja2-шаблоны
├── static/                 # CSS, JS
├── migrations/             # Alembic
├── tests/                  # pytest
└── docs/
    └── architecture.md     # Архитектурная схема
```

## Лицензия

MIT


## Известные ограничения

- **Race condition в лимитах.** На SQLite нет row-level lock. Два параллельных
  перевода могут превысить дневной лимит. В продакшене решается через
  PostgreSQL + `SELECT ... FOR UPDATE` или Redis-based rate-limit.
- **Race condition при регистрации первого админа.** Аналогично — на SQLite
  нет строгой изоляции. На PostgreSQL работает корректно.
- **Rate-limit в памяти.** Для multi-worker нужен Redis (`REDIS_URL`).
- **JWT не отзывается при logout.** Действует до истечения TTL.
