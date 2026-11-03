# Lab 2 — Контейнеризация в Docker

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Containerization-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Docker%2029-informational)

> **Цель:** упакуйте сервис из Lab 1 в контейнер. Dockerfile пишете руками, multi-stage (сборка в несколько стадий). Просканируйте образ и сделайте push в реестр. К концу недели у вас на руках *тот* артефакт, который выкатывает каждая следующая лабораторная.
> **Что сдаёте:** PR из `lab02`. В нём `app_python/Dockerfile`, `.dockerignore`, отчёт `docs/LAB02.md` и (по желанию) бонус distroless для Go в `app_go/`.

---

## Обзор

В Lab 1 сервис «работал на вашей машине». Эта лабораторная заставляет его **работать везде**. Оценивают умение *писать Dockerfile построчно*: какой базовый образ, в каком порядке, от какого пользователя, с каким entrypoint. Каркасы ниже показывают **директивы**, а **значения** оставляют пустыми. Их подставляете вы и защищаете выбор в `docs/LAB02.md`.

В этой лабораторной вы потренируетесь:
- Писать руками production-ready Dockerfile, которые дружат с кэшем слоёв
- Запускать контейнеры от пользователя non-root (defense-in-depth — защита в несколько слоёв — против побега из контейнера)
- Собирать маленькие воспроизводимые образы через multi-stage
- Сканировать образ на уязвимости Trivy и ставить гейт по коду выхода, как в CI
- Ставить теги SemVer/SHA и делать push в публичный реестр

> ⚠️ **Границы:** Compose, Kubernetes и гейты CI будут позже. Здесь один сервис, один образ, один реестр.

**Стек (май 2026):** рекомендуется Docker Engine 29.x (хранилище образов containerd по умолчанию); **23+ тоже работает** — BuildKit включён по умолчанию с версии 23, директива `# syntax=docker/dockerfile:1` учитывается · `python:3.13-slim` · Trivy v0.69.3+

---

## Состояние проекта

**Что у вас должно быть с прошлых лабораторных:**
- `app_python/` — сервис Flask/FastAPI/Django из Lab 1 с `GET /` и `GET /health`
- *(по желанию)* `app_go/` (или другой сосед на компилируемом языке) из бонуса Lab 1

**Эта лабораторная добавляет:**
- `app_python/Dockerfile` — сначала одностадийный, потом переписанный в multi-stage
- `app_python/.dockerignore`
- `app_python/docs/LAB02.md` — отчёт для сдачи
- Опубликованный образ `ghcr.io/<you>/devops-info-service:1.0.0` (или эквивалент на Docker Hub)
- *(бонус)* `app_go/Dockerfile` — multi-stage с финальной стадией distroless

К Lab 9 Kubernetes тянет этот образ. К Lab 13 ArgoCD выкатывает его новые версии. Тег, который вы запушите сегодня, — ссылка на артефакт на следующие 14 недель.

---

## Подготовка

```bash
docker version              # Engine 29.x recommended; 23+ works (BuildKit on by default since 23); rootless OK
docker buildx version       # BuildKit is required (bundled)
trivy --version             # v0.69.3 or newer — NOT v0.69.4 (see warning below)
```

> ⚠️ **Предупреждение Trivy про цепочку поставки.** Ставьте **v0.69.3** или более поздний чистый патч. **Никогда не ставьте `v0.69.4`**. Этот релиз скомпрометирован (CVE-2026-33634, атака на цепочку поставки в марте 2026). Если в Lab 3 подключите Trivy к GitHub Actions, зафиксируйте `trivy-action@v0.35.0+` / `setup-trivy@v0.2.6+`.

> 🌐 **Зеркало БД Trivy.** В сетях с лимитами и ограничениями обычное скачивание БД с `ghcr.io/aquasecurity/trivy-db` возвращает `DENIED`. Если так вышло, передавайте `--db-repository public.ecr.aws/aquasecurity/trivy-db:2` каждому вызову `trivy image`. Эту операционную ловушку стоит запомнить: кампусная сеть легко на ней спотыкается.

Создайте структуру. Содержимое напишете сами:

```
app_python/
├── Dockerfile             # YOU WRITE — Task 1 → refactor in Task 3
├── .dockerignore          # YOU WRITE — Task 1
├── README.md              # extend with a Docker section in Task 5
├── docs/
│   └── LAB02.md           # YOU WRITE — submission report
└── (your Lab 1 sources)
```

---

## Задача 1 — Production Dockerfile (3 балла)

### 1.1 — Одностадийный Dockerfile

`ВАША ЗАДАЧА`: напишите `app_python/Dockerfile` по каркасу ниже. Каждый `___` заполняете вы. **Готового примера целиком в этой лабораторной нет.** Когда Dockerfile дают полностью, это уже не учёба, а перепечатка. Лекция (слайды 11, 13, 14, 15) объясняет *зачем* каждая строка. Держите её рядом.

