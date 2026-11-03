# Lab 7 — Наблюдаемость и логирование на стеке Loki

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Logging%20%26%20Observability-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Loki%20|%20Alloy%20|%20Grafana-informational)

> **Цель:** поднимите Loki + Grafana Alloy + Grafana через Docker Compose, отправьте логи Python-приложения из Lab 1 структурированным JSON и научитесь спрашивать эти логи на LogQL.
> **Что сдаёте:** PR (pull request) из `lab07`: каталог `monitoring/` (compose-стек, конфиги Alloy и Loki, JSON дашборда), патч JSON-логирования в `app_python/` и `monitoring/docs/LAB07.md` со снятыми доказательствами.

---

## Обзор

В этой лабораторной вы потренируетесь:
- Читать River-синтаксис Alloy и **связывать компоненты по имени** (`discovery.docker` → `discovery.relabel` → `loki.source.docker` → `loki.write`), а не копировать готовый YAML-pipeline (цепочка обработки логов)
- Собрать стек Docker Compose по списку сервисов: какие образы, какие порты, какие тома, какая сеть
- Писать структурированные JSON-логи из Python-сервиса
- Писать на **LogQL** селекторы потоков, фильтры строк, парсеры и функции метрик, чтобы отвечать на реальные вопросы о своём приложении
- Укрепить стек: лимиты ресурсов, compactor для retention (срок хранения), health-проверки, Grafana без анонимного доступа

> ⚠️ **Границы:** Loki в режиме single-binary на локальном Docker-хосте. Без объектного хранилища, без multi-tenancy, без TLS. Это вернётся в лабораторных эпохи K8s (12, 16). Под прод не тюньте. Тюньте так, чтобы понять, как компоненты связаны.

> 🪦 **Заметка про Promtail.** До конца 2025 стандартным агентом Loki был **Promtail**. **2 марта 2026** у него наступил конец поддержки (End-of-Life): новых релизов нет, исправлений безопасности тоже. Преемник, **Grafana Alloy**, собирает в один бинарник Promtail, агент Prometheus и OpenTelemetry Collector. В этой лабораторной используется Alloy. Конфиги Promtail ещё встречаются в унаследованных системах, поэтому слайды 14–15 лекции показывают старый синтаксис и как он ложится на Alloy. Новый Promtail в 2026 **не поднимайте**. Блок Promtail `pipeline_stages` в Alloy **один к одному не переносится**: эквиваленты разнесены между `loki.relabel` и `loki.process`, имена части стадий другие.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**
- `app_python/` из Lab 1 — сервис на Flask/FastAPI с `/` и `/health`
- Рабочий Docker-образ из Lab 2 (запушен в registry или с локальным тегом)
- Уверенная работа с Docker Compose v2 из Lab 6 (стек из шаблона Jinja2)

**Эта лабораторная добавляет:**
- `monitoring/docker-compose.yml` — стек Loki + Alloy + Grafana
- `monitoring/loki/config.yml` — готовый каркас
- `monitoring/alloy/config.alloy` — **River-pipeline пишете вы**
- `monitoring/grafana/dashboards/lab07.json` — ваш экспортированный дашборд на 4 панели
- `monitoring/docs/LAB07.md` — отчёт для сдачи
- Доработка JSON-логирования в `app_python/app.py`

К Lab 16 вы задеплоите ту же схему на Kubernetes: `kube-prometheus-stack` плюс Helm-чарт Loki. То, что разберёте на этой неделе (кардинальность меток, LogQL, конфиги, смонтированные из ConfigMap), пригодится дальше.

---

## Подготовка

Версии в этой лабораторной. Зафиксируйте их в compose-файле:

| Компонент | Тег | Вышел |
|---|---|---|
| `grafana/loki` | `3.7.0` | март 2026 |
| `grafana/alloy` | `v1.16.1` | 6 мая 2026 |
| `grafana/grafana` | `13.0.1` | 12 мая 2026 |

```bash
docker --version           # 28.x or 29.x
docker compose version     # v2.x — note the space, not a hyphen
curl --version             # for the verification commands below
```

Создайте дерево каталогов (файлы заполните сами):

```
monitoring/
├── docker-compose.yml             # YOU write this (§1.4)
├── loki/
│   └── config.yml                 # provided plumbing — copy from labs/lab07/loki/
├── alloy/
│   └── config.alloy               # YOU write this (§1.3)
├── grafana/
│   └── dashboards/
│       └── lab07.json             # exported from Grafana UI in Task 3
└── docs/
    └── LAB07.md                   # your submission report
```

Каркас из репозитория курса для этой лабораторной:
- `labs/lab07/loki/config.yml` — готовый конфиг Loki, кладётся как есть (см. §1.2)
- `plumbing/echo/` — необязательный второй источник логов (см. §2.2)

---

## Задание 1 — Поднимите стек Loki + Alloy + Grafana (3 балла)

### 1.1 — Сначала прочитайте, потом пишите

