# Lab 10 — Пакетный менеджер Helm

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Helm-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Helm%204-informational)

> **Цель:** превратите манифесты Kubernetes из Lab 9 в написанный вручную Helm-чарт на Helm 4, который едет через dev / staging / prod с отдельным файлом values на окружение, запускает lifecycle hook и (бонус) публикуется в GHCR по OCI.
> **Что сдать:** PR из `lab10`, который добавляет `k8s/lab10-app/` (чарт) и `k8s/lab10-app/HELM.md` (ваш отчёт).

---

## Обзор

В Lab 9 вы написали четыре сырых манифеста (Deployment и Service для web, Deployment и Service для echo). Для одного окружения этого хватает. Для трёх вы копируете YAML и надеетесь, что drift не всплывёт.

Эта лабораторная вытаскивает каждое зашитое значение из тех манифестов в **чарт** с **отдельным файлом values на окружение**. Оценивается навык **писать шаблоны**: `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`, `templates/service.yaml`, `templates/_helpers.tpl` и один hook — с нуля. Не схема «`helm create` и подправить». С нуля.

> ⚠️ **Границы:** без субчартов, без library chart, без `values.schema.json`. Только анатомия чарта, иерархия values, один hook и (бонус) push в OCI.

**Что потренируете:**
- Анатомию чарта Helm 4: `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`
- Шаблоны Go и Sprig (`include`, `nindent`, `default`, `toYaml`, `printf`, `trunc`, `quote`)
- Жизненный цикл релиза: `install / upgrade / rollback / list / history / uninstall`
- Выкладку на несколько окружений через иерархию values и наслоение **`-f a -f b`**
- Lifecycle hook (`pre-install` / `pre-upgrade`) с весами и политиками удаления
- Публикацию чартов как OCI-артефактов в GHCR

> 📚 Идёт в паре с **Lecture 10 — Управление пакетами Helm**. Перед стартом перечитайте слайды 5–11 (анатомия чарта, шаблоны, `_helpers.tpl`, жизненный цикл, несколько окружений, hook).

---

## Состояние проекта

**К этому моменту у вас должно быть:**
- Кластер k3d на Kubernetes 1.36 (Lab 9 — `k3d cluster create devops ...`)
- Ваш **образ web из Lab 2** в публичном registry (Docker Hub / GHCR)
- Манифесты Lab 9 в `k8s/`: `web-deployment.yaml`, `web-service.yaml`, `echo-deployment.yaml`, `echo-service.yaml`

**Эта лабораторная добавляет:**
- `k8s/lab10-app/` — написанный вручную чарт Helm 4, который **заменяет** те четыре манифеста
- `k8s/lab10-app/values-prod.yaml` (и по желанию `-dev`, `-staging`) — дельты на окружение
- `k8s/lab10-app/templates/hooks/db-migrate.yaml` — один hook на pre-install/pre-upgrade
- `k8s/lab10-app/HELM.md` — ваш отчёт к сдаче

К Lab 13 ArgoCD выкатит этот чарт через GitOps. К Lab 14 Argo Rollouts заменит шаблон Deployment, который вы напишете здесь. Гигиена шаблонов, которую вы соберёте на этой неделе, переживёт Lab 10.

---

## Подготовка

```bash
helm version            # must report v4.1.x (course pins to 4.1.4)
kubectl get nodes       # your Lab 9 cluster must be up
```

> **Helm 3 и Helm 4.** Helm 3 только на поддержке (багфиксы до 8 июля 2026, фиксы безопасности до 11 ноября 2026). На курсе стандарт — **Helm 4**. Заголовок `apiVersion: v2` в `Chart.yaml` на Helm 4 по-прежнему верный: это версия формата чарта для Helm 3 и новее, **а не** версия бинарника Helm. `apiVersion: v1` — только Helm 2.

Инструкция по установке: <https://helm.sh/docs/intro/install/>.

Каждый файл в `k8s/lab10-app/` вы напишете сами. **Не запускайте `helm create`** (см. типичные ловушки).

---

## Задание 1 — Основы Helm и установка (1 балл)

### 1.1 — Поставьте Helm 4 и посмотрите настоящий чарт

`ВАША ЗАДАЧА`: поставьте Helm 4.1.x (или обновитесь до него), затем снимите метаданные и значения по умолчанию с публичного чарта на OCI, **не устанавливая его**:

```bash
helm show chart  oci://registry-1.docker.io/bitnamicharts/nginx
helm show values oci://registry-1.docker.io/bitnamicharts/nginx | head -40
```

Посмотрите на форму: `apiVersion`, `name`, `version`, `appVersion`, `dependencies`. В `values.yaml` заметьте, как наружу выведен каждый числовой и строковый параметр.

### 1.2 — Объясните три понятия своими словами

В `HELM.md` напишите по 2–3 предложения на **Chart**, **Release**, **Values**. Не копируйте документацию: на устном вы должны ответить сами. Подсказки:

