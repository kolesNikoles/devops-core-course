# Lab 14 — Progressive Delivery с Argo Rollouts

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Progressive%20Delivery-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Argo%20Rollouts%201.8.4-informational)

> **Цель:** замените `kind: Deployment` на `kind: Rollout`. Список canary `steps:` напишите сами, манифесты active и preview `Service` напишите сами и (в бонусе) напишите PromQL для метрического гейта, который сам обрывает релиз, если метрика просела.
> **Что сдать:** PR из `lab14` с `k8s/lab10-app/templates/rollout.yaml`, двумя `Service` для blue-green, необязательным `AnalysisTemplate` и `docs/LAB14.md`.

---

## Обзор

До Lab 9 rolling update у вас шёл обычным `Deployment`: Kubernetes меняет Pod пачками по 25% и перестаёт мерить в ту секунду, когда readiness-probe становится зелёным. Всплеск 5xx, который бьёт только по 5% запросов, всё равно уезжает на 100% пользователей. **Progressive delivery** замыкает этот цикл: режьте трафик ступенями, каждую ступень закрывайте метриками, и пусть контроллер откатывает сам — без человека, который в пятницу смотрит в Grafana.

В этой лабораторной вы:
- берёте CRD `Rollout` как замену `Deployment` без переделки шаблона Pod — тот же pod template, новый блок `spec.strategy:`
- **пишете список canary `steps:` с нуля** — каждый `setWeight` и каждый `pause` ваши
- **пишете с нуля манифесты active и preview `Service`** для blue-green
- **пишете запрос к Prometheus и условия гейта** в `AnalysisTemplate` для бонуса
- гоняете rollout командой `kubectl argo rollouts <verb>` и учите глаголы по именам

> ⚠️ **Границы:** в основном пути нет интеграции с traffic-router — `setWeight` приближает долю репликами (Lecture 14, слайд 12). Один вариант с traffic-router лежит в бонусе. Без mesh и без контроллера уведомлений. Держитесь ядра canary + blue-green и каждую строку стратегии пишите сами.

> 🪨 **Учебное ядро.** Главное доказательство этой лабораторной — один экран: `kubectl argo rollouts get rollout lab10-app-web` показывает **`Status: ॥ Paused`**, **`Message: CanaryPauseStep`** и **`SetWeight: 25, ActualWeight: 25`** на шаге 1/6. Затем, после `promote --full`, та же команда на **шаге 6/6 `Healthy`**. `Rollout`, который сразу уходит на 100% без видимой паузы, progressive delivery не упражнял. **Сохраните этот снимок.**

---

## Состояние проекта

**К этой лабораторной у вас уже должно быть:**
- Кластер k3d на Kubernetes **1.36** (Lab 9)
- Helm-чарт в `k8s/lab10-app/` из Lab 10 — шаблон `Deployment`, шаблон `Service`, `_helpers.tpl` и `values.yaml`
- ArgoCD **3.4**, который ведёт этот чарт через `ApplicationSet` (Lab 13)
- Служебные сервисы `echo` и `health` из Lab 9 / 13 — у `health` есть `/metrics`, он пригодится в бонусе
- Экземпляр Prometheus из **Lab 8** (Docker-Compose) — где он окажется относительно кластера, решите в подготовке

**Эта лабораторная добавляет:**
- `k8s/lab10-app/templates/rollout.yaml` — **пишете вы** (задание 2: стратегия canary)
- `k8s/lab10-app/templates/service-active.yaml` — **пишете вы** (задание 3: blue-green active)
- `k8s/lab10-app/templates/service-preview.yaml` — **пишете вы** (задание 3: blue-green preview)
- `k8s/lab10-app/templates/analysistemplate.yaml` — **пишете вы** (бонус)
- `k8s/lab10-app/values-bluegreen.yaml` — переключатель потока blue-green (задание 3)
- `docs/LAB14.md` — отчёт к сдаче

Служебные файлы курса для этой лабораторной:
- `labs/lab14/prometheus-pointer.md` — что делать, если Prometheus из Lab 8 всё ещё стек Docker-Compose снаружи кластера (бонусу нужен URL изнутри кластера). **Не правьте.**

К **Lab 16** голый Prometheus внутри кластера сменится на kube-prometheus-stack и `ServiceMonitor`. `AnalysisTemplate`, который вы напишете здесь, как раз читает метрики с этой цепочки.

---

## Подготовка

| Компонент | Версия | Заметки |
|---|---|---|
| Контроллер Argo Rollouts | **v1.8.4** | Вышел 13 февраля 2026. НЕ берите `latest` |
| Плагин `kubectl argo rollouts` | **v1.8.4** | Та же версия, что у контроллера |
| Kubernetes (k3s) | `v1.36.1-k3s1` | из Lab 9 |
| Helm | `v4.1.x` | из Lab 10 |
| Prometheus | `3.x` | экземпляр из Lab 8 — см. заметку про кластер ниже |

```bash
kubectl get nodes                  # 3-node k3d cluster from Lab 9 must be Running
helm version                       # 4.1.x from Lab 10
kubectl argo rollouts version      # not installed yet — that's Task 1
```

