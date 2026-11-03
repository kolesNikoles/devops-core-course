# Plumbing — сервис `health`

Второй сервис из plumbing (служебные сервисы на поддержке курса). Появляется в **Lab 13** как
**третий** сервис в вашей топологии — дополнительная цель, чтобы `ApplicationSet`
в ArgoCD сгенерировал несколько Application (паттерн имеет смысл только при ≥ 3 приложениях).

## Эндпоинты

| Путь | Поведение |
|------|-----------|
| `GET /` | JSON со статусом — сервис, версия, hostname, версия Go, uptime, счётчик запросов |
| `GET /healthz` | Возвращает `ok` (200) для probe |
| `GET /metrics` | Текстовый формат Prometheus — `health_requests_total`, `health_uptime_seconds` |

## Где используется

- **Lab 13** — третий Application в вашем ArgoCD ApplicationSet (Python-сервис + echo + health)
- **Lab 14** — ещё одна цель rollout, чтобы сравнить blue-green и canary
- **Lab 16** — входит в scrape kube-prometheus на весь кластер

CI репозитория курса публикует готовый образ в `ghcr.io/inno-devops-labs/health:vX.Y.Z`.

## Конфигурация

| Переменная | По умолчанию |
|---------|---------|
| `PORT` | `8082` |
| `VERSION` | `v1.0.0` |
