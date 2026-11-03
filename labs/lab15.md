# Lab 15 — StatefulSet и постоянное хранилище

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-StatefulSets-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-StatefulSet%20%7C%20PVC%20%7C%20K8s%201.36-informational)

> **Цель:** Запустите нагрузки, которые *помнят*. Дайте каждому Pod стабильное имя и свой диск. Докажите, что данные `web-1` остаются с `web-1` (а не уезжают на случайный Pod) после `kubectl delete`.
> **Что сдать:** PR из `lab15`. Чарт из Lab 12 переведён с Deployment на StatefulSet + headless Service (Service без ClusterIP). В `k8s/docs/LAB15.md` — доказательство, что PVC живёт на каждом Pod: запись в `web-1`, удаление `web-1`, чтение того же значения; запись в `web-0` и проверка, что значения не пересекаются.

---

## Обзор

До Lab 14 каждый Pod был одноразовым скотом. Убили — пришёл новый, со случайным именем и пустым диском, и никто не заметил. Это ломается, как только нагрузке нужно помнить что-то *своё на каждый Pod*: реплика базы, партиция очереди, шард поиска. `Deployment` даёт взаимозаменяемые Pod и общее хранилище (или никакого). **StatefulSet** даёт каждому Pod стабильное порядковое имя (ordinal), свой `PersistentVolumeClaim` и упорядоченный жизненный цикл.

В этой лабораторной вы отработаете:

- **Манифест StatefulSet от полей API** — `apiVersion`, `serviceName`, `replicas`, треугольник меток, шаблон Pod и список `volumeClaimTemplates`
- **Headless Service с нуля** — и *почему* именно `clusterIP: None` создаёт DNS на каждый Pod
- Главное доказательство: **маркер в том `web-1` → `kubectl delete pod web-1` → чтение маркера у нового `web-1` → значения СОВПАДАЮТ**. `data-web-1` привязан к ordinal 1, а не к конкретному экземпляру Pod
- Вторая проверка: данные `web-0` — *его собственные*, это не данные `web-1`

> ⚠️ **Границы:** StorageClass `local-path` по умолчанию в k3d, один StatefulSet, три реплики, доступ `ReadWriteOnce`. Без multi-AZ, без операторов, без failover. В бонусе — стратегии обновления. Выберите один из трёх путей.

> 🪨 **Педагогическое ядро.** Lab 12 доказала, что PVC переживает удаление Pod. Там был **один общий PVC** на одну реплику Deployment. Lab 15 доказывает более глубокое. С `volumeClaimTemplates` каждый ordinal `N` привязан к своему PVC `data-<sts>-N`, и привязка держится при повторном планировании, рестарте и даже отказе узла. **Если в отчёт попадёт только один захват, пусть это будет запись-удаление-чтение на `web-1`, где маркер вернулся.**

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**

- Lab 9 — Deployment `web` + Service на k3d (1.36), 3 узла
- Lab 10 — Helm-чарт `lab10-app`
- Lab 11 — Secret уже подключены
- Lab 12 — счётчик `/visits`, который пишет в `$DATA_DIR/visits`, PVC смонтирован по этому пути, и доказательство запись-удаление-чтение

**Эта лабораторная добавляет (в тот же чарт):**

- `templates/statefulset.yaml` — **пишете вы** (в этой лабораторной он вместо Deployment; `deployment.yaml` оставьте в дереве, но один из двух выключайте через values)
- `templates/service-headless.yaml` — **пишете вы** (рядом с обычным Service из Lab 9; оба живут вместе)
- Небольшая правка `values.yaml`: переключение с `Deployment` на `StatefulSet` и `replicaCount: 3`
- `k8s/docs/LAB15.md` — отчёт для сдачи

Обычный Service (ClusterIP из Lab 9) **не удаляйте**. Оставьте оба. Обычный Service балансирует запросы между Pod для клиента, которому всё равно, кто ответит. Headless Service нужен, чтобы обратиться к конкретному Pod.

> 📚 В паре с **Lecture 15 — StatefulSet и постоянное хранилище**. Перед стартом перечитайте слайд 5 (три гарантии), 6–7 (headless Service и DNS на каждый Pod), 8 (`volumeClaimTemplates`) и 19 (ловушка осиротевшего PVC).

---

## Подготовка

Нужен кластер k3d из Lab 9 на Kubernetes **1.36**, чарт `lab10-app` и `kubectl`.

```bash
kubectl get nodes                       # k3d-devops-server-0 + 2 agents on v1.36.x
kubectl get storageclass                # local-path (default) — what your PVCs will use
helm list -n lab15 2>/dev/null
```

> 💡 Замечание про кластер из Lab 12 всё ещё в силе. k3d ставит `local-path` (`rancher.io/local-path`) как `StorageClass` по умолчанию, с пометкой `(default)`. Только RWO, динамически выделяется через hostPath. Для счётчика на 3 реплики этого хватает. По *интерфейсу* это то же, что EBS / GCE PD / Ceph RBD в проде, так что манифест StatefulSet отсюда без правок заработает на настоящем кластере.

Дерево каталогов (файлы заполните сами):

```
lab10-app/                              # your Helm chart from Lab 10
├── templates/
│   ├── statefulset.yaml                # YOU write this (§Task 2)
│   ├── service-headless.yaml           # YOU write this (§Task 3)
│   ├── service.yaml                    # exists from Lab 9 — keep as-is
│   └── deployment.yaml                 # exists from Lab 9 — keep, gate behind values
└── values.yaml                         # add: kind toggle, replicaCount, persistence.*
k8s/docs/
└── LAB15.md                            # your submission report
```