> ⚠️ **Для бонуса важно, где стоит Prometheus.** В Lab 8 Prometheus крутился в стеке Docker-Compose на ноутбуке. `AnalysisTemplate` **внутри** кластера не достучится до `http://localhost:9090` на хосте. Три варианта, см. `labs/lab14/prometheus-pointer.md`:
>
> 1. **Смотреть на шлюз хоста** — k3d отдаёт `host.k3d.internal`; для бонуса сегодня хватит, но Lab 16 всё равно заменит это стеком внутри кластера.
> 2. **Поднять маленький Prometheus в кластере** в namespace `monitoring` (указатель на манифест в `labs/lab14/`).
> 3. **Пропустить бонус.** Основным заданиям 1–4 Prometheus не нужен.

Каталог, который вы соберёте:

```
k8s/lab10-app/
├── templates/
│   ├── rollout.yaml                # YOU write — replaces deployment.yaml (Task 2)
│   ├── service-active.yaml         # YOU write (Task 3)
│   ├── service-preview.yaml        # YOU write (Task 3)
│   └── analysistemplate.yaml       # YOU write (Bonus)
├── values.yaml                     # add a `strategy.type` toggle
└── values-bluegreen.yaml           # YOU write (Task 3) — override `strategy.type`
docs/
└── LAB14.md                        # your submission report
```

---

## Задание 1 — Основы Argo Rollouts (2 балла)

### 1.1 — Зачем отдельный контроллер, а не Deployment

В `docs/LAB14.md` напишите по **3** предложения на каждый пункт:

1. По какому сигналу `Deployment` решает, что rollout прошёл, и почему этого мало, если баг стреляет на 5% запросов?
2. Как Argo Rollouts сдвигает «трафик» без traffic-router (Lecture 14, слайд 12) — и где предел точности на 5 репликах?
3. Какие **три** новых поля верхнего уровня добавляет CRD `Rollout`, которых нет у `Deployment`? (Подсказка: Lecture 14, слайды 6–11 — `strategy.canary`, `strategy.blueGreen`, `analysis`/`backgroundAnalysis`.)

### 1.2 — Поставьте контроллер, зафиксированный на v1.8.4

`ВАША ЗАДАЧА`: поставьте контроллер в namespace `argo-rollouts` из манифеста релиза **v1.8.4** — не `latest` и не `main`. Шаблон URL релиза: `https://github.com/argoproj/argo-rollouts/releases/download/<version>/install.yaml`.

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f ___        # YOUR TASK: the v1.8.4 install.yaml URL
kubectl -n argo-rollouts rollout status deploy/argo-rollouts --timeout=120s
```

> 💡 Почему `latest` запрещён: релиз, который ломает analysis для `bluegreen` (у 1.8.0–1.8.3 был ровно этот баг — см. Lecture 14, слайд 21), молча уедет в лабораторные студентов. Фиксируйте версию, которую проверили преподаватели.

### 1.3 — Поставьте kubectl-плагин той же версии

`ВАША ЗАДАЧА`: скачайте бинарник плагина **v1.8.4** под свою OS/arch, сделайте его исполняемым и положите в `PATH` как `kubectl-argo-rollouts`. Проверьте `kubectl argo rollouts version` — и клиент, и сервер должны ответить `v1.8.4`.

```bash
# Linux amd64 example — adapt for darwin-arm64 / windows-amd64
curl -fSL -o kubectl-argo-rollouts ___       # YOUR TASK: the v1.8.4 plugin binary URL
chmod +x kubectl-argo-rollouts
sudo mv kubectl-argo-rollouts /usr/local/bin/

kubectl argo rollouts version                # expect: v1.8.4 client + v1.8.4 server
```

> 💡 Плагин — **не** подкоманда `kubectl`. Это отдельный бинарник, который `kubectl` вызывает снаружи. Если `kubectl argo rollouts version` говорит «unknown command», бинарника нет в `PATH`.

### 1.4 — Поставьте дашборд (необязательно, но полезно)

`ВАША ЗАДАЧА`: примените манифест дашборда v1.8.4, сделайте port-forward на `localhost:3100` и снимите скриншот пустого дашборда (заполните его, когда в задании 2 появится rollout).

```bash
kubectl apply -n argo-rollouts -f ___        # YOUR TASK: the v1.8.4 dashboard-install.yaml URL
kubectl argo rollouts ___                    # YOUR TASK: which subcommand opens the local UI on :3100?
                                             #            (one of: dashboard, ui, serve, web)
```

### 1.5 — Доказательства

**Вставьте в `docs/LAB14.md`:**

- 3 ответа из §1.1
- `kubectl -n argo-rollouts get pods` — Pod контроллера `Running`
- `kubectl argo rollouts version` — клиент и сервер оба `v1.8.4`
- скриншот пустого дашборда на `http://localhost:3100`

---

## Задание 2 — Canary-выкладка (3 балла) ← **главное задание**

### 2.1 — `Deployment` → `Rollout`

В чарте из Lab 10 есть `templates/deployment.yaml`. **Переименуйте** его в `templates/rollout.yaml` и смените kind. Всё внутри `spec.template:` остаётся как было — те же лейблы, тот же контейнер, те же порты, те же probe. CRD Rollout переиспользует pod template вашего `Deployment` байт в байт. **Селектор `Service` оставьте прежним** — Argo Rollouts добавляет лейбл `rollouts-pod-template-hash`, чтобы отличить stable от canary, но ваш текущий селектор (`app.kubernetes.io/name` + `instance`) по-прежнему матчит оба.

`ВАША ЗАДАЧА`: напишите `templates/rollout.yaml`. Каркас ниже показывает куски, **идентичные вашему Deployment** (pod template заново не собирайте), и **полностью зачищает список canary `steps:`** — каждый шаг пишете вы.

