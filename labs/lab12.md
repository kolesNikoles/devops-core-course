# Lab 12 — ConfigMap и Persistent Volumes

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Configuration%20%26%20Storage-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-ConfigMaps%20%7C%20PVC%20%7C%20K8s%201.36-informational)

> **Цель:** вынесите несекретный конфиг в ConfigMap и сделайте так, чтобы данные приложения пережили `kubectl delete pod` за счёт PersistentVolumeClaim.
> **Что сдать:** PR из ветки `lab12`, который расширяет чарт Lab 10/11 двумя ConfigMap и PVC, счётчиком визитов на стороне приложения и `docs/LAB12.md` с доказательством персистентности: запись, удаление Pod, чтение из нового Pod.

---

## Обзор

В Lab 11 вы спрятали *секреты* (пароли БД, API-токены). Но не каждое значение — секрет: уровни логов, feature flags, имена приложений, конфиги `NGINX` — им место в **ConfigMap**. Контейнеры при этом *эфемерны*: Pod перезапустился — записываемый слой исчез. Всё, что приложение писало на локальный диск, пропало. Лечение — **PersistentVolumeClaim**: Pod забирает хранилище по имени, том идёт за Pod с узла на узел, данные переживают любой конкретный Pod.

В этой лабораторной вы потренируетесь:

- Трём паттернам инъекции ConfigMap (одна переменная, массовый `envFrom`, volume mount) и **выбору подходящего под ситуацию**
- Проектированию PVC: режим доступа, размер, storage class — **прочитав, что кластер реально даёт**, а не угадав
- Главному доказательству: **запись → `kubectl delete pod` → чтение из нового Pod → то же значение**. PVC, который просто перешёл в `Bound`, персистентность НЕ доказывает.

> ⚠️ **Объём:** одна реплика и хранилище `ReadWriteOnce`. `ReadWriteMany` (одновременная запись с нескольких узлов) требует сетевую ФС (NFS/EFS/CephFS) и в эту лабораторную не входит. StatefulSet и `volumeClaimTemplates` (PVC на каждую реплику) будут в Lab 15.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**
- Lab 9 — Deployment `web` и Service на k3d (1.36)
- Lab 10 — Helm-чарт `lab10-app`, в который упакованы этот Deployment и Service
- Lab 11 — Secret, подключённые в тот же чарт

**Эта лабораторная добавляет (в уже существующий чарт):**
- **Счётчик визитов** на стороне приложения: пишет в файл под `DATA_DIR` (по умолчанию `/data`)
- `templates/configmap.yaml` — два ConfigMap: один смонтирован файлом, второй массово залит в env
- `templates/pvc.yaml` — PVC под флагом `.Values.persistence.enabled`
- Патчи Deployment, которые всё это подключают
- `docs/LAB12.md` — доказательство персистентности и ваши рассуждения о решениях

> 📚 Идёт в паре с **Lecture 12 — ConfigMap и Persistent Volumes**. Перед стартом перечитайте слайды 6–8 (паттерны инъекции), 13–15 (PV/PVC/StorageClass, режимы доступа, reclaim policy) и 16 (ловушка `subPath`).

---

## Подготовка

Нужны работающий кластер k3d из Lab 9 на Kubernetes 1.36, чарт `lab10-app` и `kubectl`.

```bash
kubectl get nodes                # should show k3d-devops-server-0 + 2 agents on v1.36.x
helm list -n lab12 2>/dev/null   # namespace you'll use below
```

**Прежде чем писать хоть одну строку YAML**, посмотрите, что кластер даёт даром. От этого зависят ответы на «какой режим доступа?» и «какой storage class?»:

```bash
kubectl get storageclass
# YOUR TASK: read the output. Which one has the (default) marker? What provisioner backs it?
# Write the answer in docs/LAB12.md — this is the storage class your PVC will use.
```

> 💡 k3d (k3s) привозит **`local-path`** (`rancher.io/local-path`) с пометкой `(default)`. Только RWO, без снапшотов — для счётчика визитов в одном Pod хватит, в прод такое не ставят. На настоящем облачном кластере тот же чарт привяжется к тому **AWS EBS** (`ebs.csi.aws.com`) или **GCE PD** (`pd.csi.storage.gke.io`) через его CSI-драйвер, и чарт менять не придётся.

---