```dockerfile
# syntax=docker/dockerfile:1
FROM ___                  # YOUR TASK: pin a slim Python base. Why slim? Why pinned? (slide 15)
WORKDIR ___               # YOUR TASK: absolute path. Not / and not ~
RUN ___                   # YOUR TASK: create a NUMERIC non-root user (uid >= 10000)
COPY ___ ___              # YOUR TASK: one file FIRST — the one that changes rarely
RUN pip install --___ -r ___  # YOUR TASK: which flag stops pip from baking its cache into the layer?
COPY ___ ___              # YOUR TASK: app source LAST. Explain the layer-cache reasoning in docs
USER ___                  # YOUR TASK: switch to the numeric UID from above
EXPOSE ___                # YOUR TASK: which port does your Lab 1 app listen on?
# YOUR TASK (recommended): HEALTHCHECK pinging GET /health. slim has NO curl + NO wget — pick deliberately
CMD [___]                 # YOUR TASK: exec form, not shell form (PID 1 trap → SIGTERM lost)
```

**Каждый пропуск — строка в критериях.** «Хороший» ответ совпадает с логикой лекции. Ответ, который «повезло» собрать, но который вы не защитите в `docs/LAB02.md`, потеряет баллы.

### 1.2 — `.dockerignore`

`ВАША ЗАДАЧА`: напишите `app_python/.dockerignore`, чтобы `COPY . .` не утащил в образ `.git/`, виртуальные окружения, конфиги IDE, `__pycache__/` и `.env`.

```gitignore
# YOUR TASK: list at minimum
# - VCS metadata (the Toyota 2022 .git leak — slide 16)
# - Python caches (__pycache__, *.pyc)
# - Virtualenvs (.venv, venv/)
# - Local env files (.env)
# - Editor dirs (.idea/, .vscode/)
# - Docs/tests (not needed at runtime — smaller context, faster builds)
```

Подсказка: синтаксис как у `.gitignore`, но читает его контекст сборки Docker, *не* git. Для этой задачи `.dockerignore` обязателен. Нет файла — сгорает вся строка критериев про `.dockerignore`.

### 1.3 — Подтверждение

**Вставьте в `docs/LAB02.md`:**

- Содержимое вашего `Dockerfile` (да, файл целиком: он короткий, и оценивают именно его)
- Содержимое `.dockerignore`
- Одно предложение обоснования на каждую строку с комментарием `# YOUR TASK`: *почему такое значение*, а не *что это такое*
- Вывод `docker build -t devops-info-service:lab02 ./app_python`, где виден финальный хеш образа (числа иллюстративные, у вас будут другие)

<details>
<summary>💡 Почему каждое правило важно (прочитайте до того, как писать)</summary>

- **Зафиксированный slim-базис** — `python:3.13` ≈ 1,2 ГБ, `python:3.13-slim` ≈ 150 МБ. Меньше образ — быстрее pull (автоскейл Kubernetes в Lab 9). Меньше пакетов — меньше CVE (задача 4). Тег `:latest` меняется, поэтому две сборки «из того же Dockerfile» могут дать разные образы.
- **Числовой non-root UID** — CVE побега из контейнера на уровне ядра (например CVE-2022-0185) даёт root на хосте, **если ваш процесс и так root**. `USER 10001` это закрывает. Числовой UID, в отличие от `USER appuser`, проходит admission-политики `runAsNonRoot: true`. Вы встретите их в Lab 9 и дальше.
- **Порядок слоёв (зависимости до кода)** — каждая инструкция — слой в кэше. `COPY . . && pip install` заново ставит все зависимости при любом изменении кода на один символ. `COPY requirements.txt . && pip install && COPY . .` переиспользует слой зависимостей, пока не изменится сам `requirements.txt`.
- **`--no-cache-dir`** — каталог pip `~/.cache/pip` — это десятки МБ wheel, которые в рантайме не нужны. Слои только добавляются. Удаление кэша в следующем `RUN` не вычищает его из образа: он просто прячется за новым слоем.
- **Exec-форма `CMD`** — shell-форма: `/bin/sh -c "python app.py"`. Тогда sh становится PID 1, приложение PID 2, сигналы до него не доходят. Exec-форма `["python","app.py"]`: приложение *и есть* PID 1.
- **`.dockerignore`** — Toyota, 2022: данные клиентов утекли, потому что `.git/` попал в выкаченный образ. Обычная защита от этого.