```yaml
# k8s/lab10-app/templates/rollout.yaml
apiVersion: ___                                    # YOUR TASK: which apiGroup/version owns Rollout?
                                                   #            (hint: NOT apps/v1 — see lecture 14 slide 7)
kind: ___                                          # YOUR TASK
metadata:
  name: {{ include "lab10-app.fullname" . }}-web   # SAME -web suffix as Lab 10's Deployment
                                                   # → rendered as `lab10-app-web` on the default release;
                                                   # every CLI command below targets this name.
  labels:
    {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}             # keep your Lab 10 default — needs ≥ 4 for visible 25/50/75 weights
  selector:
    matchLabels:
      {{- include "lab10-app.selectorLabels" . | nindent 6 }}
  template:
    # ⬇️ IDENTICAL to your Lab 13 Deployment pod template
    metadata:
      labels:
        {{- include "lab10-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          # ⬇️ pod template body is IDENTICAL to your Lab 10 Deployment — copy it verbatim
          # ⬇️ (ports, probes, resources, env). Shown elided here for brevity.
          # See your k8s/lab10-app/templates/deployment.yaml for the body to paste.
  strategy:
    canary:
      # YOUR TASK: write the 7-element step list for a 25 → 50 → 75 → 100 canary
      # with a MANUAL pause between every weight change. Format hints below.
      #
      # Step semantics (lecture 14 slide 7):
      #   - `- setWeight: ___`           shift traffic/replica ratio to the listed percentage
      #   - `- pause: {}`                HARD manual gate — rollout waits for `promote`
      #   - `- pause: { duration: ___ }` timed pause (e.g. "30s", "2m") — rollout resumes itself
      #
      # The headline proof of this lab is a PAUSED canary at SetWeight 25 (the lab14 refs
      # quote `CanaryPauseStep` at Step 1/6). So step 1 must shift to 25% and step 2 must be
      # a MANUAL gate, not a timed pause — otherwise the rollout never sits there long
      # enough for the `kubectl argo rollouts get` capture to show `Paused`.
      #
      # Required progression: 25 → pause → 50 → pause → 75 → pause → 100
      # (the trailing 100 doesn't need a pause; the rollout marks itself Healthy at step 6/6)
      steps:
        - setWeight: ___                            # YOUR TASK
        - pause: ___                                # YOUR TASK: which pause form? (see hint above)
        - setWeight: ___                            # YOUR TASK
        - pause: ___                                # YOUR TASK
        - setWeight: ___                            # YOUR TASK
        - pause: ___                                # YOUR TASK
        - setWeight: ___                            # YOUR TASK
```

**Зачем каждый шаг:**
- **`setWeight: 25` на шаге 1** — из-за него в `kubectl argo rollouts get` появляется `ActualWeight: 25`. Без этого первый вес будет 100, и progressive delivery на снимке не видно.
- **`pause: {}` (без duration)** стоит, пока вы не выполните `promote`. За таким гейтом живая команда как раз смотрит метрики.
- **`pause: { duration: 30s }`** — альтернатива, удобная для демо в CI, где человека не дождаться. Оба варианта допустимы, но **шаг 2 обязан быть `pause: {}`**, чтобы canary зримо постоял на 25% для снимка.
- **Хвост на 100 без паузы** — контроллер сам помечает rollout как `Healthy`. Явный `setWeight: 100` в конце нормален, а лишний `- pause: {}` после него оставляет rollout висеть.

### 2.2 — Запустите первый rollout и снимите паузу

Примените чарт (через sync ArgoCD, если подключён ApplicationSet из Lab 13, иначе `helm upgrade`). Первый apply `Rollout` создаёт ресурс; контроллер помечает его `Healthy` на 100%, не проходя шаги (шаги бегут только для **следующих** ревизий). Теперь сдвиньте тег образа, чтобы пошёл настоящий canary:

```bash
helm upgrade lab10-app ./k8s/lab10-app --set image.tag=v2
# or commit values-dev.yaml with the new tag and let ArgoCD sync it
```

Дальше смотрите:

```bash
kubectl argo rollouts ___ rollout lab10-app-web --watch   # YOUR TASK: which verb shows live status?
                                                       #            (one of: get, describe, status, list)
```

`ВАША ЗАДАЧА`: поймайте момент, когда rollout стоит на шаге 1/6 с `Message: CanaryPauseStep`, `SetWeight: 25`, `ActualWeight: 25`. Это главный артефакт Lab 14.

### 2.3 — Пройдите все гейты через promote

```bash
kubectl argo rollouts ___ lab10-app-web --full            # YOUR TASK: which verb advances past ALL remaining pauses?
                                                       #            (full = skip the manual gates AND wait for them)
```

Снова снимите `kubectl argo rollouts get rollout lab10-app-web`: **шаг 6/6, `Status: ✔ Healthy`, `SetWeight: 100`**.

### 2.4 — Abort посреди rollout

`ВАША ЗАДАЧА`: запустите третий rollout (поднимите тег до `v3`), дождитесь шага 1/6 на паузе в 25% и сделайте abort. Убедитесь, что трафик вернулся на stable ReplicaSet, а rollout пишет `Degraded`.