---

## Задание 1 — Понятия: когда важны идентичность и диск на каждый Pod (2 балла)

Прежде чем писать хоть одну строку YAML, запишите, *зачем* вы её пишете. Задание только на документацию. Ответы — в `k8s/docs/LAB15.md` (тот же файл дополняете в каждом следующем задании).

`ВАША ЗАДАЧА`: по 2–3 предложения на каждый пункт:

1. **Три гарантии.** Что StatefulSet даёт такого, чего нет у Deployment? Подсказка — список со слайда 5: стабильная идентичность, стабильное хранилище, упорядоченный жизненный цикл. Своими словами, по одному предложению: *какой нагрузке нужна каждая и почему*.

2. **Почему Deployment не годится для Postgres.** Возьмите **два** пункта из списка: случайные имена Pod, общий PVC (или его нет), параллельный старт, IP Pod меняется при рестарте. На каждый — одно предложение про конкретный отказ в кластере Postgres (таблица на слайде 3 Lecture 15).

3. **Headless Service.** Своими словами: что `clusterIP: None` делает с DNS и почему StatefulSet *требует* управляющий headless Service, чтобы обращаться к отдельным Pod. (Подсказка: обычный Service отдаёт один VIP; headless отдаёт N A-записей *плюс* записи на каждый Pod.)

4. **Когда своё писать не стоит.** В 2–3 предложениях: когда вы бросите сырой StatefulSet и возьмёте оператор (например CloudNativePG, Strimzi, Rook)? Подсказка: failover, согласованные бэкапы, восстановление на момент времени.

`ВАША ЗАДАЧА`: заполните в `LAB15.md` таблицу сравнения Deployment и StatefulSet. Колонки заданы. Строки заполняете вы.

| Свойство | Deployment | StatefulSet |
|---|---|---|
| Имена Pod | `ВАША ЗАДАЧА` | `ВАША ЗАДАЧА` |
| Хранилище | `ВАША ЗАДАЧА` | `ВАША ЗАДАЧА` |
| Порядок масштабирования | `ВАША ЗАДАЧА` | `ВАША ЗАДАЧА` |
| Сетевая идентичность | `ВАША ЗАДАЧА` | `ВАША ЗАДАЧА` |
| Порядок завершения | `ВАША ЗАДАЧА` | `ВАША ЗАДАЧА` |

### Подтверждение

**Вставьте в `k8s/docs/LAB15.md`:**

- Все четыре ответа выше, своими словами
- Заполненную таблицу
- Одно предложение: примените практическое правило про оператор к приложению `visits`. Запустили бы *вы* его сырым StatefulSet или взяли оператор? Почему?

---

## Задание 2 — Переведите Deployment в StatefulSet (3 балла)

Главный манифест. Ниже **скелет**. То, что по сути совпадает с шаблоном Deployment из Lab 10, уже заполнено (хелперы чарта, форма контейнера). То, что *и есть навык StatefulSet* — `replicas`, `selector.matchLabels`, всё тело шаблона Pod и весь блок `volumeClaimTemplates` — пустое. Напишите это.

### 2.1 — Напишите `templates/statefulset.yaml`

```yaml
# lab10-app/templates/statefulset.yaml
apiVersion: apps/v1                                     # GA since K8s 1.9 — the only group/version
kind: StatefulSet
metadata:
  name: {{ include "lab10-app.fullname" . }}-web        # same `-web` component suffix as your Lab 10 Deployment;
                                                        # rendered name on the default release will be `lab10-app-web`,
                                                        # pods will be `lab10-app-web-0`, `lab10-app-web-1`, `lab10-app-web-2`.
  labels:
    {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  serviceName: ___                                      # YOUR TASK: must equal the name of your headless Service in Task 3
                                                        #            ⚠️ if this is missing or wrong, the controller
                                                        #            still creates pods but per-pod DNS silently breaks
  replicas: ___                                         # YOUR TASK: how many ordinals do you need to
                                                        #            demonstrate per-pod identity AND prove the
                                                        #            data doesn't cross between pods?
  selector:
    matchLabels:
      ___: ___                                          # YOUR TASK: the label that matches pod template labels
                                                        #            AND the headless Service selector — immutable
  template:
    metadata:
      labels:
        ___: ___                                        # YOUR TASK: MUST equal selector.matchLabels above
    spec:
      containers:
        - name: app                                     # YOUR TASK: write the full container spec —
          # YOUR TASK: image, ports, env (DATA_DIR from Lab 12),
          # YOUR TASK: volumeMounts (data → /data, matching the volumeClaimTemplate name),
          # YOUR TASK: readinessProbe pointing at /health (cheap; not the visit counter).
          # The shape is identical to your Lab 10 Deployment container — copy that, then
          # change the volume name to match the volumeClaimTemplate you'll write below.
  volumeClaimTemplates:
    # YOUR TASK: the whole list. Each entry is a PVC template — it LOOKS like a normal PVC
    #            with metadata + spec, but it lives INSIDE the StatefulSet. The `metadata.name`
    #            becomes the PREFIX of the per-pod claim:  data-<sts>-0,  data-<sts>-1,  data-<sts>-2.
    #            Required fields:
    #              - metadata.name              (e.g. "data" — matches the volumeMount above)
    #              - spec.accessModes           (which one for one-Deployment-replica-style writes?)
    #              - spec.resources.requests.storage  (size — counter is one int)
    #              - spec.storageClassName      (optional; omit = use cluster default = local-path on k3d)
```

