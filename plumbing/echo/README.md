# Plumbing — сервис `echo`

Крошечный HTTP-сервис на Go, который курс поставляет как **plumbing** (служебные сервисы на поддержке курса).
**Не меняйте этот код.** Выкатите его рядом со своим сервисом начиная с **Lab 9**.

## Зачем он нужен

До Lab 8 включительно вы запускали один Python-сервис. Когда дойдёте до Kubernetes (Lab 9),
**второй** Pod рядом с ним — самая маленькая топология, в которой `Service`,
kube-DNS и сеть между Pod действительно что-то значат. `echo` и есть этот второй Pod.

## Эндпоинты

| Путь | Поведение |
|------|-----------|
| `GET /ping` | Возвращает `pong\n` — минимальный smoke-тест |
| `* /echo` | Возвращает JSON с телом запроса, заголовками, hostname, версией, uptime и монотонно растущим счётчиком запросов — удобно проверять балансировку между репликами |
| `GET /healthz` | Возвращает `ok` (200) — подключите его к `readinessProbe` и `livenessProbe` |
| `GET /metrics` | Текстовый формат Prometheus — счётчик `echo_requests_total` и gauge `echo_uptime_seconds` |

## Локальная сборка

```bash
cd plumbing/echo
docker build -t echo:dev .
docker run --rm -p 8081:8081 echo:dev
curl -s localhost:8081/ping
curl -s localhost:8081/echo -d 'hello'
```

## Где используется

- **Lab 9** — выкатывается как 2-й сервис в вашем Namespace `default`
- **Lab 10** — упакован как subchart вашего Helm-чарта (бонусный трек)
- **Labs 13–14** — вторая цель ArgoCD ApplicationSet и canary в Argo Rollouts

CI репозитория курса публикует готовый образ в `ghcr.io/inno-devops-labs/echo:vX.Y.Z`.

## Конфигурация

| Переменная | По умолчанию | Назначение |
|---------|---------|---------|
| `PORT` | `8081` | Порт прослушивания |
| `VERSION` | `v1.0.0` | Попадает в JSON `/echo`; по ней удобно проверить canary |