```bash
helm upgrade lab10-app ./k8s/lab10-app --set image.tag=v3
# wait for the paused 25% step:
kubectl argo rollouts get rollout lab10-app-web

kubectl argo rollouts ___ lab10-app-web                   # YOUR TASK: which verb cancels the in-progress rollout?
kubectl argo rollouts get rollout lab10-app-web           # expect: Status: ✖ Degraded, traffic on stable

# Now resume from step 0 once you've "fixed" the image (e.g. re-bump back to v2):
kubectl argo rollouts ___ rollout lab10-app-web           # YOUR TASK: which verb restarts a Degraded rollout?
```

### 2.5 — Доказательства

**Вставьте в `docs/LAB14.md`:**

- содержимое блока `spec.strategy.canary` из вашего `templates/rollout.yaml` (только список `steps:` — 8–10 строк)
- **главный снимок** из §2.2 — `kubectl argo rollouts get rollout lab10-app-web` с `Paused`, `CanaryPauseStep`, `Step: 1/6`, `SetWeight: 25, ActualWeight: 25`
- снимок **`promote --full`** из §2.3 — та же команда, теперь `Step: 6/6, Status: ✔ Healthy, SetWeight: 100`
- снимки **abort и retry** из §2.4 — `Degraded` после abort, затем `Healthy` после `retry`
- 2–3 предложения, что вы видели на дашборде в окне паузы: одна из 4 реплик была на новом ReplicaSet? (Lecture 14, слайд 8)

---

## Задание 3 — Blue-green (3 балла)

### 3.1 — Зачем вторая стратегия

В `docs/LAB14.md` ответьте в 2–3 предложениях:
- Какой тип workload каноничен для blue-green вместо canary? (Подсказка: Lecture 14, слайд 5 — миграции схемы.)
- Сколько копий ваших Pod крутится в окне cutover (момент переключения)? Почему это важно для стоимости кластера?

### 3.2 — Напишите два манифеста `Service` с нуля

Blue-green нужны **два** Service на один и тот же набор лейблов Pod. При promote Argo Rollouts переписывает их селекторы: `activeService` всегда смотрит на живой ReplicaSet, `previewService` — на подготовленный. В чарте Lab 10 был **один** Service; теперь вы пишете **два**.

`ВАША ЗАДАЧА`: напишите с нуля `templates/service-active.yaml` и `templates/service-preview.yaml`. Они должны:

- брать helper `selectorLabels` чарта (чтобы контроллер rollout мог менять селектор)
- отличаться только `metadata.name` — одно имя `<fullname>` (то, которым уже пользуются вызывающие), другое `<fullname>-preview`
- отдавать тот же `port` / `targetPort`, что Service из Lab 10

(Каркаса нет — к этому месту вы написали ~6 манифестов Service в Lab 9, 10, 13. Если нужна форма API, `kubectl explain service.spec` — одна команда в шелле.)

> 💡 **Имя важно.** Имя `activeService` в spec `Rollout` должно совпасть с существующим `Service` *точно*. Опечатка даёт ошибки `Rollout` вроде `service "app-python-actv" not found`, и контроллер это сам не вылечит. Берите значение через `include "lab10-app.fullname" .` из того же helper, что и у Service.

### 3.3 — Настройте стратегию blue-green

`ВАША ЗАДАЧА`: создайте `values-bluegreen.yaml`, который **переопределяет** блок `strategy:` вашего Rollout. В `values.yaml` — один булев переключатель (например `strategy.type: canary` по умолчанию; в новом файле переопределите на `bluegreen`) и ветка `{{- if eq .Values.strategy.type "bluegreen" }}` в `rollout.yaml`. Обе стратегии рядом не кладите: Argo Rollouts отвергает Rollout, где объявлены и `canary`, и `blueGreen`.

Ниже показана только внешняя форма blue-green. **Четыре значения**, которые задают поведение cutover, пустые.

```yaml
# in rollout.yaml, inside spec:
  strategy:
    {{- if eq .Values.strategy.type "bluegreen" }}
    blueGreen:
      activeService: ___                # YOUR TASK: name of the Service production traffic hits
                                        #            (hint: matches your <fullname>)
      previewService: ___               # YOUR TASK: name of the Service QA / smoke tests hit
                                        #            (hint: matches <fullname>-preview)
      autoPromotionEnabled: ___         # YOUR TASK: true or false?
                                        #            (this lab needs manual promote — see lecture 14 slide 9)
      scaleDownDelaySeconds: ___        # YOUR TASK: how many seconds to keep the OLD ReplicaSet
                                        #            alive after promotion, for instant `undo`?
                                        #            (suggested: 300 = 5 min — short enough to not pay
                                        #             for two ReplicaSets all day, long enough that you
                                        #             can `undo` if you spot a bug in the first minute)
      # prePromotionAnalysis:           # optional — covered in the bonus
    {{- else }}
    canary:
      steps:
        # ... your Task 2 step list ...
    {{- end }}
```

### 3.4 — Прогоните cutover blue-green

