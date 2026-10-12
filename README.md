# PushPilot REST API

[Спецификация OpenAPI 3.0.3](openapi.yaml) · [Сайт](https://pushpilot.ru) · [SDK](https://github.com/pushplatform/sdks)

Публичный REST API для регистрации устройств и push-подписок, отправки уведомлений,
чтения статусов, истории и аналитики. Контракт сверен с зарегистрированными маршрутами,
обработчиками и DTO сервера. Подробности и границы реализации — в [docs/contract.md](docs/contract.md),
источник сверки — в [docs/verification.md](docs/verification.md).

`sent` означает **принятие запроса push-провайдером**, а не получение уведомления
устройством. `202 Accepted` означает принятие запроса в очередь PushPilot.

## Авторизация, приложения и окружения

Получите адрес своего инстанса, `application_id` и ключ проекта у администратора
вашей организации. Приложение (`ios`, `android` или `web`), его окружения и credentials
провайдера должны быть настроены заранее. Их создание и управление ключами относятся
к административным операциям и не входят в этот публичный контракт. Публичного CRUD
окружений нет.

Для server API передавайте `Authorization: Bearer <API_KEY>` и для JSON-запросов
`Content-Type: application/json`. Ключ имеет формат `sk_test_` или `sk_live_` + 32
строчных hex-символа. Храните секретные ключи на своём backend. Scopes проверяются
сервером; это не OAuth и не отдельные HTTP-заголовки.

| Scope | Операции |
| :--- | :--- |
| `devices:write` | Регистрация установки/токена, login/logout, legacy PATCH, удаление подписки; backend Web Browser API |
| `devices:read` **или** `devices:write` | Чтение установки с подписками |
| `messages:send` | Отправка push |
| `messages:read` | Статус, история сообщений и отправок, аналитика |

Организация и проект определяются ключом. В регистрации передавайте
`environment: development` или `production`; окружение токена совпадает с установкой.
У ключа может быть ограничение `environment_id`. История и статусы сообщений в этом
случае фильтруются по точному ID окружения. Префикс `sk_test_`/`sk_live_` сам по себе
не заменяет ограничение ключа. Детали проверок окружения и исключения описаны в контракте.

В браузере используйте [Web Browser API](docs/web-browser.md): публичный `wbpub_…`
для новой установки, затем её секретный `wbinst_…` и разрешённый `Origin`.

## Быстрый старт: Android / FCM

Команды выполняются из корня этого репозитория. `api.example.invalid` — намеренно
нерабочий placeholder, не production URL. Замените его адресом своего инстанса
без завершающего `/`, задайте настоящий ключ и отредактируйте JSON-файлы примеров.
Все UUID в файлах вымышлены.

```sh
export BASE_URL='https://api.example.invalid'
export API_KEY='REPLACE_WITH_YOUR_PROJECT_API_KEY'
```

1. В [examples/installation.json](examples/installation.json) замените `application_id`
   на ID Android-приложения. `device_id` — стабильный ID установки в вашей системе.
   Зарегистрируйте устройство:

```sh
curl --fail-with-body -sS -X POST "$BASE_URL/v1/installations" \
  -H "Authorization: Bearer $API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @examples/installation.json
```

Ответ `201` содержит `id` установки и массив `subscriptions`. Сохраните `id`:

```sh
export INSTALLATION_ID='REPLACE_WITH_ID_FROM_INSTALLATION_RESPONSE'
```

2. В [examples/token.json](examples/token.json) замените `token` настоящим FCM-токеном,
   полученным на устройстве. Зарегистрируйте его в том же окружении:

```sh
curl --fail-with-body -sS -X POST "$BASE_URL/v1/installations/$INSTALLATION_ID/tokens" \
  -H "Authorization: Bearer $API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @examples/token.json
```

Ответ `201` содержит `subscription_id`, маскированный `token_prefix` и `status`.
Полный токен не возвращается. Повтор регистрации установки или того же токена
также возвращает `201`.

```sh
export SUBSCRIPTION_ID='REPLACE_WITH_ID_FROM_TOKEN_RESPONSE'
```

3. В [examples/message.json](examples/message.json) замените `application_id`,
   `recipients[0].installation_id` и `recipients[0].subscription_id` полученными ID.
   Для новой логической отправки задавайте новый `idempotency_key`.
   Формат тела:

```json
{
  "application_id": "11111111-1111-4111-8111-111111111111",
  "recipients": [{
    "installation_id": "22222222-2222-4222-8222-222222222222",
    "subscription_id": "33333333-3333-4333-8333-333333333333"
  }],
  "provider": "fcm",
  "platform": "android",
  "priority": "normal",
  "notification_type": "alert",
  "ttl_seconds": 3600,
  "idempotency_key": "example-order-001-notification-v1",
  "payload": {
    "title": "PushPilot",
    "body": "Пример уведомления",
    "custom_data": {"order_id": "example-order-001"}
  }
}
```

```sh
curl --fail-with-body -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @examples/message.json
```

Ответ `202` содержит `message_id`, `delivery_ids`, `status: pending`, `enqueued_at`,
`expires_at` и фактический `ttl_seconds`. Сохраните `message_id`; `delivery_ids`
соответствуют отдельным получателям. Отправка асинхронная, результат проверяется отдельно:

```sh
export MESSAGE_ID='REPLACE_WITH_ID_FROM_MESSAGE_RESPONSE'

curl --fail-with-body -sS "$BASE_URL/v1/messages/$MESSAGE_ID" \
  -H "Authorization: Bearer $API_KEY"

curl --fail-with-body -sS "$BASE_URL/v1/installations/$INSTALLATION_ID" \
  -H "Authorization: Bearer $API_KEY"
```

При необходимости привяжите пользователя из вашей системы через
[examples/login.json](examples/login.json), затем снимите связь при выходе:

```sh
curl --fail-with-body -sS -X POST "$BASE_URL/v1/installations/$INSTALLATION_ID/login" \
  -H "Authorization: Bearer $API_KEY" \
  -H 'Content-Type: application/json' --data-binary @examples/login.json

curl --fail-with-body -sS -X POST "$BASE_URL/v1/installations/$INSTALLATION_ID/logout" \
  -H "Authorization: Bearer $API_KEY"
```

Оба запроса возвращают `204` без тела. Logout сохраняет токены; для прекращения
push на конкретную подписку выполните отписку:

```sh
curl --fail-with-body -sS -X DELETE "$BASE_URL/v1/installations/$INSTALLATION_ID/subscriptions/$SUBSCRIPTION_ID" \
  -H "Authorization: Bearer $API_KEY"
```

Успешное удаление и повторное удаление возвращают `204` без тела.

## История и аналитика

```sh
curl --fail-with-body -sS --get "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $API_KEY" \
  --data-urlencode 'limit=50' --data-urlencode 'offset=0'

curl --fail-with-body -sS --get "$BASE_URL/v1/deliveries" \
  -H "Authorization: Bearer $API_KEY" \
  --data-urlencode "message_id=$MESSAGE_ID" --data-urlencode 'status=sent'

curl --fail-with-body -sS --get "$BASE_URL/v1/analytics/deliveries" \
  -H "Authorization: Bearer $API_KEY" \
  --data-urlencode 'from=2026-10-01T00:00:00Z' \
  --data-urlencode 'to=2026-11-01T00:00:00Z'
```

История использует `limit`/`offset`: по умолчанию `50`/`0`, `limit` ограничивается
диапазоном `1..200`. Ответ содержит `items`, `limit`, `offset`; история сообщений
также возвращает `organization_id`. `total`, cursor и `has_more` отсутствуют.
Сортировка — `created_at DESC`, без дополнительного ключа при одинаковом времени.
Продолжайте с `offset + limit`; короткая страница завершает обход. При новых записях
offset-пагинация может давать повторы/пропуски.

`from` включается, `to` не включается. История сообщений поддерживает `application_id`,
`from`, `to`; история отправок дополнительно `message_id`, `status`, `platform`.
В строках истории сообщений нет агрегированного `status`, а `payload` сериализуется
в **base64-строку**. Для статуса используйте `GET /v1/messages/{messageID}`.

Аналитика группирует отправки по `status`, `platform`, `application_id`. Возвращает
`items`, `usage`, `quota`, `sent_semantics: provider_accepted` и
`delivered_evidence: device_confirmation_only`. У ключа без ограничения окружения
`usage` относится ко всей организации, включая другие проекты; фильтр `application_id`
на `usage` не действует. Границы периода `usage` применяются к `period_start`.
Подробности — [в контракте](docs/contract.md#аналитика).

## Доступные операции

| Метод и путь | Назначение |
| :--- | :--- |
| `POST /v1/installations` | Регистрация / обновление метаданных установки |
| `GET /v1/installations/{id}` | Установка и маскированные подписки |
| `PATCH /v1/installations/{id}` | Legacy-привязка пользователя через `external_user_id` |
| `POST /v1/installations/{id}/tokens` | Регистрация токена |
| `DELETE /v1/installations/{id}/subscriptions/{subscription_id}` | Отписка установки |
| `POST /v1/installations/{id}/login` | Привязка внешнего ID пользователя |
| `POST /v1/installations/{id}/logout` | Снятие привязки пользователя |
| `POST /v1/messages` | Постановка push в очередь |
| `GET /v1/messages/{messageID}` | Статус и отправки сообщения |
| `GET /v1/messages` | История сообщений |
| `GET /v1/deliveries` | История отправок |
| `GET /v1/analytics/deliveries` | Аналитика |

Дополнительные операции браузерной интеграции перечислены в [docs/web-browser.md](docs/web-browser.md)
и OpenAPI. Owner API, административные маршруты (включая `/v1` с `admin:*`),
session API Console, формы сайта, платёжные webhooks и health endpoints не входят
в публичный контракт интеграторов. Группы, отправка по `user_id`, отмена/повтор
сообщения и публичный callback подтверждения доставки не зарегистрированы.

## Статусы, TTL и idempotency

| Статус отправки | Значение |
| :--- | :--- |
| `pending` | Ожидание обработки |
| `queued` | Поставлена в очередь |
| `sending` | Идёт попытка обращения к провайдеру |
| `sent` | Провайдер принял запрос, доставка на устройство не подтверждена |
| `delivered` | Требуется явное подтверждение устройства; публичного receipt API нет |
| `failed` | Постоянная ошибка или исчерпаны попытки |
| `expired` | TTL истёк до принятия провайдером |

Агрегированный статус сообщения: `pending`, `processing`, `sent`, `delivered`, `failed`.
При незавершённых отправках он `processing`, даже если часть уже завершилась ошибкой.
После завершения смеси успешных и ошибочных отправок он `failed`; проверяйте каждую
запись `deliveries`. TTL очереди минимум 60 секунд; верхние пределы:
`critical_voip` — 600, `transactional` — 7200, `normal` — 86400, `bulk` — 604800.
TTL провайдера имеет [особенности реализации](docs/contract.md#ttl-и-повторные-попытки).

`idempotency_key` задаётся **в JSON-теле** `POST /v1/messages`, до 255 символов.
Действующий кэш хранится 24 часа и определяется парой **организация + ключ**.
Повтор проходит текущую валидацию получателей и затем возвращает прежние ID/времена
с `202` и `status: pending`, независимо от текущего статуса отправки. Тело запроса
не сравнивается: разные приложения, проекты и окружения одной организации должны
использовать разные ключи. Заголовок `Idempotency-Key` не обрабатывается, `cached`
в HTTP-ответ не включён, ответ `409` при другом теле не предусмотрен. Не переиспользуйте
старые ключи после 24 часов: оставшаяся запись кэша может вызвать `500`.
Гарантии exactly-once внешней отправки нет; подробности гонок и повторов — в контракте.

## Ошибки

Обычный ответ ошибки:

```json
{"error":{"code":"validation_error","message":"Validation failed","details":[{"field":"DeviceID","issue":"DeviceID is required"}]}}
```

`details` необязателен; имена полей могут быть именами Go DTO (`DeviceID`, `ApplicationID`),
а не JSON-ключами. При ошибке авторизации server API возвращает `401 unauthorized`
и `WWW-Authenticate: Bearer realm="API"`, без раскрытия причины недействительности ключа.

| HTTP | Типичные коды / действие |
| :--- | :--- |
| `400` | `invalid_request`, `validation_error`, `provider_platform_mismatch`, `mixed_environments`; исправьте запрос |
| `401` | `unauthorized`, в browser API `browser_auth_required`; проверьте ключ/credential |
| `403` | `forbidden`, `insufficient_scope`, `project_scope_required`, `environment_scope_mismatch`, `organization_suspended`, `origin_not_allowed` |
| `404` | `not_found`, `application_not_found`, `recipient_not_found`, `environment_not_found`, `subscription_not_found`, в browser API также `installation_not_found` |
| `409` | `conflict`, в browser API `installation_ownership_required` |
| `429` | Web Browser API: `rate_limit_exceeded`, `Retry-After: 60` |
| `500` | `internal_error`, для отправки `enqueue_failed` |

`POST /v1/messages` сейчас сводит ошибки enqueue, в том числе превышение лимита,
к `500 enqueue_failed`; не рассчитывайте на `429` или `Retry-After` этого маршрута.
При временной ошибке повторяйте с тем же idempotency key и задержкой, избегая
параллельных повторов. Ошибка `organization_suspended` может прийти как JSON-текст
с `Content-Type: text/plain`. Ошибки несуществующего пути/метода могут быть текстовыми
`404`/`405`; общий JSON envelope для них не гарантирован.

## Состав репозитория и версионирование

| Файл или каталог | Содержимое |
| :--- | :--- |
| [openapi.yaml](openapi.yaml) | Каноническая OpenAPI-спецификация публичного API |
| [docs/contract.md](docs/contract.md) | Payload, окружения, TTL, idempotency, ограничения |
| [docs/web-browser.md](docs/web-browser.md) | Браузерная авторизация и curl |
| [docs/verification.md](docs/verification.md) | Источник и объём статической сверки |
| [examples/](examples/) | JSON-тела с безопасными placeholders |
| [CHANGELOG.md](CHANGELOG.md) | История изменений контракта |

Версия документа указана в `info.version` и отслеживается отдельно от префикса `/v1`.
Изменения контракта отражаются в changelog; несовместимые изменения требуют описания
перехода клиентов.

## Обратная связь

Об ошибках документации пишите в [Issues](https://github.com/pushplatform/api/issues).
Укажите версию контракта, операцию и ожидаемый результат. Не публикуйте API-ключи,
credentials или токены устройств.

[PushPilot](https://pushpilot.ru) · [Все SDK](https://github.com/pushplatform/sdks) · [GitHub](https://github.com/pushplatform)
