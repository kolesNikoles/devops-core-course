# Lab 17 — Деплой Cloudflare Workers на edge

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Edge%20Computing-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![type](https://img.shields.io/badge/type-Exam%20Alternative-purple)

> **Цель:** Соберите serverless HTTP API на глобальном edge Cloudflare — это V8 isolate, не контейнер — и честно сравните его со своими замерами Kubernetes из Lab 9.
> **Что сдать:** PR из `lab17` с каталогом `edge-api/` (проект Worker) и файлом `docs/LAB17.md` (сравнение и доказательства).

## Обзор

Cloudflare Workers выполняет ваш код в **V8 isolate** — изолированной JS-куче внутри уже запущенного процесса `workerd` в 330+ точках присутствия (PoP). Холодный старт — **<5 мс**, нижняя граница памяти — около 3 МБ на изолят, регион выбирать не нужно: каждый деплой глобальный. Это другой край спектра абстракций относительно Kubernetes, который вы собирали в Lab 9–16.

В этой лабораторной вы:

- соберёте каркас Worker через **Wrangler 4.95+** (в 4.x по умолчанию режим **local-first**; `--remote` — opt-in)
- напишете форму обработчика `export default { async fetch(...) }` из runtime V8
- подключите состояние на edge через **Workers KV** и секреты Wrangler
- прочитаете метаданные запроса на edge (`request.cf.colo`) и покажете модель маршрутизации по PoP
- сделаете **честное сравнение по своим замерам**: Workers и ваш Deployment в Kubernetes из Lab 9

> ⚠️ **Границы:** эта лабораторная **не** выкатывает Docker-образ из Lab 2. У Workers нет `FROM`, нет Ingress, нет HPA, нет файловой системы. Вы соберёте API на Workers с теми же эксплуатационными вопросами (маршруты, health, конфиг, состояние, логи, деплои) в модели edge — и напишете, где постановка задачи из-за этого меняется.

> ⚠️ **Это бонусная лабораторная и альтернатива экзамену.** Чтобы заменить финальный экзамен, сдайте и Lab 17 (12 баллов), и Lab 18 (12 баллов) до нужного порога.

---

## Требования к альтернативе экзамена

| Требование | Детали |
|---|---|
| **Дедлайн** | за 1 неделю до даты экзамена |
| **Минимальный балл** | **10/12** по каждой из Lab 17 и Lab 18 |
| **Нужно сдать** | и Lab 17, **и** Lab 18 |
| **Всего баллов** | Вместе они дают вариант **замены экзамена** |

> Сама по себе Lab 17 — **бонусная** лабораторная: **10 баллов** за основные задания + **2 балла** за бонус = **максимум 12 баллов**.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**

- **Lab 9** — Deployment и Service в Kubernetes на k3d, тот же HTTP-сервис, который вы растите с Lab 1. Цифры холодного старта, задержки и ресурсов понадобятся в задании 3.

**Эта лабораторная добавляет:**

- `edge-api/` — Cloudflare Worker (один `src/index.js`, один `wrangler.toml`, один KV namespace)
- `docs/LAB17.md` — отчёт, в том числе сравнение Kubernetes и Workers

> **Про региональную связность.** В некоторых странах и сетях (в том числе в части России) сервисы Cloudflare бывают частично закрыты. Если `npx wrangler whoami` или `npx wrangler deploy` падают с невнятными сетевыми ошибками, дело может быть в сетевом пути. На VPN берите режим **full-tunnel** / глобальной маршрутизации: при split-tunnel трафик Node.js может обойти VPN и уйти в ограниченную сеть.

> **Нет аккаунта Cloudflare?** Все задания, кроме живого деплоя на `workers.dev` и фиксации `request.cf`, можно закрыть через `wrangler dev --local` (`workerd` локально крутит тот же runtime V8 isolate). Чем заменить доказательство — в подразделе **Documented-gap** задания 1.6. Бонус и полные баллы за задание 3 по-прежнему требуют настоящего деплоя.

---

## Подготовка

Нужно:

```bash
node --version           # 18+
npm --version
npx wrangler --version   # 4.95.0 or newer (4.x is local-first)
```

Если Wrangler ещё не стоит, вы поставите его через генератор C3 в задании 1.

---

## Задание 1 — Каркас, сборка и деплой API на Worker (4 балла)

### 1.1 — Каркас проекта

Возьмите C3 (`create-cloudflare`) с шаблоном **Hello World / Worker only**:

```bash
npm create cloudflare@latest -- edge-api
cd edge-api
```

Когда спросят, выберите: *Hello World example* → *Worker only* → **JavaScript** (можно TypeScript, если так удобнее; скелет ниже на JS) → Git: **Yes** → Deploy now: **No**.

Дальше:

```bash
npx wrangler --version   # confirm 4.95 or newer
```

> Зачем 4.95+? В Wrangler 4.x команда `wrangler dev` стала **local-first**: Worker выполняется в `workerd` на вашей машине. Флаг `--remote` теперь opt-in. На старой 3.x `dev` ходит в preview-инфраструктуру Cloudflare. Поведение другое, и запросы тратятся.

### 1.2 — Настройте `wrangler.toml`

Генератор C3 создаёт `wrangler.jsonc`. Оставьте его или переименуйте в `wrangler.toml`: работают оба. Здесь берём TOML, так короче. Откройте файл и заполните значения сами:

```toml
# wrangler.toml — YOUR TASK: fill every value
name = "___"                    # YOUR TASK: pick a worker name (lowercase, hyphens, your-handle-ish)
main = "___"                    # YOUR TASK: path to your entry file (e.g. src/index.js)
compatibility_date = "___"      # YOUR TASK: ISO date — pin runtime behavior; pick today (YYYY-MM-DD)

# Plaintext vars (Task 2 will add real ones)
[vars]
APP_NAME = "edge-api"
```

> **Зачем `compatibility_date`?** Он фиксирует семантику runtime `workerd`: Cloudflare может менять значения по умолчанию и не ломать ваш Worker. Сегодняшняя дата подойдёт. Пустым не оставляйте.

### 1.3 — Напишите обработчик `fetch`

Откройте `src/index.js`. Скаффолд C3 даёт Hello World — **удалите его**. Точка входа Worker — один объект, экспортированный как `default`, с одним асинхронным методом `fetch(request, env, ctx)`. Вся форма такая:

```javascript
// src/index.js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // YOUR TASK: route on url.pathname for "/", "/health", "/echo"
    // YOUR TASK: GET /         → Response.json with service info
    //            keys (mirror Lab 1 shape): { service, version, framework, runtime, edge: { colo }, path }
    // YOUR TASK: GET /health   → Response.json({ status: ___, edge: true }) with HTTP 200
    // YOUR TASK: /echo
    //            POST → echo the request body back (same Content-Type)
    //            GET  → 405 Method Not Allowed, JSON body { error: ___, allow: ___ }
    // YOUR TASK: anything else → 404 with JSON body { error: ___, path: url.pathname }

    return ___;
  }
};
```

Требования:

- **`GET /`** возвращает минимум пять ключей верхнего уровня: `service`, `version`, `framework` (`"cloudflare-workers"`), `runtime` (`"v8-isolate"`), `edge.colo` (читается из `request.cf?.colo` — см. заметку ниже) и `path`. Держите форму JSON как у `/` в Lab 1, чтобы сравнение было честным.
- **`GET /health`** возвращает `{ status: "ok", edge: true }` с HTTP 200. Именно это поможет увидеть `wrangler tail` в задании 3.
- **`POST /echo`** читает тело запроса (`await request.text()` или `await request.json()`) и возвращает его. `GET /echo` отвечает **405**, а в JSON заложен смысл `Allow: POST`.
- **Всё остальное** отвечает **404** телом JSON (не HTML — тот же урок, что в Lab 1).

> **`request.cf?.colo`** Cloudflare заполняет на настоящем edge. В `wrangler dev --local` там `undefined` или заглушка (`"TBS"`). Так и задумано. В обработчике `/` нужен optional chaining, иначе будет исключение.

> **`Response.json(obj, init?)`** — современный способ отдать JSON из Worker: сам ставит `Content-Type: application/json`. Доступен с `compatibility_date: 2022-01-31`. На более старой дате пришлось бы руками писать `new Response(JSON.stringify(obj), { headers: { "content-type": "application/json" } })`. Ещё одна причина зафиксировать свежую дату в п. 1.2.

### 1.4 — Локальный запуск на `workerd`

```bash
npx wrangler dev
```

Wrangler поднимает процесс `workerd` на `http://localhost:8787`. Это **тот же** runtime V8 isolate, который Cloudflare запускает на edge. Локально поведение совпадает с продом (кроме `request.cf` и глобальной маршрутизации).

`ВАША ЗАДАЧА`: пробейте каждый маршрут через `curl` и сохраните JSON. Что проверить:

- `curl -s http://localhost:8787/ | jq .`
- `curl -s http://localhost:8787/health | jq .`
- `curl -s -X POST -H 'content-type: application/json' -d '{"ping":42}' http://localhost:8787/echo | jq .`
- `curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8787/echo` (GET → ждите `HTTP 405`)
- `curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8787/nope` (ждите `HTTP 404`)

### 1.5 — Глобальный деплой

```bash
npx wrangler login         # OAuth in browser; opens a localhost callback
npx wrangler whoami        # confirms account + lists your workers.dev subdomain
npx wrangler deploy
```

> ⚠️ **Ловушка headless и удалённой VM:** `wrangler login` открывает браузер локально и слушает `127.0.0.1:8976`, чтобы принять OAuth callback. Если вы по SSH на удалённой VM и локального браузера нет, callback до браузера не дойдёт. Два варианта: (a) `ssh -L 8976:localhost:8976 <user@host>` — пробросить OAuth callback на ноутбук; (b) экспортировать `CLOUDFLARE_API_TOKEN=<token>` из токена, который вы создаёте в дашборде (Account → API Tokens → шаблон "Edit Cloudflare Workers"). В §1.6 запишите, какой путь взяли.

В выводе будет `https://<worker-name>.<your-subdomain>.workers.dev`. Пробейте его `curl` и проверьте: JSON тот же, только `edge.colo` теперь **настоящий** код PoP (`"FRA"`, `"WAW"`, `"SVO"`, ...).

### 1.6 — Что приложить

**Вставьте в `docs/LAB17.md`:**

- вывод `npx wrangler --version` (нужна 4.95+)
- все пять захватов `curl` из 1.4 на `localhost:8787` (с **точными командами**, которые вы запускали)
- одно из двух:
  - **(a)** захват `curl` для `GET /` по выкатанному URL `workers.dev` с **настоящим** `edge.colo`, *или*
  - **(b)** подраздел **Documented-gap**: напишите, что аккаунта CF нет (или сеть блокирует деплой), вставьте **локальный** захват `GET /` и 2–3 предложения, что было бы иначе на настоящем edge (`colo` заполнен, задержка ниже 50 мс в вашем регионе, маршрутизация anycast).

---

## Задание 2 — Конфиг, секреты и состояние на edge (3 балла)

### 2.1 — Зачем это

Worker уже отвечает. Дальше проверка: можете ли вы дать ему **конфигурацию** (vars), **учётные данные** (секреты) и **состояние** (KV) и ни разу не положить их в Git. Ту же задачу разделения вы уже решали в Lab 1 (переменные окружения), Lab 11 (секреты OpenBao) и Lab 12 (ConfigMap и PVC). Смысл тот же, форма примитивов — edge.

### 2.2 — Переменная открытым текстом

`APP_NAME` вы уже добавили в `wrangler.toml` в п. 1.2. `ВАША ЗАДАЧА`: прочитайте её в Worker (`env.APP_NAME`) и выведите в поле `service` ответа `GET /`. Одним предложением в `docs/LAB17.md` напишите, почему vars **не годятся для секретов** (подсказка: они лежат в дашборде и в Git, их читает любой соавтор).

### 2.3 — Секреты Wrangler

Создайте **два** секрета. Wrangler спросит значение. В репозиторий оно не попадёт:

```bash
npx wrangler secret put ___        # YOUR TASK: name your first secret
npx wrangler secret put ___        # YOUR TASK: name your second secret
npx wrangler secret list           # names only, no values
```

Потом прочитайте их внутри Worker. Привязки лежат на `env`:

```javascript
// YOUR TASK: in your fetch handler, read the secret you put above
const token = env.___;             // YOUR TASK: must match the name in `wrangler secret put`
```

Покажите один из них на **закрытом** маршруте. Например, `GET /admin` отвечает 401, пока не выполнится `request.headers.get("authorization") === \`Bearer ${env.YOUR_SECRET}\``. Смысл — доказать полный путь секрета: `wrangler secret put` → `env` → обработка запроса.

> **Локальная разработка и секреты:** `wrangler dev` берёт локальные переопределения из `.dev.vars` (файл в gitignore, по форме как `.env`). Для настоящего деплоя считается только `wrangler secret put`.

### 2.4 — Workers KV (обязательно)

KV — eventually-consistent хранилище ключ–значение у Workers (чтение может ненадолго отставать от записи). Подходит для флагов, счётчиков и небольших блобов конфига. Два шага: создать namespace, затем привязать его.

```bash
npx wrangler kv namespace create ___      # YOUR TASK: pick a binding name (e.g. SETTINGS)
```

Команда печатает фрагмент конфига. Скопируйте выданный **id** в `wrangler.toml`:

```toml
[[kv_namespaces]]
binding = "___"     # YOUR TASK: same binding name you used above — this is what env.<binding> exposes
id      = "___"     # YOUR TASK: the namespace id Wrangler just printed (a 32-char hex string)
```

Дальше напишите маршрут `/counter`: он увеличивает счётчик визитов в KV.

```javascript
// YOUR TASK: inside fetch, handle url.pathname === "/counter"
// 1. read the current value with env.<binding>.get("visits") (returns string or null)
// 2. compute the next integer
// 3. write it back with env.<binding>.put("visits", String(n))
// 4. return Response.json({ visits: n })
```

Прогоните туда-обратно из CLI и убедитесь, что namespace настоящий:

```bash
npx wrangler kv key put --binding=___ "visits" "0"        # YOUR TASK: binding name
npx wrangler kv key get --binding=___ "visits"            # YOUR TASK: same binding
```

> **D1, R2, Durable Objects** сидят на той же модели `env.<binding>`:
> - **D1** — SQLite на регион, с репликацией. Берите, когда нужны join.
> - **R2** — хранилище блобов, совместимое с S3, **без платы за egress**. Для ассетов и бэкапов.
> - **Durable Objects** — акторы со строгой согласованностью и своим состоянием. Для чатов, таблиц лидеров, rate-limiter.
> Здесь правильный выбор — KV. Бонус позволяет заменить его на один из остальных.

### 2.5 — Что приложить

**Вставьте в `docs/LAB17.md`:**

- вывод `npx wrangler secret list`: **имена** двух секретов (значений быть не должно)
- `curl`, который доказывает закрытый маршрут из 2.3: один вызов без заголовка (401), один с ним (200)
- прогон `npx wrangler kv key put` / `kv key get` из 2.4 (показательные id подойдут; ваши будут другими)
- `curl http://localhost:8787/counter` (или выкатанный URL), **три** вызова подряд. В третьем ответе должно быть `visits: 3`, *и* после повторного деплоя счётчик продолжается, а не сбрасывается, если вы деплоили
- одно предложение, почему vars не годятся для секретов

---

## Задание 3 — Наблюдаемость, эксплуатация и сравнение с Kubernetes (3 балла)

Это практический вывод Lecture 16. Workers и Kubernetes решают одну задачу — запустить мой HTTP-код в проде — и платят за это очень разную цену.

### 3.1 — Живые логи

Добавьте в обработчик `fetch` хотя бы один вызов `console.log()`: логируйте путь и colo (он может быть `undefined`).

```javascript
// YOUR TASK: add console.log with at least the path and request.cf?.colo
console.log(___);
```

Затем в другом терминале:

```bash
npx wrangler tail
```

Из третьего шелла пробейте любой маршрут. Строка лога приходит с edge почти сразу.

`ВАША ЗАДАЧА`: сохраните **одну** строку `wrangler tail` и вставьте буквальную запись (с метками времени и всем остальным) в `docs/LAB17.md`.

### 3.2 — Версии и откат

Выкатите минимум **две** версии Worker. Между ними хватит мелочи (поднимите поле `version` в `GET /`). Затем выведите список и откатитесь:

```bash
npx wrangler deployments list      # YOUR TASK: capture this — version IDs + timestamps
npx wrangler rollback              # YOUR TASK: capture this — confirms the rollback
```

Если аккаунта CF нет, опишите это как пробел: что напечатала бы каждая команда (Wrangler показывает нумерованный список версий с автором и временем; `rollback` просит подтверждение и атомарно возвращает предыдущую версию).

### 3.3 — Таблица Kubernetes и Workers (ради неё всё)

Откройте заметки **Lab 9**: холодный старт свежего Pod, задержка запроса с ноутбука до Service в k3d, ресурсы, которые вы зарезервировали на Pod. Если забыли, на свежем Deployment снова запустите `kubectl get pods -w` и один раз `time curl ...`, когда он станет Ready. Затем заполните таблицу **своими замерами**, не дефолтами из лекции.

`ВАША ЗАДАЧА`: заполните каждую ячейку этой таблицы в `docs/LAB17.md`. У каждой цифры укажите источник.

| Аспект | Kubernetes (Lab 9, ваши цифры) | Cloudflare Workers (эта лабораторная, ваши цифры) |
|---|---|---|
| Холодный старт (первый запрос после scale-from-zero) | ___ с (`kubectl get pods -w` от `Pending` → `Ready` + первый `200`) | ___ мс (`time curl` по свежему деплою) |
| Нижняя граница памяти на экземпляр | ___ Mi (`resources.requests.memory` вашего Pod) | ~3 MB (V8 isolate, по документации) |
| География | ___ (ваш единственный узел k3d — какой регион или ноутбук) | 330+ PoP, все сразу |
| Цена за 1 млн запросов в простое | ___ ($/мес при нуле трафика — счёт за узел, если арендуете; на k3d $0, но напишите, сколько стоил бы управляемый аналог) | **Free tier:** 100k req/day × 30 = ~3M/мес за $0. **Платный ($5/мес):** 10M запросов + 50 мс CPU |
| Примитив состояния | ___ (например, PVC + local-path из Lab 12) | Workers KV (эта лабораторная) |
| Долгие задачи | да — любой процесс | максимум 30 с wall-clock (платный), CPU 10 мс (бесплатный) / 50 мс (платный, burst до 5 мин) |
| Лучший сценарий | ___ (ваш вывод в одну строку) | ___ (ваш вывод в одну строку) |

> **Смысл:** по холодному старту и нижней границе цены Workers выигрывает на API с рваной нагрузкой, где важна задержка. Kubernetes выигрывает на долгих, stateful, нативных, многосервисных нагрузках. Честным сравнение делают цифры из ваших собственных лабораторных.

### 3.4 — Рефлексия (3 предложения, в `docs/LAB17.md`)

`ВАША ЗАДАЧА`: ответьте своими словами —

1. Что в Workers было **проще**, чем в вашей схеме Kubernetes из Lab 9?
2. Где было **теснее**? (У V8 isolate нет `fs`, нет `Buffer` по умолчанию, нет нативных бинарников, на free tier CPU 10 мс.)
3. Что изменилось из-за того, что Workers — **не хост Docker**? (Подумайте: нет `Dockerfile`, нет сканирования, нет Ingress, нет HPA. Нет и выбора базового образа, не на что опереться с multi-stage из Lab 2.)

### 3.5 — Что приложить

**Вставьте в `docs/LAB17.md`:**

- строка `wrangler tail` из 3.1
- захваты `deployments list` и `rollback` из 3.2 (или эквивалент documented-gap)
- полностью заполненная таблица из 3.3 с вашими замерами
- рефлексия из трёх предложений из 3.4

---

## Бонус — свой домен или D1/R2/Durable Objects (2 балла)

Выберите **одно**:

**Вариант A — маршрутизация дальше `workers.dev`.** Привяжите Worker к Custom Domain, которым вы владеете (зона в Cloudflare). Домена нет — опишите процедуру целиком: как выглядит `wrangler.toml`, какие DNS-записи создаёт Cloudflare, какой сертификат выпускается. И покажите блок `routes`:

```toml
# YOUR TASK: routes block for a custom domain you control (or document precisely)
[[routes]]
pattern        = "___"            # YOUR TASK: e.g. api.example.com
custom_domain  = true
```

**Вариант B — второй примитив состояния.** Добавьте **D1**, **R2** или **Durable Object**, привяжите и вызовите с одного маршрута.

```bash
# YOUR TASK: pick ONE
npx wrangler d1 create ___        # SQLite per region
npx wrangler r2 bucket create ___ # blob storage, no egress fees
# Durable Objects need a class export — see the docs
```

Затем пропишите привязку в `wrangler.toml` и вызовите её из нового маршрута. В `docs/LAB17.md` — **один абзац**: почему для сделанного сценария вы берёте это, а не KV (согласованность? join? блобы? координация?).

---

## Как сдавать

```bash
git switch -c lab17
git add edge-api/ docs/LAB17.md
git commit -m "feat(lab17): cloudflare worker edge API + K8s comparison"
git push -u origin lab17
```

Откройте **два** PR:

- `your-fork:lab17` → `course-repo:master` *(на ревью)*
- `your-fork:lab17` → `your-fork:master` *(сольётся в ваш main, когда закончите)*

Если деплоили, в описание PR положите живой URL `workers.dev`. Иначе сошлитесь на раздел documented-gap в `docs/LAB17.md`. **Значения секретов в коммит не кладите.**

Чеклист PR:

```text
- [ ] Task 1 done — scaffold, fetch handler, 3 routes + 404 + 405, local run, (deploy or documented gap)
- [ ] Task 2 done — var, ≥2 secrets, KV namespace + counter route, round-trip CLI
- [ ] Task 3 done — wrangler tail capture, deployments list + rollback, K8s-vs-Workers table filled, reflection
- [ ] Bonus done — Custom Domain config OR D1/R2/Durable Object wired (with justification)
```

---

## Критерии приёмки

### Задание 1 (4 балла)

- ✅ Wrangler 4.95+ установлен; `whoami` работает (если есть аккаунт) или пробел задокументирован
- ✅ в `wrangler.toml` заполнены `name`, `main`, `compatibility_date` значениями, которые выбрал студент
- ✅ обработчик `fetch` реализует `/`, `/health`, `/echo` (GET даёт 405, POST делает echo) и JSON 404
- ✅ все пять локальных захватов `curl` есть в `docs/LAB17.md`
- ✅ есть захват деплоя на `workers.dev` (с настоящим `edge.colo`) или раздел documented-gap

### Задание 2 (3 балла)

- ✅ в `wrangler.toml` есть `[vars]` и блок `[[kv_namespaces]]` со студенческими `binding` и настоящим `id`
- ✅ захват `wrangler secret list` показывает два именованных секрета (значения не в коммите)
- ✅ закрытый маршрут даёт 401 без авторизации и 200 с правильным заголовком
- ✅ `/counter` растёт за три вызова; прогон KV из CLI сохранён

### Задание 3 (3 балла)

- ✅ в обработчике есть `console.log()`; в `docs/LAB17.md` лежит одна буквальная строка `wrangler tail`
- ✅ есть захват `deployments list` и `rollback` (или пробел задокументирован)
- ✅ таблица Kubernetes и Workers заполнена целиком **собственными замерами студента из Lab 9** и цифрами его Worker
- ✅ рефлексия из трёх предложений отвечает на все три вопроса

### Бонус (2 балла)

- ✅ конфиг Custom Domain работает или описан точно, или
- ✅ D1 / R2 / Durable Object создан, привязан и вызывается из маршрута — с обоснованием на один абзац

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|---|---:|---|
| **Задание 1** — Каркас, сборка, деплой | **4** | Обработчик и маршруты верные, локальный прогон сохранён, деплой или задокументированный пробел |
| **Задание 2** — Конфиг, секреты, KV | **3** | Переменная, 2 секрета и счётчик KV работают, значения не утекли |
| **Задание 3** — Наблюдаемость и сравнение | **3** | Захват tail, список деплоев и откат, таблица сравнения **по замерам студента**, рефлексия |
| **Бонус** — Custom Domain или второй примитив состояния | **2** | Один вариант сделан или точно описан |
| **Итого** | **12** | 10 за основное + 2 бонус |

**Оценивание:**

- **10/10 за основное:** Worker выкачен (или пробел явно обоснован), показана сохранность в KV, таблица сравнения стоит на настоящих цифрах, рефлексия конкретная
- **8–9/10:** Worker работает, в доказательствах или разборе мелкие пробелы
- **6–7/10:** базовый деплой есть, но не хватает KV, наблюдаемости или глубины сравнения
- **<6/10:** реализация не закончена

---

## Материалы

<details>
<summary>📚 Основная документация Cloudflare Workers</summary>

- [Обзор Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Старт с Wrangler](https://developers.cloudflare.com/workers/get-started/guide/)
- [Миграция Wrangler v3 → v4 (local-by-default)](https://developers.cloudflare.com/workers/wrangler/migration/update-v3-to-v4/)
- [Команды Wrangler](https://developers.cloudflare.com/workers/wrangler/commands/)
- [Тарифы Workers (free tier: 100k req/day)](https://developers.cloudflare.com/workers/platform/pricing/)
- [Как устроены Workers (V8 isolate)](https://developers.cloudflare.com/workers/reference/how-workers-works/)

</details>

<details>
<summary>🔐 Конфиг, секреты и состояние</summary>

- [Переменные окружения](https://developers.cloudflare.com/workers/configuration/environment-variables/)
- [Секреты](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Workers KV](https://developers.cloudflare.com/kv/) · [D1](https://developers.cloudflare.com/d1/) · [R2](https://developers.cloudflare.com/r2/) · [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Как выбрать хранилище](https://developers.cloudflare.com/workers/platform/storage-options/)
- [Request API и `request.cf`](https://developers.cloudflare.com/workers/runtime-apis/request/)

</details>

<details>
<summary>📊 Наблюдаемость и деплои</summary>

- [Обзор наблюдаемости](https://developers.cloudflare.com/workers/observability/)
- [`wrangler tail`](https://developers.cloudflare.com/workers/observability/logs/real-time-logs/)
- [Версии и деплои](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/)
- [Откаты](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/rollbacks/)

</details>

<details>
<summary>⚠️ Типичные ловушки (из реальных пробных прогонов)</summary>

- **В V8 isolate нет файловой системы.** `import fs from "node:fs"` падает на деплое. Каталога `/tmp` нет. Сохранять можно в KV, D1, R2, DO — или никак.
- **Node API по умолчанию нет.** `Buffer`, `process`, `crypto` (вариант Node) отсутствуют. Чтобы взять подмножество, добавьте `compatibility_flags = ["nodejs_compat"]` в `wrangler.toml`. Это включают явно не просто так: V8 isolate — не Node.
- **KV — eventually consistent.** `put`, за которым сразу идёт `get` с другого PoP, до **~60 секунд** может отдать старое значение. Для счётчиков и флагов нормально. Для «нажал ли пользователь кнопку покупки» — нет. Когда нужна гарантия read-your-writes, берите Durable Objects.
- **Content-Type у `Response.json()`.** Ставится сам начиная с `compatibility_date: 2022-01-31`. Если дата старая (или вы скопировали древний пример), `content-type: application/json` придётся выставить руками. Зафиксируйте свежую дату в `wrangler.toml`.
- **В 4.x `wrangler dev` по умолчанию локальный.** Он запускает `workerd` на ноутбуке: `request.cf` — заглушка, KV — локальный эмулятор на SQLite, секреты берутся из `.dev.vars`. `wrangler dev --remote` включайте, только если в разработке нужны настоящие метаданные edge (это ест квоту).
- **`wrangler secret put` требует, чтобы Worker уже был на Cloudflare.** Если `wrangler deploy` ещё ни разу не запускали, команда секрета падает. Сначала выкатите заглушку, потом добавьте секреты, потом выкатите настоящую версию.
- **Поддомен `workers.dev` выбирают один раз на аккаунт.** Первый деплой просит указать `<you>.workers.dev`. Позже его не сменить без поддержки на уровне аккаунта.
- **В локальной разработке `request.cf?.colo` равен `undefined`** (или постоянная заглушка `"TBS"`). Всегда ставьте optional chaining.

</details>

<details>
<summary>🐍 По желанию: Python Workers</summary>

- [Python Workers](https://developers.cloudflare.com/workers/languages/python/) — крутится на Pyodide; **расширений на C нет** (нет NumPy, нет Pillow). Если в Lab 1 вы пошли в Python и хотите остаться на этом пути, так можно, но тесно.
- [Пакеты Python Worker](https://developers.cloudflare.com/workers/languages/python/packages/)

</details>

---

## Что дальше

| Lab | Что добавляет сверх Lab 17 |
|---:|---|
| 18 | Воспроизводимые сборки на Nix — вторая бонусная лабораторная и альтернатива экзамену. Вместе: 20% оценки, полная замена экзамена |
| — | Workers AI, R2 + R2 Data Catalog, Hyperdrive, Pages Functions — остальная платформа Cloudflare для разработчиков, если после курса захотите копнуть глубже |

**Удачи.** 🌍

> **Помните:** V8 isolate ≠ контейнеры. По холодному старту и нижней границе цены Workers выигрывает на глобально распределённых API. Kubernetes выигрывает на долгих, stateful, нативных, многосервисных нагрузках. Сначала назовите нагрузку, runtime выбирайте в конце.