```bash
helm upgrade lab10-app ./k8s/lab10-app -f values-bluegreen.yaml --set image.tag=v1
# Confirm the active Service serves v1:
kubectl port-forward svc/lab10-app-web 8080:80 &
curl -s localhost:8080/ | jq .service

# Bump to v2 — the controller spins up the new ReplicaSet behind the PREVIEW Service:
helm upgrade lab10-app ./k8s/lab10-app -f values-bluegreen.yaml --set image.tag=v2

# In another shell, validate v2 via the preview Service (prod still on v1):
kubectl port-forward svc/lab10-app-web-preview 8081:80 &
curl -s localhost:8081/ | jq .service       # YOUR TASK: confirm v2

# Confirm the two Services have DIFFERENT pod IPs in their endpoints right now —
# this is the proof the controller wrote two different selectors:
kubectl get endpoints lab10-app-web lab10-app-web-preview -o wide

# Promote — the active Service flips to v2 instantly (no new pods scheduled):
kubectl argo rollouts ___ lab10-app-web                  # YOUR TASK: same verb as Task 2 manual gate

# Test instant rollback BEFORE scaleDownDelaySeconds expires:
kubectl argo rollouts ___ lab10-app-web                  # YOUR TASK: which verb rolls back to the previous ReplicaSet?
kubectl get endpoints lab10-app-web                   # active flipped back to v1 pod IPs
```

### 3.5 — Доказательства

**Вставьте в `docs/LAB14.md`:**

- оба манифеста Service целиком (каждый в своём блоке кода)
- ваш блок `blueGreen:` со всеми четырьмя значениями и одно предложение, зачем такой `scaleDownDelaySeconds`
- **два снимка `kubectl get endpoints`** из §3.4: `lab10-app-web` и `lab10-app-web-preview` смотрят на **разные IP Pod** в окне cutover — это главный артефакт blue-green
- снимки `promote` и `undo`: IP активного Service перещёлкиваются
- 2 предложения: скорость `undo` у blue-green и `abort` у canary — что быстрее и почему

---

## Задание 4 — Документация (2 балла)

`ВАША ЗАДАЧА`: напишите `docs/LAB14.md` с этими разделами, по порядку:

1. **Подготовка** — снимки задания 1 (версии контроллера / плагина / дашборда, три отличия Rollout от Deployment)
2. **Стратегия canary** — ваш список `steps:`, снимки из §2.5 (Paused на 25% → Healthy на 6/6 → Degraded после abort → Healthy после retry) и абзац, что дашборд показывал в окне паузы
3. **Стратегия blue-green** — два манифеста Service, блок `blueGreen:`, снимок расхождения IP эндпоинтов, доказательства promote и undo
4. **Сравнение стратегий** — когда вы берёте canary, а когда blue-green? Опирайтесь на таблицу со слайда 5 Lecture 14. Возьмите `app-python` (ваш сервис) и обоснуйте, какую стратегию вы бы взяли *в проде* для обычного релиза фичи и для миграции схемы
5. **Справка по CLI** — **пять глаголов**, которые вы запускали (`get`, `promote`, `abort`, `retry`, `undo` — плюс `version` и `dashboard`), по предложению на глагол, что он делает
6. **Трудности и выводы** — хотя бы одна настоящая (сюрприз с селектором лейблов, слишком короткий scaleDownDelay, неверный apiVersion, …) — см. типичные ловушки

---

## Бонус — авто-abort по метрике (2 балла)

**Цель:** добавьте `AnalysisTemplate`, который спрашивает Prometheus и **сам делает abort** canary, когда метрика просела — без человека.

На лекции (слайды 10–11) показали *форму* AnalysisTemplate. Вы пишете сам запрос, условие успеха, лимит провалов и частоту. Возьмите метрики, которые инструментировали в **Lab 8**. `app_requests_total{status}` — каноническая основа RED для доли ошибок.

### Бонус.1 — Напишите AnalysisTemplate

`ВАША ЗАДАЧА`: напишите `k8s/lab10-app/templates/analysistemplate.yaml`. Kind и внешняя форма `spec.metrics:` даны. **Запрос, условие успеха, лимит провалов и интервал — ваши.**

```yaml
# k8s/lab10-app/templates/analysistemplate.yaml
apiVersion: argoproj.io/v1alpha1
kind: ___                                          # YOUR TASK: which kind? (hint: slide 10)
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: ___                                # YOUR TASK: how often to sample?
                                                   #   Rule from slide 18: ≥ scrape_interval × 4
                                                   #   so if Prometheus scrapes every 15s, 60s is the floor.
      count: ___                                   # YOUR TASK: how many samples before declaring success?
                                                   #   3–5 is typical — fewer = jittery, more = slow gates.
      successCondition: ___                        # YOUR TASK: a Go expression over `result` (slide 10)
                                                   #   You're computing "fraction of non-5xx requests".
                                                   #   The query below returns a single float in [0, 1].
                                                   #   What threshold makes sense for 99.x% availability?
                                                   #   (NOT 0.95 — that's the slide-16 anti-pattern.)
      failureLimit: ___                            # YOUR TASK: how many failed samples before AUTO-ABORT?
                                                   #   0 = abort on first miss (too jittery)
                                                   #   2 = the lecture's recommendation
      provider:
        prometheus:
          address: ___                             # YOUR TASK: in-cluster Prometheus URL
                                                   #   See labs/lab14/prometheus-pointer.md for the three options.
                                                   #   Format: http://<service-name>.<namespace>:9090
          query: |
            ___                                    # YOUR TASK: PromQL for the
                                                   # "fraction of non-5xx requests" metric over a 2-minute window.
                                                   #
                                                   # Use the Lab 8 metric: app_requests_total{status="..."}
                                                   # The shape is:
                                                   #   sum(rate(<good>[2m])) / sum(rate(<all>[2m]))
                                                   # — see slide 10 for the literal template.
                                                   #
                                                   # IMPORTANT: when the canary pod has just started, no requests
                                                   # have hit it yet → the numerator and denominator are both 0
                                                   # → Prometheus returns an EMPTY VECTOR → AnalysisTemplate
                                                   # crashes on `result[0]` with NaN/index-out-of-range.
                                                   # Fix: append `or on() vector(0)` so the empty case returns 0
                                                   # rather than nothing. (See Common Pitfalls.)
                                                   #
                                                   # The `service-name` arg is yours to use:
                                                   # filter by `service="{{args.service-name}}"` or by the
                                                   # `app.kubernetes.io/name` label your `web` pods carry.
```