## Задание 1 — Доработка персистентности приложения (2 балла)

До любой работы с Kubernetes приложению нужно что-то, что стоит хранить. Добавьте счётчик визитов, который пишет в файл, и докажите локально, что он работает, — и только потом идите дальше.

### 1.1 — Добавьте счётчик визитов

`ВАША ЗАДАЧА`: в `app_python/app.py` (или аналог на вашем языке) добавьте счётчик.

Требования:

- `GET /` увеличивает счётчик; значение лежит в файле под `DATA_DIR` (переменная окружения, по умолчанию `/data`).
- На старте прочитайте файл. Если его нет — берите `0`, не падайте.
- Добавьте `GET /visits`, который возвращает JSON `{"visits": N}`.
- Пишите **атомарно**: сначала во временный файл в том же каталоге, потом `os.replace(tmp, final)`. Падение посреди записи не должно портить счётчик.
- На старте создайте каталог данных, если его нет (`os.makedirs(..., exist_ok=True)`).

Подсказки — набросок частей (не копируйте, выведите код сами):

- **константы** — `DATA_DIR = env("DATA_DIR","/data")`, `COUNTER = DATA_DIR + "/visits"`
- **read_count** — открыть и `int(...)`; на `FileNotFoundError` вернуть `0`
- **write_count(n)** — `mkstemp` в `DATA_DIR`, запись, `os.replace(tmp, COUNTER)` (атомарно)
- **`GET /`** — `c = read_count() + 1`; `write_count(c)`; ответить
- **`GET /visits`** — ответить `{ "visits": read_count() }`

### 1.2 — Локальный тест персистентности в Docker Compose

`ВАША ЗАДАЧА`: в `app_python/docker-compose.yml` смонтируйте каталог хоста в `DATA_DIR`. Дёрните `/` три раза, перезапустите контейнер, дёрните `/visits` — счётчик должен продолжиться, а не обнулиться.

Каркас (заполните пропуски):

```yaml
services:
  app:
    # ... your build/image from Lab 2 ...
    environment:
      DATA_DIR: # YOUR TASK
    volumes:
      - # YOUR TASK: host-path : DATA_DIR
```

### 1.3 — Доказательства

Вставьте в `docs/LAB12.md`:

- Три вызова (`curl localhost:8080/` × 3) и чтение `/visits`.
- `docker compose restart app`, затем `/visits` с **тем же** счётчиком, не с 0.
- `cat ./data/visits` с хоста с тем же целым (это доказывает, что значение реально сохранилось, а не гонка).

---

## Задание 2 — ConfigMap: три паттерна, одно решение на значение (3 балла)

Лекция разобрала три способа залить ConfigMap в Pod. **Один паттерн на *значение*, а не все три на каждое.** Вы отдадите два ConfigMap (один как файлы, другой как переменные окружения), подключите каждый в Pod тем паттерном, который подходит его форме, и защитите выбор для трёх примеров в таблице ниже.

### 2.1 — Напишите два ConfigMap

Два ConfigMap поедут в `templates/configmap.yaml`. Форма и *лейблы* заданы; **данные** заполняете вы из `values.yaml` чарта.

Каркас — `templates/configmap.yaml` (один файл, два ConfigMap через `---`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "lab10-app.fullname" . }}-file
  labels: {{- include "lab10-app.labels" . | nindent 4 }}
data:
  config.json: |-                # YOUR TASK: load files/config.json via .Files.Get
                                 #            and indent 4 — hint: `| indent 4`
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "lab10-app.fullname" . }}-env
  labels: {{- include "lab10-app.labels" . | nindent 4 }}
data:                            # YOUR TASK: ≥3 K/V pairs from .Values
                                 # ⚠️ keys must match [A-Z_][A-Z0-9_]* (Lec 12 slide 7)
                                 # invalid names are silently skipped by envFrom