**Почему каждый пропуск важен — прочитайте, прежде чем заполнять:**

- **`spec.serviceName`** — связка StatefulSet и DNS. Из этого имени контроллер собирает FQDN на каждый Pod: `<pod>.<serviceName>.<ns>.svc.cluster.local`. **Нет `serviceName` или он неверный — тихий отказ: Pod поднимаются, но `kubectl exec web-0 -- getent hosts web-1.<wrong-svc>` ничего не возвращает.** В документации StatefulSet это поле прямо названо обязательным.
- **`replicas`** — выберите число, которым сразу доказываются **две** вещи: (a) идентичность по ordinal (нужно минимум 2, чтобы было видно `web-0 ≠ web-1`) и (b) свой PVC у каждого Pod (чем больше Pod, тем убедительнее). В эталоне берут 3.
- **Треугольник меток** — та же ошибка №1 в YAML для Kubernetes, что и в Lab 9. `selector.matchLabels` совпадает с `template.metadata.labels`. `spec.selector` headless Service в задании 3 совпадает с *теми же* метками.
- **`volumeClaimTemplates`** — это **не PVC**. Это *шаблон*: контроллер штампует из него по одному PVC на ordinal. Блок **после создания не меняется**. Чтобы увеличить размер, правьте PVC напрямую (сработает, только если у StorageClass стоит `allowVolumeExpansion: true`). Повторная правка шаблона изменение не разнесёт. **Не откладывайте это на потом. Слайд 19 Lecture 15 называет это ловушкой изменения размера.**
- **Спека контейнера.** `volumeMounts[].name` совпадает с `volumeClaimTemplates[].metadata.name`. Назовёте по-разному — Pod стартует с `emptyDir` и без сообщения об ошибке. Записи просто не сохраняются.

### 2.2 — Обновите `values.yaml` и закройте старый Deployment

Добавьте в `values.yaml`:

```yaml
# YOUR TASK: add the keys this lab needs. Required:
#   - a toggle that picks StatefulSet over Deployment (so deployment.yaml doesn't render)
#   - replicaCount (Task 2)
#   - persistence.size, persistence.storageClass (Task 2 volumeClaimTemplates)
# Suggested shape:
# kind: StatefulSet            # toggle: "StatefulSet" | "Deployment"
# replicaCount: ...
# persistence:
#   size: ...
#   storageClass: ""           # "" → cluster default = local-path on k3d
```

`ВАША ЗАДАЧА`: в `templates/deployment.yaml` (файл из Lab 10) оберните всё целиком в `{{- if eq .Values.kind "Deployment" }} ... {{- end }}`, чтобы при переключении он не рендерился. Такое же условие поставьте в начале `statefulset.yaml`: `eq .Values.kind "StatefulSet"`. **Оба файла оставьте в дереве.** Проверяющие смотрят, что историю вы не стёрли.

### 2.3 — Выкатите и проверьте

```bash
helm upgrade --install lab10-app ./lab10-app -n lab15 --create-namespace
kubectl get statefulset -n lab15                # YOUR TASK: should show 3/3 READY
kubectl get pods -n lab15 -l app.kubernetes.io/name=lab10-app  # YOUR TASK: ordinal pods, NOT random suffixes
kubectl get pvc -n lab15                        # YOUR TASK: 3 PVCs named data-<sts>-0/1/2, all Bound
```

> 💡 **Лестница отладки, если с первого раза не завелось:**
> 1. `kubectl describe statefulset <name>` — если контроллер отказывается катить, поле названо в Events внизу.
> 2. `kubectl get pvc` — PVC в `Pending` при `local-path` это **норма** (`WaitForFirstConsumer`). `Pending` без Pod-потребителя — неверный access mode или storage class.
> 3. `kubectl describe pod <name>-0` — Events внизу, если CrashLoopBackOff или Pod так и не стартовал.
> 4. **`spec.selector.matchLabels` после создания не меняется.** Ошиблись на первом apply: `helm uninstall lab10-app -n lab15`, `kubectl delete pvc -n lab15 -l app.kubernetes.io/name=lab10-app`, затем снова `helm upgrade --install`.

### 2.4 — Подтверждение

**Вставьте в `k8s/docs/LAB15.md`:**

- `kubectl get sts,pod,pvc -n lab15` — **3/3** ready, имена Pod по ordinal, **3 PVC** с именами `data-<sts>-0/1/2`, все в `Bound`
- Одно предложение: какой `accessModes` взяли для `volumeClaimTemplates` и почему (`RWO` здесь верный — обоснуйте; `RWX` на local-path неверен)
- Одно предложение: **почему ваш `serviceName` совпадает с именем headless Service** из задания 3 и что сломается, если не совпадёт
- Нужный кусок `statefulset.yaml`, именно блок `volumeClaimTemplates`, чтобы проверяющий видел, как вы описали PVC на каждый Pod

---

## Задание 3 — Headless Service и DNS на каждый Pod (2 балла)

Здесь `clusterIP: None` отрабатывает свою роль. У обычного Service есть виртуальный IP: одна запись, kube-proxy балансирует. У **headless** Service IP нет вовсе. DNS отдаёт A-записи *каждого* подходящего Pod *плюс* записи вида `<pod>.<service>.<ns>.svc.cluster.local`. Именно вторая форма делает pod-0 адресуемым напрямую, так что pod-1 не может выдать себя за него.

### 3.1 — Напишите `templates/service-headless.yaml`

В скелете ниже только *kind* и `apiVersion`. Остальные поля ваши.

