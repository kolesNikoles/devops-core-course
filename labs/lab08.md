# Lab 8 — Метрики и мониторинг с Prometheus

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Metrics%20%26%20Monitoring-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Prometheus%203.11%20|%20Grafana%2013%20|%20prometheus--client-informational)

> **Цель:** инструментируйте Python-приложение из Lab 1 метриками Prometheus, поставьте Prometheus 3.x рядом со стеком Loki с прошлой недели, чтобы он забирал эти метрики, и спроектируйте дашборд Grafana вокруг **метода RED**.
> **Что сдаёте:** PR из `lab08`: эндпоинт `/metrics` и инструментация в `app_python/`, сервис `prometheus` в `monitoring/docker-compose.yml`, файл `monitoring/prometheus/prometheus.yml`, JSON дашборда RED на 6 панелей и `monitoring/docs/LAB08.md` со снятыми доказательствами.

---

## Обзор

В Lab 7 у сервиса появились глаза (логи). На этой неделе вы даёте ему **пульс** (метрики). Вы:
- Инструментируете HTTP-сервис библиотекой `prometheus_client`: объявляете **Counter**, **Histogram** и эндпоинт `/metrics`
- Подключаете **Prometheus** к Compose-стеку Lab 7, чтобы он забирал `/metrics` у приложения каждые несколько секунд
- Пишете **PromQL** для **метода RED** (Rate / Errors / Duration). Запросы ваши, из лекции их не копируют
- Проектируете дашборд Grafana вокруг трёх вопросов RED и saturation (насыщение)
- Укрепляете стек: retention (срок хранения), лимиты ресурсов, health-проверки, персистентность

> ⚠️ **Границы:** Prometheus в режиме single-binary на локальном Docker-хосте. Без Alertmanager, без `remote_write`, без Mimir. Это уже за пределами Core. Навык здесь — **инструментирование и PromQL**, а не федерация Prometheus.

> ✨ **Заметка про Prometheus 3.x.** В лабораторной Prometheus 3.11.3 (актуальный stable на май 2026; ветка 3.5 LTS поддерживается до июля 2026). По сравнению с 2.x имена метрик и меток по умолчанию в UTF-8, в комплекте приёмник OTLP, а **native histograms доведены до GA**. В обязательных заданиях берите классические гистограммы. Native histograms можно включить сверх программы.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**
- `app_python/` из Lab 1 — сервис на Flask/FastAPI с `/` и `/health`
- Рабочий Docker-образ из Lab 2
- Compose-стек `monitoring/` из Lab 7: `loki`, `alloy`, `grafana`, `app`, все в сети `logging`
- Доработка JSON-логирования в `app_python/` из Lab 7 (эта лабораторная кладёт *ещё один* слой в тот же `app.py`)

**Эта лабораторная добавляет:**
- Эндпоинт `/metrics` и инструментацию `Counter` + `Histogram` в `app_python/app.py`
- `monitoring/prometheus/prometheus.yml` — заполняете вы, начиная с каркаса
- Сервис `prometheus` в `monitoring/docker-compose.yml` (стек Lab 7 расширяется, а не заменяется)
- `monitoring/grafana/dashboards/lab08.json` — ваш экспортированный дашборд RED на 6 панелей
- `monitoring/docs/LAB08.md` — отчёт для сдачи

К Lab 16 вы задеплоите это как `kube-prometheus-stack` на k3d: CRD `ServiceMonitor` сами находят таргеты. Всё с этой недели — типы метрик, кардинальность, PromQL, RED — переносится как есть.

---

## Подготовка

Версии в этой лабораторной. Зафиксируйте их в compose-файле и в `requirements.txt`:

| Компонент | Тег / версия | Заметки |
|---|---|---|
| `prom/prometheus` | `v3.11.3` | Актуальный stable (май 2026); `v3.5.0` LTS тоже подходит |
| `grafana/grafana` | `13.0.1` | Уже есть в стеке Lab 7 |
| `prometheus-client` (Python) | `0.23.1` | Минимум `0.23+`; `0.25.x` — последняя линия патчей |

```bash
docker --version           # 28.x or 29.x
docker compose version     # v2.x
python --version           # 3.12+
```

Дерево каталогов, которое вы соберёте:

```
app_python/
├── app.py                            # YOU add instrumentation here (§1)
└── requirements.txt                  # YOU add prometheus-client (§1.2)

monitoring/
├── docker-compose.yml                # YOU extend with a `prometheus` service (§2.2)
├── prometheus/
│   └── prometheus.yml                # YOU fill the scaffold (§2.3)
├── grafana/
│   └── dashboards/
│       └── lab08.json                # exported from Grafana UI in Task 3
└── docs/
    └── LAB08.md                      # your submission report
```

Каркас из репозитория курса для этой лабораторной:
- `labs/lab8/prometheus/prometheus.yml` — **каркас** с маркерами ВАША ЗАДАЧА. Скопируйте его в `monitoring/prometheus/prometheus.yml` и заполните пропуски.

---

## Задание 1 — Инструментируйте Python-приложение (3 балла)

### 1.1 — Сначала прочитайте, потом пишите