- *Chart* — пакет на диске; *Release* — один запущенный экземпляр этого пакета; *Values* заполняют шаблон.
- Helm 4 хранит состояние релиза в Secret `helm.sh/release.v1` в namespace релиза (Tiller давно нет, это был Helm 2).

### 1.3 — Доказательство работы

Вставьте в `HELM.md`:

- Вывод `helm version` с v4.1.x
- Первые ~20 строк одного из вызовов `helm show` выше
- Ваши объяснения Chart, Release и Values по 2–3 предложения

---

## Задание 2 — Соберите чарт (3 балла)

**Цель:** напишите чарт с нуля. Каркас ниже — **форма каталога**, которую нужно получить. **Не** запускайте `helm create`: уедете с бойлерплейтом, который ссылается на `serviceaccount.yaml`, которого вы не писали, и сломаете `helm lint` в ту секунду, когда поправите `values.yaml` (см. типичные ловушки).

```
k8s/lab10-app/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml                # Task 3
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml             # web
    ├── service.yaml                # web
    ├── echo-deployment.yaml        # echo (Task 2.6)
    ├── echo-service.yaml           # echo (Task 2.6)
    ├── hooks/
    │   └── db-migrate.yaml         # Task 4
    └── NOTES.txt                   # optional; printed after install
```

### 2.1 — `Chart.yaml`

`ВАША ЗАДАЧА`: заполните метаданные. Важны и `version` (SemVer чарта: поднимайте при изменениях чарта), и `appVersion` (строка версии на стороне образа: поднимайте при смене образа). Они живут независимо. `kubeVersion` заставляет Helm отказаться ставить чарт на более старый кластер.

```yaml
apiVersion: ___                  # YOUR TASK: chart-format version (Helm 3/4 expects this value)
name: ___                        # YOUR TASK: chart name (must match the directory name)
description: ___                 # YOUR TASK: one sentence; shows in `helm show chart`
type: application                # 'application' (default) or 'library' — leave as-is for this lab
version: ___                     # YOUR TASK: SemVer for the chart, e.g. 0.1.0
appVersion: "___"                # YOUR TASK: quoted string, image-side version (your Lab 2 tag)
kubeVersion: "___"               # YOUR TASK: a range, e.g. ">=1.33.0" — refuses older clusters
# dependencies:                  # OPTIONAL — not used in this lab
#   - name: ...
#     version: ...
#     repository: ...
```

### 2.2 — `values.yaml`

`ВАША ЗАДАЧА`: перечислите каждое значение, на которое сошлются шаблоны, с разумным умолчанием. **Каждое зашитое число и строка из манифестов Lab 9 становится значением здесь.** И у `web`, и у `echo` свой блок.

Минимальная поверхность (можно добавить больше):

```yaml
# YOUR TASK: fill every default. The keys below are the contract between
# values.yaml and your templates — your templates will fail to render if you
# rename a key here without updating the template, and vice versa.

replicaCount: ___                # default replicas for `web`

image:
  repository: ___                # your Lab 2 image (GHCR or Docker Hub), e.g. ghcr.io/<you>/devops-info-service
  tag: ""                        # empty → templates fall back to .Chart.AppVersion
  pullPolicy: ___                # IfNotPresent | Always | Never (Lab 9's pitfalls call out the k3d-import case)

service:
  type: ___                      # NodePort for local k3d; LoadBalancer in prod
  port: ___                      # the Service port (clients hit this)
  # the container's listen port lives under `containerPort` below

containerPort: ___               # what your Lab 2 app listens on (Lab 9 set this — match it)

resources:
  requests:
    cpu: ___
    memory: ___
  limits:
    cpu: ___
    memory: ___

probes:
  liveness:
    path: ___                    # /health from Lab 1
    initialDelaySeconds: ___
    periodSeconds: ___
  readiness:
    path: ___
    periodSeconds: ___

ingress:
  enabled: false                 # not used in this lab; reserved for Lab 16

echo:
  replicaCount: ___              # default replicas for the echo sidecar service (Lab 9 had a specific count)
  image:
    repository: ghcr.io/inno-devops-labs/echo
    tag: v1
  service:
    type: ClusterIP
    port: 80
    targetPort: 8081
  resources:                     # echo is a tiny Go binary — size it smaller than web (Lab 9 §3.1)
    requests:
      cpu: ___
      memory: ___
    limits:
      cpu: ___
      memory: ___

# Hook config (Task 4)
hooks:
  dbMigrate:
    enabled: true
    image: busybox:1.37          # any small image — the hook is a fake migration
```

### 2.3 — `templates/_helpers.tpl`

Именованные шаблоны избавляют от копипасты блоков меток по каждому манифесту. Вы определите **четыре**: `name`, `fullname`, `labels`, `selectorLabels`. Это мышечная память Helm: такие блоки вы будете писать в каждом чарте дальше по карьере.

`ВАША ЗАДАЧА`: реализуйте тела. Форма функций дана. Заполните тело каждого блока `define` так, чтобы:

| Шаблон | Что должен вернуть |
|---|---|
| `lab10-app.name` | Имя чарта (или `nameOverride`, если пользователь его задал), обрезанное до 63 символов (лимит метки K8s), без хвостового `-`. |
| `lab10-app.fullname` | Уникальное на релиз имя: `<release-name>-<chart-name>` (или `nameOverride`, если задан), обрезка до 63, хвостовой `-` срезан. Именно его обычно ставят в metadata.name. |
| `lab10-app.labels` | Полный набор меток: `helm.sh/chart`, все метки селектора (включите `selectorLabels`), `app.kubernetes.io/version`, `app.kubernetes.io/managed-by`. |
| `lab10-app.selectorLabels` | **Строгое подмножество** `labels`: только `app.kubernetes.io/name` и `app.kubernetes.io/instance`. Селекторы после создания не меняются. Никогда не кладите сюда `version` (см. типичные ловушки). |

```handlebars
{{/* YOUR TASK: return the chart name, truncated to 63 chars, no trailing dash */}}
{{- define "lab10-app.name" -}}
{{- /* your body — hint: `default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-"` is the canonical form */ -}}
{{- end }}

{{/* YOUR TASK: return a unique-per-release name, truncated to 63, no trailing dash.
     IMPORTANT: when the release name already contains the chart name (e.g. you ran
     `helm install lab10-app k8s/lab10-app`), the canonical helper collapses to JUST
     the release name — otherwise you'd get `lab10-app-lab10-app`. Lab 11 references
     `deploy/lab10-app-web`, so your fullname helper MUST handle the collapse. The
     `helm create` boilerplate has the canonical form; cribbing the pattern (not the
     whole file) is fair game.
     Shape: `if contains $name .Release.Name → .Release.Name`, else
            `printf "%s-%s" .Release.Name $name`, then `| trunc 63 | trimSuffix "-"` */}}
{{- define "lab10-app.fullname" -}}
{{- /* your body */ -}}
{{- end }}

{{/* YOUR TASK: return the full label set (5 labels including the helm.sh/chart line) */}}
{{- define "lab10-app.labels" -}}
{{- /* your body — must include `lab10-app.selectorLabels`; emit `helm.sh/chart`,
       `app.kubernetes.io/version`, `app.kubernetes.io/managed-by` */ -}}
{{- end }}

{{/* YOUR TASK: return ONLY the two selector labels (name + instance). No version. */}}
{{- define "lab10-app.selectorLabels" -}}
{{- /* your body */ -}}
{{- end }}
```

Функции, которые понадобятся (все из Sprig): `default`, `trunc`, `trimSuffix`, `printf`, `replace`, `quote`. См. [документацию Sprig](https://masterminds.github.io/sprig/) и [руководство по шаблонам чарта](https://helm.sh/docs/chart_template_guide/).

### 2.4 — `templates/deployment.yaml` (web)

`ВАША ЗАДАЧА`: зашаблонизируйте Deployment web из Lab 9. Форма YAML дана, чтобы не изобретать велосипед. Каждый `{{ ___ }}` — токен, который нужно заполнить (многим понадобится конвейер вроде `| nindent 4` или `| default ...`).

> 💡 **Именуйте web и echo по-разному.** Оба Deployment в одном релизе. Если `metadata.name` сойдётся в одно значение, второй `helm install` упадёт с конфликтом. Соглашение: дописывайте суффикс компонента к fullname (здесь, например, `{{ include "lab10-app.fullname" . }}-web`, а `-echo` — на манифестах echo в 2.6). Lab 11 будет ссылаться на `deploy/lab10-app-web` напрямую. Суффикс сохраните.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ ___ }}-web                            # YOUR TASK: `include` your fullname template (suffix already present)
  labels:
    {{- ___ | nindent 4 }}                       # YOUR TASK: `include` your labels template
spec:
  replicas: {{ ___ }}                            # YOUR TASK: from .Values.replicaCount (with a sane default)
  selector:
    matchLabels:
      {{- ___ | nindent 6 }}                     # YOUR TASK: `include` your SELECTOR labels template
  template:
    metadata:
      labels:
        {{- ___ | nindent 8 }}                   # YOUR TASK: `include` your SELECTOR labels template
    spec:
      containers:
        - name: web
          image: "{{ ___ }}:{{ ___ | default .Chart.AppVersion }}"   # YOUR TASK: repo + tag from values
          imagePullPolicy: {{ ___ }}             # YOUR TASK: from .Values.image.pullPolicy
          ports:
            - name: http
              containerPort: {{ ___ }}           # YOUR TASK: from .Values.containerPort
              protocol: TCP
          livenessProbe:
            httpGet:
              path: {{ ___ }}                    # YOUR TASK: from values
              port: http
            initialDelaySeconds: {{ ___ }}
            periodSeconds: {{ ___ }}
          readinessProbe:
            httpGet:
              path: {{ ___ }}
              port: http
            periodSeconds: {{ ___ }}
          {{- with .Values.resources }}
          resources:
            {{- ___ | nindent 12 }}              # YOUR TASK: render the whole sub-tree as YAML (one Sprig fn)
          {{- end }}
```