```yaml
# lab10-app/templates/service-headless.yaml
apiVersion: v1                                          # Services live in core v1
kind: Service
metadata:
  name: ___                                             # YOUR TASK: must equal spec.serviceName from your StatefulSet
                                                        #            (Task 2). Pick a name that makes the per-pod FQDN
                                                        #            readable, e.g. <chart>-headless.
  labels:
    {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  clusterIP: ___                                        # YOUR TASK: ONE specific value — the field is what makes this
                                                        #            Service "headless". A typo (e.g. an empty string,
                                                        #            or omitting the field) gives you a regular VIP
                                                        #            and silently defeats per-pod DNS.
  selector:
    ___: ___                                            # YOUR TASK: must match the StatefulSet pod template labels
                                                        #            EXACTLY — same label triangle as Task 2.
  ports:
    - ___                                               # YOUR TASK: the full port list — `name`, `port`, `targetPort`.
                                                        #            One entry pointing at your app's HTTP port. The
                                                        #            `port` field on a headless Service is informational
                                                        #            (no kube-proxy is involved), but you still need it
                                                        #            for DNS SRV records and for kubectl's port column.
```

**Почему каждый пропуск важен:**

- **`metadata.name`** — вторая половина FQDN. Ошибётесь в имени — `serviceName` в StatefulSet указывает в пустоту, DNS на каждый Pod сломан, а контроллер StatefulSet молчит (он не проверяет, что Service существует).
- **`spec.clusterIP`** — единственное поле, которым headless отличается от обычного. Опечатка, `""` или `192.x.x.x` — и Service либо не применится, либо получит VIP и начнёт балансировать. Это тихо ломает *весь смысл* лабораторной. **Допустимое значение ровно одно.**
- **`spec.selector`** — третий угол треугольника меток. Для отладки смотрите `kubectl get endpoints <headless-svc>`: там должны быть IP Pod вашего StatefulSet. Пустой список — опечатка в селекторе.
- **`spec.ports`** — порты всё равно объявляйте. Headless Service не балансирует, но kube-DNS берёт имя порта для SRV-записей (обнаружение сервисов внутри кластера), а `kubectl get svc` показывает `<none>` в ClusterIP, только если вы заодно забыли порт.

### 3.2 — Примените и проверьте DNS на каждый Pod

```bash
kubectl apply ...                                                # the `helm upgrade` from Task 2 already deployed it
kubectl get svc -n lab15                                         # YOUR TASK: confirm CLUSTER-IP column shows "None"
                                                                 #            for your headless Service
kubectl get endpoints <your-headless-svc> -n lab15               # YOUR TASK: MUST list 3 pod IPs (one per replica)
```

`ВАША ЗАДАЧА`: докажите, что DNS-запись на конкретный Pod реально резолвится. **В busybox и в тонком образе из Lab 2 может не быть `nslookup` и `dig`.** Возьмите один из вариантов ниже:

```bash
# Option A — getent (in glibc; works in most images, fails in busybox/alpine without bind-tools)
kubectl exec -n lab15 <pod-0> -- getent hosts <pod-1>.<headless>.lab15.svc.cluster.local

# Option B — throwaway debug pod with full networking tools
kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -n lab15 -- \
  nslookup <pod-1>.<headless>.lab15.svc.cluster.local
```

Ожидайте одну A-запись. Она совпадает с текущим IP `<pod-1>`, шаблон FQDN — `<pod>.<headless>.<ns>.svc.cluster.local`. У данного ordinal имя не меняется, даже если pod-1 уехал на другой узел.

### 3.3 — Посмотрите упорядоченный старт

```bash
kubectl get pods -n lab15 -w                       # in one terminal — watch ordinal-by-ordinal startup
kubectl scale statefulset <name> -n lab15 --replicas=1   # in another — terminates -2 then -1 (reverse)
kubectl scale statefulset <name> -n lab15 --replicas=3   # back up — brings -1 then -2 in order
```

`ВАША ЗАДАЧА`: в `LAB15.md` напишите 2–3 предложения про поведение `OrderedReady` по умолчанию, которое вы увидели, и один пример, когда уместен `podManagementPolicy: Parallel` (подсказка: gossip-кластеры в духе Cassandra, где все узлы равны; большинству баз нужен режим по умолчанию).

### 3.4 — Подтверждение

**Вставьте в `k8s/docs/LAB15.md`:**

- `kubectl get svc -n lab15`: headless Service с `CLUSTER-IP: None` рядом с обычным Service, чтобы проверяющий видел, что оба живут вместе
- `kubectl get endpoints <headless> -n lab15`: **3** IP Pod
- Вывод резолва DNS из §3.2, с явно указанным шаблоном FQDN
- Захват наблюдения за Pod из §3.3 (нужные строки: ordinal 0 в Ready до старта 1; при scale-down 2 завершается раньше 1)
- Ваши 2–3 предложения про `OrderedReady` и `Parallel`

---

## Задание 4 — Доказательство: PVC на каждый Pod (2 балла) ← **главное задание**

В этом весь смысл лабораторной. **Работающий StatefulSet — ещё не доказательство. Доказательство такое:** записать уникальный маркер в том `web-1`, удалить `web-1`, дождаться, пока контроллер создаст его заново, прочитать маркер у **нового** `web-1` — то же значение. Затем записать *другой* маркер в том `web-0` и показать, что маркеры не пересекаются: значение `web-0` — его собственное, не значение `web-1`.

### 4.1 — Запишите маркер на конкретный Pod

