# Архитектура FakeBank

## Общая схема

```
┌──────────────────────────────────────────────────────┐
│                     HTTP (Flask)                     │
│  Blueprints: auth, dashboard, transfer, cards,       │
│  loans, deposits, payments, history, profile,        │
│  admin, api, errors                                  │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│           Формы (Flask-WTF) + валидация              │
│         CSRF, honeypot, sanitize_text                │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│              Сервисы (services/*)                    │
│  banking, cards, loans, deposits, payments           │
│  • бизнес-логика                                     │
│  • проверка лимитов (single / daily / hourly)        │
│  • Decimal через _d() с is_finite()                  │
│  • запись Transaction + Notification                 │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│           Модели (SQLAlchemy) + Alembic              │
│  User, Transaction, Card, Loan, Deposit,             │
│  Notification, AuditLog, LoginAttempt                │
│  • ondelete="CASCADE" / "SET NULL"                   │
│  • naming_convention для FK/PK/индексов              │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│              SQLite / PostgreSQL                     │
│         PRAGMA foreign_keys=ON (SQLite)              │
└──────────────────────────────────────────────────────┘
```

## Слои

### HTTP-слой (`routes/`)
Только:
1. Парсинг формы через WTForms
2. Проверка PIN (для чувствительных операций)
3. Вызов сервиса
4. `db.session.commit()` / `rollback()`
5. `flash()` + редирект

Роут **не должен** содержать бизнес-логику, расчёты, проверки лимитов.

### Сервисный слой (`services/`)
Вся бизнес-логика:
- нормализация сумм через `_d()`
- проверка лимитов (`check_transfer_limits`, `check_daily_limit`)
- проверка владельца объекта (IDOR)
- изменение балансов, создание транзакций, уведомлений
- выброс `ValueError` с понятным сообщением

### Слой моделей (`models.py`)
- Декларативные классы SQLAlchemy
- Каскады на уровне ORM (`cascade="all, delete-orphan"`) **и** БД (`ondelete=...`)
- Свойства: `is_locked`, `is_active`, `check_password`, `check_pin`

## Кросс-слойные модули

| Модуль | Назначение |
|---|---|
| `extensions.py` | Singleton'ы Flask + PRAGMA FK + naming_convention |
| `constants.py` | Бизнес-константы (лимиты, ставки) |
| `config.py` | Конфигурация + `_required()` для секретов |
| `security.py` | JWT, fingerprint, Fernet-шифрование, IP |
| `audit.py` | Асинхронно-совместимый журнал действий |
| `waf.py` | Простейший WAF на `before_request` |
| `honeypot.py` | Honeypot-поля и тайминг-проверка |
| `validators.py` | Валидация пароля, телефона, сумм, username |
| `totp_utils.py` | TOTP + backup-коды (SHA-256) |
| `email_utils.py` | Отправка писем через Flask-Mail |
| `decorators.py` | `@admin_required` |

## Поток данных: перевод

```
POST /transfer/send
  │
  ├─ TransferForm.validate_on_submit()       # типы, диапазоны, regex
  ├─ _check_pin_or_flash(pin)                # Argon2 verify
  │
  ▼
services.banking.transfer(sender, account, name, amount, comment)
  │
  ├─ _d(amount)                              # Decimal + is_finite
  ├─ sender.balance >= amount
  ├─ recipient_account != sender.account_number
  ├─ check_transfer_limits(sender, amount)
  │    ├─ amount <= TRANSFER_MAX_SINGLE
  │    ├─ daily_sum(sender) + amount <= TRANSFER_DAILY_LIMIT
  │    └─ hourly_count(sender) < TRANSFER_HOURLY_COUNT
  ├─ if recipient exists:
  │    ├─ recipient.is_active_flag
  │    └─ recipient.currency == sender.currency
  ├─ sender.balance -= amount
  ├─ recipient.balance += amount             # если внутренний
  ├─ db.session.add(Transaction(kind="transfer", ...))
  └─ notify(...)
  │
  ▼
db.session.commit()
  │
  ▼
audit("transfer_done", "amount=... to=...")
```

## Обработка ошибок

- `ValueError` из сервисов → `flash(str(e), "danger")` + `rollback()`
- HTTP-ошибки → `routes/errors.py`:
  - 400 — некорректный запрос (WAF)
  - 401 — не авторизован
  - 403 — доступ запрещён (honeypot, admin_required)
  - 404 — не найдено
  - 413 — payload слишком большой
  - 423 — аккаунт заблокирован
  - 429 — rate-limit
  - 500 — внутренняя ошибка

## Безопасность: контрольные точки

| Угроза | Мера | Где |
|---|---|---|
| SQL-инъекции | ORM + параметризация | `models.py`, `services/*` |
| XSS | Jinja autoescape + CSP nonce | `templates/*`, `app.py` |
| CSRF | Flask-WTF | `extensions.py` |
| Брутфорс пароля | Limiter + lockout | `routes/auth.py`, `models.py` |
| Брутфорс 2FA | TTL + лимит попыток | `routes/auth.py` |
| IDOR | Проверка владельца | `services/*` |
| Утечка сессии | Fingerprint + pwd_iat | `app.py` |
| NaN/Infinity | `_d()` → `is_finite()` | `services/banking.py` |
| Обход лимитов | Проверка в сервисах | `services/*` |
| Каскадное удаление | FK ON DELETE + PRAGMA | `models.py`, `extensions.py` |
| Слабые секреты | `_required()` в prod | `config.py` |
| Мусорные суммы | `_parse_delta()` | `routes/admin.py` |
| Мусорные query-параметры | `_safe_days()` | `routes/history.py` |

## Жизненный цикл сессии

1. **POST /login** → `LoginForm.validate_on_submit()`
2. Проверка пароля (`Argon2.verify`)
3. Если `totp_enabled`:
   - `session["pending_2fa_uid"]`, `["pending_2fa_at"]`, `["pending_2fa_fp"]`
   - редирект на `/2fa`
4. После успешного 2FA: `_finish_login(u)`
   - `session.clear()`
   - `login_user(u)`
   - `session["fp"] = fingerprint_request()`
   - `session["pwd_iat"] = int(u.password_changed_at.timestamp())`
5. **На каждом запросе** (`before_request`):
   - Проверка `pwd_iat` — если пароль сменили в другой сессии, выкинуть
6. **dashboard.index**: мягкая проверка fingerprint (только для UX)

## Миграции

- `flask db init` — создать `migrations/`
- `flask db migrate -m "msg"` — автогенерировать миграцию
- `flask db upgrade` — применить
- `flask db downgrade` — откатить

`render_as_batch=True` в `migrate.init_app()` — для корректной работы `ALTER TABLE` в SQLite.

## Известные ограничения

- Rate-limit в памяти — для multi-worker нужен Redis (`REDIS_URL`)
- WAF упрощённый, не production-grade
- JWT не отзывается при logout (только по TTL)
- Нет шифрования чувствительных полей в БД
- `MAIL_SUPPRESS_SEND=1` по умолчанию — письма не уходят