> **`indent` и `nindent`.** `nindent N` = перевод строки плюс отступ на N пробелов. `indent N` = только отступ. Когда действие `{{- ... -}}` стоит на своей строке и съело предыдущий перевод строки, `indent` даёт невалидный YAML. В 9 случаях из 10 берите `nindent`.

### 2.5 — `templates/service.yaml` (web)

`ВАША ЗАДАЧА`: то же упражнение, сильно короче.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ ___ }}-web                            # YOUR TASK: fullname template (`-web` suffix mirrors the Deployment)
  labels:
    {{- ___ | nindent 4 }}                       # YOUR TASK: labels template
spec:
  type: {{ ___ }}                                # YOUR TASK: from .Values.service.type
  ports:
    - port: {{ ___ }}                            # YOUR TASK: from .Values.service.port
      targetPort: http                           # named ports beat numbers; this points at the deployment's `name: http`
      protocol: TCP
      name: http
  selector:
    {{- ___ | nindent 4 }}                       # YOUR TASK: SELECTOR labels template (NOT the full labels)
```

### 2.6 — `templates/echo-deployment.yaml` и `templates/echo-service.yaml`

`ВАША ЗАДАЧА`: тот же приём, но для sidecar echo. Два заметных отличия:

1. Образ берётся из `.Values.echo.image.repository` и `.Values.echo.image.tag` (без `default .Chart.AppVersion`: образ echo не зависит от `appVersion` вашего приложения).
2. Поставьте другой различитель метки (например `app.kubernetes.io/component: echo`) на шаблон Pod'а и на селектор Service, чтобы Service **web** случайно не выбрал Pod'ы echo. Можно:
   - расширить хелпер `selectorLabels` необязательным компонентом (передать `.Values.echo` в `include`) **или**
   - вручную дописать маленький дополнительный ключ селектора только в этом манифесте.

Остановитесь и выберите. Оба варианта годные.

### 2.7 — Проверьте, отрендерьте, установите

Идите в таком порядке: `helm lint <chart>`, `helm template <release> <chart>` (посмотрите на отрендеренный YAML), `helm install <release> <chart> --dry-run --debug`, затем настоящий `helm install --create-namespace` в свежий namespace. Закончите `helm list` (ждите `REVISION 1`) и `kubectl get deploy,svc,pods`, чтобы убедиться, что оба Pod'а Ready.

```bash
# Release name MUST be `lab10-app` (matches chart name). Lab 11 references `deploy/lab10-app-web`;
# if your fullname helper has the canonical collapse-when-contained branch (see 2.3), this works.
# If you used a too-simple fullname (just `printf "%s-%s" .Release.Name .Chart.Name`), you'll get
# `lab10-app-lab10-app-web` and Lab 11 won't find your Deployment. Run `helm template` first to verify.
helm install lab10-app k8s/lab10-app -n helm-demo --create-namespace
# expected: helm list shows STATUS deployed, REVISION 1
# expected (verify with `kubectl get deploy -n helm-demo`): deploy/lab10-app-web and deploy/lab10-app-echo
```

### 2.8 — Доказательство работы

Вставьте в `HELM.md`:

- Вывод `helm lint k8s/lab10-app` (должно быть `0 chart(s) failed`)
- Первые ~30 строк `helm template lab10-app k8s/lab10-app`: ваши values текут в отрендеренный Deployment (образ, реплики, probe, resources)
- `helm list -n helm-demo` с **REVISION 1** у релиза `lab10-app`
- `kubectl get deploy,svc,pods -n helm-demo`: Pod'ы и web, и echo в Ready (имена deploy: `lab10-app-web`, `lab10-app-echo`)

---

## Задание 3 — Values для нескольких окружений (3 балла)

**Цель:** ведите ≥ 2 окружения из одного чарта через иерархию базовых values и перекрытий.

### 3.1 — Создайте `values-prod.yaml` (и по желанию `values-dev.yaml` / `values-staging.yaml`)

`ВАША ЗАДАЧА`: напишите файл перекрытия **только из дельт** — лишь ключи, которые отличаются от `values.yaml`. Предлагаемая форма:

| | dev (умолчания в `values.yaml`) | prod (`values-prod.yaml`) |
|---|---|---|
| `replicaCount` | 1 | 4 |
| `image.tag` | `""` (→ `appVersion`) | зафиксированный SemVer, например `v1.2.0` |
| `service.type` | `NodePort` | `LoadBalancer` |
| `resources.requests.cpu` | `100m` | `500m` |
| `resources.limits.memory` | `256Mi` | `1Gi` |

Для ≥ 3 окружений добавьте между ними `values-staging.yaml`.

### 3.2 — Обновите существующий релиз слоями values

Это **главное доказательство** Задания 3: тот же чарт, тот же релиз, переход с REVISION 1 на REVISION 2, потому что `-f` наложил prod-перекрытие поверх базы.

```bash
helm upgrade lab10-app k8s/lab10-app -n helm-demo \
  -f k8s/lab10-app/values.yaml \
  -f k8s/lab10-app/values-prod.yaml