`ВАША ЗАДАЧА`: четыре шага придумайте сами. Каждый — это `kubectl exec` или `kubectl delete`. Манифесты не меняете.

```bash
# YOUR TASK: write a UNIQUE marker (epoch seconds, your name, anything distinctive) into
#            web-1's mounted volume — into /data/marker, on web-1 only
#
# YOUR TASK: kubectl delete pod <sts>-1 -n lab15
#            (the StatefulSet controller will immediately reschedule it — same name,
#             possibly different node, same PVC reattached)
#
# YOUR TASK: kubectl wait --for=condition=Ready pod/<sts>-1 -n lab15 --timeout=60s
#            (so the next exec doesn't race the pod's startup)
#
# YOUR TASK: read /data/marker from the NEW <sts>-1 — assert it MATCHES the value you wrote
```

Если маркер вернулся **тем же**, вы доказали главную гарантию: `data-<sts>-1` привязан к ordinal 1, а не к экземпляру Pod. Новый Pod получил PVC старого.

Если маркера **нет** (файл не найден) → `volumeClaimTemplates` не смонтировался в `DATA_DIR`, или вы случайно писали в `emptyDir` (имена томов не совпали — см. подсказки в задании 2).

Если маркер вернулся **другим или пустым** → вы не используете `volumeClaimTemplates`, а монтируете общий PVC дважды. Для RWO это неверно.

### 4.2 — Докажите, что маркеры не пересекаются

`ВАША ЗАДАЧА`: запишите **другой** маркер в том `web-0`. Затем прочитайте оба. Покажите, что у `web-0` свой маркер, а у `web-1` свой: они разные. (Это половина доказательства про *изоляцию* по Pod. Без неё вы показали только то, что PVC одного Pod пережил удаление, а не то, что Pod-N владеет своим PVC.)

```bash
# YOUR TASK: write a DIFFERENT marker into <sts>-0 's /data/marker
# YOUR TASK: read /data/marker from <sts>-0 AND from <sts>-1
#            They must be DIFFERENT — each pod's marker is its own.
```

### 4.3 — Ещё одна проверка: счётчик визитов

Счётчик `/visits` из Lab 12 теперь лежит на PVC конкретного Pod. Погоняйте трафик на каждый Pod отдельно и покажите, что счётчики *независимы*:

```bash
kubectl port-forward -n lab15 pod/<sts>-0 8080:8080 &
kubectl port-forward -n lab15 pod/<sts>-1 8081:8080 &
# YOUR TASK: curl localhost:8080/visits   — pod-0's counter only
# YOUR TASK: curl localhost:8081/visits   — pod-1's counter only
# YOUR TASK: hit pod-0 a few extra times; pod-1's count must NOT change
```

### 4.4 — Подтверждение

**Вставьте в `k8s/docs/LAB15.md` (это главный раздел, его нельзя пропустить):**

- **Захват из §4.1**, рядом: `kubectl exec <sts>-1 -- cat /data/marker` (исходный) → `kubectl delete pod <sts>-1` → `kubectl wait ...` → `kubectl exec <sts>-1 -- cat /data/marker` (новый Pod, **то же значение**)
- **Захват из §4.2**: маркеры обоих Pod, значения явно разные
- Расхождение счётчика визитов из §4.3 — 2–3 вывода curl: счётчик pod-0 не связан со счётчиком pod-1
- Одно предложение, **почему** это работает (PVC привязан к ordinal через `volumeClaimTemplates`, контроллер подключает его снова при повторном планировании)

---

## Задание 5 — Документация (1 балл)

Соберите всё в `k8s/docs/LAB15.md`. Обязательные разделы, по порядку:

1. **Понятия** — четыре ответа и таблица сравнения из задания 1
2. **Манифесты** — короткие абзацы про выборы в `statefulset.yaml` и `service-headless.yaml`: какой `accessModes`, какой storage class, какой `serviceName`, какой `clusterIP`. Назовите *конкретный отказ, который дал бы каждый неверный выбор*.
3. **Инвентарь ресурсов** — `kubectl get sts,pod,pvc,svc -n lab15`: StatefulSet, Pod по ordinal, PVC на каждый ordinal, оба Service (обычный и headless)
4. **Сетевая идентичность** — резолв DNS из задания 3.2 с объяснением шаблона FQDN, плюс наблюдение за упорядоченным стартом и scale-down
5. **Доказательство PVC на каждый Pod** — захваты из задания 4: запись-удаление-ожидание-чтение на `web-1` с тем же маркером, проверка, что `web-0` и `web-1` не пересекаются, плюс расхождение счётчика визитов
6. **Ловушки, на которые вы наступили** — хотя бы одна настоящая и как вы её отладили (строка `kubectl describe`, которая вас разблокировала, пропущенное поле, опечатка)

> Весь вывод `kubectl` и `curl` явно пометьте как снятый с вашего кластера. Не вставляйте вывод, который сами не получили.

---

## Бонус — Стратегии обновления для stateful-нагрузок (2 балла)

Обновлять primary в stateful-кластере *куда* важнее, чем подменять stateless-фронтенд. Цена автоматической ошибки — «восстановление из бэкапа». Kubernetes даёт две стратегии (`RollingUpdate` с `partition` и `OnDelete`). Экосистема даёт третью: операторы, которые обновляют за вас.

**Проблема:** нужно провести canary stateful-нагрузки. Сначала обновить *один* Pod, проверить, потом катить дальше. Наивный `RollingUpdate` у Deployment выкатывает новый образ на все реплики, человеческого шлюза там нет.