```

`ВАША ЗАДАЧА`: создайте `files/config.json` минимум с ключами `appName`, `environment`, `features` (объект с `visitsCounter: true`). Те же значения добавьте в `values.yaml`.

### 2.2 — Выберите паттерн инъекции на каждое значение

Паттернов инъекции три (Lecture 12, слайды 6–8). Ниже каркас каждого отдельным блоком-патчем Deployment — **нужные оставьте, лишние удалите**, затем обоснуйте выбор в `docs/LAB12.md`. Как минимум в итоге Deployment использует паттерн B (ConfigMap в форме env) **и** паттерн C (ConfigMap в форме файла). Паттерн A необязателен и даёт баллы, только если вы можете защитить конкретную причину переименовать ключ или взять ровно один.

**Паттерн A — одна переменная, переименование и необязательность.** Когда нужно ОДНО значение, возможно ПЕРЕИМЕНОВАННОЕ внутри контейнера.

```yaml
env:
  - name:                            # YOUR TASK: env name inside the container
    valueFrom:
      configMapKeyRef:
        name:                        # YOUR TASK: which of your two ConfigMaps holds this?
        key:                         # YOUR TASK: key in the ConfigMap (may differ from name)
        optional:                    # YOUR TASK: true/false — start if missing?
```

**Паттерн B — `envFrom` (пачкой).** КАЖДЫЙ ключ ConfigMap становится переменной окружения. Без переименования и без фильтра: отбирайте ключи в самом ConfigMap, а не в инъекции.

```yaml
envFrom:
  - configMapRef:
      name:                          # YOUR TASK
```

**Паттерн C — volume mount (весь каталог, НЕ `subPath`).** Для структурированных файлов (JSON/YAML/conf), которые читает приложение. Обновляется сам, без пересборки образа (на этом держится бонус).

```yaml
volumeMounts:
  - name:                            # YOUR TASK: must match the volume name below
    mountPath:                       # YOUR TASK: dir where config.json appears
                                     # ⚠️ NO subPath: here — see Bonus + Pitfalls
volumes:
  - name:                            # YOUR TASK: same name as the volumeMount above
    configMap:
      name:                          # YOUR TASK: which ConfigMap supplies the files?
