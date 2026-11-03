# Каталог микросервисов

Вариант 1112, сервис объявлений. Формат описания взят из задания, к нему добавлено поле `Data Owner`, потому что задание требует назвать владельца данных.

Общие правила для всех сервисов:

- Данные сервиса меняет только он сам. Другие сервисы получают их через API или события, общих БД нет.
- События публикуются через Transactional Outbox, потребители идемпотентны (дедупликация по `eventId`).
- Каждый сервис сам проверяет JWT по открытым ключам User Service (JWKS, кэш в сервисе) и сам проверяет права. Внутренние REST-вызовы идут с токеном вызывающего сервиса ([README, раздел 4](README.md#4-границы-сервисов-ответственность-и-владение-данными)).
- Поиск, карточка объявления и список категорий доступны без токена.
- Все сервисы развёрнуты в Kubernetes, минимум 2 пода, разнесённых по зонам; для Search API минимум 3 пода, по одному в каждой зоне. Число подов меняет HPA по метрике из поля `Scalability` ([раздел 8 отчёта](README.md#8-независимое-развёртывание-и-масштабирование)).
- Внешний трафик проходит через единый API Gateway. Это инфраструктурный компонент без бизнес-логики и своих данных, поэтому в каталог он не входит.
- Команды: **Marketplace** (Listing, Search), **Engagement** (Chat, Favorites, Notification), **Trust & Platform** (User, Report, а также шлюз, Kafka и мониторинг).

## User Service

```text
Service: User Service

Responsibility:
Регистрация и вход, профиль пользователя, выдача JWT и публикация ключей для их проверки,
блокировка и удаление пользователей.

Owned Data:
User
Credential (хеш пароля)
RefreshToken
Хранилище: PostgreSQL

Data Owner:
Команда Trust & Platform

API:
POST /users
POST /auth/login
POST /auth/refresh
GET /users/me
PATCH /users/me
DELETE /users/me
GET /users/{id}  (публичный профиль: имя, дата регистрации)
GET /.well-known/jwks.json

Dependencies:
Нет синхронных зависимостей от других сервисов

Communication:
REST + Kafka
Публикует: user.events (UserRegistered, UserContactChanged, UserBlocked, UserUnblocked, UserDeleted)
Потребляет: report.events (решение о блокировке)

Scalability:
Горизонтальное масштабирование по CPU, потому что хеширование паролей требует вычислений.
Экземпляры не хранят состояния. Публичные профили кэшируются.
При сбое пользователи с действующим токеном продолжают работать до его истечения (15 минут).
```

## Listing Service

```text
Service: Listing Service

Responsibility:
Создание и редактирование объявлений, фотографии, жизненный цикл объявления
(DRAFT, ACTIVE, SOLD, EXPIRED, ARCHIVED, HIDDEN), дерево категорий и схема атрибутов категории.

Owned Data:
Listing
ListingPhoto (метаданные; сами файлы лежат в объектном хранилище)
Category
CategoryAttribute (схема атрибутов категории)
Проекция заблокированных продавцов (копия из user.events, источник истины остаётся в User)
Хранилище: PostgreSQL с read-репликами, объектное хранилище

Data Owner:
Команда Marketplace

API:
POST /listings
GET /listings/{id}
PATCH /listings/{id}
POST /listings/{id}/publish
POST /listings/{id}/sold
DELETE /listings/{id}  (архивация)
GET /listings?sellerId={id}
POST /listings/{id}/photos/upload-url
GET /categories
GET /categories/{id}
PUT /categories/{id}  (администратор)
GET /internal/listings/export?cursor={c}  (выгрузка для перестроения индекса)

Dependencies:
Объектное хранилище: ссылки для загрузки фото (через ACL)
Синхронных зависимостей от других сервисов нет

Communication:
REST + Kafka
Публикует: listing.events, category.events
Потребляет: user.events (UserBlocked, UserUnblocked, UserDeleted), report.events (ReportResolved)

Scalability:
Горизонтальное масштабирование по CPU и RPS, чтение идёт в реплики.
Карточка объявления и категории кэшируются на CDN и в экземплярах (категории надолго, карточка на секунды).
Запись редка: около 135 RPS в пик, из них около 95 приходится на ссылки для загрузки фото. Задача истечения срока объявлений (30 дней) работает отдельным воркером.
```

## Search Service

```text
Service: Search Service

Responsibility:
Полнотекстовый поиск по объявлениям, фильтры по категории, цене и городу, сортировка, подсказки.
Построение и обновление поискового индекса.

Owned Data:
SearchDocument (плоская денормализованная проекция объявления, источник истины остаётся в Listing)
Хранилище: OpenSearch

Data Owner:
Команда Marketplace

API:
GET /search?q={text}&categoryId={id}&priceFrom={n}&priceTo={n}&city={c}&sort={s}&cursor={c}
GET /search/suggest?q={text}

Dependencies:
Синхронных зависимостей нет.
Listing Service: только выгрузка для перестроения индекса, в обычной работе не используется

Communication:
REST + Kafka
Потребляет: listing.events (в том числе ListingPriceChanged и ListingArchived), category.events

Scalability:
Три независимо масштабируемых части:
Search API масштабируется по RPS и p99, минимум 3 пода.
Search Indexer масштабируется по отставанию consumer group.
OpenSearch масштабируется добавлением реплик индекса, потому что индекс небольшой (около 10 ГБ на реплику)
и целиком помещается на узел, а нагрузка растёт по чтению.
Популярные запросы кэшируются на несколько секунд.
```

## Favorites Service

```text
Service: Favorites Service

Responsibility:
Добавление объявлений в избранное и удаление, список избранного пользователя,
отслеживание изменений цены и статуса избранных объявлений.

Owned Data:
Favorite (userId, listingId, дата)
ListingSnapshot (id, название, цена, превью, статус; копия из listing.events только для избранных объявлений)
Хранилище: PostgreSQL, партиции по hash(userId)

Data Owner:
Команда Engagement

API:
POST /favorites  (тело: listingId)
DELETE /favorites/{listingId}
GET /favorites?cursor={c}
GET /favorites/ids  (идентификаторы для отметки сердечка в выдаче)

Dependencies:
Listing Service: снимок объявления при добавлении.
Если он недоступен, избранное сохраняется со статусом SNAPSHOT_PENDING, снимок догружается повтором

Communication:
REST + Kafka
Публикует: favorites.events (FavoritedListingPriceChanged, FavoritedListingSold)
Потребляет: listing.events, user.events (UserDeleted)

Scalability:
Горизонтальное масштабирование по RPS. Данные разбиты по userId, все запросы идут с ключом пользователя.
Лимит 1 000 избранных на пользователя защищает от раздувания.
```

## Chat Service

```text
Service: Chat Service

Responsibility:
Диалоги между покупателем и продавцом по объявлению, отправка и хранение сообщений,
доставка в реальном времени, статус прочтения.

Owned Data:
Chat (покупатель, продавец, снимок объявления: название и превью)
Message
ChatParticipantState (последнее прочитанное сообщение)
Проекция заблокированных пользователей (копия из user.events)
Хранилище: PostgreSQL, партиции по hash(chatId), Redis pub/sub для доставки между подами

Data Owner:
Команда Engagement

API:
POST /chats  (тело: listingId)
GET /chats
GET /chats/{id}/messages?before={messageId}
POST /chats/{id}/messages  (идемпотентный, по clientMessageId)
POST /chats/{id}/read
WS /chat/ws
GET /internal/chats/{id}/messages/{messageId}  (для Report Service)

Dependencies:
Listing Service: данные объявления и продавец при создании чата.
Если он недоступен, новые чаты не создаются (503), существующие диалоги работают

Communication:
REST + WebSocket + Kafka
Публикует: chat.events (MessageSent без текста сообщения)
Потребляет: user.events (UserBlocked, UserUnblocked, UserDeleted)

Scalability:
Горизонтальное масштабирование по числу WebSocket-соединений.
Сообщение уходит на под с соединением получателя через Redis pub/sub. Доставка не гарантируется,
поэтому клиент после переподключения запрашивает историю по последнему полученному messageId.
Хранилище растёт примерно на 330 ГБ в год сырых данных (около 2 ТБ на диске с индексами и репликами).
```

## Report Service

```text
Service: Report Service

Responsibility:
Приём жалоб на объявления, пользователей и сообщения, фиксация доказательств,
очередь модератора, решения по жалобам.

Owned Data:
Report
ReportEvidence (снимок объекта жалобы на момент подачи)
ModerationDecision
Хранилище: PostgreSQL

Data Owner:
Команда Trust & Platform

API:
POST /reports  (targetType: LISTING, USER или MESSAGE; targetId; reason)
GET /reports/{id}
GET /moderation/reports?status={s}  (роль MODERATOR)
POST /moderation/reports/{id}/take
POST /moderation/reports/{id}/resolve  (решение: DISMISS, HIDE_LISTING, BLOCK_USER)

Dependencies:
Listing Service: снимок объявления.
Chat Service: сообщение как доказательство.
Если владелец недоступен, жалоба принимается в статусе PENDING_EVIDENCE, доказательство догружается повтором

Communication:
REST + Kafka
Публикует: report.events (ReportResolved)
Потребляет: ничего

Scalability:
Нагрузка низкая (около 50 RPS), обычно хватает минимума: 2 пода.
Решение привязано к версии объявления, поэтому правка после жалобы не отменяет решение.
```

## Notification Service

```text
Service: Notification Service

Responsibility:
Формирование уведомлений из доменных событий, доставка по каналам (push, e-mail, SMS),
история уведомлений, настройки получателей.

Owned Data:
Notification
NotificationPreference
DeviceToken
RecipientContact (минимальная копия контактов из user.events: e-mail, телефон, язык; удаляется по UserDeleted)
Хранилище: PostgreSQL

Data Owner:
Команда Engagement

API:
GET /notifications?cursor={c}
POST /notifications/{id}/read
GET /notifications/preferences
PUT /notifications/preferences
POST /devices  (регистрация push-токена)
DELETE /devices/{id}

Dependencies:
Провайдеры e-mail, SMS и push (через ACL).
Синхронных зависимостей от других сервисов нет

Communication:
Kafka + REST
Потребляет: user.events, listing.events, favorites.events, chat.events, report.events

Scalability:
Горизонтальное масштабирование по отставанию consumer group.
При сбое уведомления задерживаются, но не теряются: непрочитанные события остаются в Kafka.
Дедупликация по eventId и ограничение частоты push по получателю защищают от повторов и спама.
```