**Задание:** выберите **один** из трёх путей ниже и сделайте его. Каждый путь даёт все 2 балла. Не сдавайте YAML, скопированный из туториала. Разберите стратегию, решите, почему она, а не две другие, и запишите компромисс.

### Путь A — `RollingUpdate` с `partition`

Поле `partition` для StatefulSet — то же, что canary. Pod с ordinal `>= partition` получают новый шаблон Pod. Остальные остаются на старом. Чтобы катить обновление дальше, уменьшайте `partition`.

`ВАША ЗАДАЧА`:

1. Добавьте в StatefulSet `spec.updateStrategy`. Возьмите `type: RollingUpdate` и поставьте `rollingUpdate.partition` на **самый старший ordinal**, чтобы при смене образа обновился только верхний Pod.
2. Поменяйте `image.tag` в `values.yaml` на новую версию (пересоберите образ из Lab 2 со свежим тегом или на время смените `name:` контейнера — подойдёт что угодно, лишь бы хеш шаблона Pod изменился).
3. `helm upgrade ...`. Проверьте, что на новый шаблон перезапустился *только* Pod с наибольшим ordinal. Остальные остаются на старом.
4. Спустите `partition` вниз (от старшего → ... → 0). Каждый раз применяйте заново и между шагами снимайте, как идёт выкат.

YAML пишете вы. Скелета нет.

### Путь B — `OnDelete`

`type: OnDelete` — режим «при смене spec контроллер *ничего* не делает, выкат ведут люди». Так на самом деле выглядит большинство обновлений баз в проде.

`ВАША ЗАДАЧА`:

1. Добавьте в StatefulSet `spec.updateStrategy.type: OnDelete`.
2. Поменяйте `image.tag` и выполните `helm upgrade ...`. Проверьте, что **ни один Pod не перезапустился**. Контроллер специально не трогает Pod.
3. `kubectl delete pod <sts>-2` (или тот ordinal, который хотите обновить первым). Покажите, что на новый шаблон вернулся *только этот Pod*. Остальные сидят на старом, пока вы их тоже не удалите.
4. Опишите один реальный сценарий, где `OnDelete` — правильный ответ (подсказка: жёсткие окна изменений; согласованные обновления primary и replica, где человек между шагами смотрит лаг репликации).

YAML пишете вы. Скелета нет.

### Путь C — Оператор базы данных

Бросьте ручной StatefulSet для одной stateful-нагрузки и запустите её через оператор под эту задачу. Опишите, что оператор умеет такого, чего сырой StatefulSet не умеет.

`ВАША ЗАДАЧА`:

1. Поставьте **один** оператор: **CloudNativePG** (PostgreSQL), **Strimzi** (Kafka) или оператор хранилища вроде **Rook** (Ceph) / **Longhorn**.
2. Опишите Custom Resource (например CloudNativePG `kind: Cluster` с `instances: 3`).
3. Убедитесь, что оператор *сам* создаёт нижележащие StatefulSet, Service и PVC. После применения CR выполните `kubectl get sts,svc,pvc -n <op-ns>`.
4. В `LAB15.md` опишите, что он добавляет: автоматические выборы primary и failover, непрерывные бэкапы WAL или топиков, согласованные rolling-обновления, готовые метрики `PodMonitor`. Ничего из этого сырой StatefulSet не даёт.

### Подтверждение (какой бы путь вы ни выбрали)

**Вставьте в `k8s/docs/LAB15.md`:**

- Фрагмент обновлённого манифеста (ваш блок `updateStrategy` или YAML CR для пути C)
- Захват `kubectl rollout status` / `kubectl get pods -w` с частичным выкатом (пути A и B) или `kubectl get sts,svc,pvc` под управлением оператора (путь C)
- **Один абзац**, *почему* этот путь, а не два других: компромиссы, что приложению `visits` на самом деле нужно и каким продовым инструментом вы занялись бы дальше

---

## Как сдать

```bash
git switch -c lab15
git add lab10-app/ k8s/docs/LAB15.md
git commit -m "feat(lab15): convert chart to StatefulSet with per-pod PVCs + headless Service"
git push -u origin lab15
```

Откройте **два** PR:

- `your-fork:lab15` → `course-repo:master` *(на ревью)*
- `your-fork:lab15` → `your-fork:master` *(вливается в ваш main)*

Чеклист PR:

```text
- [ ] Task 1 done — concepts, comparison table, operator rule-of-thumb in LAB15.md
- [ ] Task 2 done — StatefulSet with serviceName + replicas + label triangle + volumeClaimTemplates, 3 ordinal pods + 3 Bound per-pod PVCs
- [ ] Task 3 done — headless Service with clusterIP: None, per-pod DNS resolution captured, ordered startup observed
- [ ] Task 4 done — write→delete-pod→read SAME marker on web-1; web-0 marker is its own; visits counters independent per pod
- [ ] Task 5 done — LAB15.md with all 6 sections + real evidence
- [ ] Bonus done — Path A (partition) OR Path B (OnDelete) OR Path C (operator) with justification
```

---

## Критерии приёмки

### Задание 1 — Понятия (2 балла)
- ✅ Три гарантии StatefulSet описаны своими словами
- ✅ Два конкретных режима отказа Deployment для Postgres объяснены
- ✅ Поведение DNS у headless Service (`clusterIP: None`) объяснено
- ✅ Есть заметка «когда брать оператор»
- ✅ Таблица сравнения заполнена (все пять строк)

