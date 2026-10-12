# Источник и объём сверки

Дата: 2026-10-12. Документация подготовлена по локальному рабочему дереву
`../pushpilot-server`, HEAD `3d8f136afbc8440cda58beb4b31cc9576ecb2372`.
В нём уже имелись незакоммиченные изменения, в том числе административные дополнения
роутера; сверка относится к прочитанному рабочему дереву, а не только к HEAD.
Серверная OpenAPI не использовалась как источник истины вместо регистрации маршрутов.

| Область | Проверенные исходники сервера |
| :--- | :--- |
| Подключение маршрутов и границы public/admin | `cmd/api/main.go`, `internal/handler/message_handler.go`, `internal/handler/web_browser_handler.go` |
| Авторизация и scopes | `internal/auth/middleware.go`, `scopes.go`, `key_generator.go`; `internal/middleware/suspension.go` |
| Установки и токены | `internal/handler/device_handler.go`, `dto/device.go`, SDK unsubscribe в `admin_device_handler.go`; `internal/service/device_service.go` |
| Отправка, история и аналитика | `internal/handler/message_handler.go`; `internal/models/queue_models.go` |
| TTL и idempotency | `internal/service/message_enqueue_service.go`; уникальность кэша в `migrations/000008_queue_and_events.up.sql` |
| Payload worker и провайдеры | `internal/worker/processor.go`, `retry.go`; `internal/provider/apns/apns_provider.go`, `fcm/delivery_provider.go`, `fcm/payload.go`, `webpush/webpush_provider.go` |
| Web credentials / proof / лимиты | `internal/service/web_browser_auth.go`, `internal/handler/web_browser_handler.go` |
| Формат ошибок | `internal/handler/errors.go`, `dto/common.go`, `response.go` |

Публичный контракт включает 23 операции интеграции: 12 основных и 11 операций
Web Browser / его backend. Операция SDK unsubscribe включена по маршруту и проверке
`devices:write`, хотя реализована в файле административного обработчика.
Маршруты `/owner/v1`, `/admin/v1` и группы `/v1` с обязательным `admin:*` исключены.
Session API Console, публичные формы сайта, платежные webhooks и служебные probes
не относятся к контракту интеграции push.

Сверены методы/пути, request DTO и обязательные поля, scopes, response DTO и прямая
JSON-сериализация моделей, фильтры, пагинация, ошибки и ограничения. Важные расхождения
с ожидаемым контрактом описаны явно: `500` вместо rate-limit `429` на send,
base64 payload истории, область idempotency и поведение старого кэша, ограничения
environment scope и TTL провайдера.

OpenAPI проверена однократно локальным `@apidevtools/swagger-parser` (с проверкой
спецификации и разрешением ссылок); JSON-примеры и JSON-блок README — по схемам
через Ajv. Curl-пути, методы, request schema и безопасные placeholders проверены
статически, shell-блоки — проверкой синтаксиса. Curl не выполнялся.
Никакие runtime-тесты, сервисы, backend/SDK/Console или базы не запускались и не менялись.
Эта проверка подтверждает соответствие исходникам, а не доступность конкретного инстанса.

Результат единого прогона: **PASS**. `swagger-parser` 13.1.0: 23 операции;
сверка полей 11 DTO/моделей; 5 JSON-файлов и 5 schema examples;
2 JSON-блока Markdown; 14 shell-блоков и 15 curl-команд. Относительные ссылки
и `git diff --check` прошли в том же прогоне. Повторных прогонов не выполнялось.
