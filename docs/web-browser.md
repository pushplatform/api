# Web Browser API

Эти маршруты зарегистрированы отдельно от server API. Они доступны только для
настроенного web-приложения, browser key и разрешённых origins. Публичный ключ
`wbpub_…` не даёт права отправлять push или читать историю.

## Авторизация

В URL используется `/v1/web/{browser_key}`. В каждом browser-запросе обязателен
`Origin`, точно совпадающий с настройкой приложения (схема, host и порт, без пути).
Для production нужен HTTPS; для development допускается HTTP localhost/loopback.
Origin является проверкой доступа приложения, а не удостоверением пользователя.

| Операция | Authorization |
| :--- | :--- |
| `GET .../config` | Не требуется; browser_key и Origin обязательны |
| Новая `POST .../installations` | `Bearer <browser_key>` |
| Повторная регистрация той же установки | `Bearer <installation_credential>` |
| Изменения установки/подписки | `Bearer <installation_credential>` |
| Backend proof/recover/revoke | Server API key с `devices:write` |

Credential `wbinst_…` возвращается при регистрации, действует 7 дней и привязан
к установке, browser key и Origin. Храните его как секрет. Browser public key
нельзя использовать вместо credential для изменения существующей установки.
Секретные server API keys не помещайте в браузер.

## Операции браузера

В таблице `{base}` означает `/v1/web/{browser_key}`.

| Метод и путь | Результат |
| :--- | :--- |
| `GET {base}/config` | `200`: application_id, environment, vapid_public_key |
| `POST {base}/installations` | `201`: id, installation_credential, credential_expires_at, created_at |
| `POST {base}/installations/{id}/tokens` | `201`: subscription_id |
| `DELETE {base}/installations/{id}/subscriptions/{subscription_id}` | `204`: удалить свою подписку |
| `POST {base}/installations/{id}/login` | `204`: external_user_id + identity_proof |
| `POST {base}/installations/{id}/logout` | `204`: снять user_id |
| `POST {base}/installations/{id}/credential/renew` | `200`: новый credential и срок |
| `DELETE {base}/installations/{id}/credential` | `204`: отозвать credential |

Повтор регистрации со своим credential и прежним device_id возвращает тот же ID
и credential. Повтор с публичным ключом и уже существующим device_id возвращает
`409 installation_ownership_required`; публичный ключ не восстанавливает владение.
Метаданные os_version/app_version/sdk_version/device_model принимаются DTO browser
регистрации, но этим обработчиком не сохраняются.

`renew` немедленно заменяет старый credential; сохраните новый до следующего запроса.
Для renew нужен ещё действующий credential. После истечения/потери/отзыва используется
backend recover после проверки вашей пользовательской сессии и принадлежности установки.
Удаление credential не удаляет подписки: чтобы прекратить push, удалите подписку отдельно.

Подписка Web Push берётся из браузерного `PushSubscription.toJSON()`. В запросе:
`provider: web_push`, `token` равен `endpoint`, `keys` содержит реальные `p256dh` и `auth`,
`environment` совпадает с установкой. Endpoint — HTTPS URL без userinfo/fragment,
token длиной 32..1024, endpoint до 2048 символов. `p256dh` — корректный P-256 public key,
`auth` — 16 байт; оба закодированы base64url без padding. Fake keys не дадут рабочую
подписку. Перенос подписки другой установки запрещён.

## Примеры curl

Запускайте из корня репозитория. Все значения ниже — placeholders;
замените адресом инстанса и настройками своего web-приложения.

```sh
export BASE_URL='https://api.example.invalid'
export BROWSER_KEY='wbpub_REPLACE_WITH_PUBLIC_BROWSER_KEY'
export SITE_ORIGIN='https://site.example.invalid'

curl --fail-with-body -sS "$BASE_URL/v1/web/$BROWSER_KEY/config" \
  -H "Origin: $SITE_ORIGIN"
```

Возьмите application_id/environment из config и замените значения в
[examples/browser-installation.json](../examples/browser-installation.json).
`device_id` должен быть новым стабильным ID браузерной установки.