### Задание 2 — Перевод в StatefulSet (3 балла)
- ✅ `templates/statefulset.yaml` написан вручную
- ✅ `spec.serviceName` задан и совпадает с именем headless Service из задания 3
- ✅ Треугольник меток согласован (selector.matchLabels = метки шаблона = селектор headless Service)
- ✅ `replicas: 3` (или обоснованно больше); Pod называются `<name>-0/1/2`
- ✅ Блок `volumeClaimTemplates` написан; PVC на каждый Pod рендерятся как `data-<name>-N`, и **все три в Bound**
- ✅ `volumeMounts[].name` контейнера совпадает с `volumeClaimTemplates[].metadata.name`
- ✅ Старый `deployment.yaml` закрыт через values, а не удалён

### Задание 3 — Headless Service и жизненный цикл (2 балла)
- ✅ `templates/service-headless.yaml` написан вручную
- ✅ Стоит `spec.clusterIP: None` (не пропущен, не "")
- ✅ Селектор точно совпадает с метками Pod StatefulSet
- ✅ Обычный Service (Lab 9) живёт рядом и не удалён
- ✅ Резолв DNS на каждый Pod снят, шаблон FQDN виден
- ✅ Наблюдали упорядоченный старт и scale-down в обратном порядке

### Задание 4 — Доказательство сохранности PVC (2 балла)
- ✅ **Маркер `web-1` переживает `kubectl delete pod web-1`** — то же значение до и после, захваты рядом
- ✅ **Маркер `web-0` — его собственный** и отличается от маркера `web-1`
- ✅ Счётчики визитов независимы по Pod
- ✅ Одно предложение «почему это работает» со ссылкой на `volumeClaimTemplates` и привязку к ordinal

### Задание 5 — документация (1 балл)
- ✅ Все 6 разделов в `k8s/docs/LAB15.md`; настоящие захваты CLI (не иллюстрации)
- ✅ В разделе про ловушки есть хотя бы одна настоящая, на которую вы наступили

### Бонус — стратегии обновления / оператор (2 балла)
- ✅ Реализован ровно один путь (A / B / C)
- ✅ YAML написан вами, а не скопирован из туториала
- ✅ Захваты соответствуют пути (уменьшение partition, ручное удаление или список ресурсов под оператором)
- ✅ Абзац, *почему этот путь, а не два других*

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Понятия | **2** | Гарантии, таблица сравнения, DNS headless, зачем оператор |
| **Задание 2** — StatefulSet | **3** | Ручной `statefulset.yaml`: serviceName, треугольник меток, volumeClaimTemplates; 3 Pod по ordinal и 3 PVC в Bound |
| **Задание 3** — Headless Service и жизненный цикл | **2** | Ручной headless Service (`clusterIP: None`); DNS на каждый Pod доказан; упорядоченный старт наблюдался |
| **Задание 4** — Доказательство сохранности | **2** | Маркер `web-1` переживает delete; `web-0` ≠ `web-1`; изоляция по Pod доказана |
| **Задание 5** — Документация | **1** | Шесть разделов в `LAB15.md` с настоящими доказательствами |
| **Бонус** — Стратегии обновления / оператор | **2** | Один из трёх путей сделан и обоснован письменно |
| **Итого** | **12** | 10 основных + 2 бонус |

**Шкала (основные 10 баллов):**

- **10/10:** все ресурсы верны, привязка PVC к Pod доказана на `web-1`, на `web-0` показано, что данные не пересекаются, документация отличная
- **8-9/10:** работает от начала до конца; мелкие пробелы в доказательствах или одно слабое объяснение
- **6-7/10:** StatefulSet выкатывается, но доказательство сохранности тонкое (только одна половина: переживает удаление или изоляция по Pod, но не обе)
- **<6/10:** нет headless Service, сломан `volumeClaimTemplates` или нет доказательства сохранности

---

## Материалы

<details>
<summary>📚 Документация Kubernetes</summary>

