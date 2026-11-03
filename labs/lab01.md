# Lab 1 — DevOps Info Service: разработка веб-приложения

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Web%20Development-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![languages](https://img.shields.io/badge/languages-Python%20|%20Go-informational)

> **Цель:** соберите заготовку своего DevOps Info Service — небольшой HTTP-сервис, который отдаёт собственные метаданные и сведения о хосте, на котором работает. Каждая следующая лабораторная наращивает этот же сервис.
> **Что сдаёте:** PR из `lab01` с каталогом `app_python/` (и по желанию `app_go/`). Образ, который вы будете собирать следующие 16 недель, начинается здесь.

---

## Обзор

В этой лабораторной вы потренируетесь:
- Выбрать веб-фреймворк на Python и обосновать выбор
- Считать сведения о хосте и runtime из `platform`, `socket`, `os`
- Подключить HTTP-маршруты, которые возвращают JSON, а не HTML
- Зафиксировать зависимости, настроить сервис через переменные окружения, написать нормальную документацию

> ⚠️ **Границы:** без базы данных, без аутентификации, без Docker. Это Lab 2 и дальше. Не мудрите. Код короткий и очевидный.

---

## Состояние проекта

**Что у вас должно быть с прошлых лабораторных:**
- Ничего. Это первая неделя.

**Эта лабораторная добавляет:**
- `app_python/` — сервис на Python с `GET /` (info) и `GET /health` (probe)
- *(бонус по желанию)* `app_go/` — сосед на компилируемом языке с теми же эндпоинтами

К Lab 9 этот сервис работает в Kubernetes. К Lab 13 его выкатывает ArgoCD. Решения этой недели — фреймворк, раскладка каталогов, конфигурация через переменные окружения — переживут Lab 1.

---

## Подготовка

Нужно только это:

```bash
python3 --version         # 3.12+ (course standardizes on 3.13)
pip3 --version
```

> 📝 **`python` и `python3`**: в Debian и Ubuntu бинарник называется `python3`, в macOS и Windows это может быть `python`. Берите тот, который есть на машине. Дальше для краткости написано `python app.py`. Если система называет интерпретатор `python3`, подставляйте `python3 app.py`.

Создайте дерево каталогов. Файлы заполните сами:

```
app_python/
├── app.py
├── requirements.txt
├── .gitignore
├── README.md
├── tests/
│   └── __init__.py            # empty; tests come in Lab 3
└── docs/
    └── LAB01.md               # your submission report
```

---

## Задача 1 — веб-приложение на Python (6 баллов)

### 1.1 — Выберите фреймворк и обоснуйте его

Можно взять **Flask 3.1**, **FastAPI 0.136** или **Django 5.2** (для этой задачи это с избытком; берите, если хотите потренироваться). В `docs/LAB01.md` напишите обоснование на один абзац и небольшую таблицу сравнения. Минимум такие колонки: footprint (сколько тянет за собой), поддержка async, OpenAPI и документация, порог входа.

### 1.2 — `GET /` возвращает полный JSON со сведениями

`ВАША ЗАДАЧА`: напишите обработчик корневого пути. Он возвращает JSON с **пятью** ключами верхнего уровня: `service`, `system`, `runtime`, `request`, `endpoints`.

Требования:

- `service`: `name`, `version`, `description`, `framework` (ваш выбор из 1.1)
- `system`: `hostname`, `platform`, `platform_version`, `architecture`, `cpu_count`, `python_version`
- `runtime`: `uptime_seconds` (int), `uptime_human` (например `"1 hour, 3 minutes"`), `current_time` (ISO 8601 UTC), `timezone`
- `request`: `client_ip`, `user_agent`, `method`, `path`
- `endpoints`: список `{path, method, description}` по каждому маршруту, который вы публикуете

Подсказки:

- `platform.system()`, `platform.machine()`, `platform.python_version()` закрывают почти весь блок `system`
- Hostname берёте через `socket.gethostname()`
- Uptime — это `datetime.now(timezone.utc) - START_TIME`. `START_TIME` фиксируйте при импорте модуля. Везде будьте **timezone-aware** (с часовым поясом): смесь naive и aware datetime бросает `TypeError`
- Как достать запрос в каждом фреймворке: Flask → `request.remote_addr` / `request.headers.get('User-Agent')`; FastAPI → `request.client.host` / `request.headers.get('user-agent')`

<details>
<summary>💡 Не складывается структура?</summary>

Набросок. Python-кода здесь нет, так и задумано:

```
imports → app object → constants (SERVICE_NAME, START_TIME, ...) → helper get_system_info() → helper get_uptime() → route handler returning the 5-key dict → error handlers → __main__ block
```

Слайд Lecture 1 про жизненный цикл и Lecture 2 про конфигурацию. Оба здесь к месту.

</details>

### 1.3 — `GET /health` возвращает короткий JSON и HTTP 200

`ВАША ЗАДАЧА`: напишите второй обработчик. Он возвращает `{status, timestamp, uptime_seconds}` и код **200**. В Lab 9 это будет цель liveness и readiness в Kubernetes. Держите обработчик дешёвым: без походов в БД и без внешних HTTP-вызовов.

### 1.4 — Настройка через переменные окружения

`ВАША ЗАДАЧА`: прочитайте `HOST`, `PORT` и `DEBUG` из окружения. Разумные значения по умолчанию: `0.0.0.0`, `5000`, `False`. Приложение должно запускаться без правок кода так:

```bash
python app.py                            # default 0.0.0.0:5000
PORT=8080 python app.py                  # custom port
HOST=127.0.0.1 PORT=3000 python app.py   # both
```

Подсказка: `PORT` приводите к `int`, а `DEBUG` к `bool` **сравнением строки в нижнем регистре** (`os.getenv("DEBUG","False").lower() == "true"`). В Python `bool("False")` равен `True`. Так в прод легко уезжает баг с вечно включённым debug.

### 1.5 — Обработчики ошибок и логирование

`ВАША ЗАДАЧА`: зарегистрируйте обработчики **404** и **500**. Они возвращают JSON, как и остальной API. Настройте стандартный `logging` на уровень INFO. В формате должны быть время и уровень. `print()` не используйте.

### 1.6 — Подтверждение

**Вставьте в `docs/LAB01.md`:**

- Четыре захвата из CLI с **точными командами**, которые вы запускали:
  - `curl -s http://localhost:5000/ | jq .` — полный JSON, ваш настоящий hostname и uptime
  - `curl -s http://localhost:5000/health | jq .`
  - `curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:5000/nope` — так вы доказываете обработчик 404
  - `PORT=8080 python app.py` в другом терминале и `curl`, который показывает, что отвечает новый порт — так вы доказываете настройку через переменные окружения
- Таблицу сравнения фреймворков из 1.1
- 2–3 предложения: на какой самой неприятной ловушке вы споткнулись и как из неё вышли

<details>
<summary>💡 Подсказки, если застряли</summary>

- Фреймворк выберите сразу. Смена посреди лабораторной съедает время.
- `GET /` собирайте последним. Сначала `GET /health` (пять строк), потом по шагам добавляйте сведения о системе, uptime и данные запроса.
- Если JSON выглядит странно (например, префикс байтов `b'...'`), вы вернули строку, а не dict. Отдайте dict фреймворку, пусть он сам сделает jsonify.

</details>

---

## Задача 2 — документация и практики (4 балла)

### 2.1 — `app_python/README.md`

`ВАША ЗАДАЧА`: напишите README для пользователя. Разделы такие (без воды, команды важнее прозы):

1. **Обзор** — один абзац
2. **Что нужно заранее** — версия Python, ОС
3. **Установка** — `python -m venv venv` … `pip install -r requirements.txt`
4. **Запуск** — пример по умолчанию и пример с другим портом
5. **Эндпоинты API** — таблица `Method | Path | Description`
6. **Конфигурация** — таблица переменных окружения, значений по умолчанию и назначения

### 2.2 — Гигиена кода

`ВАША ЗАДАЧА`: добавьте то, что нужно любому Python-сервису.

| Файл | Что должно быть |
|---|---|
| `requirements.txt` | Фреймворк **зафиксирован до конкретного патча** (например `Flask==3.1.3`, а не `Flask>=3`). Фреймворк один. Остальные уберите. |
| `.gitignore` | `__pycache__/`, `*.pyc`, `venv/`, `*.log`, каталоги IDE (`.vscode/`, `.idea/`), `.DS_Store` |
| Импорты в `app.py` | Сгруппированы (stdlib → сторонние → локальные), по PEP 8 |

### 2.3 — `app_python/docs/LAB01.md`, отчёт для сдачи

Обязательные разделы, строго в этом порядке:

1. **Выбор фреймворка** — ваш выбор и таблица сравнения
2. **Практики** — маркированный список, у каждого пункта одно предложение, *зачем*
3. **Документация API** — пример запроса и ответа на каждый эндпоинт
4. **Результаты проверок** — четыре захвата CLI из пункта 1.6
5. **Трудности и решения** — хотя бы одна настоящая (не «я в Python новичок»)
6. **Сообщество GitHub** — один абзац, см. 2.4

### 2.4 — Участие в сообществе GitHub

Это социальная половина работы разработчика. Всё это видно в вашем профиле.

`ВАША ЗАДАЧА`:
1. Поставьте **Star** репозиторию курса.
2. Поставьте **Star** проекту [simple-container-com/api](https://github.com/simple-container-com/api). Это перспективный open-source инструмент для управления контейнерами.
3. Нажмите **Follow** у преподавателя и ассистентов на GitHub:
   - Преподаватель: [@Cre-eD](https://github.com/Cre-eD)
   - Ассистент: [@Naghme98](https://github.com/Naghme98)
   - Ассистент: [@pierrepicaud](https://github.com/pierrepicaud)
4. Нажмите **Follow** минимум у 3 однокурсников с этого курса.
5. В разделе «Сообщество GitHub» файла `docs/LAB01.md` **первой строкой поставьте свой username на GitHub** (ассистент сверит star и follow меньше чем за 30 секунд). Дальше 2–3 предложения своими словами, не из лекции: *Зачем star проекту реально помогает мейнтейнерам? Что конкретного вы получаете, когда подписываетесь на других разработчиков?*

### 2.5 — Подтверждение

**Вставьте в `docs/LAB01.md`:**

- Сам `docs/LAB01.md` закрывает все шесть обязательных разделов. Это и *есть* подтверждение.
- Вывод `find app_python -maxdepth 2 -type f | sort`. По нему видно, что все обязательные файлы на месте.

---

## Бонус — сосед на компилируемом языке (2 балла)

Соберите тот же сервис ещё раз на **компилируемом языке**, в соседнем каталоге (например `app_go/`).

Зачем. Бонус Lab 2 с multi-stage Docker ужимает Go-сервис с ~900 МБ до ~15 МБ. Без статического бинарника на старте это не сработает.

`ВАША ЗАДАЧА`:

- Те же два эндпоинта (`/`, `/health`)
- Та же форма JSON, что у версии на Python (одни и те же вызовы `curl` должны проходить в оба)
- README в соседнем каталоге: как собрать и как запустить
- `go.mod` (и `go.sum`, если тянете зависимости). Бонусу Lab 2 они нужны как контекст сборки
- В `docs/LAB01.md` добавьте одну строку: **размер артефакта** для обоих. Меряйте одинаковым способом, сравнение должно быть честным:
  - Python: `du -sh app_python/venv` (venv тащит всё, что сверх интерпретатора)
  - Скомпилированный вариант: `ls -lh app_go/<binary>` (один статический бинарник)
  - Вся суть в разнице размеров. Настоящее сравнение Docker-образов будет в Lab 2.

Выберите один вариант:

| Язык | Идиоматичный HTTP | Заметки |
|---|---|---|
| **Go** *(рекомендуется)* | `net/http` | Один статический бинарник; компиляция мгновенная; позже самый маленький образ distroless |
| Rust | `actix-web` / `axum` | Строгие типы; компиляция дольше; хороший выбор, если вам важна безопасность |
| Java + Spring Boot | `spring-boot-starter-web` | Промышленный стандарт в enterprise, runtime тяжелее |
| C# + ASP.NET Core | `WebApplication.Create` | Кроссплатформенный .NET; по весу рядом с Java, эргономика новее |

Подсказки (полного кода нет):

- В Go `runtime.NumCPU()`, `runtime.GOOS`, `runtime.GOARCH`, `os.Hostname()` делают то же, что `os` и `platform` в Python. Регистр JSON-ключей другой. Struct tag (`json:"hostname"`) сохраняют ту же форму ответа.
- Для Java и C# стартуйте с `spring init` / `dotnet new web`. Не пишите `pom.xml` и `csproj` руками.

---

## Как сдавать

```bash
git switch -c lab01
git add app_python/
git add app_go/                   # only if you did the bonus
git commit -m "feat(lab01): devops info service — python (+ go bonus)"
git push -u origin lab01
```

Откройте **два** PR:

- `your-fork:lab01` → `course-repo:master` *(на ревью)*
- `your-fork:lab01` → `your-fork:master` *(сольётся в ваш main, когда закончите)*

Чеклист PR:

```text
- [ ] Task 1 done — /, /health, env config, error handlers, logging
- [ ] Task 2 done — README, requirements.txt, .gitignore, docs/LAB01.md, GitHub social
- [ ] Bonus done — app_go/ (or other) with same endpoints + size comparison
```

---

## Критерии приёмки

### Задача 1 (6 баллов)
- ✅ Сервис стартует командой `python app.py` и слушает `0.0.0.0:5000`
- ✅ `GET /` отвечает HTTP 200, все пять ключей верхнего уровня заполнены *с вашей реальной машины*
- ✅ `GET /health` отвечает HTTP 200, на месте все три обязательных поля
- ✅ Обработчик 404 возвращает JSON, а не HTML
- ✅ Переменная `PORT` меняет порт
- ✅ Логирование настроено (в stdout видна строка лога при старте)

### Задача 2 (4 балла)
- ✅ В `README.md` есть все шесть разделов
- ✅ `requirements.txt` фиксирует **точную** версию
- ✅ `.gitignore` закрывает мусор Python, IDE и ОС
- ✅ В `docs/LAB01.md` все шесть разделов, включая абзац про сообщество GitHub
- ✅ Star и follow видны в профиле студента на GitHub

### Бонус (2 балла)
- ✅ Соседнее приложение собирается и отдаёт те же два эндпоинта
- ✅ JSON той же формы (один и тот же `curl … | jq` работает с обоими)
- ✅ Сравнение размеров артефактов записано (`du -sh venv` и `ls -lh <binary>`)
- ✅ Манифест сборки на месте (`go.mod` / `Cargo.toml` / `pom.xml` / `*.csproj`), бонусу Lab 2 есть что отдать в `COPY`

---

## Критерии оценивания

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — веб-приложение на Python | **6** | Оба маршрута верные, конфигурация через переменные окружения, обработчики ошибок, логирование |
| **Задача 2** — документация и гигиена | **4** | Все требования к файлам и разделам закрыты; зависимости зафиксированы; LAB01.md полный |
| **Бонус** — сосед на компилируемом языке | **2** | Те же эндпоинты, JSON той же формы, сравнение размеров |
| **Итого** | **12** | 10 основных + 2 бонус |

---

## Материалы

<details>
<summary>📚 Документация</summary>

- [Flask 3.1](https://flask.palletsprojects.com/en/latest/) — канонический микрофреймворк
- [FastAPI](https://fastapi.tiangolo.com/) — async, OpenAPI из коробки
- [Django 5.2](https://docs.djangoproject.com/en/stable/) — full-stack; для этой лабораторной избыточен
- [Python `platform`](https://docs.python.org/3/library/platform.html) / [`socket`](https://docs.python.org/3/library/socket.html)
- [PEP 8](https://pep8.org/), [PEP 660](https://peps.python.org/pep-0660/)

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных прогонов)</summary>

- **Naive и aware datetime** — `datetime.now() - START_TIME` бросает `TypeError`, если один объект с часовым поясом, а другой без. Выберите один вариант (везде `datetime.now(timezone.utc)`) и не смешивайте.
- **`bool("False") == True`** — никогда не пишите `DEBUG = bool(os.getenv("DEBUG", "False"))`. Сравнивайте строку в нижнем регистре.
- **Предупреждение Flask 3 об устаревании** на `flask.__version__` — если нужно напечатать версию фреймворка, берите `importlib.metadata.version("flask")`.
- **Возврат dict из FastAPI и из Flask** — FastAPI сериализует сам, Flask нужен `jsonify(...)`. Не возвращайте голый dict из обработчика Flask.
- **`socket.gethostname()` внутри контейнера** позже вернёт ID контейнера, а не имя ноутбука. Так и должно быть. Вы увидите это снова в Lab 2.
- **Порт 5000 занят в macOS** — на macOS Monterey и новее *AirPlay Receiver* слушает `:5000`. Дефолт Flask с ним сталкивается: `curl` попадёт в сервис Apple. Либо выключите AirPlay Receiver (System Settings → General → AirDrop & Handoff), либо для обычного запуска возьмите `PORT=5050`.
- **`pip install -r requirements.txt` вне venv** — ставит Flask в систему, а `python app.py` может подхватить другой, уже стоявший Flask. Всегда активируйте venv (`source venv/bin/activate`) перед `pip install` и перед `python app.py`.
- **Не возвращайте Response из обработчика по умолчанию у `@app.errorhandler`** — обработчик 500 во Flask должен вернуть кортеж `(body, status)`. Голый `jsonify(...)` уходит как HTTP 200 с JSON ошибки. Тонкий баг, который потом ломает probe в Lab 9.

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- [jq](https://jqlang.github.io/jq/) — JSON в командной строке; `curl ... | jq .` пригодится постоянно
- [HTTPie](https://httpie.io/) — удобнее `curl`, когда проверяете запрос руками
- [Ruff](https://docs.astral.sh/ruff/) — быстрый линтер Python (его ставят в CI Lab 3)

</details>

---

## Что дальше

| Lab | Что добавляет этому сервису |
|---:|---|
| 2 | Multi-stage Dockerfile, сканирование образа, push в реестр |
| 3 | CI: pytest, lint, сборка образа и гейт Trivy на каждый PR |
| 7 / 8 | Структурированные JSON-логи (Loki и Alloy) и эндпоинт `/metrics` (Prometheus) |
| 9 / 10 | Выкат в Kubernetes (k3d) и Helm-чарт 4 |
| 11 / 12 | OpenBao для секретов, ConfigMap и PVC для состояния |
| 13 / 14 | GitOps через ArgoCD и canary rollout |

Держите код простым. Вы ещё вернётесь и перепишете куски. В этом смысл.