### Бонус.2 — Подключите его к canary

`ВАША ЗАДАЧА`: поправьте canary `steps:` из задания 2, чтобы шаг `analysis` срабатывал после первой смены веса, **и** добавьте непрерывный analysis на уровне spec, чтобы просадка посреди шага обрывала rollout сразу, ещё до следующего гейта.

```yaml
  strategy:
    canary:
      ___:                                         # YOUR TASK: which spec-level key runs analysis CONTINUOUSLY
                                                   #   (in parallel with steps), so a regression mid-step
                                                   #   aborts immediately? (hint: lecture 14 slide 11 "pro tip")
        templates:
          - templateName: ___                      # YOUR TASK: which AnalysisTemplate name from Bonus.1?
        args:
          - name: service-name
            value: ___                             # YOUR TASK: which service name does the PromQL filter on?
      steps:
        - setWeight: 25
        - ___:                                     # YOUR TASK: which step type runs analysis as a GATE
                                                   #   (i.e. blocks until success/failure)?
            templates:
              - templateName: ___                  # YOUR TASK
            args:
              - name: service-name
                value: ___                         # YOUR TASK
        # … rest of your Task 2 steps …
```

### Бонус.3 — Докажите авто-abort

`ВАША ЗАДАЧА`: выкатите нарочно сломанный образ (например обработчик `app.before_request`, который в половине случаев отдаёт `500`) и запустите rollout. Не трогая CLI, rollout должен стать `Degraded` за `interval × failureLimit + count × interval` секунд. Снимите:

- `kubectl argo rollouts get rollout lab10-app-web` — `Degraded` с `Message: RolloutAborted` после провала analysis
- `kubectl get analysisruns -o wide` — упавший `AnalysisRun` с `Phase: Failed` и проваленным `MeasurementCount`

### Бонус.4 — Дальше (нужно для полных 2 баллов) — выберите ОДНО

**Вариант A — несколько AnalysisTemplate.** Напишите второй AnalysisTemplate (например `p95-latency` через `histogram_quantile(0.95, ...)` по Histogram из Lab 8), сошлитесь на **оба** шаблона в одном шаге `analysis:` и опишите, как `failureLimit` ведёт себя, когда падает *любой* из шаблонов. (Один упавший AnalysisTemplate обрывает rollout — внутри шага провал склеивается по ИЛИ.)

**Вариант B — несколько метрик в одном AnalysisTemplate.** Добавьте вторую запись в `spec.metrics:` (например доля ошибок и p95 latency) внутри того же `AnalysisTemplate`. Argo Rollouts считает каждую отдельно; rollout обрывается, когда *любая* метрика превышает свой `failureLimit`. Опишите, чем это удобнее варианта A.

### Бонус.5 — Доказательства

**Вставьте в `docs/LAB14.md`:**

- полный `AnalysisTemplate`, который вы написали — запрос, interval, count, successCondition, failureLimit, URL провайдера
- снимки из §Бонус.3 (rollout `Degraded` после провала analysis и `AnalysisRun` с измерениями Prometheus, которые это вызвали)
- расширение A или B — второй шаблон или вторая метрика, с файлом и снятым вторым `AnalysisRun`
- 2–3 предложения, какое расширение выбрали и почему

---

## Как сдавать

```bash
git switch -c lab14
git add k8s/lab10-app/templates/rollout.yaml
git add k8s/lab10-app/templates/service-active.yaml k8s/lab10-app/templates/service-preview.yaml
git add k8s/lab10-app/values-bluegreen.yaml
git add k8s/lab10-app/templates/analysistemplate.yaml  # bonus only
git add docs/LAB14.md
git commit -m "feat(lab14): argo rollouts 1.8.4 — canary + blue-green + analysis"
git push -u origin lab14
```

Откройте **два** PR:

- `your-fork:lab14` → `course-repo:master` *(ревью)*
- `your-fork:lab14` → `your-fork:master` *(сольётся в ваш main, когда закончите)*

Чеклист PR:

```text
- [ ] Task 1 — controller + plugin + dashboard pinned to v1.8.4; research answers in LAB14.md
- [ ] Task 2 — Rollout written from scratch; canary 25→pause→50→pause→75→pause→100;
              PAUSED-at-25% capture + Healthy-at-6/6 capture + abort/retry captures
- [ ] Task 3 — two Service manifests written from scratch; blueGreen block with all four values;
              endpoints diff during cutover; promote + undo evidence
- [ ] Task 4 — LAB14.md with all six sections
- [ ] Bonus — AnalysisTemplate written from scratch; auto-abort demonstrated on a broken image;
              Option A or B extension documented
```

---

## Критерии приёмки

### Задание 1 — Основы (2 балла)
- ✅ Контроллер, плагин и дашборд **все v1.8.4** (не `latest`)
- ✅ `kubectl argo rollouts version` показывает клиент и сервер, оба `v1.8.4`
- ✅ Три отличия Rollout от Deployment записаны в LAB14.md