# expected: STATUS deployed, REVISION 2
# expected: deploy/lab10-app-web spec.replicas = your prod override (was 1 at REV 1)
```

Затем выполните `helm list`, `kubectl get deploy lab10-app-web -n helm-demo -o jsonpath='{.spec.replicas}'` и `helm history` и снимите доказательства.

> **Приоритет values (от низшего к высшему):**
> 1. `values.yaml` (умолчание чарта, **грузится всегда** — передавать явно не обязательно)
> 2. Каждый `-f file.yaml` по порядку — поздние файлы перекрывают ранние
> 3. `--set key=value` в CLI
>
> Поэтому `-f values.yaml -f values-prod.yaml --set replicaCount=10` заканчивается 10 репликами.

### 3.3 — Отрендерьте diff окружений без кластера

`ВАША ЗАДАЧА`: прогоните `helm template` по чарту **без** `-f values-prod.yaml` и **с** ним, затем сделайте `diff` двух выводов. Видимая дельта доказывает, что файл перекрытия что-то сделал. Кластер не нужен. Первые ~20 строк diff положите в `HELM.md`.

### 3.4 — Откат

`ВАША ЗАДАЧА`: выполните `helm rollback lab10-app 1 -n helm-demo`, затем снова спросите число реплик у Deployment. Оно должно вернуться к умолчанию dev. Снимите `helm history`: REVISION 3 с описанием отката.

### 3.5 — Доказательство работы

Вставьте в `HELM.md`:

- Содержимое `values-prod.yaml` (только дельты — файл должен быть коротким)
- `helm list -n helm-demo` **на REVISION 2**, снятый после upgrade
- `kubectl get deploy lab10-app-web -n helm-demo -o jsonpath='{.spec.replicas}'` возвращает prod-значение
- `helm history lab10-app -n helm-demo`: REVISION 1 вытеснена REVISION 2
- Первые 20 строк `diff` из 3.3 — видимая дельта между отрендеренными окружениями

---

## Задание 4 — Lifecycle hook (2 балла)

**Цель:** добавьте **один** Job-hook, который запускается и на `pre-install`, и на `pre-upgrade` (например, фейковая миграция БД). Порядок задаёт `hook-weight`, уборку — `hook-delete-policy`.

### 4.1 — Напишите `templates/hooks/db-migrate.yaml`

Четыре ключа аннотаций — весь контракт hook. Остальное — обычный Job `batch/v1`. `ВАША ЗАДАЧА`: напишите файл. Каркас:

```yaml
{{- if .Values.hooks.dbMigrate.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ ___ }}-db-migrate                  # YOUR TASK: fullname template + "-db-migrate"
  labels:
    {{- ___ | nindent 4 }}                    # YOUR TASK: labels template
  annotations:
    "helm.sh/hook": ___                       # YOUR TASK: TWO comma-separated points — pre-install,pre-upgrade
    "helm.sh/hook-weight": "___"              # YOUR TASK: a string, NOT a number (Helm quirk). Negative runs earlier.
    "helm.sh/hook-delete-policy": ___         # YOUR TASK: comma-separated policies — before-hook-creation,hook-succeeded
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
          image: {{ ___ }}                    # YOUR TASK: from .Values.hooks.dbMigrate.image
          command: ["sh", "-c"]
          args:
            - |
              echo "running pre-install/pre-upgrade migration for release {{ .Release.Name }}";
              sleep 5;
              echo "migration complete";
{{- end }}
```

**Четыре ключа аннотаций**, которые нужно выставить (и что они делают):

| Аннотация | Какое значение здесь | Что делает |
|---|---|---|
| `helm.sh/hook` | `pre-install,pre-upgrade` | Когда запускать. Несколько точек — через запятую. |
| `helm.sh/hook-weight` | Строка в кавычках, например `"-5"` | Порядок hook в одной точке. Меньше — раньше. При равенстве порядок алфавитный. **Обязательно строка в кавычках**: Helm читает её как строку и конвертирует внутри. |
| `helm.sh/hook-delete-policy` | `before-hook-creation,hook-succeeded` | Когда удалять ресурс hook. Без этого упавшие Job копятся вечно: `helm uninstall` ресурсы hook НЕ удаляет. |
| (нет — ресурс и есть сам Job) | — | Hook и есть обычный Job, просто он не входит в обычный релиз. |

### 4.2 — Проверьте, что hook срабатывает и на install, и на upgrade

`ВАША ЗАДАЧА`: снимите релиз, чтобы начать с чистого листа, и поставьте заново. Во время install выполните `kubectl get jobs -n helm-demo` (Job hook должен появиться) и `kubectl logs job/<hook-job>`, чтобы снять ваш echo `running pre-install migration`. Затем `helm upgrade` с `-f values-prod.yaml` и убедитесь, что **тот же hook срабатывает снова** на `pre-upgrade`. В конце снова спросите jobs: политика `before-hook-creation,hook-succeeded` должна была удалить предыдущий.

### 4.3 — Доказательство работы

Вставьте в `HELM.md`:

- Блок аннотаций вашего `db-migrate.yaml` (контракт из четырёх ключей)
- `kubectl get jobs -n helm-demo`, снятый **во время** install (Job на месте)
- `kubectl logs` Job hook с вашим echo `running pre-install migration`
- `kubectl get jobs -n helm-demo`, снятый **после** срабатывания политики (Job исчез)

---

## Задание 5 — Документация (1 балл)

`ВАША ЗАДАЧА`: создайте `k8s/lab10-app/HELM.md` и покройте, в этом порядке:

1. **Обзор чарта** — дерево каталогов, что рендерит каждый шаблон, контракт `values.yaml`
2. **Конфигурация** — таблица значений верхнего уровня и чем `values-prod.yaml` отличается от умолчаний
3. **Хелперы** — ваши четыре именованных шаблона и одно предложение, почему `selectorLabels` — подмножество `labels`
4. **Hook** — четыре аннотации на Job `db-migrate` и что делает каждая
5. **Операции** — точные команды `install`, `upgrade -f a -f b`, `history`, `rollback`, `uninstall`, которые вы запускали
6. **Доказательства** — все выводы из Tasks 1.3 / 2.8 / 3.5 / 4.3
7. **Трудности и выводы** — минимум одна настоящая (баг `indent`/`nindent`, неизменяемый селектор, `hook-weight` без кавычек и т.д.)

---

## Бонусное задание — Публикация в GHCR через OCI (2 балла)

**Цель:** упакуйте чарт, запушьте его в GHCR как OCI-артефакт, затем **установите его из registry**, чтобы доказать путь туда и обратно.

`ВАША ЗАДАЧА`: здесь вас меньше ведут за руку.

1. `helm package` вашего чарта в `lab10-app-<version>.tgz`.
2. Создайте **classic** GitHub PAT (Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token) со scope **`write:packages`**. Fine-grained токены OCI-push пока не умеют (май 2026). В CI вы бы взяли автоподставленный `GITHUB_TOKEN`. Локально берите PAT. **Никогда не вставляйте PAT в закоммиченный файл или в CLI.**
3. `echo $CR_PAT | helm registry login ghcr.io -u <your-username> --password-stdin` — PAT приходит через stdin и не попадает в историю шелла.
4. `helm push <tgz> oci://ghcr.io/<your-username>/charts`.
5. Сделайте пакет GHCR **публичным** (Your profile → Packages → `charts/lab10-app` → Package settings → Change visibility → Public), чтобы проверяющий мог стянуть его без кредов.
6. В свежем namespace: `helm install … oci://ghcr.io/<your-username>/charts/lab10-app --version <ver>`. Докажите, что установка пришла **из registry**, а не из локального каталога (`STATUS` покажет источник OCI).

