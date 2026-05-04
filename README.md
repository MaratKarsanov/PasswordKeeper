# 🔐 PasswordVault

> Безопасное отказоустойчивое хранилище паролей на основе микросервисной архитектуры

[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4?logo=dotnet)](https://dotnet.microsoft.com/)
[![ASP.NET Core](https://img.shields.io/badge/ASP.NET_Core-8.0-512BD4?logo=dotnet)](https://dotnet.microsoft.com/apps/aspnet)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)](https://www.docker.com/)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-MassTransit-FF6600?logo=rabbitmq)](https://www.rabbitmq.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql)](https://www.postgresql.org/)

Реализует принципы **zero-knowledge** шифрования, **Database per Service**, асинхронной коммуникации через брокер сообщений и отказоустойчивости через паттерны Retry и Circuit Breaker.

---

## 📐 Архитектура

```
                    ┌─────────────────────────────┐
                    │         Client App           │
                    └──────────────┬──────────────┘
                                   │ HTTPS
                    ┌──────────────▼──────────────┐
                    │         API Gateway          │
                    │  (YARP, JWT-валидация,       │
                    │   Rate Limiting, Redis)      │
                    └──────┬───────┬───────┬───────┘
                           │       │       │
              REST/HTTP    │       │       │   REST/HTTP
        ┌──────────────────▼─┐   ┌─▼──────────────────┐
        │    Auth Service    │   │   Vault Service     │
        │      :5001         │   │      :5002          │
        └────────┬───────────┘   └─────────┬───────────┘
                 │                         │
                 │      RabbitMQ (events)  │
                 └────────────┬────────────┘
                              │
                   ┌──────────▼──────────┐
                   │    Audit Service    │
                   │       :5003         │
                   └─────────────────────┘
```

**Правило:** сервисы **не обращаются друг к другу напрямую** за данными — только через события в очереди.

---

## 🧩 Микросервисы

### 🔀 API Gateway (YARP)
Единая точка входа для всех клиентских запросов.

- Маршрутизация через **YARP** (Yet Another Reverse Proxy от Microsoft)
- Валидация **JWT**-токенов до проксирования
- Проверка **Redis blacklist** (отозванные токены)
- **Rate Limiting** — защита от брутфорса (5 попыток входа/мин)
- Трансформ: пробрасывает `X-User-Id` заголовок в downstream-сервисы

### 🔐 Auth Service `:5001`
Аутентификация и управление токенами.

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| `POST` | `/auth/register` | Регистрация пользователя |
| `POST` | `/auth/login` | Вход, выдача токенов |
| `POST` | `/auth/refresh` | Обновление Access-токена |
| `POST` | `/auth/logout` | Отзыв токена (Redis blacklist) |

**Безопасность:** мастер-пароль хэшируется через **Argon2id**. Access + Refresh токены на основе **JWT**.

### 🗄️ Vault Service `:5002`
CRUD-операции над зашифрованными записями паролей.

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| `GET` | `/vault/entries` | Список записей (без расшифровки) |
| `POST` | `/vault/entries` | Создать запись |
| `PUT` | `/vault/entries/{id}` | Обновить запись |
| `DELETE` | `/vault/entries/{id}` | Удалить запись |
| `GET` | `/vault/entries/{id}/decrypt` | Расшифровать запись (требует мастер-пароль) |

**Zero-knowledge:** сервер никогда не хранит открытые пароли. Шифрование **AES-256-GCM**, ключ выводится из мастер-пароля через **PBKDF2**.

### 📋 Audit Service `:5003`
Журналирование событий и детектирование аномалий.

- Запись всех значимых событий (вход, смена пароля, доступ к записи)
- Детектирование подозрительной активности (серия неудачных входов)
- Предоставление журнала пользователю
- Потребляет события из **RabbitMQ** через **MassTransit**

---

## 📨 События в очереди (RabbitMQ)

```csharp
// Auth Service публикует:
UserLoggedIn      { UserId, IP, Timestamp }
UserLoginFailed   { Email, IP, Timestamp }
UserRegistered    { UserId, Timestamp }

// Vault Service публикует:
EntryAccessed     { UserId, EntryId, Timestamp }
EntryCreated      { UserId, EntryId, Timestamp }
EntryDeleted      { UserId, EntryId, Timestamp }
```

---

---

## 🛠️ Технологии

| Категория | Технология |
|-----------|-----------|
| Framework | ASP.NET Core 8 |
| API Gateway | YARP 2.2 |
| Аутентификация | JWT Bearer, Argon2id |
| Шифрование | AES-256-GCM, PBKDF2 |
| Брокер сообщений | RabbitMQ + MassTransit |
| Кэш / Blacklist | Redis |
| База данных | PostgreSQL (отдельная на каждый сервис) |
| Отказоустойчивость | Polly (Retry, Circuit Breaker) |
| Контейнеризация | Docker, Docker Compose |
| Health Checks | ASP.NET Core Health Checks |

---

## 🚀 Запуск

### Требования
- [Docker](https://www.docker.com/) и Docker Compose
- [.NET 8 SDK](https://dotnet.microsoft.com/download) (для локальной разработки)

### Быстрый старт

```bash
# Клонировать репозиторий
git clone https://github.com/<your-username>/PasswordVault.git
cd PasswordVault

# Запустить все сервисы
docker-compose up --build
```

После запуска доступны:

| Сервис | URL |
|--------|-----|
| API Gateway | `http://localhost:8080` |
| RabbitMQ Management | `http://localhost:15672` |
| Auth Service (direct) | `http://localhost:5001` |
| Vault Service (direct) | `http://localhost:5002` |
| Audit Service (direct) | `http://localhost:5003` |

---

## 📁 Структура проекта

```
PasswordVault/
├── src/
│   ├── ApiGateway/
│   │   ├── Middleware/
│   │   │   ├── JwtValidationMiddleware.cs
│   │   │   └── RateLimitingMiddleware.cs
│   │   ├── Program.cs
│   │   └── appsettings.json
│   ├── AuthService/
│   │   ├── Controllers/
│   │   ├── Services/
│   │   ├── Models/
│   │   └── Program.cs
│   ├── VaultService/
│   │   ├── Controllers/
│   │   ├── Services/
│   │   ├── Models/
│   │   └── Program.cs
│   └── AuditService/
│       ├── Consumers/       ← MassTransit consumers
│       ├── Models/
│       └── Program.cs
├── docker-compose.yml
└── PasswordVault.sln
```

---

## 🔒 Ключевые принципы безопасности

- **Zero-knowledge** — сервер хранит только зашифрованные данные, расшифровка возможна только на стороне клиента при наличии мастер-пароля
- **Database per Service** — полная изоляция данных между сервисами
- **Token Blacklist** — мгновенный отзыв токенов через Redis
- **Argon2id** — устойчивый к GPU-атакам алгоритм хэширования паролей
- **Rate Limiting** — защита от брутфорса на уровне шлюза
- **Audit Trail** — полный журнал всех операций

---

## 📄 Лицензия

MIT — см. [LICENSE](LICENSE)