### Задание 2 — Canary (3 балла)
- ✅ `templates/rollout.yaml` написан с нуля — заполнены `apiVersion`, `kind` и весь список `spec.strategy.canary.steps:`
- ✅ Список шагов — `25 → pause → 50 → pause → 75 → pause → 100`. **Пауза после 25 обязана быть `pause: {}` (ручная)**
- ✅ **Главный снимок**: `kubectl argo rollouts get rollout lab10-app-web` показывает `Status: ॥ Paused`, `Message: CanaryPauseStep`, `Step: 1/6`, `SetWeight: 25, ActualWeight: 25`
- ✅ **Снимок promote**: та же команда после `promote --full` показывает `Status: ✔ Healthy`, `Step: 6/6`, `SetWeight: 100`
- ✅ Показаны abort и retry — на снимках сначала `Degraded`, потом снова `Healthy`

### Задание 3 — Blue-Green (3 балла)
- ✅ `templates/service-active.yaml` и `templates/service-preview.yaml` **написаны с нуля** — каркаса в этой лабораторной нет
- ✅ Блок `blueGreen:` заполнен всеми четырьмя значениями (`activeService`, `previewService`, `autoPromotionEnabled`, `scaleDownDelaySeconds`)
- ✅ **Снимок diff эндпоинтов**: `kubectl get endpoints lab10-app-web lab10-app-web-preview` показывает **разные IP Pod** в окне cutover
- ✅ Показаны promote и undo — `kubectl get endpoints lab10-app-web` показывает, как активный селектор перещёлкивается обратно

### Задание 4 — Документация (2 балла)
- ✅ В `docs/LAB14.md` есть все шесть разделов
- ✅ В справке по CLI названы пять глаголов (`get`, `promote`, `abort`, `retry`, `undo`), у каждого своё предложение
- ✅ Задокументирована одна настоящая трудность (не «я раньше не видел YAML»)

### Бонус — авто-abort по метрике (2 балла)
- ✅ `AnalysisTemplate` написан с нуля — запрос, условие, interval, count и failureLimit ваши
- ✅ Запрос берёт метрику Lab 8 `app_requests_total` со страховкой `or on() vector(0)`
- ✅ Авто-abort показан на нарочно сломанном образе — на снимках `Degraded` с `RolloutAborted` из-за провала analysis (без ручного `abort`)
- ✅ Сделано расширение A или B — второй шаблон или вторая метрика, со своим доказательством `AnalysisRun`

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Основы | **2** | Контроллер, плагин и дашборд v1.8.4; отличия Rollout от Deployment |
| **Задание 2** — Canary | **3** | Шаги написаны руками; главный снимок паузы на 25%; promote до Healthy; abort и retry |
| **Задание 3** — Blue-Green | **3** | Service написаны руками; блок `blueGreen:` из четырёх значений; IP эндпоинтов расходятся в окне cutover; мгновенный `undo` |
| **Задание 4** — Документация | **2** | Шесть разделов, глаголы CLI объяснены, настоящая трудность |
| **Бонус** — Авто-abort | **2** | AnalysisTemplate написан руками и PromQL из Lab 8; авто-abort на сломанном образе; расширение A или B |
| **Итого** | **12** | 10 основных + 2 бонус |

**Оценивание:**
- **10/10:** обе стратегии от начала до конца, главный снимок paused-at-25% читается однозначно, отчёт честный
- **8–9/10:** обе работают, небольшие дыры в доказательствах (например, нет diff эндпоинтов blue-green)
- **6–7/10:** canary работает, но Service для blue-green написаны не с нуля, или нет diff IP эндпоинтов
- **<6/10:** список шагов скопирован из документации без главного кадра 25→pause, или Service не написаны

---

## Материалы

<details>
<summary>📚 Документация Argo Rollouts</summary>