Команды, которые понадобятся (найдите их сами: `helm package`, `helm registry login`, `helm push`, `helm install oci://...`).

Вставьте в `HELM.md`:

- Вывод `helm package`
- Вывод `helm push` **с digest**
- URL пакета GHCR (например `https://github.com/users/<you>/packages/container/package/charts%2Flab10-app`)
- Вывод `helm install oci://…` и `helm list`, где виден релиз из OCI

---

## Как сдавать

Тот же поток, что в Lab 9: ветка `lab10`, добавьте `k8s/lab10-app/`, коммит, push, откройте **два** PR.

```bash
git switch -c lab10
git add k8s/lab10-app/
git commit -m "feat(lab10): package web + echo as a helm 4 chart"
git push -u origin lab10
```

PR:

- `your-fork:lab10` → `course-repo:master` *(на ревью)*
- `your-fork:lab10` → `your-fork:master` *(сольётся в ваш собственный main, когда закончите)*

Чеклист PR:

```text
- [ ] Task 1 done — helm v4.1.x, public chart inspected, concepts explained
- [ ] Task 2 done — Chart.yaml, values.yaml, _helpers.tpl (4 templates), deployment + service + echo manifests, lint clean, REVISION 1 installed
- [ ] Task 3 done — values-prod.yaml + `helm upgrade -f base -f override` → REVISION 2 with replica count change
- [ ] Task 4 done — pre-install/pre-upgrade hook with the four annotations, hook fires + cleans up
- [ ] Task 5 done — HELM.md with all seven sections + evidence
- [ ] Bonus done — chart pushed to GHCR via OCI and installed from the registry
```

---

## Критерии приёмки

### Задание 1 (1 балл)
- ✅ `helm version` показывает v4.1.x
- ✅ Публичный OCI-чарт осмотрен через `helm show chart` / `helm show values`
- ✅ Chart / Release / Values объяснены своими словами

