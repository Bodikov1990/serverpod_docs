# Полная документация Serverpod 3.2.0

> Исчерпывающий обзор всех 61 страниц официальной документации Serverpod

---

## Содержание

1. [Что такое Serverpod](#1-что-такое-serverpod)
2. [Установка и создание проекта](#2-установка-и-создание-проекта)
3. [Endpoints](#3-endpoints-эндпоинты)
4. [Модели данных](#4-модели-данных)
5. [База данных (ORM)](#5-база-данных-orm)
6. [Сессии](#6-сессии-sessions)
7. [Конфигурация](#7-конфигурация)
8. [Кэширование](#8-кэширование)
9. [Логирование](#9-логирование)
10. [Стриминг](#10-стриминг)
11. [Server Events](#11-server-events)
12. [Аутентификация](#12-аутентификация)
13. [Загрузка файлов](#13-загрузка-файлов)
14. [Модули](#14-модули)
15. [Деплой](#15-деплой)
16. [Инструменты](#16-инструменты)
17. [Дополнительные возможности](#17-дополнительные-возможности)
18. [Обучение](#18-обучение)

---

## 1. Что такое Serverpod

Serverpod — это **open-source серверный фреймворк**, созданный специально для Flutter-разработчиков. Позволяет писать бэкенд на Dart, генерирует клиентский код автоматически. Текущая версия — **3.2.0**.

**Ключевые возможности:** автоматическая кодогенерация, ORM для PostgreSQL, система миграций БД, встроенное кэширование (локальное + Redis), аутентификация (Email, Google, Apple, Passkey), стриминг данных в реальном времени через WebSocket, загрузка файлов (S3, GCP, БД), планирование задач (future calls), встроенный веб-сервер Relic, логирование и мониторинг через Serverpod Insights, поддержка векторных полей (pgvector).

---

## 2. Установка и создание проекта

**Предварительные требования:** Flutter SDK, Docker (для PostgreSQL).

**Установка CLI:** \`dart pub global activate serverpod_cli\`

**Создание проекта:** \`serverpod create my_project\`

Создаёт три пакета: my_project_server, my_project_client, my_project_flutter.

**Запуск:**
- \`docker compose up -d\` — запуск PostgreSQL
- \`dart run bin/main.dart --apply-migrations\` — запуск сервера

Порты: **8080** (API), **8081** (Insights), **8082** (веб-сервер Relic).

**Serverpod Mini** — легковесная версия без БД: \`serverpod create myproject --mini\`. Переход на полную: \`serverpod create .\`

---

## 3. Endpoints (Эндпоинты)

Эндпоинты — точки входа для клиентских вызовов. Класс расширяет Endpoint, методы возвращают типизированный Future, первый параметр — Session.

\`\`\`dart
class ExampleEndpoint extends Endpoint {
  Future<String> hello(Session session, String name) async {
    return 'Hello $name';
  }
}
\`\`\`

На клиенте: \`var result = await client.example.hello('World');\`

**Допустимые типы:** bool, int, double, String, UuidValue, Duration, DateTime, ByteData, Uri, BigInt, сериализуемые модели, List, Map, Set, Record.

**Аннотации:**
- \`@doNotGenerate\` — исключает из кодогенерации
- \`requireLogin\` — требует аутентификации
- \`requiredScopes\` — ограничение по ролям
- \`@unauthenticatedClientCall\` — отключает аутентификацию
- maxRequestSize: 512 КБ по умолчанию

**Наследование:** расширение эндпоинтов, abstract endpoints, переопределение методов, getEndpointOfType<T>() на клиенте.

**Middleware:** перехват запросов на базе Relic — \`pod.server.addMiddleware(myMiddleware())\`

---

## 4. Модели данных

Определяются в .spy.yaml файлах в lib/ сервера.

\`\`\`yaml
class: Company
fields:
  name: String
  foundedDate: DateTime?
  employees: List<Employee>
\`\`\`

**Типы:** bool, int, double, String, Duration, DateTime, ByteData, UuidValue, Uri, BigInt, Vector, HalfVector, SparseVector, Bit, другие модели, List, Map, Set, Record.

**Возможности:**
- serverOnly: true — только серверный код
- scope=serverOnly — видимость полей
- immutable: true — неизменяемые классы
- extends — наследование
- sealed: true — sealed классы
- exception: — сериализуемые исключения
- enum: — перечисления (byName/byIndex)
- default, defaultModel, defaultPersist — значения по умолчанию
- Векторные поля: Vector(1536), HalfVector, SparseVector, Bit

---

## 5. База данных (ORM)

### Модели БД

\`\`\`yaml
class: Company
table: company
fields:
  name: String
\`\`\`

Добавляется id (int? по умолчанию, поддерживается UuidValue). !persist — исключает поле.

### CRUD

- **Create:** Company.db.insertRow(session, company) / Company.db.insert(session, list)
- **Read:** findById, findFirstRow, find с фильтрами
- **Update:** updateRow, update, updateById, updateWhere (с columns для частичного)
- **Delete:** deleteRow, delete, deleteWhere
- **Count:** Company.db.count(session, where: ...)

### Фильтрация

equals, notEquals, >, >=, <, <=, between, inSet, like, ilike. Логические: & (AND), | (OR), ~ (NOT). По связям: count(), none(), any(), every(). Векторные: distanceL2, distanceCosine, distanceInnerProduct, distanceL1, distanceHamming, distanceJaccard.

### Сортировка

orderBy, orderDescending, orderByList. По связям и count().

### Пагинация

limit + offset. Cursor-based через where + orderBy.

### Связи (Relations)

- **One-to-one:** relation keyword + unique index
- **One-to-many:** List<Model>?, relation
- **Many-to-many:** через junction-таблицу
- **Self-relations:** ссылки на свою таблицу
- **Bidirectional:** name= параметр
- **Referential actions:** onUpdate/onDelete: NoAction, Restrict, SetDefault, Cascade, SetNull
- **С модулями:** через bridge-таблицу

### Relation Queries (Joins)

include: для объектов, includeList: для списков, вложенные includes. Attach/Detach.

### Транзакции

session.db.transaction с isolation levels и savepoints.

### Миграции

\`serverpod create-migration\`, \`dart run bin/main.dart --apply-migrations\`. Repair-миграции, rollback, tagging. managedMigration: false для opt-out.

### Индексация

btree (default), hash, gist, spgist, gin, brin. Векторные: HNSW, IVFFLAT.

### Raw SQL

session.db.unsafeQuery с параметрами (named/positional).

### Runtime Parameters

Настройка PostgreSQL на уровне сервера и транзакций.

---

## 6. Сессии (Sessions)

Session — контекст запроса. Доступ к db, caches, storage, messages, passwords, authenticated.

Типы: MethodCallSession, WebCallSession, MethodStreamSession, StreamingSession, FutureCallSession, InternalSession (требует close()).

---

## 7. Конфигурация

Три уровня: YAML → Environment variables → Dart config object.

Файлы: config/development.yaml, passwords.yaml, generator.yaml.

Параметры: порты, БД, Redis, maxRequestSize, логирование, future calls.

Режимы: development, staging, production, test. Роли: monolith, serverless, maintenance.

---

## 8. Кэширование

Три кэша: session.caches.local, localPrio, global (Redis). Поддержка lifetime и CacheMissHandler.

---

## 9. Логирование

session.log() с уровнями. Хранение в БД: serverpod_log, serverpod_query_log, serverpod_session_log. Настройка: persistentEnabled, consoleEnabled.

---

## 10. Стриминг

Streaming Methods — методы с Stream типом. Общий WebSocket, типизированные стримы, обработка ошибок.

\`\`\`dart
Stream echoStream(Session session, Stream stream) async* {
  await for (var message in stream) { yield message; }
}
\`\`\`

---

## 11. Server Events

session.messages.postMessage('channel', msg) — локально и глобально (Redis). Получение: createStream, addListener.

---

## 12. Аутентификация

Модуль serverpod_auth_idp: Email, Google, Apple, Passkey. JWT и Server-Side Sessions. FlutterAuthSessionManager на клиенте. requireLogin, requiredScopes, custom Scope. Custom auth через authenticationHandler.

---

## 13. Загрузка файлов

Хранилища: БД, GCP (serverpod_cloud_storage_gcp), S3 (serverpod_cloud_storage_s3). session.storage для создания upload description и верификации.

---

## 14. Модули

Пакеты с серверным, клиентским и Flutter-кодом. Добавление через pubspec.yaml + generator.yaml. Создание: serverpod create --template module. Ссылки: module:auth:AuthUser.

---

## 15. Деплой

**Стратегии:** Server cluster (все функции) vs Serverless (минимальная стоимость).

**Платформы:** Serverpod Cloud (beta), GCE (Terraform), Cloud Run, AWS EC2 (Terraform), VPS (community), любой Docker хост.

**Роли:** monolith, serverless, maintenance.

---

## 16. Инструменты

- **Serverpod Insights** — companion app (Mac, Windows)
- **VS Code Extension** — подсветка YAML
- **LSP Server** — serverpod language-server
- **Run Scripts** — скрипты в pubspec.yaml

---

## 17. Дополнительные возможности

- **Health Checks** — автомониторинг + кастомные метрики
- **Custom Serialization** — Freezed, ProtocolSerialization
- **Security** — TLS/SSL через SecurityContextConfig
- **Backward Compatibility** — правила совместимости API
- **Experimental** — column override, diagnostic events, shutdown tasks

---

## 18. Обучение

- **Serverpod Academy** — 4-часовой курс
- **YouTube** — туториалы
- **Getting Started** — пошаговый гайд
- **GitHub Discussions** — поддержка
- **Discord** — сообщество

---

## Обновление до 3.0

- Dart SDK 3.8.0+, Flutter 3.32.0+
- Relic framework для веб-сервера
- session.authenticated синхронный
- Enum сериализация по умолчанию byName
- SerializableEntity → SerializableModel
- Graceful SIGTERM shutdown

---

*Источник: [docs.serverpod.dev](https://docs.serverpod.dev/) — версия 3.2.0*