Прочитайте это до правок в `app.py`. Здесь ответы на вопросы, которые ставит инструментация:
- [Типы метрик Prometheus](https://prometheus.io/docs/concepts/metric_types/) — Counter / Gauge / Histogram / Summary
- [Практики инструментирования](https://prometheus.io/docs/practices/instrumentation/) — правило низкой кардинальности и имена метрик
- [`prometheus_client` (Python)](https://github.com/prometheus/client_python) — Counter, Histogram, `generate_latest`, `CONTENT_TYPE_LATEST`

`ВАША ЗАДАЧА`: в `docs/LAB08.md` ответьте на три вопроса, по 2–4 предложения на каждый. После задания 3 вернётесь и поправите:

1. Когда брать **Counter**, а когда **Histogram**? По одному примеру каждого из *вашего* приложения.
2. Зачем запрашивать `rate(counter[5m])`, а не сырое значение счётчика? (Подсказка: при рестарте процесса счётчик обнуляется.)
3. Что такое **кардинальность меток** и почему метка `user_id` или `request_id` со временем уложит Prometheus в OOM?

### 1.2 — Добавьте клиентскую библиотеку

Добавьте точный пин в `app_python/requirements.txt`:

```txt
prometheus-client==0.23.1
```

Пересоберите образ из Lab 2, чтобы зависимость попала в контейнер.

### 1.3 — Опишите метрики (`ВАША ЗАДАЧА`)

В `app_python/app.py` вы добавите **две** метрики. Лекция (слайды 8, 10, 13) даёт имена, метки и набор бакетов, который SRE берёт по умолчанию. Ваша работа: объявить метрики, написать внятные **строки help** и выбрать набор меток. Только ограниченные перечисления.

`ВАША ЗАДАЧА`: объявите две метрики на уровне модуля:

| Переменная | Тип | Имя метрики | Метки | Заметки |
|---|---|---|---|---|
| `REQUESTS` | `Counter` | `app_requests_total` | `method`, `endpoint`, `status` | Считает каждый HTTP-запрос, на который сервис ответил |
| `LATENCY` | `Histogram` | `app_request_duration_seconds` | `method`, `endpoint` | Раскладывает длительность запроса по бакетам. Набор SRE по умолчанию: `(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0)` |

```python
# app.py — add to your Lab 1 service (Flask shown; adapt to FastAPI middleware)
import time
from flask import Flask, Response, request
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

# --- Metric definitions (low-cardinality labels only!) ---
REQUESTS = Counter(
    ___,                # YOUR TASK: metric name — see the table; ends in _total per Prom naming convention
    ___,                # YOUR TASK: help string — read by anyone debugging your metrics
    ___,                # YOUR TASK: label list — bounded enumerations only
)
LATENCY = Histogram(
    ___,                # YOUR TASK: metric name — _seconds suffix is the Prom convention for time
    ___,                # YOUR TASK: help string
    ___,                # YOUR TASK: label list — narrower than the counter's (no status)
    buckets=___,        # YOUR TASK: the SRE-default tuple from the table — bucket past your SLO ceiling
)
```

> ⚠️ **Кардинальность — способ №1 уложить Prometheus в OOM.** Каждая уникальная комбинация меток — свой временной ряд (~3–5 КБ RAM). Метки должны быть *ограниченными* перечислениями: `method` (~8), `endpoint` (десятки, нормализованные), `status` (~10). В метку **нельзя** класть `user_id`, `request_id`, сырые пути, адреса почты и метки времени. Тот же урок, что у меток Loki (Lecture 7), тот же радиус поражения.

### 1.4 — Подключите хуки (`ВАША ЗАДАЧА`)

У Flask есть `before_request` (срабатывает до обработчика) и `after_request` (срабатывает после и получает объект `Response`). Схема: *засечь время в before, в after увеличить счётчик и вызвать observe*. Тела функций пишете вы.

```python
@app.before_request
def _start_timer():
    request._t0 = ___                # YOUR TASK: capture a monotonic start time
                                     #   — hint: time.perf_counter(), NOT time.time()

@app.after_request
def _record(response):
    duration = ___                   # YOUR TASK: now - request._t0
    if ___:                          # YOUR TASK: skip /metrics — exclude its own path here
        return response              #            (otherwise scrape traffic poisons your counts)
    endpoint = ___                   # YOUR TASK: use request.url_rule.rule when matched,
                                     #            fall back to "unknown" when url_rule is None
    REQUESTS.labels(___).inc()       # YOUR TASK: method, endpoint, str(response.status_code)
    LATENCY.labels(___).observe(___) # YOUR TASK: (method, endpoint) labels + the duration
    return response

@app.route("/metrics")
def metrics():
    return Response(
        ___,                         # YOUR TASK: bytes from generate_latest()
        mimetype=___,                # YOUR TASK: CONTENT_TYPE_LATEST — wrong mimetype = silent
                                     #            parse failure on Prometheus's side
    )
```

Подсказки, *не* готовый код:
- **`request.url_rule.rule`** для совпавшего маршрута равен `"/health"` или `"/"`. На 404 там `None`. Подставьте `"unknown"`, чтобы кардинальность осталась ограниченной.
- **Статус приводите** к `str(response.status_code)`. Значение метки — строка.
- **Эквивалент на FastAPI:** одна корутина `@app.middleware("http")`. Она оборачивает `await call_next(request)` вызовами `time.perf_counter()`. Эндпоинт `/metrics` отдайте так: `Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)`.

### 1.5 — Проверьте локально

Два способа. Выберите один и дальше держитесь его:

- **Через Compose-стек Lab 7** (так лучше): пересоберите и перезапустите только сервис `app`, чтобы новый образ с `prometheus-client` слушал `localhost:8080`.
- **Отдельный процесс Python:** сначала `pip install -r requirements.txt` в venv, потом `python app.py`.

```bash
# Option A — through the Compose stack (recommended; matches Task 2 below).
# Run from repo root; the stack listens on host port 8080:
docker compose -f monitoring/docker-compose.yml up -d --build app    # picks up new app.py + requirements

# Option B — standalone Python (run from repo root; Lab 1 default port is 5000).
# Either keep the curls below on :5000, or set PORT=8080 to match Option A:
# PORT=8080 python app_python/app.py

# Generate traffic and inspect /metrics (Option A → :8080; Option B unchanged → :5000):
for i in $(seq 1 20); do curl -s localhost:8080/ > /dev/null; done
for i in $(seq 1 5);  do curl -s localhost:8080/health > /dev/null; done
curl -s localhost:8080/metrics | grep -E "^app_(requests_total|request_duration)"
```

Вывод — текстовый формат экспозиции Prometheus. **Пример. Ваши счётчики будут другими:**

```
app_requests_total{endpoint="/",method="GET",status="200"} 20.0
app_requests_total{endpoint="/health",method="GET",status="200"} 5.0
app_request_duration_seconds_bucket{endpoint="/",method="GET",le="0.005"} 12.0
app_request_duration_seconds_bucket{endpoint="/",method="GET",le="0.01"}  18.0
app_request_duration_seconds_count{endpoint="/",method="GET"} 20.0
app_request_duration_seconds_sum{endpoint="/",method="GET"} 0.0421
```

**Короткая проверка:** снимите `curl -s localhost:8080/metrics > /tmp/m1.txt`, дёрните маршрут, снимите `curl -s localhost:8080/metrics > /tmp/m2.txt`, сделайте diff. Для маршрута, который вы дёрнули, сумма должна **вырасти ровно на 1**. Строки `endpoint="/metrics"` быть не должно. Если `/metrics` считает сам себя, исключение в §1.4 написано неверно.

### 1.6 — Доказательство работы

**Вставьте в `docs/LAB08.md`:**

- Полный вывод `curl -s localhost:8080/metrics | grep ^app_`: и строки `app_requests_total{...}`, и `app_request_duration_seconds_{bucket,sum,count}`, с вашими реальными числами.
- Доказательство, что сервис не считает сам себя: `curl -s localhost:8080/metrics | grep -c 'endpoint="/metrics"'` возвращает `0`.
- Код объявления метрик и хуков (фрагмент, не весь файл), закоммиченный в `app_python/app.py`.
- Три письменных ответа из §1.1.

---

## Задание 2 — Задеплойте Prometheus и настройте scrape (3 балла)

### 2.1 — Разберитесь с pull-моделью

`ВАША ЗАДАЧА`: сумейте ответить на это в §3 файла `docs/LAB08.md` (ответы напишите сейчас, после §2.4 поправьте):

1. Почему *сорвавшийся scrape* (Prometheus сам забирает метрики) даёт здоровье таргета само собой (`up == 0`), а push-модель так не умеет?
2. Чем **job** отличается от **target** и от **интервала scrape**?
3. Почему в Docker Compose Prometheus должен снимать метрики с `app:8080` (имя сервиса) и не должен ходить на `localhost:8080`?

### 2.2 — Добавьте сервис Prometheus (`ВАША ЗАДАЧА`)

**Файл:** `monitoring/docker-compose.yml` (расширьте стек Lab 7, не подменяйте его)

Prometheus садится в ту же сеть `logging` и по имени сервиса достучится до Loki, Grafana и `app`. Образ и порты заданы, версии зафиксированы. Тома, флаги команды, healthcheck и членство в сети — ваша работа.

```yaml
  # add to monitoring/docker-compose.yml, alongside loki/alloy/grafana/app from Lab 7
  prometheus:
    image: prom/prometheus:v3.11.3
    command:
      - ___                              # YOUR TASK: --config.file= path INSIDE the container
                                         #   (where you'll mount prometheus.yml below)
      - ___                              # YOUR TASK: --storage.tsdb.retention.time=
                                         #   (pick a value — Task 4 documents the trade-off)
      - ___                              # YOUR TASK: --storage.tsdb.retention.size=
                                         #   (defence in depth against disk fill)
    ports:
      - "9090:9090"
    volumes:
      - ___                              # YOUR TASK: bind-mount prometheus.yml :ro
      - ___                              # YOUR TASK: named volume for /prometheus (TSDB)
    networks: [___]                      # YOUR TASK: which network from Lab 7?
    healthcheck:
      test: ___                          # YOUR TASK: probe /-/healthy via wget --spider or curl --fail
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

volumes:
  prometheus-data:                       # add alongside loki-data / grafana-data from Lab 7
```

Заметки. Не пропускайте: это *зачем* у пропусков.

- **`--config.file=`** указывает на место, куда смонтирован `prometheus.yml` (обычно `/etc/prometheus/prometheus.yml`). Пути разъехались — при старте `docker compose logs prometheus` пишет чистую строку «open ...: no such file». Сервис не поднимается — сначала читайте логи.
- **Retention — это флаг CLI Prometheus, а не ключ в файле конфига** (самая загугленная ловушка Prometheus 3.x). Нужны и `retention.time` (например, `15d`), и `retention.size` (например, `2GB`). Срабатывает то, что наступит раньше. Выбор запишите в §4.
- **Том TSDB.** Без персистентности каждый `docker compose down` стирает историю метрик. Смонтируйте именованный том на `/prometheus`, чтобы после рестарта дашборды не пустели.
- **Сеть `logging`** — та самая из Lab 7. Prometheus в другой сети не зарезолвит `app` / `loki` / `grafana`. Отлаживать придётся DNS, а не scrape.

### 2.3 — Заполните конфиг scrape (`ВАША ЗАДАЧА`)

**Файл:** `monitoring/prometheus/prometheus.yml`

Структура жёсткая: один блок `global`, один список `scrape_configs`, один job на таргет. Так устроен Prometheus. Навык в другом: **как часто снимать**, **на каком hostname:port живёт каждый таргет** и **как назвать каждый job** (имя видно в каждой метке дашборда и в каждом алерте).

Начните с каркаса:

```bash
cp labs/lab8/prometheus/prometheus.yml monitoring/prometheus/prometheus.yml
```

Что вы заполняете (в каркасе те же пропуски):

```yaml
global:
  scrape_interval:        # YOUR TASK: how often to scrape
                          #   — recall lecture §15: query ranges must be ≥ 4× this
  evaluation_interval:    # YOUR TASK: how often to evaluate rules (match scrape_interval)

scrape_configs:
  - job_name:             # YOUR TASK: short lowercase name, e.g. "prometheus"
    static_configs:
      - targets:          # YOUR TASK: Prometheus's own /metrics, from INSIDE the container
                          #   — this is the ONE target that uses localhost

  - job_name:             # YOUR TASK: a name you'll filter on later — e.g. up{job="..."}
    static_configs:
      - targets:          # YOUR TASK: your app's Compose service name + port
                          #   — NOT localhost; Prometheus runs in a different container

  - job_name:             # YOUR TASK
    static_configs:
      - targets:          # YOUR TASK: Loki's HTTP port on the logging network

  - job_name:             # YOUR TASK
    static_configs:
      - targets:          # YOUR TASK: Grafana's HTTP port on the logging network
```

Заметки:
- `metrics_path:` по умолчанию равен `/metrics`, поле можно не писать.
- **Имя сервиса — это hostname** в пользовательской сети Compose: `loki:3100`, `grafana:3000`, `app:8080`. Self-scrape Prometheus — единственный таргет на `localhost`, потому что этот трафик не выходит из контейнера Prometheus.
- **`scrape_interval` выбирайте сами.** Слишком короткий (1 с) жжёт CPU и раздувает TSDB. Слишком длинный (60 с) делает графики перцентилей рваными. Эталонная сдача для тихой лабораторной берёт `5s`. В проде по умолчанию `15s`. Выбор запишите в §3.

<details>
<summary>💡 Второй таргет по желанию — обвязка `echo` из курса</summary>

В репозитории есть маленький Go-сервис `plumbing/echo` (снова встретите его в Lab 9). Свои метрики он уже отдаёт на `/metrics`. Добавите его — на дашборде появится второй инструментированный сервис. Это удобно для панелей `sum by (job)`:

Добавьте сервис `echo` (`build: ../plumbing/echo`, порт `8081`, сеть `logging`) и пятый scrape job на `echo:8081`. Тот же приём, новых идей нет.

Для полного балла не требуется. Для аккуратного дашборда в задании 3 — в самый раз.

</details>

### 2.4 — Поднимите стек и проверьте

```bash
cd monitoring
docker compose up -d
docker compose ps
```

Проверьте команды, вывод которых вставите в §2.5:

```bash
# Prometheus is up
curl -s localhost:9090/-/healthy           # expect: "Prometheus Server is Healthy."

# Every job's target is UP — this is the proof your scrape config + service names are right.
# Run this AFTER one scrape_interval has elapsed.
curl -s localhost:9090/api/v1/targets \
  | jq -r '.data.activeTargets[] | "\(.labels.job): \(.health)"'
# (illustrative — every line must say "up"; if any says "down", that job's target field is wrong)
# <your_app_job>: up                       # YOUR TASK: this is the headline assertion —
# loki: up                                 #            the line for the job_name you picked in §2.3
# grafana: up                              #            for your app MUST say "up"
# prometheus: up
```

**Дерево диагностики:**
- Таргет **down** → неверное имя сервиса, неверный порт, или сервис не запущен. `docker compose logs prometheus | grep -i scrape` показывает URL, который Prometheus реально пробовал.
- У job **нет таргета вообще** → сломаны отступы YAML в `prometheus.yml`. Ошибку разбора на старте пишет `docker compose logs prometheus`.
- Таргет **unknown** → подождите один `scrape_interval` и обновите страницу. Первый scrape ещё не прошёл.

### 2.5 — Доказательство работы

**Вставьте в `docs/LAB08.md`:**

- Вывод `docker compose ps`: `prometheus` в состоянии `Up` и `(healthy)` (второе — после того как истечёт start_period).
- Точный вывод `curl -s localhost:9090/api/v1/targets | jq ...` сверху. В каждой строке job должно быть `up`, и строка **`app: up`** хорошо видна. Это главный критерий оценки задания 2.
- Скриншот `http://localhost:9090/targets` с той же картиной (тот же факт в UI).
- Три письменных ответа из §2.1.

---

## Задание 3 — Дашборд RED в Grafana (2 балла)

### 3.1 — Подключите Prometheus как источник данных

В Grafana из Lab 7:

1. **Connections → Data sources → Add data source → Prometheus**.
2. URL: `http://prometheus:9090` — имя сервиса в сети `logging`.
3. **Save & Test** → ответ должен быть зелёным.

> Бонусный трек прописывает источник сам. Один раз руками — и станет ясно, что именно кодирует provisioning YAML.

### 3.2 — Напишите PromQL (`ВАША ЗАДАЧА`)

Здесь сердце лабораторной. Лекция (слайды 16, 18, 21) покрывает все нужные функции. Запросы пишете вы.

На каждую строку — выражение PromQL, которое отвечает на вопрос. Проверьте его в **Explore** (источник Prometheus) на живом трафике. Сначала нагоните трафик:

```bash
for i in $(seq 1 200); do curl -s localhost:8080/ > /dev/null; curl -s localhost:8080/health > /dev/null; done
```

| Вопрос (RED) | Подсказка: одна именная функция | Ограничение |
|---|---|---|
| **Q1 — Rate.** Запросов в секунду, в разбивке по endpoint. | `rate` | `sum by (endpoint) (...)`, чтобы оставить `endpoint` и отбросить остальное |
| **Q2 — Errors.** Запросов 5xx в секунду, по endpoint. | `rate` с матчером `status=~"5.."` | Та же форма, что у Q1; матчер стоит **внутри** селектора, до `rate` |
| **Q3 — Duration.** Латентность p95 по endpoint. | `histogram_quantile` | `sum by (endpoint, le)` стоит **внутри** `histogram_quantile`; собирать наоборот нельзя |
| **Q4 — Error ratio.** 5xx как *доля* всех запросов. | Два `rate`, деление | Без группировки — одно число; умножьте на 100, если нужен процент |

`ВАША ЗАДАЧА`: напишите все четыре запроса. Лекцию дословно не копируйте. Подгоните их под *ваши* имена метрик из §1.3.

```promql
# Q1 — Rate (requests/sec per endpoint)
___                              # YOUR TASK: rate(...) + sum by (endpoint)

# Q2 — Errors (5xx requests/sec per endpoint)
___                              # YOUR TASK: same shape, status=~"5.." selector

# Q3 — Duration (p95 latency per endpoint)
___                              # YOUR TASK: histogram_quantile OUTSIDE sum by (endpoint, le)

# Q4 — Error ratio (5xx fraction of total)
___                              # YOUR TASK: rate(5xx) / rate(total) — no grouping
```

Заметки. Не пропускайте:

- **`rate()`** переживает обнуление счётчика (рестарт процесса). `irate()` и вычитание сырых значений этого не делают. На графиках берите `rate()`.
- **Нижняя граница окна:** при `scrape_interval: 5s` честный минимум — `rate(...[1m])`. При `15s` берите `[5m]` или длиннее.
- **Перцентиль не усредняют.** `histogram_quantile()` стоит *снаружи* `sum by (le, …)`, внутри `avg()` ему не место. Один раз соберите вложенность наоборот и прочитайте ошибку Prometheus. Это полезнее абзаца теории.
- **Застряли на имени** — возьмите форму из лекции (`http_requests_total` и похожие). Q1–Q4 всё равно должны совпасть с именами метрик, которые вы реально отдали в §1.3.

### 3.3 — Соберите дашборд: 6 панелей

Каждая панель отвечает на один вопрос. Запросы PromQL для панелей 1, 2, 3, 5 и 6 описаны ниже. Панель 4 (heatmap) вы пишете сами, это продолжение §3.2.

| # | Панель | Визуализация | Откуда запрос | Отвечает на |
|---|-------|---------------|---|---|
| 1 | **Частота запросов** | Time series | Ваш Q1 | *Насколько загружены?* |
| 2 | **Частота ошибок** | Time series | Ваш Q2 | *Как часто падаем?* |
| 3 | **Латентность p95** | Time series | Ваш Q3 | *Насколько медленно на 95-м?* |
| 4 | **Тепловая карта латентности** | Heatmap | ВАША ЗАДАЧА — тот же ряд `_bucket`, но `sum by (le)` *без* `histogram_quantile` (перцентили считает сама панель heatmap) | *Как выглядит длинный хвост?* |
| 5 | **В работе или всего** | Gauge / Stat | `sum(rate(<your counter>[1m]))` | *Насколько загружены прямо сейчас?* |
| 6 | **Аптайм сервиса** | Stat | `up{job="<your app job name>"}` | *Мы живы?* |

`ВАША ЗАДАЧА`: PromQL панели 4 напишите сами. Это единственная панель, которую не закрывают Q1–Q4. Подсказка: heatmap **не** вызывает `histogram_quantile`. Он рисует счётчики бакетов как есть. Форма та же: `sum by (le) (rate(...))`, без обёртки квантиля.

**Как собрать (Grafana 13):**
1. **Dashboards → New → New dashboard → Add visualization** → источник **Prometheus**.
2. Вставьте PromQL. Тип визуализации — из таблицы. Название панели — **по вопросу, на который она отвечает** («Ошибок в секунду по endpoint» сильнее, чем «Запрос 2»).
3. Единицы: панель 1 = `req/s`; панель 3 = `seconds`; панель 2 = `req/s` и порог, чтобы ненулевые ошибки красили панель красным.
4. В легенде `{{endpoint}}` (или `{{job}}`), чтобы линии были подписаны, а не назывались «Series 1».
5. **Save dashboard**, затем **Share → Export → Save to file**. Коммит в `monitoring/grafana/dashboards/lab08.json`.

> 💡 Нужен задел по инфраструктурным панелям? Импортируйте дашборд сообщества **ID 3662** (self-stats Prometheus) через **Dashboards → New → Import** и привяжите его к своему источнику Prometheus. На оценку идёт *ваш* дашборд RED из 6 панелей.

### 3.4 — Доказательство работы

**Вставьте в `docs/LAB08.md`:**

- Скриншот дашборда из 6 панелей с **живыми данными на каждой** (свежий трафик — прямо перед скриншотом; пустая панель баллов не даёт).
- Запросы PromQL Q1–Q4 и запрос heatmap панели 4 в markdown-таблице. У каждого подписан вопрос, на который он отвечает.
- Настоящий ответ PromQL с вашего трафика, снятый через HTTP API (проверяющий должен видеть *ваши* числа, а не только скриншот). Прогоните четыре запроса через `/api/v1/query` и сохраните вывод:
  ```bash
  curl -s --get http://localhost:9090/api/v1/query \
       --data-urlencode 'query=___' \
       | jq '.data.result'
  # YOUR TASK: substitute each of your Q1–Q4 queries in turn; paste the .data.result
  # (illustrative — your counts will differ)
  # [{"metric":{"endpoint":"/"},"value":[1716894123.456,"127"]},
  #  {"metric":{"endpoint":"/health"},"value":[1716894123.456,"54"]}]
  ```
- Путь к закоммиченному JSON: `monitoring/grafana/dashboards/lab08.json`.

---

## Задание 4 — Готовность к проду (1 балл)

Укрепите стек. Всё здесь опирается на сервис `prometheus`, который вы написали в §2.2. Пропуски уже стоят в файле. В этом задании вы *обосновываете* выбранные значения.

### 4.1 — Лимиты ресурсов

`ВАША ЗАДАЧА`: добавьте `deploy.resources` сервису `prometheus`. Числа берите разумные. RAM Prometheus растёт вместе с числом активных рядов. Для этой лабораторной пары сотен MiB с запасом. Числа и рассуждение запишите в `LAB08.md`.

```yaml
    deploy:
      resources:
        limits:
          cpus: ___                 # YOUR TASK: ceiling — what's the most you'd let it eat?
          memory: ___               # YOUR TASK: RAM ceiling — keep series count in mind
        reservations:
          cpus: ___                 # YOUR TASK: guaranteed floor at scheduling time
          memory: ___               # YOUR TASK: guaranteed memory floor
```

### 4.2 — Retention

Флаги retention вы наметили в §2.2. В `LAB08.md` объясните, зачем нужны **оба**, `retention.time` и `retention.size`. Время — ручка, которую задаёт SLA. Размер — защита в глубину на случай, если кардинальность разбежится.

### 4.3 — Health-проверка и персистентность

Блок healthcheck из §2.2 должен быть заполнен. Проверьте, что он работает:

```bash
docker compose ps                              # prometheus reports (healthy) once start_period elapses
docker compose down && docker compose up -d    # NOTE: no `-v` — that would wipe the named volumes
sleep 30                                       # wait for the first post-restart scrape (≥ one scrape_interval + start_period)
curl -s 'http://localhost:9090/api/v1/query?query=app_requests_total' | jq '.data.result | length'
# (illustrative — non-zero means your TSDB volume persisted the series through the restart;
#  exactly 0 means either the volume wasn't mounted or `down -v` was used by mistake)
```

### 4.4 — Доказательство работы

**Вставьте в `docs/LAB08.md`:**

- Вывод `docker compose ps`: **все** сервисы healthy, включая `prometheus (healthy)`.
- Вывод цепочки `down` → `up` → query сверху. История метрик пережила рестарт.
- Ваши значения лимитов и по одному предложению, почему каждое такое.

---

## Задание 5 — Документация (1 балл)

`ВАША ЗАДАЧА`: напишите `monitoring/docs/LAB08.md` с такими разделами, по порядку:

1. **Архитектура** — диаграмма Mermaid: `app → Prometheus → Grafana`, рядом путь логов из Lab 7 (одна Grafana показывает оба столпа).
2. **Запуск** — как поднять стек (`docker compose up -d`) и команды проверки из §2.4.
3. **Инструментирование** — имена метрик, типы и метки, которые вы добавили, плюс ответы из §1.1.
4. **Конфиг scrape** — имена job, ваш `scrape_interval`, pull-модель (ответы на вопросы §2.1).
5. **Дашборд** — таблица из §3.4: вопрос → PromQL → скриншот.
6. **Продакшен-конфиг** — зачем такой retention, лимиты ресурсов, healthcheck, доказательство персистентности.
7. **Метрики и логи** — один абзац: когда вы пойдёте в Prometheus, а когда в запрос LogQL из Lab 7.
8. **Трудности и решения** — хотя бы одна настоящая (формулировка «я только познакомился с Prometheus» не считается).

Включите фрагменты конфигов (не файлы целиком) и снятое из заданий 1–4. Пишите так, чтобы это читалось: в 3 часа ночи файл откроет ваш будущий дежурный.

---

## Бонусное задание — автоматизация на Ansible (2 балла)

Расширьте роль `roles/monitoring` из Lab 6 и Lab 7, чтобы один playbook выкатывал **логи и метрики** вместе.

`ВАША ЗАДАЧА`: обновите роль так, чтобы она:

- **Шаблонизировала** `prometheus/prometheus.yml` через Jinja2. Интервал scrape, retention **и список таргетов** становятся переменными роли (они живут в `defaults/main.yml`). Один и тот же шаблон верно рисует и 4 таргета, и 14.
- **Дописала** шаблон `docker-compose.yml` из Lab 7: сервис `prometheus` и том `prometheus-data`.
- **Положила provisioning** источника данных Prometheus в Grafana через `grafana/provisioning/datasources/`.
- **Положила provisioning** экспортированного JSON дашборда RED через `grafana/provisioning/dashboards/`.
- **Дождалась** Prometheus на `:9090/-/healthy`, и только потом сообщала об успехе. Тот же приём, что ожидание готовности в Lab 7, другой URL.
- Была **идемпотентной**: второй прогон даёт `changed=0`.

Подсказок меньше, чем в заданиях 1–5. Имена переменных, устройство шаблона и опрос готовности придумайте сами. Бонус Lab 7 уже закрыл механику деплоя через `docker_compose_v2` и цикл ожидания готовности.

**Переменные, без которых не обойтись** (цитатой не копируйте — объявите в своём `defaults/main.yml`):
- `prometheus_version` (например, `v3.11.3`), `prometheus_port` (`9090`)
- `prometheus_retention_time` (`15d`), `prometheus_retention_size` (`2GB`)
- `prometheus_scrape_interval` (`15s`)
- `prometheus_targets` — список словарей `{job, targets}`, чтобы `{% for %}` в Jinja2 собрал конфиг scrape

**Доказательства (вставьте в `docs/LAB08.md`):**

- Вывод первого прогона (changes > 0) и второго (`changed=0`).
- **Отрендеренный** (не шаблон) `prometheus.yml` с реального прогона.
- Скриншот Grafana: **оба** источника (Loki и Prometheus) работают на трафике.
- Путь: `ansible/roles/monitoring/` + `ansible/playbooks/deploy-monitoring.yml`.

---

## Как сдавать

```bash
git switch -c lab08
git add app_python/                                # instrumentation + requirements
git add monitoring/                                # compose + prometheus.yml + dashboard
git add ansible/roles/monitoring \
        ansible/playbooks/deploy-monitoring.yml    # only if bonus done
git commit -m "feat(lab08): prometheus metrics + RED dashboard"
git push -u origin lab08
```

Откройте **два** PR:

- `your-fork:lab08` → `course-repo:master` *(на ревью)*
- `your-fork:lab08` → `your-fork:master` *(вливается в ваш main)*

Чеклист PR:

```text
- [ ] Task 1 done — Counter + Histogram + /metrics + before/after hooks; /metrics excluded from self-counting
- [ ] Task 2 done — prometheus service in compose, prometheus.yml filled, all targets `up`
- [ ] Task 3 done — 6-panel RED dashboard, JSON committed, real PromQL response captured
- [ ] Task 4 done — limits, retention, healthcheck healthy, persistence proven
- [ ] Task 5 done — LAB08.md with all 8 sections + captures
- [ ] Bonus done — idempotent Ansible role with readiness wait, both data sources working
```

---

## Критерии приёмки

### Задание 1 — Инструментирование (3 балла)
- ✅ Объявлен Counter `app_requests_total` с метками `method, endpoint, status`
- ✅ Объявлен Histogram `app_request_duration_seconds` с метками `method, endpoint` и бакетами SRE по умолчанию
- ✅ Хуки `before_request` / `after_request` видят **каждый** запрос, кроме `/metrics`
- ✅ `/metrics` отдаёт корректный текстовый формат Prometheus и верный `Content-Type`
- ✅ `/metrics` сам себя **не** считает (`grep -c 'endpoint="/metrics"' = 0`)
- ✅ В `LAB08.md` есть три письменных ответа из §1.1

### Задание 2 — Scrape Prometheus (3 балла)
- ✅ Сервис `prometheus` добавлен в сеть `logging`, с флагами retention и healthcheck
- ✅ В `prometheus.yml` заполнены `global` и 4 scrape job из каркаса
- ✅ `curl -s localhost:9090/api/v1/targets | jq` показывает **каждый** job в состоянии `up`, и `app: up` хорошо видно
- ✅ В `LAB08.md` есть три письменных ответа из §2.1

### Задание 3 — Дашборд RED (2 балла)
- ✅ 6 панелей с подходящими визуализациями; заголовок описывает вопрос, а не запрос
- ✅ PromQL для Q1–Q4 и запрос heatmap панели 4 написаны студентом (из лекции не скопированы)
- ✅ Настоящий ответ PromQL с трафика студента снят через `/api/v1/query`
- ✅ JSON дашборда закоммичен в `monitoring/grafana/dashboards/lab08.json`

### Задание 4 — Продакшен-конфиг (1 балл)
- ✅ У `prometheus` есть лимиты ресурсов и reservations
- ✅ Флаги retention (`time` и `size`) заданы в командной строке
- ✅ Healthcheck показывает `(healthy)`; история метрик переживает `down` / `up`

### Задание 5 — Документация (1 балл)
- ✅ В `monitoring/docs/LAB08.md` есть все 8 разделов
- ✅ Ответы своими словами студента, не цитаты из лекции

### Бонус — Ansible (2 балла)
- ✅ `prometheus.yml` шаблонизируется из переменной-списка `prometheus_targets`
- ✅ Источник данных и дашборд приезжают provisioning-ом (в UI Grafana кликать не нужно)
- ✅ Деплой идемпотентный: второй прогон даёт `changed=0`
- ✅ Ожидание готовности не пускает успех, пока Prometheus и Grafana не ответят
- ✅ Оба источника видны в Grafana

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Инструментирование | **3** | Counter и Histogram объявлены верно; хуки видят запросы кроме `/metrics`; формат экспозиции корректный |
| **Задание 2** — Scrape Prometheus | **3** | Сервис в compose, конфиг scrape заполнен, `/api/v1/targets` показывает `app: up` |
| **Задание 3** — Дашборд RED | **2** | 6 панелей с PromQL, который написал студент; снят настоящий ответ запроса; JSON закоммичен |
| **Задание 4** — Продакшен-конфиг | **1** | Лимиты, retention, healthcheck, персистентность |
| **Задание 5** — Документация | **1** | Все 8 разделов, снятые данные на месте, ответы своими словами студента |
| **Бонус** — Ansible | **2** | Идемпотентный деплой из шаблонов всего стека логов и метрик |
| **Итого** | **12** | 10 основных + 2 бонуса |

---

## Материалы

<details>
<summary>📊 Prometheus</summary>

- [Обзор Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Анонс Prometheus 3.0](https://prometheus.io/blog/2024/11/14/prometheus-3-0/) — UTF-8, native histograms, OTLP
- [Справочник конфигурации](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [`scrape_config`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)
- [Типы метрик](https://prometheus.io/docs/concepts/metric_types/)
- [Native histograms](https://prometheus.io/docs/specs/native_histograms/)

</details>

<details>
<summary>🐍 Инструментирование на Python</summary>

- [`prometheus_client` (GitHub)](https://github.com/prometheus/client_python)
- [Руководство по инструментированию на Python](https://prometheus.io/docs/guides/python/)
- [Практики инструментирования](https://prometheus.io/docs/practices/instrumentation/)
- [Имена метрик](https://prometheus.io/docs/practices/naming/)

</details>

<details>
<summary>📈 PromQL и Grafana</summary>

- [Основы PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Примеры PromQL](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- [Источник данных Prometheus в Grafana](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
- [Провижининг дашбордов](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)

</details>

<details>
<summary>📚 Методы наблюдаемости</summary>

- [Метод RED (Tom Wilkie, Grafana)](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/)
- [Метод USE (Brendan Gregg)](https://www.brendangregg.com/usemethod.html)
- [Четыре золотых сигнала (Google SRE)](https://sre.google/sre-book/monitoring-distributed-systems/)

</details>

<details>
<summary>⚠️ Частые ловушки (из реальных пробных прогонов)</summary>

- **Ловушка: `/metrics` считает сам себя.** Если хук `after_request` пишет ответ на сам `/metrics`, каждый scrape увеличивает `app_requests_total{endpoint="/metrics",...}`. При интервале scrape 5 секунд это двенадцать фальшивых «запросов в минуту», и они топят настоящий трафик. Путь `/metrics` **всегда** исключайте до инкремента. Проверка: `curl … | grep -c 'endpoint="/metrics"'` возвращает `0`. Это баг инструментирования №1. В заметках к оценке эталонной сдачи его проверяют явно.
- **Кардинальность меток = OOM Prometheus.** Каждая уникальная комбинация меток — временной ряд (~3–5 КБ RAM на ряд). Метка со значением `user_id`, `request_id`, `trace_id`, сырой путь с id (`/user/123`), почта или метка времени — самый быстрый способ уронить ingester. Для метки `endpoint` **берите `request.url_rule.rule`**. `request.path` оставляет `/user/123` и `/user/456` двумя разными рядами. Правило `/user/<id>` схлопывает их в один. Тот же урок, что метки Loki в Lab 7.
- **Путаница Counter и Gauge.** Counter только растёт (и обнуляется при рестарте процесса). Если хочется `.dec()` или `.set()`, вы взяли counter там, где нужен gauge. Сырое значение counter — это «сколько запросов прошло с загрузки». На графике это почти никогда не нужно. Всегда оборачивайте в `rate()` или `increase()`.
- **Выбор бакетов гистограммы.** Бакеты гистограммы `prometheus_client` по умолчанию заканчиваются на **10 секундах**. Если вы меряете запросы к базе, которые могут идти 30 с, каждый медленный запрос падает в бакет `+Inf`, и p99 врёт. Бакетите гистограмму под *ожидаемый* диапазон и оставьте один бакет за потолком допуска. Старт — набор SRE `(.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10)`, дальше подкрутка по данным из прода. Симметричная ловушка — бакеты слишком узкие снизу: каждый быстрый запрос садится в самый маленький бакет, и 1 мс не отличить от 50 мкс.
- **DNS по имени сервиса в Compose.** Prometheus ходит к таргетам по hostname через HTTP. Внутри сети Compose `logging` hostname — это **имя сервиса** (`app`, `loki`, `grafana`). Не `localhost` и не IP хоста. Единственный законный `localhost` — self-scrape Prometheus: трафик не выходит из его контейнера. Ошибётесь здесь — `up{job="app"}` вернёт 0, и в UI явной ошибки не будет. Смотрите `docker compose logs prometheus | grep -i scrape`.
- **Retention — флаг CLI, а не настройка в файле.** На этом спотыкаются все, кто переносит статью про 2.x. Ключ `retention_period` в `prometheus.yml` **игнорируется**. Ставьте `--storage.tsdb.retention.time=15d` и `--storage.tsdb.retention.size=2GB` в списке `command:`. Оба флага могут стоять вместе. Срабатывает тот, который наступит раньше.
- **`scrape_interval` и окно PromQL разъехались.** При `scrape_interval: 15s` выражение `rate(...[15s])` вырождено: одна точка или ноль, график пустой. Минимальное окно с осмысленным результатом — `≥ 4× scrape_interval`. Scrape раз в 5 с → спокойно `[1m]`. Scrape раз в 15 с → пол это `[5m]`.
- **`histogram_quantile` снаружи, `sum by (le)` внутри. Наоборот собирать нельзя.** Рабочий запрос перцентиля: `histogram_quantile(0.95, sum by (endpoint, le) (rate(*_bucket[5m])))`. Завернуть `histogram_quantile` *внутрь* агрегатора (например, `avg(histogram_quantile(...))`) — математический нонсенс. Среднее квантилей квантилем не становится. Соберите так один раз, посмотрите на результат, запомните.
- **`prometheus-client` версии >=0.23 пишет `# HELP`, `# TYPE` и `# UNIT`.** Prometheus разбирает все три строки. Свой текст `/metrics` руками, без строки `# TYPE`, Prometheus молча считает нетипизированным gauge, и `rate()` по нему вернёт NaN. Всегда берите `generate_latest()`.

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- [`promtool`](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) — лежит в образе Prometheus; `promtool check config /etc/prometheus/prometheus.yml` проверяет конфиг до reload
- [`hey`](https://github.com/rakyll/hey) — ровный трафик на приложение, чтобы наполнить дашборды (`hey -n 5000 -c 10 http://localhost:8080/`)
- [`jq`](https://jqlang.github.io/jq/) — разбор ответов `/api/v1/targets` и `/api/v1/query` в CLI

</details>

---

## Что дальше

| Lab | Что это добавляет сервису |
|---:|---|
| 9 | k3d Kubernetes: задеплойте инструментированное приложение и обвязку `echo` как Pod, scrape через sidecar |
| 10 | Helm-чарт Helm 4: упакуйте приложение и values для Prometheus |
| 12 | ConfigMap и PVC: конфиг Prometheus заново из ConfigMap; TSDB лежит на PVC |
| 16 | Helm-чарт `kube-prometheus-stack` с CRD `ServiceMonitor`, которые сами находят приложение. Тот же PromQL, обвязка уже родная для K8s |

---

**Удачи!** 🚀

> **Запомните:** счётчикам нужен `rate()`. Перцентилям нужен `histogram_quantile()` *снаружи* `sum by (le)`. Метка не должна быть уникальной на каждый запрос и на каждого пользователя. **RED — каждому сервису. USE — каждому ресурсу.** И `/metrics` не должен считать сам себя.