```sh
curl --fail-with-body -sS -X POST "$BASE_URL/v1/web/$BROWSER_KEY/installations" \
  -H "Origin: $SITE_ORIGIN" \
  -H "Authorization: Bearer $BROWSER_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @examples/browser-installation.json

export INSTALLATION_ID='REPLACE_WITH_ID_FROM_BROWSER_RESPONSE'
export INSTALLATION_CREDENTIAL='REPLACE_WITH_CREDENTIAL_FROM_BROWSER_RESPONSE'
```

Сохраните настоящий `PushSubscription.toJSON()` из своего браузера в локальный файл
`/tmp/pushpilot-browser-subscription.json`. Утилита `jq` преобразует его в запрос:

```sh
export DEVICE_ENVIRONMENT='development'

jq --arg environment "$DEVICE_ENVIRONMENT" \
  '{provider: "web_push", token: .endpoint, endpoint: .endpoint, keys: .keys, environment: $environment}' \
  /tmp/pushpilot-browser-subscription.json > /tmp/pushpilot-browser-token-request.json

curl --fail-with-body -sS -X POST "$BASE_URL/v1/web/$BROWSER_KEY/installations/$INSTALLATION_ID/tokens" \
  -H "Origin: $SITE_ORIGIN" \
  -H "Authorization: Bearer $INSTALLATION_CREDENTIAL" \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/pushpilot-browser-token-request.json
```

Это файлы с вашей реальной подпиской: не добавляйте их в Git. `jq` в примере
не создаёт ключей, а переносит их из браузера. Для отправки на эту подписку ваш backend
использует `POST /v1/messages` с `provider: web_push`, `platform: web` и полученными ID.

Для продления credential:

```sh
curl --fail-with-body -sS -X POST "$BASE_URL/v1/web/$BROWSER_KEY/installations/$INSTALLATION_ID/credential/renew" \
  -H "Origin: $SITE_ORIGIN" \
  -H "Authorization: Bearer $INSTALLATION_CREDENTIAL"
```

## Операции доверенного backend

| Метод и путь | Тело / ответ |
| :--- | :--- |
| `POST /v1/web/identity-proofs` | installation_id, browser_key, origin, external_user_id → `200 {identity_proof}` |
| `POST /v1/web/installation-credentials/{id}/recover` | browser_key, origin → `200` новый credential |
| `POST /v1/web/installation-credentials/{id}/revoke` | Без тела → `204` |

Backend сначала проверяет свою сессию и соответствие installation_id пользователю
сайта, затем обращается к PushPilot с ключом проекта и scope `devices:write`.
PushPilot проверяет организацию/проект/web-платформу/окружение, но не сессию вашего сайта.
Поля `browser_key`/`origin` для proof/recover передаются в JSON; заголовок Origin
для этих server-маршрутов не требуется.

Proof одноразовый, действует 1 минуту, привязан к installation_id, application_id,
environment, origin и external_user_id. Передайте его браузеру, который вызовет
login с `external_user_id` и `identity_proof`. Не выдавайте proof для user ID,
взятого из запроса посетителя без проверки его сессии.

## Ошибки и лимиты

Ошибки browser API используют JSON envelope `error.code/message`:
`origin_not_allowed` (`403`), `browser_auth_required` (`401`),
`installation_ownership_required` (`409`), `application_scope_mismatch` (`400` при
регистрации, `403` для server proof), `invalid_web_subscription` (`400`),
`installation_not_found` / `subscription_not_found` (`404`).

При лимите — `429 rate_limit_exceeded`, `Retry-After: 60`. Текущие лимиты за окно
60 секунд: 1200 запросов на browser key, 120 обычных запросов на browser key и socket
peer, 10 регистрационных запросов на browser key и socket peer, 60 запросов
на credential. Server proof/recover также используют browser rate limiter.
Лимиты разделяются между экземплярами сервера.

Browser ответы имеют `Cache-Control: no-store`. CORS разрешает GET/POST/DELETE и
заголовки Authorization/Content-Type; expose — Retry-After. Preflight OPTIONS
возвращает `204`, запрещённые метод/заголовок — `403 preflight_rejected`.
Обычный server API имеет отдельную политику CORS; используйте browser-маршруты для сайта.