```

Подберите паттерн каждому из этих значений и зафиксируйте выбор в `docs/LAB12.md` (по одному предложению):

| Значение | Паттерн? | Почему? |
|---|---|---|
| `LOG_LEVEL` (одна строка, которую фреймворк читает из env) | ВАША ЗАДАЧА | ВАША ЗАДАЧА |
| `config.json` (структурированный файл с feature flags) | ВАША ЗАДАЧА | ВАША ЗАДАЧА |
| ≥ 10 булевых feature flags, и все должны стать переменными окружения | ВАША ЗАДАЧА | ВАША ЗАДАЧА |

> 💡 **Подсказка** — ни один из трёх ответов не «все три». Одно значение — один паттерн. Смысл в *решении*, а не в том, чтобы размножить паттерны.

### 2.3 — Доказательства

`ВАША ЗАДАЧА`: после `helm upgrade --install lab10-app k8s/lab10-app -n lab12 --create-namespace` (тот же путь чарта, что в Lab 10/11) снимите:

```bash
# YOUR TASK: get the pod name into $POD
# YOUR TASK: cat the mounted config.json from inside the pod — shows file mount works
# YOUR TASK: printenv | grep -E '<your env keys>' — shows env injection works
# YOUR TASK: kubectl describe pod $POD | grep -A2 -E 'Environment|Mounts' — shows the wiring
```

Вставьте в `docs/LAB12.md`:

- Три захвата выше (содержимое файла, переменные окружения, выдержка из описания Pod)
- Заполненную таблицу решений из 2.2
- Один абзац: *если в ConfigMap 30 ключей, а нужны только 2, стали бы вы всё равно брать `envFrom`? Почему да или почему нет?*

---

## Задание 3 — Persistent Volumes: сначала прочитайте кластер, потом напишите claim (3 балла)

У PVC четыре ручки: `accessModes`, `resources.requests.storage`, `storageClassName` и (это менее очевидно) **reclaim policy**, которую он наследует от StorageClass. Первые три задайте явно — но только после того, как посмотрите, что кластер уже предлагает.

### 3.1 — Сначала осмотрите хранилище кластера

```bash
kubectl get storageclass
kubectl get sc <default-sc-name> -o yaml | grep -E 'provisioner|reclaimPolicy|volumeBindingMode'
```

`ВАША ЗАДАЧА`: в `docs/LAB12.md` ответьте:

- У какого StorageClass стоит пометка `(default)`? Какой provisioner за ним?
- Какой у него `reclaimPolicy` (Delete или Retain)? Что будет с данными, если удалить PVC?
- Какой у него `volumeBindingMode`? Почему при `WaitForFirstConsumer` PVC в `Pending` — это **норма**, а не баг?

### 3.2 — Выберите `accessModes` для счётчика визитов

Четыре режима (Lecture 12, слайд 14):

- **`ReadWriteOnce`** — чтение и запись с одного *узла* (несколько Pod на этом узле можно). Облачные блочные диски (EBS/PD/Azure Disk) умеют только это.
- **`ReadOnlyMany`** — только чтение со многих узлов. Раздача контента.
- **`ReadWriteMany`** — чтение и запись со многих узлов сразу. Нужны NFS/EFS/CephFS — медленнее и дороже.
- **`ReadWriteOncePod`** — чтение и запись ровно с *одного* Pod на весь кластер (GA в K8s 1.29).

`ВАША ЗАДАЧА`: выберите ОДИН режим для счётчика визитов и обоснуйте его в `docs/LAB12.md`. Подсказка: одна реплика Deployment пишет в один файл. Не хватайтесь за самый гибкий вариант — берите тот, который реально подходит.

### 3.3 — Напишите PVC

Каркас — `templates/pvc.yaml`:

```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "lab10-app.fullname" . }}-data
  labels: {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  accessModes:
    -                              # YOUR TASK: RWO | ROX | RWX | RWOP — pick + defend
  resources:
    requests:
      storage:                     # YOUR TASK: size — counter is one int, don't ask 100Gi
  {{- with .Values.persistence.storageClass }}
  storageClassName: {{ . }}        # omit field → cluster default; "" → also default
  {{- end }}
{{- end }}
```

Добавьте в `values.yaml`:

```yaml
persistence:
  enabled: true
  size: # YOUR TASK
  storageClass: # YOUR TASK: "" to use the cluster default, or a specific name (e.g. local-path)
```

### 3.4 — Смонтируйте его на Deployment

Пропатчьте `templates/deployment.yaml` — только новые куски:

```yaml
volumeMounts:
  - name:                          # YOUR TASK: invent a volume name (match below)
    mountPath:                     # YOUR TASK: must equal DATA_DIR from Task 1
volumes:
  - name:                          # YOUR TASK: same name as the volumeMount above
    {{- if .Values.persistence.enabled }}
    persistentVolumeClaim:
      claimName:                   # YOUR TASK: same name as the PVC you authored above
    {{- else }}
    emptyDir: {}                   # fallback: data dies with the pod
    {{- end }}
```

### 3.5 — Доказательство персистентности (главное)

Это сердце лабораторной. **PVC в статусе `Bound` значит, что том есть, а не то, что данные пережили перезапуск Pod.** Персистентность доказывается так: записали, удалили Pod, прочитали из нового Pod.

`ВАША ЗАДАЧА`: четыре шага придумайте сами. Понадобятся `kubectl exec ... -- sh -c 'echo ... > $DATA_DIR/marker'`, `kubectl delete pod`, `kubectl wait` и финальный `kubectl exec ... cat $DATA_DIR/marker`.

Для доказательства возьмите **отдельный файл** `marker` — не переиспользуйте `/data/visits`. Счётчик растёт сам, а вам нужна неизменная строка для сравнения. Оба файла лежат на одном PVC, так что доказательство для `marker` заодно доказывает и `visits`.

```bash
# YOUR TASK: write a unique marker (e.g. epoch seconds) to $DATA_DIR/marker
#            inside the running pod (DATA_DIR defaults to /data — adjust if you changed it)
# YOUR TASK: delete the pod and wait for the Deployment to spin up a fresh one
# YOUR TASK: read $DATA_DIR/marker from the NEW pod
# YOUR TASK: prove the marker is the SAME — the write survived the pod's death
```

Сдать нужно реальный захват, где *одна и та же* строка до и после. Если строки разные (или второй `cat` отвечает "No such file or directory"), PVC на самом деле не смонтирован в каталог данных, либо storage class ведёт себя неожиданно — смотрите `kubectl describe pvc` и `kubectl describe pod`.

> 💡 От `kubectl exec` может прилететь `Unknown stream id 5` — это безвредное предупреждение websocket на стороне клиента, не ошибка. Если данные сходили туда и обратно, всё хорошо.

### 3.6 — Доказательства

Вставьте в `docs/LAB12.md`:

- Осмотр storage class из 3.1 и ваши ответы
- Решение по режиму доступа из 3.2 с обоснованием
- `kubectl get pvc`: claim в `Bound` (или `Pending`, если `WaitForFirstConsumer` и ни один Pod его ещё не взял — напишите, какой из случаев у вас)
- **Доказательство персистентности из 3.5: имя исходного Pod, записанный marker, команда удаления, имя нового Pod, прочитанный marker — то же значение, рядом**

---

## Задание 4 — Документация (2 балла)

Соберите всё в `docs/LAB12.md`. Обязательные разделы, по порядку:

1. **Изменения приложения** — устройство счётчика, `DATA_DIR`, зачем писать атомарно, захват персистентности Compose из задания 1
2. **Устройство ConfigMap** — ваш `files/config.json`, два ConfigMap, таблица решений из 2.2, захваты проверки из 2.3
3. **Устройство PVC** — осмотр StorageClass, обоснование режима доступа, YAML PVC, монтирование, **главное доказательство персистентности из 3.5**
4. **ConfigMap и Secret** — когда что брать (область RBAC, tmpfs против обычной ФС, история про encryption-at-rest). Сошлитесь на Lab 11.
5. **Ловушки, на которые вы сами наткнулись** — минимум одна настоящая и как вы её отладили

---

## Бонусное задание — горячая перезагрузка ConfigMap (2 балла)

Поменяйте значение в `lab10-app-env`. Посмотрите на запущенный Pod. Переменная окружения **не меняется**. Поменяйте значение в `lab10-app-file`. Посмотрите на смонтированный файл. Он **когда-нибудь** меняется — через минуты. А если смонтировать его одним файлом через `subPath`? Не изменится **никогда**.

Из-за этих трёх поведений каждая команда в итоге выбирает стратегию hot reload. Вам нужно:

1. **Воспроизведите поведение по умолчанию.** `ВАША ЗАДАЧА`: измените значение через `kubectl edit configmap` и засеките, сколько пройдёт, пока (a) переменная внутри запущенного Pod увидит изменение — меряйте в Pod через `printenv` — и (b) смонтированный файл увидит изменение — меряйте `cat` в цикле. Зафиксируйте результат.

2. **Объясните `subPath`.** `ВАША ЗАДАЧА`: одним абзацем в `docs/LAB12.md` объясните, почему монтирование через `subPath` — разовая копия, которая больше не обновляется, и когда вы всё равно на этот компромисс пойдёте (подсказка: чистые пути монтирования, приложению не виден симлинк `..data`).

3. **Выберите и реализуйте ОДНУ стратегию.** К команде установки пока не тянитесь — разберите три варианта и выберите:

   - **A — аннотация с контрольной суммой (родная для GitOps, без лишнего контроллера):** в шаблоне Pod лежит аннотация, значение которой — `sha256sum` отрендеренного ConfigMap. Поменялись данные → поменялась аннотация → Deployment делает rollout. Всё это живёт в вашем шаблоне Helm.
   - **B — контроллер Reloader** (Stakater): контроллер в кластере смотрит на ConfigMap и Secret и патчит Deployment-владельцев, чтобы запустить rollout. В чарте ноль логики, одна аннотация.
   - **C — наблюдение в самом приложении:** приложение через `inotify` следит за смонтированным файлом и перечитывает его при изменении. Без рестарта, сразу. Так из коробки делают NGINX, Envoy, Prometheus и Loki.

   `ВАША ЗАДАЧА`: в `docs/LAB12.md` напишите 3–4 предложения: что выбрали и **почему** (ложится ли на GitOps? сколько лишних частей? умеет ли ваше приложение вообще вариант C?). Потом реализуйте. **Команд установки и кусков аннотаций в этой лабораторной нет** — идите в лекцию, в документацию или в upstream README и разберитесь сами.

4. **Покажите, что это работает.** `ВАША ЗАДАЧА`: поменяйте значение ConfigMap и поймайте либо (a) rolling restart для A/B, либо (b) перечитывание на месте для C. Захватите доказательства.

> 💡 Если выбрали A, хеш должен меняться, когда меняются **данные** ConfigMap. `helm upgrade` без изменений Pod перекатывать НЕ должен. Если два прогона `helm upgrade --dry-run` дают одну и ту же аннотацию — сделано верно.

---

## Как сдавать

```bash
git switch -c lab12
git add app_python/ k8s/lab10-app/ docs/LAB12.md
git commit -m "feat(lab12): configmaps + persistence with hot-reload strategy"
git push -u origin lab12
```

Откройте **два** PR:

- `your-fork:lab12` → `course-repo:master` *(на ревью)*
- `your-fork:lab12` → `your-fork:master` *(вливается в ваш собственный main)*

Чеклист PR:

```text
- [ ] Task 1 done — counter + /visits + DATA_DIR + atomic write + Compose persistence
- [ ] Task 2 done — two ConfigMaps, three patterns understood, decision table filled in
- [ ] Task 3 done — PVC bound, mounted, write→delete-pod→read proof captured
- [ ] Task 4 done — docs/LAB12.md complete with all five sections
- [ ] Bonus done — default behavior measured, strategy chosen + implemented + demonstrated
```

---

## Критерии приёмки

### Задание 1 (2 балла)
- ✅ `GET /` увеличивает счётчик и сохраняет его; `GET /visits` возвращает текущее значение
- ✅ Переменная `DATA_DIR` учитывается; запись атомарная (временный файл и rename)
- ✅ Перезапуск Compose сохраняет счётчик (захват есть)

### Задание 2 (3 балла)
- ✅ `files/config.json` и ConfigMap `lab10-app-file` (volume mount)
- ✅ ConfigMap `lab10-app-env` с ≥ 3 ключами в форме env (`envFrom`)
- ✅ Таблица решений заполнена для каждого из трёх примеров, с обоснованием
- ✅ Захваты проверки: файл читается в Pod, переменные на месте, `describe pod` показывает проводку

### Задание 3 (3 балла)
- ✅ Осмотр StorageClass записан (какой именно, какой provisioner, какой `volumeBindingMode`)
- ✅ Режим доступа выбран и обоснован (здесь верен `ReadWriteOnce` — защитите его)
- ✅ PVC доходит до `Bound` (или `Pending` объяснён через `WaitForFirstConsumer`)
- ✅ **Запись → удаление Pod → чтение из нового Pod, тот же marker, захват рядом**

### Задание 4 (2 балла)
- ✅ Все пять разделов `docs/LAB12.md` заполнены реальными захватами (не примерами из текста)

### Бонус (2 балла)
- ✅ Поведение по умолчанию измерено: задержка обновления переменной и задержка обновления файла
- ✅ Ловушка `subPath` объяснена своими словами
- ✅ Выбрана одна стратегия (A/B/C) с письменным обоснованием, реализована и показана

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — персистентность приложения | **2** | Счётчик, атомарная запись, доказательство в Compose |
| **Задание 2** — ConfigMap | **3** | Два ConfigMap, понимание трёх паттернов, таблица решений |
| **Задание 3** — PVC | **3** | Верный режим доступа, верные размер и storage class, главное доказательство персистентности |
| **Задание 4** — документация | **2** | Все пять разделов LAB12.md с реальными доказательствами |
| **Бонус** — hot reload | **2** | Поведение по умолчанию измерено, стратегия выбрана, реализована и показана |
| **Итого** | **12** | 10 основных + 2 бонус |

**Как оцениваем:**
- **10/10:** все четыре задания; доказательство персистентности однозначно (один и тот же marker в обоих Pod); решения по ConfigMap защищены, а не просто выбраны
- **8–9/10:** ConfigMap и PVC работают; доказательство персистентности есть, но обоснование тонкое или одно решение по паттерну неверное
- **6–7/10:** PVC в `Bound`, но нет доказательства запись→удаление→чтение, ИЛИ использован только один паттерн ConfigMap
- **< 6:** нет доказательства персистентности, или ConfigMap не подключён в Pod

---

## Материалы

<details>
<summary>📚 Документация</summary>

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) — концепции
- [Configure a Pod with a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) — три паттерна инъекции
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) — PV, PVC, StorageClass
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) — `reclaimPolicy`, `volumeBindingMode`
- [Access Modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) — RWO/ROX/RWX/RWOP
- [Helm `.Files.Get`](https://helm.sh/docs/chart_template_guide/accessing_files/) — вложить файл в ConfigMap
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner) — StorageClass по умолчанию в k3d

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных пробных прогонов — прочитайте это ДО отладки)</summary>

- **Переменные из ConfigMap в запущенном Pod не обновляются.** Поменяйте `LOG_LEVEL` в ConfigMap — `printenv` в запущенном Pod всё равно показывает старое значение, и так будет всегда, пока Pod не перезапустится. Это причина номер один у «мой ConfigMap обновился, а ничего не работает». Переменные запекаются на старте Pod; дальше их уже не трогают.
- **Монтирование через `subPath` НИКОГДА не обновляется само.** `subPath: config.json` — это разовая копия файла на старте Pod. ConfigMap может меняться каждые пять секунд, файл в контейнере — нет. Монтирование всего каталога обновляется само (в пределах `kubelet --sync-frequency`, по умолчанию 60 с плюс TTL кэша — суммарная задержка может быть пара минут). Хотите hot reload — монтируйте весь каталог.
- **PVC при `WaitForFirstConsumer` остаётся в `Pending` — это норма, не ошибка.** StorageClass по умолчанию в k3d (и большинство облачных SC в кластерах на несколько AZ) ждёт, пока Pod реально смонтирует PVC, чтобы PV можно было создать в той же зоне, что и Pod. Если PVC в `Pending` и ни один Pod его ещё не взял, просто задеплойте Pod — «чинить» StorageClass не надо.
- **`kubectl exec ... Unknown stream id` безвреден.** Это сообщение websocket на стороне клиента из-за безобидного расхождения control-frame. Данные ходят нормально. Не гоняйтесь за ним.
- **Удаление PVC может НЕ удалить PV (и данные).** Зависит от `reclaimPolicy`: `Delete` (значение по умолчанию у облачных SC) стирает том; `Retain` оставляет PV и диск после того, как PVC исчез, — убирать придётся руками. Ставьте `Retain` на всё, о чём будете жалеть, и напишите runbook по уборке **до** того дня, когда он понадобится.
- **Невалидные имена переменных из `envFrom` молча пропускаются.** Ключ ConfigMap `feature.x.enabled` переменной не станет: `envFrom` выпускает только ключи по шаблону `[A-Z_][A-Z0-9_]*`. Ни ошибки, ни предупреждения — просто пусто.
- **`hostPath` ради «настоящей» персистентности привязывает Pod к узлу.** Только для демо. Даже на локальном кластере берите PVC: local-path provisioner даёт настоящий PVC поверх каталога хоста, и перенос Pod работает как положено.
- **Не затирайте `/data/visits` своим marker.** Для доказательства из 3.5 берите *отдельный* файл (например, `/data/marker`). Счётчик визитов переписывает `visits` на каждый `GET /`, и сравнение рядом начинает врать.

</details>

<details>
<summary>🛠️ Полезный CLI (без YAML — только рецепты exec)</summary>

- `kubectl get sc` / `kubectl describe sc <name>` — какое хранилище даёт кластер
- `kubectl describe pvc <name>` — смотрите Events внизу, если PVC завис
- `kubectl describe pod <name> | grep -A4 'Mounts\|Volumes'` — что реально смонтировано
- `kubectl exec <pod> -- mount | grep /data` — убедитесь, что том смонтирован
- `kubectl get events --field-selector involvedObject.name=<pvc>` — ошибки provisioning
- Для доказательства персистентности: запишите marker командой `kubectl exec deploy/<name> -- sh -c 'echo $(date +%s) > /data/marker'` (НЕ `/data/visits` — счётчик его перезаписывает), затем `kubectl delete pod -l <selector>`, затем `kubectl wait --for=condition=Ready pod -l <selector> --timeout=60s`, затем `kubectl exec deploy/<name> -- cat /data/marker`

</details>

---

## Что дальше

| Lab | Что добавляет |
|---:|---|
| 13 | ArgoCD и GitOps — ваши ConfigMap и PVC реконсилируются из git |
| 14 | Argo Rollouts — canary нового ConfigMap, прежде чем его увидят все Pod |
| 15 | StatefulSet и `volumeClaimTemplates` — PVC *на* каждую реплику, для Postgres и Kafka |
| 16 | kube-prometheus-stack — скрейп `/metrics` с вашего приложения, алерт на регрессии счётчика визитов |

> **Запомните:** цель лабораторной — не «PVC в `Bound`». Цель — «значение, которое я записал в Pod A, вернулось из Pod B после того, как я удалил Pod A». Вот это персистентность. Всё остальное — обвязка.