- [Документация Argo Rollouts](https://argoproj.github.io/argo-rollouts/) — начните с *Features*
- [Спецификация Rollout](https://argoproj.github.io/argo-rollouts/features/specification/) — каждое поле CRD
- [Стратегия canary](https://argoproj.github.io/argo-rollouts/features/canary/)
- [Стратегия blue-green](https://argoproj.github.io/argo-rollouts/features/bluegreen/)
- [Analysis и progressive delivery](https://argoproj.github.io/argo-rollouts/features/analysis/)
- [CLI `kubectl argo rollouts`](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts/) — все подкоманды
- [Релиз Argo Rollouts v1.8.4](https://github.com/argoproj/argo-rollouts/releases/tag/v1.8.4) — зафиксируйте именно этот тег

</details>

<details>
<summary>📦 Служебный код курса</summary>

- `labs/lab14/prometheus-pointer.md` — три способа дать AnalysisTemplate внутри кластера доступный URL Prometheus
- `plumbing/health/README.md` — эндпоинт `/metrics`, на который бонусный запрос тоже можно направить

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных прогонов)</summary>

- **`kubectl rollout restart deployment/lab10-app-web` на Rollout не работает.** Подкоманды core `kubectl rollout` целятся в `Deployment` / `DaemonSet` / `StatefulSet`. CRD `Rollout` принадлежит Argo Rollouts, а не rollout API Kubernetes. Берите `kubectl argo rollouts restart lab10-app-web` — тот же глагол, другой бинарник. Симптом: `error: deployments.apps "lab10-app-web" not found`, хотя `kubectl get rollouts` его показывает.

- **AnalysisTemplate падает на `result[0]` с NaN, когда трафика нет.** Canary-Pod только что стартовал, запросов ещё не было → `sum(rate(app_requests_total{...}[2m]))` возвращает **пустой вектор**, а не 0. Выражение `successCondition` `result[0] >= 0.99` тогда падает с ошибкой, и AnalysisRun показывает `Inconclusive` или `Error`, а не оценку. Фикс: допишите к PromQL `or on() vector(0)` — Prometheus вернёт `0` для пустого случая, и выражение посчитается чисто. Это самый частый провал бонуса.

- **`scaleDownDelaySeconds` легко стреляет в ногу, пока трафик ещё на preview.** Слишком большое значение (например `3600`) — каждый cutover blue-green оставляет **старый** ReplicaSet на час: жжёте CPU и путаете всех, кто видит залежавшиеся Pod. Слишком маленькое (например `30`) — к моменту, когда вы заметите баг v2 в проде, ReplicaSet v1 уже некуда откатить через `undo`. 300 секунд (5 мин) — здравый дефолт; запишите свой выбор.

- **`kubectl set image` нужен плагин, а не core kubectl.** `kubectl set image rollout/lab10-app-web web=ghcr.io/...:v2` падает с `the server doesn't have a resource type "rollout"`. Используйте `kubectl argo rollouts set image lab10-app-web web=ghcr.io/...:v2`. Ещё лучше меняйте образ через Helm / GitOps и пусть ArgoCD синхронизирует: тогда изменение лежит в Git.

- **Prometheus не в кластере, и AnalysisTemplate до него не достучится.** Prometheus из Lab 8 крутится в Docker-Compose на ноутбуке. Изнутри k3d `http://localhost:9090` резолвится в localhost **Pod**, а не хоста. Три фикса описаны в `labs/lab14/prometheus-pointer.md`: (a) `host.k3d.internal:9090` изнутри k3d (сегодня работает, в Lab 16 сломается, когда вы уйдёте дальше); (b) минимальный Prometheus в namespace `monitoring` кластера; (c) дождитесь kube-prometheus-stack из Lab 16 и до тех пор пропустите бонус.

- **Загрязнение селектора из чарта Lab 13.** Argo Rollouts вешает на лейблы Pod `rollouts-pod-template-hash`, чтобы отличить stable от canary. Если helper `selectorLabels` из Lab 10 содержит что-то кроме `name` + `instance` (например `version` — антипаттерн Lecture 15), мутация контроллера дерётся с helper, и Service начинает выбирать *оба* ReplicaSet. Держите `selectorLabels` на минимуме из двух ключей (то же правило, что у helper в Lab 10).

- **Promote ручного гейта, пока canary ещё не встал на паузу.** `kubectl argo rollouts promote lab10-app-web` сразу возвращается, если активной паузы нет: следующего шага он не ждёт. Если дёрнуть его слишком рано из скрипта, rollout проскочит момент главного снимка и уйдёт прямо на 100%. Ждите, пока `kubectl argo rollouts get` покажет `Paused / CanaryPauseStep`, и только потом promote.

- **`autoPromotionEnabled: true` в blue-green молча пропускает ручной гейт.** По умолчанию `false`, но куча туториалов ставит `true` для демо в CI. С `true` новый ReplicaSet уходит в promote, как только станет Ready — окно проверки через preview Service длится **ноль секунд**. В values лабораторной всегда ставьте `false`. `true` — только в полностью автоматическом pipeline (цепочка автоматической сборки и выкладки) с prePromotionAnalysis.

- **`apiVersion: apps/v1` в Rollout.** Срабатывает привычка с `Deployment`: Rollout живёт в `argoproj.io/v1alpha1`, а не в `apps/v1`. `kubectl apply` отвечает `no matches for kind "Rollout" in version "apps/v1"`, и файл молча не применяется, если вы гоняете `helm template | kubectl apply -f -`. Всегда перепроверяйте первые две строки.

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- `alias kr='kubectl argo rollouts'` — экономит 18 символов на каждой команде
- [`argo-rollouts/dashboard`](http://localhost:3100) — локальный UI, самый чистый способ снять скриншоты заданий 2 и 3
- [`kubectl get analysisruns -A`](https://argoproj.github.io/argo-rollouts/features/analysis/) — каждый прогон analysis, который контроллер когда-либо породил, включая упавшие, на которых rollout сделал abort

</details>

---

## Что дальше

| Lab | Что добавляет к этому стеку |
|---:|---|
| 15 | StatefulSet — stateful-нагрузка в модель Rollout не встаёт; вместо этого упорядоченное имя Pod и PVC на каждый Pod |
| 16 | **kube-prometheus-stack** и CRD `ServiceMonitor` — вместо URL Prometheus, который вы собрали руками, нормальный стек мониторинга в кластере; `AnalysisTemplate` из этой лабораторной продолжает работать, только с `address: http://prometheus-operated.monitoring:9090` |

---

**Удачи!** 🚦

> **Помните:** `Rollout` — это `Deployment` плюс стратегия. Без метрических гейтов вы только замедлили плохой деплой; с ними откат автоматический. Заголовок этой лабораторной — `SetWeight: 25, ActualWeight: 25, Message: CanaryPauseStep` — момент, когда «задеплоено» уже значит «выпущено, *если canary пережил гейт*».