Прочитайте это, прежде чем трогать конфиг. Здесь ответы на вопросы из блоков ВАША ЗАДАЧА:
- [Обзор Loki](https://grafana.com/docs/loki/latest/get-started/overview/) — distributor / ingester / querier; чанки и индекс
- [Компоненты Alloy](https://grafana.com/docs/alloy/latest/) — `discovery.*`, `loki.source.*`, `loki.write`, как они ссылаются друг на друга по имени
- [Введение в LogQL](https://grafana.com/docs/loki/latest/query/) — селекторы, фильтры, парсеры, метрические запросы

`ВАША ЗАДАЧА`: в `docs/LAB07.md` ответьте на три вопроса, по 2–4 предложения на каждый. По ходу сборки вернётесь и поправите:

1. Чем индекс **меток** (label) Loki отличается от полнотекстового индекса Elasticsearch и почему при 100 ГБ/день это дешевле?
2. Что такое **поток** (stream) в Loki и что будет с памятью, если сделать `user_id` меткой на сервисе с 10 млн пользователей?
3. Компоненты Alloy ссылаются друг на друга по **имени** (например, `loki.write.default.receiver`). Как называется имя export и почему оно отличается от имени блока (`loki.write "default"`)?

### 1.2 — Конфиг Loki (готовый каркас — не переписывайте)

Конфиг Loki — **единственный** полностью написанный файл в этой лабораторной. В репозитории курса он лежит как готовый каркас. Скопируйте его в свой стек:

```bash
cp labs/lab07/loki/config.yml monitoring/loki/config.yml
```

Писать его не нужно. В документации вы **должны** уметь объяснить:
- Почему `store: tsdb` + `schema: v13` (и что это сменило)
- Почему `object_store: filesystem` для прода не годится
- Почему блок `compactor` обязателен, когда задан `retention_period`

> ⚠️ **Ловушка, которую надо запомнить:** `retention_period` без блока `compactor` — логи не удаляются никогда. Loki молча глотает лимит. Это причина №1, по которой команды пробивают дисковый бюджет. Сошлитесь на это в §3 документации («Выбор конфигурации»).

### 1.3 — River-конфиг Alloy (ВАША ЗАДАЧА)

**Файл:** `monitoring/alloy/config.alloy`

Alloy — это связанные компоненты, а не YAML-pipeline. Четыре блока, каждый кормит следующий. Имена блоков ниже менять нельзя. Навык — в аргументах, метках и ссылках между ними.

`ВАША ЗАДАЧА`: заполните пропуски. Имя блока говорит, какую роль он играет. Комментарии задают контракт.

```alloy
// 1) Discover Docker containers via the daemon socket
discovery.docker "containers" {
  host             = ___           // YOUR TASK: docker socket URL
  refresh_interval = "5s"
  ___                              // YOUR TASK: add a `filter { ... }` block — only
                                   //   scrape containers labelled `logging=alloy`
}

// 2) Promote useful Docker metadata to Loki labels — keep cardinality LOW
discovery.relabel "containers" {
  targets = ___                    // YOUR TASK: previous block's exported targets
  rule {
    source_labels = [___]          // YOUR TASK: Docker meta-label for container name
    regex         = ___            // YOUR TASK: strip the leading "/"
    target_label  = "container"
  }
  ___                              // YOUR TASK: second `rule { ... }` — copy Docker
                                   //   label `app` into Loki label `app`
}

// 3) Tail the discovered containers' log streams
loki.source.docker "default" {
  host       = ___                 // YOUR TASK: same socket as block 1
  targets    = ___                 // YOUR TASK: feed from the *relabel* block
  forward_to = [___]               // YOUR TASK: which loki.write receiver?
}

// 4) Ship everything to Loki
loki.write "default" {
  endpoint {
    url = ___                      // YOUR TASK: Loki push URL — which host, port, path?
  }
}
```

Заметки по каждому блоку. Не пропускайте: это *зачем* у пропусков.

- **Блок 1.** `filter` делает сбор логов opt-in. Без него Alloy отправит stdout каждого контейнера, включая свой и Loki, и дашборды замусорятся.
- **Блок 2.** `discovery.relabel` переписывает метаданные меток до снятия логов. Первое правило приводит имя контейнера к нормальному виду (Docker ставит впереди `/`). Второе копирует пользовательскую метку Docker в метку Loki. **Остановитесь и подумайте о кардинальности**, прежде чем добавлять третье правило. Всё, у чего больше ~50 разных значений, сюда не кладут.
- **Блок 3.** `loki.source.docker` читает stdout/stderr контейнера с хвоста. В `targets` должен прийти выход **relabel**, а не сырой discovery. Иначе метки в Loki не попадут.
- **Блок 4.** `loki.write` — единственный блок, который ходит по HTTP. URL эндпоинта решает, переедет ли стек в другую сеть. Берите имя сервиса compose, а не `localhost`.

<details>
<summary>💡 Подсказки по синтаксису River</summary>

- Блок записывается как `<kind>.<subkind> "label" { … }`. **Label** в кавычках (например, `"containers"`, `"default"`) — так на блок ссылаются остальные.
- У каждого компонента есть задокументированные **exports**. `discovery.docker` отдаёт `.targets`. `discovery.relabel` отдаёт `.output`. `loki.write` отдаёт `.receiver`. Связь пишется полным именем export, например `discovery.relabel.containers.output`.
- Alloy поднимает живой UI на порту **12345** и показывает граф компонентов. Порванная связь будет красной в UI раньше, чем вы заметите её в Loki.
- Справочник: [`discovery.docker`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.docker/), [`discovery.relabel`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/), [`loki.source.docker`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/), [`loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/).
- Путь push-эндпоинта Loki описан [здесь](https://grafana.com/docs/loki/latest/reference/loki-http-api/#ingest-logs).

</details>

### 1.4 — Стек Docker Compose (ВАША ЗАДАЧА)

**Файл:** `monitoring/docker-compose.yml`

В стеке **четыре** сервиса. Теги образов и порты заданы, версии зафиксированы. Тома, членство в сети, переменные окружения и `depends_on` — ваша работа. Без этих связей стек не заведётся.

| Сервис | Образ | Проброшенные порты | Зачем он здесь |
|---|---|---|---|
| `loki` | `grafana/loki:3.7.0` | `3100:3100` | Хранилище и движок запросов |
| `alloy` | `grafana/alloy:v1.16.1` | `12345:12345` (живой UI) | Коллектор: читает docker.sock и пушит в Loki |
| `grafana` | `grafana/grafana:13.0.1` | `3000:3000` | Дашборды и UI Explore |
| `app` | ваш образ из Lab 2 | `8080:8080` | Источник логов: сервис Lab 1 с JSON на выходе (задание 2) |

`ВАША ЗАДАЧА`: напишите compose-файл. Разобраться нужно вот в чём:

1. **Тома `loki`.** Один bind mount делает конфиг из §1.2 видимым внутри контейнера по пути `/etc/loki/config.yml`. Один именованный том хранит `/loki`, чтобы чанки пережили рестарт. Команда контейнера: `-config.file=/etc/loki/config.yml`.
2. **Тома `alloy`.** Один bind mount конфига из §1.3, **только на чтение**. Одно монтирование Docker-сокета хоста, тоже **только на чтение**: Alloy должен находить контейнеры и читать их логи. В команде Alloy три флага: `run`, `--server.http.listen-addr=0.0.0.0:12345`, `--storage.path=/var/lib/alloy/data`, затем путь к конфигу.
3. **Переменные `grafana`.** Только в этом задании включите анонимный доступ с ролью администратора (`GF_AUTH_ANONYMOUS_ENABLED`, `GF_AUTH_ANONYMOUS_ORG_ROLE`). **В задании 4 вы это выключите.** Комментарий оставьте сразу, чтобы не забыть. Положите `/var/lib/grafana` в именованный том: дашборд из задания 3 должен пережить `docker compose down`.
4. **Метки `app`.** На контейнере должны быть `logging: "alloy"` (включает сбор Alloy через ваш `filter` из §1.3) и `app: "devops-python"` (становится меткой Loki через правило relabel из §1.3). Нет **обеих** меток — дашборды в задании 3 пустые.
5. **Сеть.** Опишите пользовательскую bridge-сеть, назовите её `logging` и посадите на неё **все четыре** сервиса. Тогда они резолвятся по имени сервиса. `loki` и `grafana` ждут зависимости перед стартом (`depends_on:`).

```yaml
# monitoring/docker-compose.yml — YOUR TASK
services:
  loki:
    image: grafana/loki:3.7.0
    command: ___                              # YOUR TASK: -config.file=/etc/loki/config.yml
    ports:
      - "3100:3100"
    volumes:
      - ___                                   # YOUR TASK: bind-mount loki/config.yml
      - ___                                   # YOUR TASK: named volume for /loki
    networks: [___]                           # YOUR TASK

  alloy:
    image: grafana/alloy:v1.16.1
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - ___                                   # YOUR TASK: path to mounted config.alloy
    ports:
      - "12345:12345"
    volumes:
      - ___                                   # YOUR TASK: bind-mount config.alloy :ro
      - ___                                   # YOUR TASK: bind-mount the Docker socket :ro
    networks: [___]                           # YOUR TASK: same network as loki
    depends_on: [___]                         # YOUR TASK: must start after which service?

  grafana:
    image: grafana/grafana:13.0.1
    ports:
      - "3000:3000"
    environment:
      - ___                                   # YOUR TASK: enable anonymous access (DEV ONLY — Task 4 turns this off)
      - ___                                   # YOUR TASK: anonymous role = Admin
    volumes:
      - ___                                   # YOUR TASK: persist /var/lib/grafana
    networks: [___]                           # YOUR TASK
    depends_on: [___]                         # YOUR TASK

  app:
    image: ___                                # YOUR TASK: your Lab 2 image (or `build:` block)
    ports:
      - "8080:8080"
    labels:
      ___: ___                                # YOUR TASK: opt into Alloy collection
      ___: ___                                # YOUR TASK: the `app` Loki label (matches §1.3 rule)
    networks: [___]                           # YOUR TASK

volumes:
  ___:                                        # YOUR TASK: declare named volume for loki
  ___:                                        # YOUR TASK: declare named volume for grafana

networks:
  ___:                                        # YOUR TASK: declare the user-defined bridge
    name: logging
```

> ⚠️ **Про безопасность:** монтирование `/var/run/docker.sock` даёт Alloy **широкий** обзор хоста. По сути это root на демоне. Для лабораторной нормально. В проде берут socket proxy (`docker-socket-proxy` от Tecnativa) или file-based discovery. Этот компромисс запишите в документацию.

### 1.5 — Поднимите стек и проверьте

```bash
cd monitoring
docker compose up -d
docker compose ps
```

Проверьте:

```bash
# Loki — ingester takes ~15s after process start to report ready.
# If you see "Ingester not ready" the first time, wait and retry; do NOT debug.
curl -s http://localhost:3100/ready          # expect: "ready"

# Loki's view of discovered labels — once Alloy is wired correctly, you should
# see at LEAST "container" and (after §2.2) "app" in the list:
curl -s http://localhost:3100/loki/api/v1/labels | jq -c .data
# (illustrative — your set will differ)
# ["container","service_name"]

# Alloy live component graph — open in your browser
#   macOS:  open http://localhost:12345
#   Linux:  xdg-open http://localhost:12345   (or just paste the URL)

# Grafana — open in your browser at http://localhost:3000
# Add data source: Connections → Data sources → Loki → URL http://loki:3100 → Save & Test
```

### 1.6 — Доказательство работы

**Вставьте в `docs/LAB07.md`:**

- Вывод `docker compose ps`: все четыре сервиса в состоянии `Up` (после задания 4 у loki и grafana ещё и `(healthy)`; для alloy и app healthcheck не обязателен)
- Вывод `curl -s http://localhost:3100/ready` — буквально слово `ready`
- Вывод `curl -s http://localhost:3100/loki/api/v1/labels | jq -c .data` — в списке есть `container` (и `app` после §2.2)
- Скриншот живого UI Alloy на `:12345`: четыре блока связаны, красных рёбер нет
- Скриншот Grafana **Explore** с запросом `{container=~".+"}`: логи хотя бы из двух контейнеров

---

## Задание 2 — Подключите приложение и отправьте JSON-логи (3 балла)

### 2.1 — Добавьте JSON-логирование в Python-приложение из Lab 1

Выберите **одну** библиотеку. Обе подходят: обе пишут JSON, который Loki разбирает через `| json`.

| Библиотека | Когда брать | Компромисс |
|---|---|---|
| **`python-json-logger`** | Ставится поверх `logging.basicConfig`. Правок в приложении минимум. | Модель stdlib `logging`: handlers, formatters, уровни. |
| **`structlog`** | Новый код, где нужны переменные контекста, процессоры и типизированные события. | Другой API, удобнее в работе. |

`ВАША ЗАДАЧА`: доработайте `app_python/app.py`, чтобы события ниже писали в stdout **одну JSON-строку на событие**. Именно stdout снимает Docker, и именно его увозит Alloy:

| Событие | Когда | Обязательные поля JSON |
|---|---|---|
| **Старт** | При старте процесса | `level`, `msg`, `host`, `port`, `service` |
| **Каждый HTTP-запрос** | После ответа | `level`, `msg`, `method`, `path`, `status`, `client_ip`, `duration_ms` |
| **Каждая ошибка** | В обработчике 500 или в `try/except` | `level="ERROR"`, `msg`, `error_type`, строка исключения |

Подсказки:

- Flask: `@app.before_request` засекает время старта, `@app.after_request` пишет лог. Либо возьмите встроенный логгер `werkzeug` и подмените его formatter на JSON.
- FastAPI: один middleware со скобками `time.perf_counter()` чище, чем лог на каждом маршруте.
- **Не** кладите `request_id` и `user_id` в **метку**. Им место в теле JSON. Фильтр на запросе: `| json | user_id="alice"`. (Lecture 7, слайд 12.)
- Один JSON-объект на строку. Без висячих запятых. Без pretty-print. Иначе `docker logs` разрежет запись по каждому внутреннему переносу строки, и дашборды развалятся.

Обновите `app_python/requirements.txt`:
```
# python-json-logger>=3.4.0    # YOUR TASK: pin one of these to an exact version
# structlog>=25.4.0
```

Пересоберите образ (Dockerfile из Lab 2), чтобы новая зависимость оказалась в контейнере.

### 2.2 — Добавьте приложение в стек

Сервис `app` вы уже описали в §1.4. Две метки обязательны:

```yaml
    labels:
      logging: "alloy"          # opts the container into Alloy collection (§1.3 filter)
      app: "devops-python"      # becomes a Loki label via the §1.3 relabel rule
```

**По желанию, но лучше сделать — `plumbing/echo`.** В репозитории курса лежит маленький Go-сервис `plumbing/echo` (он снова появится в Lab 9). Добавьте его вторым источником логов, чтобы дашборд смотрел не на одно приложение. Те же две метки, порт `8081`, образ или строка `build:` — на ваш выбор.

### 2.3 — Нагоните трафик и выполните запрос

```bash
# Generate at least 20 successful + 5 not-found + 1 error event
for i in $(seq 1 20); do curl -s http://localhost:8080/ > /dev/null; done
for i in $(seq 1 5);  do curl -s http://localhost:8080/nope > /dev/null; done
# Force an error: hit an endpoint that raises (add a /boom route or trip a 500 deliberately)
curl -s http://localhost:8080/boom > /dev/null
```

### 2.4 — Напишите запросы LogQL (ВАША ЗАДАЧА)

Для каждой строки напишите LogQL, который на неё отвечает. Из лекции не копируйте: сначала ответьте на вопрос, потом проверьте запрос в Explore.

| # | Вопрос | Подсказка |
|---|---|---|
| Q1 | Все логи вашего Python-приложения | Селектор потока по метке `app`, больше ничего |
| Q2 | Только JSON-строки, где `level` равен `ERROR` | Селектор потока → `\| json` → равенство поля `level` |
| Q3 | Частота ошибок в минуту, по контейнерам | Метрический запрос: `sum by (container) (rate(... [1m]))`. После селектора потока цепочка `\| json \| level="ERROR"`, затем всё лог-выражение оберните в `rate(...[1m])` |
| Q4 | Все запросы к пути, в котором есть `health` (фильтр по **полю JSON `path`**, а не по подстроке сырой строки) | `\| json`, затем равенство поля или regex-фильтр по `path` |

> 💡 **Порядок по цене:** селектор потока → фильтр строки → парсер → фильтр поля → метрика. Сначала самый дешёвый фильтр. Фильтры идут слева направо.

### 2.5 — Доказательство работы

**Вставьте в `docs/LAB07.md`:**

- Одна полная JSON-строка из `docker logs <app-container> | head -n 1`. В ней все обязательные поля из §2.1
- Вывод `curl -s http://localhost:3100/loki/api/v1/label/container/values | jq -c .data`. В списке есть имя контейнера вашего приложения
- Вывод прогона Q1–Q4 через HTTP query API Loki. Проверяющий должен видеть данные, а не только скриншот:
  ```bash
  curl --get http://localhost:3100/loki/api/v1/query_range \
       --data-urlencode 'query=<YOUR QUERY HERE>' \
       --data-urlencode 'limit=5' | jq '.data.result[0].values[0]'
  ```
  Покажите, что каждый из Q1–Q4 возвращает хотя бы одну строку лога *вашего* приложения. (Q3 возвращает числовой ряд, а не строку лога. Для него покажите `.data.result[0]`.)
- Сами четыре запроса LogQL в блоке кода и по одному предложению, что делает каждый

---

## Задание 3 — Соберите дашборд логов (2 балла)

### 3.1 — Один вопрос на панель

Дашборд отвечает на **один вопрос** для **одной аудитории**. Четыре панели ниже закрывают самые частые операционные вопросы по одному сервису. Запросы LogQL из задания 2 покрывают почти всё нужное. Один запрос новый.

| Панель | Визуализация | На какой вопрос отвечает | Откуда запрос |
|---|---|---|---|
| 1 | Logs | *«Что происходит прямо сейчас?»* | Ваш Q1 (если приложений несколько, расширьте до `app=~"devops-.*"`) |
| 2 | Time series | *«Насколько загружено каждое приложение?»* | **Частота** запросов по приложениям (напишите сами: та же форма, что у Q3, без фильтра ошибок) |
| 3 | Logs | *«Что сломалось?»* | Ваш Q2 |
| 4 | Pie chart или Stat | *«Насколько шумный каждый уровень?»* | Распределение `level` за 5 минут: `count_over_time` + `| json` + `sum by (level)` |

`ВАША ЗАДАЧА`: LogQL для панелей 2 и 4 напишите сами (из §2.4 не копируйте). Панели 1 и 3 берут Q1 и Q2 дословно.

### 3.2 — Соберите дашборд

1. **Dashboards → New → New dashboard → Add visualization**. Источник данных — **Loki**.
2. Вставьте LogQL. Тип визуализации возьмите из таблицы выше.
3. Название панели — **вопрос, на который она отвечает**, а не источник данных. «Ошибки в минуту» сильнее, чем «Loki query #3».
4. Сохраните дашборд. Дальше **Share → Export → Save to file** и коммит JSON-модели в `monitoring/grafana/dashboards/lab07.json`.

> 💡 Пустая панель почти всегда значит, что не хватает меток. Сначала прогоните запрос панели в **Explore**. Данные в Explore есть, а панель пустая — смотрите диапазон времени и источник данных по умолчанию.

### 3.3 — Доказательство работы

**Вставьте в `docs/LAB07.md`:**

- Скриншот дашборда из 4 панелей с живыми данными на каждой (свежий трафик — прямо перед скриншотом; пустая панель баллов не даёт)
- Четыре запроса LogQL в markdown-таблице и вопрос к каждому
- Путь к закоммиченному JSON: `monitoring/grafana/dashboards/lab07.json`

---

## Задание 4 — Готовность к проду (1 балл)

Укрепите стек. В текущем виде это ещё игрушка.

### 4.1 — Лимиты ресурсов

`ВАША ЗАДАЧА`: добавьте `deploy.resources.limits` (cpus и memory) и `reservations` каждому из четырёх сервисов. Числа берите разумные. Loki и Grafana нужно больше памяти, чем Alloy и приложению. Приложению память почти не нужна. Выбор запишите в `LAB07.md`.

### 4.2 — Закройте доступ к Grafana

`ВАША ЗАДАЧА`:
- Поставьте `GF_AUTH_ANONYMOUS_ENABLED=false` (комментарий DEV-ONLY в §1.4 как раз про это).
- Пароль администратора возьмите из файла `.env`: `GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}`.
- Добавьте `.env` в **`.gitignore`** и закоммитьте изменение `.gitignore`.
- Закоммитьте `.env.example` с именем переменной и пустым плейсхолдером, чтобы проверяющий видел контракт.

### 4.3 — Health-проверки

`ВАША ЗАДАЧА`: добавьте блок `healthcheck:` сервисам **Loki** (проверка `:3100/ready`) и **Grafana** (проверка `:3000/api/health`). Внутри контейнера используйте `wget --spider` или `curl --fail`. Подберите разумные `interval`, `timeout`, `retries` и `start_period`. Loki после старта готов примерно через 15 с, и `start_period` должен дать эту паузу.

### 4.4 — Доказательство работы

**Вставьте в `docs/LAB07.md`:**

- Вывод `docker compose ps`, где и Loki, и Grafana в состоянии `(healthy)`
- Скриншот **страницы входа** Grafana (анонимный доступ выключен)
- Содержимое `monitoring/.env.example` и подтверждение, что `.env` есть в `.gitignore` и **не** закоммичен (пустой вывод `git log -- monitoring/.env` — достаточное доказательство)

---

## Задание 5 — Документация (1 балл)

`ВАША ЗАДАЧА`: напишите `monitoring/docs/LAB07.md` с такими разделами, по порядку:

1. **Архитектура** — диаграмма Mermaid: app → Alloy → Loki → Grafana, на схеме отмечены сеть и Docker-сокет
2. **Запуск** — как поднять стек (`docker compose up -d`) и команды проверки из §1.5
3. **Выбор конфигурации** — три ответа из §1.1 и короткая заметка по каждому блоку Loki (schema, retention, compactor) и каждому компоненту Alloy
4. **Логирование приложения** — какую библиотеку взяли, почему и какую схему полей пишете
5. **Дашборд** — каждая панель, её LogQL, вопрос, на который она отвечает, и скриншот
6. **Продакшен-конфиг** — лимиты ресурсов и почему так, аутентификация Grafana, компромисс с Docker-сокетом
7. **Трудности и решения** — хотя бы одна настоящая (формулировка «я только познакомился с Loki» не считается)

Включите **фрагменты** конфигов (не файлы целиком) и снятое из заданий 1–4. Пишите так, чтобы это читалось: в 3 часа ночи файл откроет ваш будущий дежурный, то есть вы сами.

---

## Бонусное задание — автоматизация на Ansible (2 балла)

Автоматизируйте весь стек на Ansible. Опирайтесь на Lab 6.

`ВАША ЗАДАЧА`: создайте роль `roles/monitoring`, которая поднимает весь стек идемпотентно.

Роль должна:
- **Шаблонизировать** `loki/config.yml`, `alloy/config.alloy` и `docker-compose.yml` через Jinja2. Теги образов, порты, retention и лимиты ресурсов становятся переменными роли (они живут в `defaults/main.yml`)
- **Выкатывать** стек модулем `community.docker.docker_compose_v2`
- **Дождаться** Loki на `:3100/ready` *и* Grafana на `:3000/api/health`, и только потом сообщить об успехе (берите `ansible.builtin.uri` с повторами)
- Быть **идемпотентной**: второй прогон даёт `changed=0` (главный критерий Lab 5 всё ещё в силе)

Подсказок меньше, чем в заданиях 1–5. Раскладку каталогов, имена переменных и схему опроса готовности придумайте сами. Механику деплоя через `docker_compose_v2` вы уже разобрали в Lab 6.

**Доказательства (вставьте в `docs/LAB07.md`):**

- Вывод первого прогона (changes > 0) и второго (`changed=0`)
- **Отрендеренный** (не шаблон) `docker-compose.yml` с реального прогона
- Путь: `ansible/roles/monitoring/` + `ansible/playbooks/deploy-monitoring.yml`

---

## Как сдавать

```bash
git switch -c lab07
git add monitoring/
git add app_python/                            # JSON logging upgrade
git add ansible/roles/monitoring \
        ansible/playbooks/deploy-monitoring.yml   # only if bonus done
git commit -m "feat(lab07): loki + alloy + grafana observability stack"
git push -u origin lab07
```

Откройте **два** PR:

- `your-fork:lab07` → `course-repo:master` *(на ревью)*
- `your-fork:lab07` → `your-fork:master` *(вливается в ваш main)*

Чеклист PR:

```text
- [ ] Task 1 done — stack up, Alloy pipeline wired, container labels visible
- [ ] Task 2 done — JSON logging in app_python, 4 LogQL queries verified via /query_range
- [ ] Task 3 done — 4-panel dashboard built, JSON exported into the repo
- [ ] Task 4 done — limits, healthchecks, anonymous off, .env gitignored
- [ ] Task 5 done — LAB07.md with all 7 sections + captures + screenshots
- [ ] Bonus done — idempotent Ansible role with readiness wait
```

---

## Критерии приёмки

### Задание 1 — Деплой стека (3 балла)
- ✅ `docker compose up -d` поднимает loki, alloy, grafana, app
- ✅ `curl -s :3100/ready` возвращает `ready`
- ✅ `curl -s :3100/loki/api/v1/labels` содержит `container` (и `app` после §2.2)
- ✅ Живой UI Alloy показывает pipeline из четырёх блоков, порванных связей нет
- ✅ Grafana Explore с запросом `{container=~".+"}` возвращает логи из ≥ 2 контейнеров

### Задание 2 — Интеграция приложения и JSON-логирование (3 балла)
- ✅ Приложение пишет один JSON-объект на строку; обязательные поля есть у каждого типа события
- ✅ Контейнер приложения помечен `logging=alloy` и `app=devops-python`
- ✅ Каждый из Q1–Q4 возвращает реальные данные через `/loki/api/v1/query_range` (выводы CLI вставлены)
- ✅ Q3 и Q4 используют JSON-парсер и фильтры полей (не regex по сырой строке)

### Задание 3 — Дашборд (2 балла)
- ✅ 4 панели, по одному операционному вопросу, на всех живые данные
- ✅ Заголовок панели описывает вопрос, а не запрос
- ✅ JSON дашборда закоммичен в `monitoring/grafana/dashboards/lab07.json`

### Задание 4 — Продакшен-конфиг (1 балл)
- ✅ У всех четырёх сервисов есть лимиты CPU и памяти и reservations
- ✅ Анонимный доступ Grafana выключен; пароль администратора из `.env`; `.env` в gitignore
- ✅ Loki и Grafana в `docker compose ps` показывают `(healthy)`

### Задание 5 — Документация (1 балл)
- ✅ В `monitoring/docs/LAB07.md` есть все семь разделов
- ✅ Ответы из §1.1 написаны своими словами
- ✅ Настоящие скриншоты и выводы CLI (без заглушек)

### Бонус — Ansible (2 балла)
- ✅ Роль шаблонизирует все три конфига из переменных
- ✅ Деплой идемпотентный: второй прогон даёт `changed=0`
- ✅ Ожидание готовности не пускает успех, пока Loki и Grafana не ответят
- ✅ Оба прогона playbook вставлены в документацию

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Деплой стека | **3** | Все четыре сервиса подняты; pipeline Alloy связан верно (четыре пункта ВАША ЗАДАЧА в §1.3 и тома, сеть, метки в §1.4); доказательства `/ready` и `/labels` |
| **Задание 2** — Интеграция приложения | **3** | JSON-логирование сделано; обязательные поля на месте; каждый из Q1–Q4 возвращает реальные данные через HTTP API |
| **Задание 3** — Дашборд | **2** | 4 панели с подходящим LogQL; JSON дашборда закоммичен |
| **Задание 4** — Продакшен-конфиг | **1** | Лимиты есть, анонимный доступ выключен, healthcheck в состоянии healthy |
| **Задание 5** — Документация | **1** | Все семь разделов, настоящие снятые данные, ответы своими словами |
| **Бонус** — Ansible | **2** | Идемпотентный деплой из шаблонов, с ожиданием готовности |
| **Итого** | **12** | 10 основных + 2 бонуса |

---

## Материалы

<details>
<summary>📚 Loki</summary>

- [Обзор Loki](https://grafana.com/docs/loki/latest/get-started/overview/)
- [Справочник конфигурации Loki](https://grafana.com/docs/loki/latest/configure/)
- [Хранилище TSDB](https://grafana.com/docs/loki/latest/operations/storage/tsdb/)
- [Retention и compactor](https://grafana.com/docs/loki/latest/operations/storage/retention/)
- [HTTP API — push и query](https://grafana.com/docs/loki/latest/reference/loki-http-api/)

</details>

<details>
<summary>⚡ Grafana Alloy (преемник Promtail)</summary>

- [Документация Alloy](https://grafana.com/docs/alloy/latest/)
- [`discovery.docker`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.docker/)
- [`discovery.relabel`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/)
- [`loki.source.docker`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/)
- [`loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/)
- [Миграция с Promtail на Alloy](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/)

</details>

<details>
<summary>🔍 LogQL и Grafana</summary>

- [Справочник LogQL](https://grafana.com/docs/loki/latest/query/) — каждый оператор с примерами
- [Дашборды Grafana](https://grafana.com/docs/grafana/latest/dashboards/)
- [Источник данных Loki](https://grafana.com/docs/grafana/latest/datasources/loki/)
- [Explore для логов](https://grafana.com/docs/grafana/latest/explore/logs-integration/)

</details>

<details>
<summary>📝 Структурированное логирование</summary>

- [`structlog`](https://www.structlog.org/en/stable/)
- [`python-json-logger`](https://github.com/nhairs/python-json-logger)
- [The Twelve-Factor App: Logs](https://12factor.net/logs)
- [HOWTO по Python `logging`](https://docs.python.org/3/howto/logging.html)

</details>

<details>
<summary>⚠️ Частые ловушки (из реальных пробных прогонов)</summary>

- **«Ingester not ready, waiting 15s» на первом push.** Ingester Loki начинает отвечать ready примерно через 15 с после того, как процесс уже жив. Пустой ответ сразу после `docker compose up -d` — это старт, а не баг. Подождите и повторите. В задании 4 поставьте `healthcheck.start_period: 30s`, чтобы compose не объявлял сервис healthy раньше времени.
- **Метки с высокой кардинальностью = OOM Loki.** В метку Loki нельзя класть `user_id`, `request_id`, `trace_id`, `email` и любое значение уровня одного запроса (правило Alloy `discovery.relabel` или любой другой способ). Каждая уникальная комбинация — это поток. Каждый поток стоит индекса и памяти. На 10 млн пользователей ingester упадёт. Значения уровня запроса идут в **тело JSON**, фильтр на запросе: `| json | user_id="alice"`.
- **Права на Docker-сокет внутри контейнера Alloy.** На обычной установке Linux `/var/run/docker.sock` принадлежит root и группе `docker`. Внутри контейнера Alloy пользователь процесса может в этой группе не состоять. Если в логах Alloy при чтении сокета `permission denied`, для лабораторной запустите контейнер с `user: root` (для dev допустимо) либо передайте `group_add: ["${DOCKER_GID}"]`, предварительно сняв GID группы docker на хосте командой `getent group docker | cut -d: -f3`.
- **Монтирования `subPath` не обновляются, когда меняется ConfigMap.** В эту стену вы упрётесь в Lab 12, когда будете поднимать Loki на Kubernetes заново. Пока bind mount Compose для `loki/config.yml` *подхватывает* правки после `docker compose restart loki`, менять конфиг просто. Запомните поведение сейчас: позже оно понадобится.
- **Синтаксис конфига Promtail в Alloy один к одному не переносится.** Плоский список `pipeline_stages` у Promtail в Alloy разбит между `loki.relabel` (переписывание меток) и `loki.process` (разбор и правка строк). Имена стадий другие (`docker:` → блок `stage.docker {}`, `json:` → `stage.json {}` и так далее). Есть конфиг Promtail, который надо перенести, — берите [официальный инструмент миграции](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/), руками не переводите. Матрица переименований шире, чем кажется.
- **dict из middleware FastAPI и `after_request` во Flask.** Middleware FastAPI ждёт `await`. Хуки Flask — нет. Не делайте `await` у `request.json()` внутри Flask `after_request`: вызов блокирующий. Возьмите паттерн одного фреймворка и держитесь его.
- **`docker logs` режет JSON-строку.** Pretty-print (`json.dumps(obj, indent=2)`) заставляет `docker logs` (а значит и Alloy) считать каждую строку с отступом отдельной записью. Парсер `| json` упадёт на каждой. Всегда один JSON-объект на строку, без отступов.
- **Анонимный доступ Grafana в задании 4 остался включённым «случайно».** В §1.4 легко сказать себе «выключу, я запомню». Не запомните. Спасёт только комментарий в compose-файле. Оставьте его.
- **Порт 8080 сталкивается с Lab 6.** Стек Compose из Lab 6 по умолчанию тоже занимает порт хоста 8080. Если перед этой лабораторной вы не сделали `docker compose down`, `docker compose up -d` упадёт с *«Bind for 0.0.0.0:8080 failed: port is already allocated»*. Либо остановите Lab 6 (`cd ../app_python && docker compose down`), либо задайте другой порт хоста стеку Lab 6 (`app_host_port: 8084` в group_vars Lab 6). Метки контейнера `logging=alloy` и `app=devops-python` в любом случае должны стоять на контейнере `app` **этой** лабораторной.
- **`/ready` у Loki отдаёт 503, пока не появится `(healthy)`.** Healthcheck в задании 4 опрашивает `/ready`. Пока ingester Loki не поднимется (около 15 с после старта процесса), `/ready` отвечает 503, и `docker compose ps` показывает `(starting)`. Поставьте `start_period: 30s`, чтобы compose не мигал в `(unhealthy)` и обратно в `(healthy)`. Иначе легко решить, что стек сломан.

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- [`logcli`](https://grafana.com/docs/loki/latest/query/logcli/) — CLI для LogQL; для разового запроса быстрее, чем UI Grafana
- [`jq`](https://jqlang.github.io/jq/) — аккуратный вывод JSON; везде цепочка `curl … | jq`
- [`hey`](https://github.com/rakyll/hey) — гоняет трафик на приложение, чтобы наполнить дашборды

</details>

---

## Что дальше

| Lab | Что добавляет |
|---:|---|
| 8 | Столп **метрик**: Prometheus и `/metrics` на приложении из Lab 1, PromQL по методу RED |
| 9 | Тот же сервис и обвязка `echo` на k3d Kubernetes |
| 12 | ConfigMap и PVC: конфиг Loki заново из ConfigMap; ловушка `subPath` |
| 16 | `kube-prometheus-stack` и Helm-чарт Loki: весь этот стек заново, уже на K8s |

---

**Удачи!** 🚀

> **Запомните:** Loki индексирует *метки*, а не текст. Кардинальность меток держите низкой. Поля одного запроса живут в теле JSON и фильтруются на запросе через `| json | field="value"`. Pipeline Alloy — четыре связанных компонента, а не YAML-список. Связи — это имена export (`.targets`, `.output`, `.receiver`), а не метки блоков.