- [StatefulSet — понятия](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Туториал StatefulSet Basics](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
- [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
- [Шаблоны volume claim](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#volume-claim-templates)
- [Стратегии обновления и выкат через partition](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies)
- [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [DNS для Service и Pod (записи на каждый Pod)](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pods)

</details>

<details>
<summary>🤖 Операторы и cloud-native хранилище</summary>

- [Паттерн Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) · [OperatorHub.io](https://operatorhub.io/)
- [CloudNativePG](https://cloudnative-pg.io/) — оператор PostgreSQL (1.29, по умолчанию PG 18.3)
- [Strimzi](https://strimzi.io/) — Kafka на Kubernetes
- [Rook](https://rook.io/) — продовое хранилище на Ceph (v1.18 + Ceph 19 Squid)
- [Longhorn](https://longhorn.io/) — лёгкое блочное хранилище CNCF (1.9)

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных пробных прогонов — прочитайте ДО отладки)</summary>

- **Пропущен или неверен `spec.serviceName`.** Контроллер StatefulSet **не** валит apply. Он спокойно создаёт Pod с порядковыми именами, но DNS на каждый Pod тихо ломается: FQDN `<pod>.<serviceName>` не указывает ни на один Service. Симптом: Pod в Ready, а `kubectl exec web-0 -- getent hosts web-1.<wrong>` ничего не возвращает. **Всегда применяйте StatefulSet вместе с парным headless Service и перед коммитом grep-ом проверьте, что имена в обоих файлах совпадают.**

- **Опечатка в `clusterIP: None` или поле пропущено.** У обычного Service с `clusterIP: <some-vip>` та же форма, что у headless, кроме этого одного поля. Уберёте его или поставите `""` — получите обычный VIP, который балансирует. Это тихо ломает DNS на каждый Pod, то есть весь смысл лабораторной. Допустимое значение **ровно одно**: литерал `None`.

- **PVC из `volumeClaimTemplates` остаются после удаления StatefulSet — ловушка со счётом.** Так задумано: данные переживают ошибки. Следствие: `helm uninstall lab10-app -n lab15` оставляет три PVC. На облачном StorageClass за них капают деньги, а владельца не видно. Снесите их явно: `kubectl delete pvc -n lab15 -l app.kubernetes.io/name=lab10-app`. **Та же ловушка, когда чините неизменяемый блок `volumeClaimTemplates`. `helm uninstall` PVC не стирает. Следующий `helm install` может подхватить claim, которые не совпадают с шаблоном.**

- **Упорядоченный старт блокирует rolling update, если один Pod не проходит liveness.** По умолчанию `podManagementPolicy: OrderedReady`: Pod-N ждёт Ready у Pod-(N-1), и только потом стартует. Зависший pod-0 — и Pod 1, 2, ... так и не стартуют. Снаружи это «StatefulSet застрял». Причина **всегда** в логах и events pod-0. Не хватайтесь за `Parallel`, чтобы обойти поломку. Чините pod-0. `Parallel` — только если приложение находит соседей через gossip и все узлы равны (как в Cassandra).

- **В busybox и тонких образах нет `nslookup` и `dig`.** В Python-образе из Lab 2 их, скорее всего, тоже нет. Два чистых способа доказать DNS на каждый Pod, не раздувая образ приложения:
  - `kubectl exec <pod> -- getent hosts <fqdn>` — сработает, если в образе есть glibc (у большинства баз не на busybox она есть)
  - `kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -- nslookup <fqdn>` — одноразовый отладочный Pod с полным набором утилит

- **`volumeClaimTemplates` после создания не меняется.** Слайд 19 Lecture 15 называет это ловушкой изменения размера. Правите блок в чарте, делаете `helm upgrade` — контроллер StatefulSet молча пропускает изменение. Чтобы реально увеличить том: `kubectl edit pvc data-<sts>-N` для каждого ordinal, и только если у StorageClass стоит `allowVolumeExpansion: true` (дефолтный `local-path` в k3d это *умеет*; многие облачные SC тоже). Для более глубоких правок (storage class, access mode) сделайте `kubectl delete sts --cascade=orphan` и примените шаблон заново. PVC переживают удаление с orphan.

- **Разъезд треугольника меток.** Три места держите в синхроне: `StatefulSet.spec.selector.matchLabels`, `StatefulSet.spec.template.metadata.labels` и `Service.spec.selector` (и у обычного Service, и у headless). Селектор **после создания не меняется**. Ошиблись: `helm uninstall`, `kubectl delete pvc -l ...`, затем `helm install`. `kubectl edit` не спасёт.

- **Путаница `serviceName` с обычным Service.** Поле `serviceName` у StatefulSet указывает на **headless** Service, а не на обычный ClusterIP Service из Lab 9. Это разные Service. Селектор у каждого свой, и оба селектора закрывают одни и те же Pod. Обычный Service — чтобы из любой точки кластера сделать `curl http://web/` (с балансировкой). Headless Service — чтобы `curl http://web-0.web-headless/` попал в конкретный Pod.

</details>

<details>
<summary>🛠️ Полезный CLI (без YAML — только рецепты)</summary>

```bash
# Inspect what the cluster offers before you write YAML
kubectl get storageclass
kubectl get sc local-path -o yaml | grep -E 'provisioner|reclaimPolicy|volumeBindingMode|allowVolumeExpansion'

# The four together tell the whole StatefulSet story
kubectl get sts,pod,pvc,svc -n lab15

# Confirm per-pod PVC binding (the headline)
kubectl get pvc -n lab15 -l app.kubernetes.io/name=lab10-app
# Expect: data-<sts>-0, data-<sts>-1, data-<sts>-2 — all Bound

# Per-pod DNS without nslookup
kubectl exec -n lab15 <sts>-0 -- getent hosts <sts>-1.<headless>.lab15.svc.cluster.local
# Or with full tooling:
kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -n lab15 -- bash

# Persistence proof (Task 4)
kubectl exec -n lab15 <sts>-1 -- sh -c 'echo "MARKER-$(date +%s)" > /data/marker'
kubectl exec -n lab15 <sts>-1 -- cat /data/marker
kubectl delete pod -n lab15 <sts>-1
kubectl wait --for=condition=Ready pod/<sts>-1 -n lab15 --timeout=60s
kubectl exec -n lab15 <sts>-1 -- cat /data/marker   # MUST match the value above
```

</details>

---

## Что дальше

| Lab | Что это добавляет в стек |
|---:|---|
| 16 | **kube-prometheus-stack** — снять метрики с каждого Pod вашего StatefulSet через `ServiceMonitor` |
| 17 *(бонус)* | **Cloudflare Workers** — когда кластер избыточен: изоляты V8, мгновенный холодный старт |
| 18 *(бонус)* | **Nix** — воспроизводимые сборки и dev shell без единого контейнера |

---

**Удачи!** 💾

> **Запомните:** StatefulSet — это *управляемые питомцы*: стабильная идентичность и диск на каждый Pod. Вся лабораторная сводится к одному захвату. Запишите маркер в `web-1`, удалите `web-1`, прочитайте маркер у нового `web-1` — то же значение, потому что `data-web-1` был привязан к ordinal 1, а не к Pod, который там жил. Если из лабораторной выживет только один захват, пусть это будет он.