Ссылки: [Практики Dockerfile](https://docs.docker.com/build/building/best-practices/) · [Справочник Dockerfile](https://docs.docker.com/reference/dockerfile/)

</details>

---

## Задача 2 — Сборка, запуск и проверка (2 балла)

### 2.1 — Соберите, запустите, дёрните эндпоинты

`ВАША ЗАДАЧА`: соберите образ, запустите его и докажите, что оба эндпоинта отвечают так же, как в Lab 1. Сохраните настоящий вывод CLI.

```bash
# YOUR TASK: build it
docker build -t devops-info-service:lab02 ./app_python

# YOUR TASK: run it (background, host:container port mapping is yours to choose)
docker run -d --rm -p ___:___ --name lab02 devops-info-service:lab02

# YOUR TASK: hit both endpoints and capture the JSON
curl -s http://localhost:___/ | jq .
curl -s http://localhost:___/health | jq -c .
```

### 2.2 — Докажите, что контейнер не root

`ВАША ЗАДАЧА`: запустите тот же образ, подставив `id -u` как entrypoint, и сохраните вывод. **Цифра `0` в выводе недопустима.**

```bash
docker run --rm devops-info-service:lab02 id -u
# YOUR TASK: paste the captured number into docs/LAB02.md
```

Если видите `0`, строка `USER ___` отсутствует или стоит выше `COPY`, который её затирает. Поменяйте порядок, пересоберите, проверьте снова.

### 2.3 — Посмотрите, из чего собран образ

```bash
docker images devops-info-service --format '{{.Tag}}\t{{.Size}}'
docker history devops-info-service:lab02
```

`ВАША ЗАДАЧА`: прочитайте свой вывод `docker history`. В `docs/LAB02.md` найдите **один самый большой слой** и одним предложением объясните, почему он такой: базовый образ, зависимости или ваш код?

### 2.4 — Подтверждение

**Вставьте в `docs/LAB02.md`:**

- Хвост лога сборки (последние ~10 строк, включая шаг `=> exporting to image`)
- JSON обоих эндпоинтов с реальных запусков `curl`
- Строка `id -u`, которая доказывает non-root (иллюстрация — `10001`, если выбрали другой UID, число будет другим)
- Строка `docker images` с вашим реальным размером (иллюстрация — для Flask на slim типично `~130 MB`)
- Разбор «самого большого слоя» в одно предложение

---

## Задача 3 — Multi-stage сборка (2 балла)

### 3.1 — Перепишите в две стадии

`ВАША ЗАДАЧА`: перепишите `app_python/Dockerfile` (или заведите `Dockerfile.multi`) как multi-stage сборку. Стадия **builder** ставит зависимости в изолированный каталог. Стадия **runtime** копирует только нужное: без `pip`, без тулчейна сборки, без кэшей apt.

```dockerfile
# syntax=docker/dockerfile:1
# ---------- Stage 1: builder ----------
FROM ___ AS builder           # YOUR TASK: same slim base (or a fatter one if you need gcc for wheels)
WORKDIR ___                   # YOUR TASK: pick a build location
COPY requirements.txt .
RUN ___                       # YOUR TASK: install deps into an ISOLATED dir — venv at /opt/venv OR pip --prefix=/install

# ---------- Stage 2: runtime ----------
FROM ___ AS runtime           # YOUR TASK: same slim base — minimal final surface
RUN ___                       # YOUR TASK: numeric non-root user again (stages don't share USER)
WORKDIR ___                   # YOUR TASK: absolute path
COPY --from=builder ___ ___   # YOUR TASK: copy ONLY the deps dir from the builder
ENV PATH="___:$PATH"          # YOUR TASK: prepend your venv/prefix bin dir
COPY ___ ___                  # YOUR TASK: app source LAST
USER ___                      # YOUR TASK: numeric UID
EXPOSE ___                    # YOUR TASK: same port as Task 1
CMD [___]                     # YOUR TASK: exec form
```

### 3.2 — Соберите и сравните

```bash
# If you replaced Dockerfile in-place:
docker build -t devops-info-service:lab02-multi ./app_python

# If you kept the single-stage as Dockerfile and put the multi-stage in Dockerfile.multi,
# pass it explicitly with -f (otherwise docker build picks the default Dockerfile):
docker build -f app_python/Dockerfile.multi -t devops-info-service:lab02-multi ./app_python

docker images devops-info-service --format '{{.Tag}}\t{{.Size}}'
```

`ВАША ЗАДАЧА`: сохраните обе строки (`lab02` и `lab02-multi`) и напишите в `docs/LAB02.md` **один абзац** с выводом.

> ⚠️ **Честный результат, который может выйти:** у приложения на **чистом Python** (Flask) multi-stage образ бывает *чуть больше* (например 133 против 130 МБ), чем одностадийный. У Flask нет скомпилированных wheel, выбрасывать тулчейн нечего, а копия venv добавляет несколько МБ метаданных. **Так и задумано.** Запишите измерение. Не рисуйте уменьшение, которого не было. Резкий выигрыш даёт **компилируемый язык** (бонус): там multi-stage уменьшает образ в 60 раз.

### 3.3 — Подтверждение

**Вставьте в `docs/LAB02.md`:**

- Содержимое вашего `Dockerfile.multi`
- Строки `docker images` с реальными размерами `lab02` и `lab02-multi`
- Один абзац: multi-stage уменьшил образ, оставил примерно тем же или чуть раздул? Почему? (Если в ответе нет скомпилированных wheel и тулчейна сборки, лекцию вы не усвоили.)
- `curl -s localhost:___/health` из multi-stage контейнера — тот же ответ, что в задаче 2

<details>
<summary>💡 Идеи multi-stage</summary>

- **Зачем две стадии?** Builder может нуждаться в компиляторе C, чтобы собрать wheel (например `cryptography`, `psycopg2`). Runtime — нет. `COPY --from=builder` выбрасывает всё, что runtime не нужно.
- **Как называть стадии.** `FROM image AS builder`, затем `COPY --from=builder /src /dst`. Имя после `AS` — идентификатор стадии.
- **Где это особенно видно.** На компилируемых языках. `golang:1.25` ≈ 900 МБ, multi-stage до `distroless/static` ≈ 15 МБ. Тот же бинарник, в 60 раз меньше, ноль shell, ноль пакетного менеджера. Это и есть бонус.

Ссылка: [Multi-stage сборки](https://docs.docker.com/build/building/multi-stage/)

</details>

---

## Задача 4 — Сканирование Trivy (2 балла)

### 4.1 — Полный скан и скан с гейтом

`ВАША ЗАДАЧА`: прогоните два сканирования multi-stage образа и сохраните оба вывода.

```bash
# Full informational scan — lists every CVE by severity
trivy image devops-info-service:lab02-multi

# CI-style gate — fails (non-zero exit) on HIGH/CRITICAL findings.
# This is the exact gate you'll add to GitHub Actions in Lab 3.
trivy image --severity HIGH,CRITICAL --exit-code 1 devops-info-service:lab02-multi
echo "exit code: $?"
```

> 🌐 Если скачивание БД падает с `ghcr.io/aquasecurity/trivy-db: DENIED`, добавьте `--db-repository public.ecr.aws/aquasecurity/trivy-db:2`. AWS держит анонимное зеркало. Aqua публикует его туда специально: анонимные токены ghcr.io упираются в rate limit.

### 4.2 — Таблица по серьёзности

`ВАША ЗАДАЧА`: впишите в `docs/LAB02.md` **свои** числа (числа ниже иллюстративные: у вас будут другие, это зависит от свежести базового образа):

| Серьёзность | Число *(иллюстрация — ваши числа будут другими)* |
|-----------|--------:|
| CRITICAL  | 2 |
| HIGH      | 5 |
| MEDIUM    | 33 |
| LOW       | 63 |
| UNKNOWN   | 5 |
| **Итого** | **108** |

### 4.3 — Разберите одну находку и закройте её

`ВАША ЗАДАЧА`: выберите **одну** находку (лучше HIGH или CRITICAL: приоритет проще защитить). В `docs/LAB02.md` напишите **минимум три предложения**:

1. Пакет и идентификатор CVE (например `libncursesw6 6.5+20250216-2 — CVE-2025-69720`)
2. Почему её пометили: что говорит advisory апстрима
3. Что сделаете: пересоберёте на свежем `python:3.13-slim` (часто проходит само, когда Debian выкладывает патч), зафиксируете исправленную версию библиотеки, перейдёте на базу меньше (у distroless CVE почти нет, см. бонус) или прямо напишете, что **сегодня это не чинится** (патча апстрима ещё нет)

> 🚫 **Не глушите находки через `.trivyignore`**, лишь бы гейт стал зелёным. Если исправления у CVE ещё нет, напишите это прямо. Прятать незакрытые CVE от CI — так в прод уезжает следующий Log4Shell.

### 4.4 — Подтверждение

**Вставьте в `docs/LAB02.md`:**

- Полный вывод Trivy (или скриншот: длинно, но таблица сверху — главная часть)
- Ваша таблица серьёзности с реальными числами
- Команда скана с гейтом и её код выхода
- Разбор одной находки и что с ней делать

<details>
<summary>💡 Зачем сканировать и зачем гейт по коду выхода</summary>

- `trivy image <ref>` сообщает CVE и по пакетам ОС, **и** по зависимостям языка (например `requirements.txt`).
- `--severity HIGH,CRITICAL --exit-code 1` — идиома «завалить pipeline» (pipeline — цепочка автоматической сборки и выкладки). LOW и MEDIUM вы отсекаете, чтобы гейт не шумел, а опасное всё равно ловите.
- Меньше база — меньше пакетов — меньше CVE. Slim, потом distroless, потом scratch: так убывает поверхность уязвимостей. Бонус показывает это на числах (Python slim ≈ 108 CVE, Go distroless ≈ 0).

Ссылки: [Документация Trivy](https://trivy.dev/) · [Команда Trivy `image`](https://trivy.dev/latest/docs/target/container_image/)

</details>

---

## Задача 5 — Push в реестр и документация (1 балл)

### 5.1 — Тег и push

`ВАША ЗАДАЧА`: опубликуйте образ в **GHCR** (так лучше: для публичных пакетов бесплатно и стыкуется с Actions, которые вы напишете в Lab 3) или в Docker Hub.

Для **GHCR** (рекомендуется): заведите Personal Access Token (classic) со scope `write:packages`, затем:

```bash
# Put your token in a shell variable FIRST — do NOT paste the token onto the docker login
# line (it would land in your shell history). --password-stdin reads from stdin only.
export GHCR_PAT='ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

echo "$GHCR_PAT" | docker login ghcr.io -u <github-username> --password-stdin
docker tag devops-info-service:lab02-multi ghcr.io/<github-username>/devops-info-service:___   # YOUR TASK: real version tag
docker push ghcr.io/<github-username>/devops-info-service:___                                  # YOUR TASK: same tag
```

> 🤖 **В CI (Lab 3) PAT не нужен.** GitHub Actions на каждый запуск выдаёт эфемерный `GITHUB_TOKEN`. У него есть `write:packages` на пространство GHCR этого репозитория. Docker-job в Lab 3 берёт
> `password: ${{ secrets.GITHUB_TOKEN }}`. Долгоживущий PAT, который можно утечь, там не нужен. PAT — только для *ноутбука сегодня*.

> 📢 **После первого push пакеты GHCR по умолчанию PRIVATE.** Критерии приёмки ниже требуют, чтобы образ *тянулся публично*. Когда первый `docker push` прошёл, откройте **свой профиль GitHub → Packages → `devops-info-service` → Package settings (справа) → Change visibility → Public**. Подтвердите, введя имя пакета. Без этого шага `docker pull` из чистого кэша ниже пройдёт только потому, что логин ещё закэширован. Проверяющий запустит `docker pull` без вашего токена и получит `401 Unauthorized`.

`ВАША ЗАДАЧА`: выберите стратегию тегов и подставьте настоящий тег в оба пропуска выше. **Не `:latest`**. Смотрите слайд 20. Один вариант из таблицы:

| Стратегия | Пример | Когда брать |
|------------|---------------|-------------|
| SemVer     | `1.0.0`       | Настоящий релиз сервиса |
| Git SHA    | `git-3a7f29c` | Сборки в CI, тег всегда уникален |
| CalVer     | `2026.05.0`   | Релизы по календарю |

Проверьте, что образ и правда тянется из **по-настоящему чистого состояния**. Сначала `docker logout`, чтобы закэшированный логин не спрятал приватный пакет:

```bash
docker logout ghcr.io                                            # forget the PAT so we test the public path
docker rmi  ghcr.io/<github-username>/devops-info-service:___   # YOUR TASK: your tag
docker pull ghcr.io/<github-username>/devops-info-service:___   # YOUR TASK: must succeed anonymously (package must be Public)
```

Если после `docker logout` команда `docker pull` отвечает `unauthorized`, пакет всё ещё приватный. Переключите видимость на Public, как в заметке выше, и стяните образ ещё раз.

### 5.2 — Допишите `app_python/README.md`

`ВАША ЗАДАЧА`: добавьте в README из Lab 1 раздел **Docker** из трёх подразделов:

1. **Сборка** — ваша команда `docker build`
2. **Запуск** — ваша команда `docker run` (проброс портов хост:контейнер)
3. **Pull из реестра** — публичная команда pull с вашим настоящим URL реестра

Содержимое Dockerfile в README вставлять не нужно. Поставьте ссылку на файл в репозитории.

### 5.3 — `app_python/docs/LAB02.md`, отчёт для сдачи

Обязательные разделы, строго в этом порядке:

1. **Решения по Dockerfile** — файл из задачи 1 и строка *зачем* на каждый `# YOUR TASK`
2. **Итог multi-stage** — размеры одностадийного и multi-stage с вашими числами и вывод в одну строку (включая честное «без изменений, потому что чистый Python», если измерили именно это)
3. **Скан Trivy** — таблица серьёзности с реальными числами, одна разобранная находка, что с ней делать, код выхода скана с гейтом
4. **Сборка, запуск, push** — сохранённые выводы из задач 2 и 5, включая публичный URL реестра
5. **Трудности и решения** — хотя бы одна настоящая. Ловушка HEALTHCHECK «в slim нет curl» и зеркало БД Trivy оба считаются. Берите то, на что реально наткнулись.

### 5.4 — Подтверждение

**Вставьте в `docs/LAB02.md`:**

- Вывод `docker push`, где виден digest
- Вывод `docker pull` из чистого состояния (образ публичный, его можно стянуть)
- URL реестра (например `https://github.com/<you>/devops-info-service/pkgs/container/devops-info-service`)
- Раздел Docker в обновлённом `README.md` (или ссылка на него в PR)

---

## Бонус — multi-stage на distroless (Go) (2 балла)

Снова упакуйте бонусное приложение из Lab 1 (лучше Go) в multi-stage сборку. Финальная стадия — **distroless или `scratch`**. Именно здесь обещание лекции «в 60 раз меньше, ноль CVE» становится видно.

`ВАША ЗАДАЧА`: заполните каркас. За руку здесь ведут меньше, чем в основных задачах: два Dockerfile вы уже написали.

```dockerfile
# syntax=docker/dockerfile:1
# ---------- build stage: full Go toolchain ----------
FROM ___ AS build             # YOUR TASK: pin a Go base (e.g. golang:1.25). Not :latest
WORKDIR ___                   # YOUR TASK: pick a build dir
COPY go.mod go.sum* ./
RUN go mod download
COPY . .
RUN ___                       # YOUR TASK: build a FULLY STATIC binary. Which env var disables cgo? (slide 17)
                              # WHY this matters: cgo links against the build image's libc dynamically.
                              # A static binary (no libc dep) is what makes `FROM scratch` / distroless/static
                              # viable — those images literally don't ship libc.

# ---------- final stage: ~nothing but the binary ----------
FROM ___                      # YOUR TASK: gcr.io/distroless/static-debian12 OR scratch — choose + defend in docs
COPY --from=build ___ ___     # YOUR TASK: copy the static binary out of the build stage
USER ___                      # YOUR TASK: distroless ships `nonroot:nonroot`; or pick a numeric UID
EXPOSE ___                    # YOUR TASK: same port as Lab 1
ENTRYPOINT [___]              # YOUR TASK: exec form — single static binary, no shell needed
```

### Требования бонуса

- Multi-stage Dockerfile в `app_go/`, финальная стадия distroless или `scratch`
- Статически слинкованный бинарник. В `app_go/docs/LAB02.md` объясните, *почему* статическая линковка делает distroless и scratch возможными (в рантайме нет libc)
- Приложение в контейнере работает: отвечают и `/`, и `/health`
- Скан Trivy distroless-образа. Сравните число находок с Python multi-stage и объясните, *почему* их сильно меньше (нет shell, нет пакетного менеджера, поверхность атаки почти нулевая)
- Размер и число CVE записаны (лекция обещает ~15 МБ / ~0 CVE; фиксируйте свои числа)

**Ожидаемый выигрыш** *(иллюстрация — ваши числа будут другими)*: Python slim ≈ **130 МБ / ~100 CVE** → Go distroless ≈ **8 МБ / 0 CVE**.

<details>
<summary>💡 Distroless и scratch: когда что брать</summary>

- **`gcr.io/distroless/static-debian12`** — пустое пользовательское пространство, кроме CA-сертификатов, `/etc/passwd` (пользователь `nonroot`) и `tzdata`. Для Go это разумный дефолт.
- **`scratch`** — пусто. CVE около нуля, потому что в образе ничего нет. Работает только у полностью статических бинарников. Если бинарник ходит по HTTPS, CA-сертификаты добавляете сами.
- **Компромисс** — без shell нельзя сделать `docker exec ... sh` и отлаживаться в проде. Под атакой это *полезное свойство*, при разработке — *раздражение*. Если нужно локально, держите рядом вариант distroless `*-debug`.

Ссылки: [distroless](https://github.com/GoogleContainerTools/distroless) · [Multi-stage сборки](https://docs.docker.com/build/building/multi-stage/)

</details>

---

## Как сдавать

```bash
git switch -c lab02
git add app_python/Dockerfile app_python/.dockerignore app_python/README.md app_python/docs/LAB02.md
git add app_go/                   # only if you did the bonus
git commit -m "feat(lab02): containerize devops info service (multi-stage + trivy + push)"
git push -u origin lab02
```

Откройте **два** PR:

- `your-fork:lab02` → `course-repo:master` *(на ревью)*
- `your-fork:lab02` → `your-fork:master` *(сольётся в ваш main, когда закончите)*

Чеклист PR:

```text
- [ ] Task 1 — Dockerfile + .dockerignore written by hand
- [ ] Task 2 — build / run / non-root verified with real curl + id -u captures
- [ ] Task 3 — multi-stage Dockerfile with size comparison
- [ ] Task 4 — Trivy full + gated scan; one finding explained
- [ ] Task 5 — image pushed to GHCR/Docker Hub with SemVer or SHA tag (not :latest only)
- [ ] Bonus  — Go distroless image, ~order-of-magnitude smaller, ~0 CVEs (if attempted)
```

---

## Критерии приёмки

### Задача 1 — Dockerfile (3 балла)
- ✅ `app_python/Dockerfile` написан по каркасу (каждый `___` заполнен)
- ✅ Зафиксированный slim-базис (не `latest` и не полный образ)
- ✅ `WORKDIR` задан явно
- ✅ **Числовой** non-root `USER` (uid ≥ 10000)
- ✅ Зависимости ставятся до кода приложения (порядок дружит с кэшем); есть `--no-cache-dir`
- ✅ `CMD` в exec-форме (JSON-массив)
- ✅ Есть `app_python/.dockerignore` с исключениями VCS, venv, кэша и IDE

### Задача 2 — Сборка и запуск (2 балла)
- ✅ Образ собирается чисто
- ✅ `GET /` и `GET /health` отдают тот же JSON, что в Lab 1
- ✅ `docker run ... id -u` печатает ненулевой UID
- ✅ В документации есть размер из `docker images` и одна строка про «самый большой слой»

### Задача 3 — Multi-stage (2 балла)
- ✅ Две стадии и `COPY --from=builder`
- ✅ В финальном образе нет тулчейна сборки и кэша pip
- ✅ В документации реальные размеры, включая **честное** объяснение, если multi-stage не уменьшил образ чистого Python-приложения

### Задача 4 — Trivy (2 балла)
- ✅ Прогнаны и полный скан, и гейт `--severity HIGH,CRITICAL --exit-code 1`
- ✅ Числа по серьёзности ваши, не из примера
- ✅ Одна находка разобрана, с планом исправления
- ✅ Ничего не спрятано через `.trivyignore`

### Задача 5 — Реестр и документация (1 балл)
- ✅ Образ запушен и **публично тянется** (`docker rmi && docker pull` с чистого состояния проходит)
- ✅ Настоящий тег версии (SemVer или SHA), а не один только `:latest`
- ✅ В `README.md` есть раздел Docker
- ✅ `docs/LAB02.md` полный: все 5 разделов, реальные доказательства, URL реестра

### Бонус (2 балла)
- ✅ Multi-stage Dockerfile для Go, финальная стадия distroless или `scratch`
- ✅ Статически слинкованный бинарник; оба эндпоинта отвечают
- ✅ Скан Trivy distroless-образа и сравнение числа находок
- ✅ В `app_go/docs/LAB02.md` есть размеры, объяснение статической линковки и разбор безопасности

---

## Критерии оценивания

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — Dockerfile  | **3** | Зафиксированный slim-базис, числовой non-root, слои дружат с кэшем, `CMD` в exec-форме, `.dockerignore` |
| **Задача 2** — Сборка и запуск | **2** | Собирается чисто; оба эндпоинта совпадают с Lab 1; non-root проверен |
| **Задача 3** — Multi-stage | **2** | Две стадии, в финальном образе нет тулчейна, размер записан как есть (даже если «без изменений, и вот почему») |
| **Задача 4** — Trivy       | **2** | Полный скан и скан с гейтом, реальные числа, одна разобранная находка, ничего не спрятано |
| **Задача 5** — Реестр    | **1** | Публично тянется, настоящий тег версии, полный `LAB02.md` и раздел Docker в README |
| **Бонус** — Distroless Go | **2** | Multi-stage на distroless или scratch, сравнение размера и CVE |
| **Итого** | **12** | 10 основных + 2 бонус |

**Как оценивают:**
- **10/10** — образ маленький, безопасный, воспроизводимый; есть дисциплина multi-stage; скан понят; push чистый; в документации рассуждение идёт дальше «так сказали на лекции».
- **8–9/10** — рабочий multi-stage образ от non-root, просканирован и запушен, большинство решений защищено.
- **6–7/10** — контейнер работает и запущен не от root, но сборка только одностадийная или скан с разбором поверхностные.
- **<6/10** — запуск от root, нет `.dockerignore`, нет скана, или Dockerfile скопирован и объяснить его нельзя.

---

## Материалы

<details>
<summary>📚 Документация Docker</summary>

- [Практики Dockerfile](https://docs.docker.com/build/building/best-practices/)
- [Справочник Dockerfile](https://docs.docker.com/reference/dockerfile/)
- [Multi-stage сборки](https://docs.docker.com/build/building/multi-stage/)
- [`.dockerignore`](https://docs.docker.com/reference/dockerfile/#dockerignore-file)
- [Заметки к релизу Docker Engine 29](https://docs.docker.com/engine/release-notes/)
- [Хранилище образов containerd](https://docs.docker.com/engine/storage/containerd/)

</details>

<details>
<summary>🔒 Безопасность и сканирование</summary>

- [Документация Trivy](https://trivy.dev/) · [Сканирование Trivy `image`](https://trivy.dev/latest/docs/target/container_image/)
- [Distroless](https://github.com/GoogleContainerTools/distroless)
- [Зачем контейнеры non-root](https://docs.docker.com/build/building/best-practices/#user)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- *Container Security* — Liz Rice (O'Reilly, 2020)

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- [Hadolint](https://github.com/hadolint/hadolint) — линтер Dockerfile
- [Dive](https://github.com/wagoodman/dive) — интерактивный разбор слоёв образа
- [GHCR](https://docs.github.com/packages/working-with-a-github-packages-registry/working-with-the-container-registry) · [Docker Hub](https://hub.docker.com/)

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных прогонов)</summary>

- **`COPY . .` до `pip install`** — убивает кэш слоёв. Изменение кода на один символ заново ставит все зависимости. Самая частая ошибка этой лабораторной. Лечится так: сначала копируете `requirements.txt` и ставите зависимости, и только потом остальной код. (Слайд 14 лекции.)
- **`USER appuser` (не число) при `runAsNonRoot: true`** — в Docker такой `USER` работает. Admission-политика Kubernetes в Lab 9 его не пропустит: без разбора `/etc/passwd` UID не проверить. Всегда ставьте **числовой** `USER 10001`.
- **БД Trivy, `ghcr.io ... DENIED`** — кампус и закрытые сети режут rate limit анонимного токена ghcr.io. Ставьте `--db-repository public.ecr.aws/aquasecurity/trivy-db:2`. Aqua выкладывает зеркало туда специально для этого случая.
- **В `python:3.13-slim` нет ни `curl`, ни `wget`** — HEALTHCHECK ломается молча. Либо `python -c 'import urllib.request; urllib.request.urlopen("http://127.0.0.1:8080/health")'`, либо `apt-get install curl` (и лишние CVE впридачу). Не добавляйте строку в надежде, что она заработает.
- **Trivy `v0.69.4` вредоносный** — компрометация цепочки поставки, CVE-2026-33634 (март 2026). Фиксируйте `v0.69.3` или более поздний чистый патч. В CI: `trivy-action@v0.35.0+`.
- **BuildKit выключен** — старые Docker Engine (ниже 23) игнорируют `# syntax=docker/dockerfile:1` и не умеют возможности `COPY --from=builder`. В Docker 29 BuildKit включён по умолчанию. Если сборка падает на синтаксисе `RUN --mount=type=cache`, Docker у вас старый.
- **Нет `.dockerignore`** — `COPY . .` кладёт в образ `.git/`, `.venv/`, конфиги IDE и **`.env` с секретами**. Слои не забывают: секрет, попавший в образ, остаётся там, даже если удалить файл в следующем слое.
- **`COPY secrets.env / && rm secrets.env`** — так секрет не удалить. Он лежит в предыдущем слое, его видно через `docker history` и `docker save | tar -xvf`. Каждый слой считайте неизменяемым.
- **Для чистого Python multi-stage чуть БОЛЬШЕ** — у Flask нет скомпилированных wheel, стадии builder нечего выбрасывать, а копия venv добавляет ~3 МБ. Так и должно быть, это правильно. Запишите измерение, не прячьте его. Большой выигрыш multi-stage — у компилируемых языков (бонус).
- **Кэш образа протухает после `apt-get update`** — если разнести `RUN apt-get update && apt-get install foo` на два слоя, `update` закэшируется навсегда, а `install` пойдёт по старому списку пакетов. Всегда держите их в одном `RUN`. Тот же урок для любого пакетного менеджера: связанную работу не разносите по слоям.
- **Hostname контейнера — это его ID, а не имя ноутбука** — `socket.gethostname()` внутри контейнера вернёт что-то вроде `a3f29c8b1d04`. Так и задумано (сноска в Lab 1). Вы видите пространства имён PID и UTS.
- **`docker run` по умолчанию идёт от root** — даже при `USER 10001` в Dockerfile флаг `docker run --user root` его перекрывает. Dockerfile задаёт *значение по умолчанию*, рантайм может его сменить. В Lab 9 политики Kubernetes PSA и Pod Security навязывают это уже со стороны оркестратора.

</details>

---

## Что дальше

Образ, который вы соберёте на этой неделе, идёт за сервисом до конца курса:

| Lab | Что добавляется |
|---:|---|
| 3  | CI: pytest, ruff, сборка образа и гейт Trivy на каждый PR |
| 7  | Структурированные JSON-логи: Alloy → Loki |
| 8  | Эндпоинт `/metrics` и инструментация Prometheus |
| 9  | Выкат в Kubernetes на k3d, probe на `/health` |
| 10 | Упаковка этого образа в чарт Helm 4 |
| 13 | GitOps на ArgoCD: новые теги уезжают сами |

---

**Жёстко:** хороший Dockerfile маленький, безопасный и воспроизводимый. Если на месте только два из трёх, это баг. Запускайтесь не от root, сканируйте каждый образ, не выкатывайте `:latest`. Всё остальное в курсе — вариации этих трёх правил.