### Задание 2 (3 балла)
- ✅ В `Chart.yaml` верные `apiVersion`, SemVer `version` и `appVersion`
- ✅ В `values.yaml` наружу выведено каждое зашитое значение из Lab 9 (реплики, образ, порты, resources, probe, блок echo)
- ✅ `_helpers.tpl` определяет и использует **четыре** именованных шаблона: `name`, `fullname`, `labels`, `selectorLabels`
- ✅ `selectorLabels` — строгое подмножество `labels` (без `version`)
- ✅ Все четыре шаблона манифестов рендерятся через `include`, `nindent`, `default`, `with`
- ✅ Liveness и readiness probe сохранены и настраиваются через values
- ✅ `helm lint` чистый, `helm install` доходит до REVISION 1

### Задание 3 (3 балла)
- ✅ `values-prod.yaml` (и по желанию dev/staging) содержит **только дельты** относительно `values.yaml`
- ✅ `helm upgrade ... -f values.yaml -f values-prod.yaml` даёт REVISION 2
- ✅ Настоящее значение (число реплик) заметно меняется между ревизиями
- ✅ `helm rollback` возвращает релиз на REVISION 1

### Задание 4 (2 балла)
- ✅ Один Job-hook с `helm.sh/hook: pre-install,pre-upgrade`
- ✅ Задан `helm.sh/hook-weight` (строка в кавычках)
- ✅ Задан `helm.sh/hook-delete-policy`, чтобы Job убирался
- ✅ Hook срабатывает и на install, и на upgrade; логи сняты

### Задание 5 (1 балл)
- ✅ `HELM.md` покрывает все семь обязательных разделов, выводы вставлены

### Бонус (2 балла)
- ✅ Чарт упакован в `.tgz`
- ✅ Запушен в `oci://ghcr.io/<username>/charts`
- ✅ Установлен *из registry* (не из локального каталога)
- ✅ URL GHCR и вывод установки из OCI лежат в `HELM.md`

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Основы | **1** | v4.1.x, публичный чарт осмотрен, понятия объяснены |
| **Задание 2** — Сборка чарта | **3** | чистые шаблоны, четыре хелпера, lint чистый, install доходит до REVISION 1 |
| **Задание 3** — Несколько окружений | **3** | наслоение `-f a -f b` даёт REVISION 2 с видимым изменением значения; откат работает |
| **Задание 4** — Hook | **2** | hook pre-install/pre-upgrade с весом и политикой удаления; срабатывает и убирается |
| **Задание 5** — Документация | **1** | полный `HELM.md` с доказательствами |
| **Бонус** — push в OCI | **2** | чарт упакован, запушен в GHCR, установлен *из registry* |
| **Итого** | **12** | 10 основных + 2 бонус |

**Оценивание:**
- **10/10:** чистые шаблоны с нуля, upgrade на несколько окружений виден между ревизиями, hook срабатывает и убирается, документация подробная
- **8–9/10:** чарт ставится и обновляется, мелкие дыры в хелперах или аннотациях hook
- **6–7/10:** базовый чарт ставится, но нет upgrade на несколько окружений или hook не доведён
- **<6/10:** сдан бойлерплейт `helm create`, зашитые значения, нет hook или чарт не проходит lint

---

## Материалы

<details>
<summary>📚 Документация</summary>

- [Документация Helm](https://helm.sh/docs/) — Helm 4
- [Руководство по шаблонам чарта](https://helm.sh/docs/chart_template_guide/)
- [Лучшие практики чартов](https://helm.sh/docs/chart_best_practices/)
- [Hook чартов](https://helm.sh/docs/topics/charts_hooks/)
- [Реестры на OCI](https://helm.sh/docs/topics/registries/)
- [Справочник функций Sprig](https://masterminds.github.io/sprig/) — библиотека функций, в которую вы будете заглядывать каждую неделю
- [Встроенные объекты](https://helm.sh/docs/chart_template_guide/builtin_objects/) — `.Release`, `.Chart`, `.Values`, `.Files`, `.Capabilities`

</details>

<details>
<summary>🛠️ Инструменты и реестры</summary>

- [Artifact Hub](https://artifacthub.io/) — публичный поиск чартов (больше 15 000)
- [helm-docs](https://github.com/norwoodj/helm-docs) — автогенерация README из values
- [chart-testing](https://github.com/helm/chart-testing) — lint и установка чартов в CI
- [GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) — OCI-реестр и для образов, и для чартов

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных пробных прогонов)</summary>

- **`helm create` затирает вашу работу.** Запуск внутри уже существующего каталога чарта без спроса переписывает написанные вручную `Chart.yaml`, `values.yaml` и хелперы. Не запускайте. Пишите файлы сами: эта лабораторная оценивает именно этот навык.
- **У хелпера fullname нет ветки схлопывания.** Наивный `printf "%s-%s" .Release.Name .Chart.Name` даёт `lab10-app-lab10-app`, когда имя релиза и имя чарта совпадают (а они совпадают: релиз — `lab10-app`). Дальше `-web` дописывается, и получается `lab10-app-lab10-app-web`. Lab 11 ссылается на `deploy/lab10-app-web`, так что следующая лабораторная молча ломается. Каноническая ветка `if contains $name .Release.Name` (списанная с `helm create`) как раз делает схлопывание. Проверьте `helm template lab10-app k8s/lab10-app | grep '^  name:'` ДО установки.
- **Замена всего `values.yaml` от `helm create` ломает `helm lint`.** Сгенерированный бойлерплейт везёт `templates/serviceaccount.yaml`, который ссылается на `.Values.serviceAccount.create`. Стоит выкинуть этот блок из `values.yaml` — `helm lint` падает, потому что ключ исчез. Либо правьте умолчания на месте, **либо** (так и надо) не запускайте `helm create` вообще. *(Ровно та яма, которую эталонный прогон этой лабораторной выкопал себе сам.)*
- **`indent` и `nindent`.** `nindent N` = перевод строки плюс N пробелов; `indent N` = только N пробелов. Если действие `{{- ... -}}` уже съело предыдущий перевод строки, `indent` даёт невалидный YAML: `helm lint` это поймает, `helm install` откажется. Берите `nindent`, пока не уверены.
- **Изменяемые метки селектора ломают rolling update.** Когда `Deployment` уже есть, K8s отвергает правки `spec.selector.matchLabels`. Если в метках селектора есть `version`, каждый подъём `appVersion` запускает upgrade чарта, который K8s не примет. Придётся `kubectl delete deploy ...`, и нулевой простой потеряется. Держите в `selectorLabels` только `name` и `instance`.
- **`hook-weight` обязан быть строкой в кавычках.** `"helm.sh/hook-weight": -5` (без кавычек) молча парсится как число, и Helm его игнорирует: порядок hook станет недетерминированным. Всегда в кавычках: `"helm.sh/hook-weight": "-5"`.
- **Ресурсы hook переживают `helm uninstall`.** Job hook *не* входят в релиз. Без `hook-delete-policy` упавшие Job копятся вечно, и `helm uninstall` их не уберёт. Всегда ставьте `before-hook-creation,hook-succeeded`.
- **Забыли `helm dependency update` после правки `Chart.yaml`.** Если добавите запись `dependencies:` (в этой лабораторной не нужно, но для бонуса легко добавить), субчарт не скачается в `charts/`, пока не выполните `helm dependency update`. Тогда `helm install` падает с путано сформулированной ошибкой про отсутствующую зависимость.
- **Подвох авторизации OCI.** `helm registry login ghcr.io` берёт конфиг Docker, а не отдельное хранилище кредов Helm. Если для `ghcr.io` завалялся старый логин Docker, сначала может понадобиться `docker logout ghcr.io`. Всегда отдавайте PAT через `--password-stdin`. Никогда не кладите его в командную строку: он осядет в истории шелла.
- **`apiVersion: v2` и `v1` в `Chart.yaml`.** `v2` — версия формата чарта, которую ввёл Helm 3 и которую по-прежнему использует Helm 4. Это **не** версия бинарника Helm. `v1` — только Helm 2. Helm 4 на чарте `v1` падает сразу. Helm 3 на чарте `v2` работает. Всегда пишите `apiVersion: v2`.
- **Пустой `image.tag` в `values.yaml`.** `tag: ""` даёт запасной путь `{{ .Values.image.tag | default .Chart.AppVersion }}`. `tag: latest` ломает воспроизводимость (убивает откаты и diff ArgoCD в Lab 13). В умолчаниях оставьте пустым. SemVer зафиксируйте в `values-prod.yaml`.

</details>

<details>
<summary>📖 Учебные материалы</summary>

- [Краткое руководство](https://helm.sh/docs/intro/quickstart/)
- [Использование Helm](https://helm.sh/docs/intro/using_helm/)
- [Три больших понятия](https://helm.sh/docs/intro/using_helm/#three-big-concepts)
- *Learning Helm* (2e, 2024) — Butcher, Farina, Dolitsky (O'Reilly)

</details>

---

## Что дальше

| Lab | Что добавляет к этому чарту |
|---:|---|
| 11 | Управление секретами через OpenBao: `values.yaml` лежит в git, а пароль БД — нет |
| 12 | ConfigMap и PVC добавляются новыми шаблонами в этот же чарт |
| 13 | ArgoCD выкатывает этот чарт через GitOps; `helm template` становится шагом рендера |
| 14 | Argo Rollouts заменяет шаблон Deployment на Rollout для canary-выкладки |
| 16 | kube-prometheus-stack (сам по себе Helm-чарт) скрейпит ваше приложение через ServiceMonitor, который вы добавите в этот чарт |

---

**Удачи!** ⛵

> **Запомните:** шаблонизируйте всё, не зашивайте ничего (кроме разумных умолчаний в `values.yaml`). Один чарт, N файлов values. Никогда не комментируйте health-проверку: сделайте из неё значение. И **никогда** не запускайте `helm create` в этой лабораторной.
