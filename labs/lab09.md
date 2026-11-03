# Lab 9 — Основы Kubernetes на k3d

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Kubernetes-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Kubernetes%201.36%20%7C%20k3d-informational)

> **Цель:** поднять локальный кластер k3d (1 server + 2 agents), выкатить **два** сервиса рядом — ваш образ Python из Lab 2 и служебный Go-сервис `echo` курса — и доказать, что они ходят друг к другу через `Service` и kube-DNS.
> **Что сдать:** PR из `lab09`, который добавляет `k8s/` (манифесты Deployment и Service для обоих сервисов плюс бонусный Ingress) и `k8s/docs/LAB09.md` со снятыми выводами.

---

## Обзор

В этой лабораторной вы потренируетесь:
- Управлять кластером командами `kubectl get / describe / logs / exec / apply / rollout`
- **Писать манифесты Deployment и Service прямо по API** — выбрать верный `apiVersion/kind`, собрать треугольник меток и селекторов, задать requests/limits, повесить probe (проверку живости и готовности) на дешёвый эндпоинт
- Читать сетевую модель K8s: виртуальный IP `Service` → короткое имя kube-DNS → список endpoints → IP Pod'ов
- Импортировать локально собранные образы в containerd кластера k3d
- Масштабировать `Deployment`, выкатывать новую версию и откатывать её

> ⚠️ **Границы:** только обычный YAML. Helm будет в Lab 10. Пока без persistent volumes, без secrets и без сбора метрик: это Labs 11, 12 и 16. Не лезьте в `kubectl edit` и ничего не шаблонизируйте. Сначала выучите сырой API.

> 🪨 **Педагогическое ядро.** До Lab 8 у вас работал **один** сервис. С одним сервисом на вопрос «зачем стабильный IP и DNS-имя?» честно ответить трудно. Здесь сервисов **два**. Изнутри Pod'а `web` вы выполните `curl http://echo/ping` и получите `pong`. Один запрос туда-обратно сразу проверяет kube-DNS, селекторы меток и балансировку Service. **Если снимете только один вывод, снимайте этот.**

---

## Состояние проекта

**К этому моменту у вас должно быть:**
- `app_python/` из Lab 1 — сервис на Flask/FastAPI с `/` и `/health`
- Рабочий образ контейнера из Lab 2 (запушен в GHCR или Docker Hub, **или** собран локально и протегирован)
- Опыт с `docker compose up` (Labs 6–8): вы уже упёрлись в его потолок

**Эта лабораторная добавляет:**
- `k8s/web-deployment.yaml` — **вы пишете** (Deployment для образа из Lab 2)
- `k8s/web-service.yaml` — **вы пишете** (Service перед `web`)
- `k8s/echo-deployment.yaml` — **вы пишете** (Deployment для служебного `echo` курса)
- `k8s/echo-service.yaml` — **вы пишете** (ClusterIP Service для `echo`)
- `k8s/ingress.yaml` — **вы пишете** (Ingress через Traefik, только бонус)
- `k8s/docs/LAB09.md` — ваш отчёт к сдаче

К Lab 10 вы упакуете всё это в один Helm-чарт на **Helm 4**; к Lab 13 ArgoCD выкатит их через `ApplicationSet`. Метки и селекторы должны быть верными *уже сейчас*: дальше они расползутся везде.

Служебный код курса для этой лабораторной:
- `plumbing/echo/` — Go HTTP-сервис, который вы выкатите в Задании 3. **Не меняйте его.** Готовый образ публикует CI курса в `ghcr.io/inno-devops-labs/echo:v1`. Список эндпоинтов — в `plumbing/echo/README.md`.

---

## Подготовка

Версии в этой лабораторной. Фиксируйте их там, где выбор за вами:

| Компонент | Версия | Релиз |
|---|---|---|
| Kubernetes (k3s) | `v1.36.1-k3s1` | 22 апреля 2026 (1.36 «Haru») |
| k3d | 5.7+ | — |
| kubectl | 1.36.x (вровень с кластером) | — |
| образ служебного `echo` | `ghcr.io/inno-devops-labs/echo:v1` | публикует CI курса |

```bash
docker --version           # 28.x or 29.x — k3d runs k3s inside Docker
k3d version                # 5.7+
kubectl version --client   # 1.36.x — match the cluster you'll create
```

> ⚠️ **Не берите Kubernetes, вшитый в Docker Desktop.** Он отстаёт от upstream, и посреди лабораторной вылезет version skew.

Создайте дерево каталогов (файлы заполните сами):

```
k8s/
├── web-deployment.yaml          # YOU write this (§2)
├── web-service.yaml             # YOU write this (§2)
├── echo-deployment.yaml         # YOU write this (§3)
├── echo-service.yaml            # YOU write this (§3)
├── ingress.yaml                 # YOU write this (bonus only)
└── docs/
    └── LAB09.md                 # your submission report
```

---

## Задание 1 — Локальный кластер k3d (2 балла)

### 1.1 — Почему k3d, а не minikube / kind / Docker Desktop

В `docs/LAB09.md` ответьте на каждый вопрос в 2–3 предложениях:

1. Что k3d на самом деле запускает внутри Docker и чем это отличается от minikube, который поднимает виртуалку?
2. Зачем этой лабораторной **несколько нод** (`--agents 2`)? Что видно на 3 нодах и не видно на одной?
3. Какой ingress-контроллер и какую реализацию LoadBalancer k3d привозит по умолчанию и что изменится в бонусном Ingress, если заменить k3d на kind?

(Это есть на слайде 13 Lecture 9 и в [документации k3d](https://k3d.io/).)

### 1.2 — Создайте кластер

`ВАША ЗАДАЧА`: поднимите кластер с именем `devops`, **1 server + 2 agents**, Kubernetes **1.36.1**. Пробросьте порты LoadBalancer `80→8080` и `443→8443` на хост, чтобы бонусный Ingress открывался.

```bash
# YOUR TASK lines below:
#   --image:  which k3s tag = K8s 1.36.1?  (see hint after the block)
#   --agents: how many agents for a multi-node cluster?
#   first -p: host port that maps cluster :80
#   second -p: host port that maps cluster :443
k3d cluster create devops \
  --image rancher/k3s:___-k3s1 \
  --agents ___ \
  -p "___:80@loadbalancer" \
  -p "___:443@loadbalancer"
```

> 💡 Теги образов k3s **дописывают** `-k3s1` к версии upstream K8s. Возьмите самый свежий тег `v1.36.x-k3s1` у [`rancher/k3s` на Docker Hub](https://hub.docker.com/r/rancher/k3s/tags).

### 1.3 — Проверьте

```bash
kubectl config use-context k3d-devops    # k3d sets this automatically — sanity-check
kubectl version                          # server MUST report 1.36.x; client can be ±1 minor (skew is OK)
kubectl cluster-info
kubectl get nodes -o wide                # YOUR TASK: expect exactly THREE nodes — 1 server + 2 agents
```

> 💡 Если `kubectl` печатает `WARNING: version difference between client (1.34) and server (1.36) exceeds the supported version skew`, это предупреждение, а не ошибка: лабораторная всё равно засчитается. Чтобы заглушить его, поставьте kubectl 1.36.x вровень с кластером.

### 1.4 — Доказательство работы

**Вставьте в `docs/LAB09.md`:**

- Ответы на три вопроса из §1.1
- `kubectl version`, где client и server показывают `1.36.x`
- `kubectl get nodes -o wide` с **3** нодами, все `Ready`, все `v1.36.1+k3s1`
- Вашу команду `k3d cluster create` (чтобы проверяющий смог поднять ваше окружение заново)

---

## Задание 2 — Выкатите сервис `web` (3 балла)

Здесь навык — писать манифесты. Каркас ниже показывает, какие поля требует `Deployment`. **Значения** выводите сами из API K8s и из работы Lab 1 и Lab 2.

### 2.1 — Deployment

`ВАША ЗАДАЧА`: напишите `k8s/web-deployment.yaml`. Три набора меток и селекторов в этом файле должны совпасть (metadata у Deployment, `spec.selector.matchLabels` и `spec.template.metadata.labels`). Этот треугольник меток — ошибка №1 в YAML для K8s. Service из §2.2 выберет ту же метку.

```yaml
# k8s/web-deployment.yaml
apiVersion: ___                       # YOUR TASK: which apiGroup/version owns Deployment?
kind: ___                             # YOUR TASK
metadata:
  name: ___                           # YOUR TASK: lab's headline name
  labels:
    ___: ___                          # YOUR TASK: the label the Service selector will match
spec:
  replicas: ___                       # YOUR TASK: how many web pods? (lecture 9 + Task 4 scale)
  selector:
    matchLabels:
      ___: ___                        # MUST match the pod template labels below
  strategy:
    type: ___                         # YOUR TASK: which strategy for zero-downtime updates?
    rollingUpdate:
      maxUnavailable: ___             # YOUR TASK: 0 = strict zero-downtime
      maxSurge: ___
  template:
    metadata:
      labels:
        ___: ___                      # MUST match selector.matchLabels EXACTLY
    spec:
      containers:
        - name: ___                   # YOUR TASK
          image: ___                  # YOUR TASK: your Lab 2 image. Two valid forms:
                                      #   - GHCR pull: ghcr.io/<you>/devops-info-service:1.0.0  (public package — Lab 2 Task 7)
                                      #   - Local tag: devops-info-service:lab02-multi          (must `k3d image import` — see §2.3)
          imagePullPolicy: ___        # YOUR TASK: which policy lets a k3d-imported image work?
                                      #            (hint: see Setup note about local images + Common Pitfalls)
          ports:
            - name: ___               # YOUR TASK: name this port (e.g. `http`). Lab 10's Helm chart
                                      #            and Lab 16's ServiceMonitor both target the port BY NAME.
                                      #            Pick a name now or rewrite both labs later.
              containerPort: ___      # YOUR TASK: the port your Lab 1 app listens on
              protocol: TCP
          resources:
            requests:
              cpu: ___                # YOUR TASK: scheduler reserves this — see lecture 9 slide 15
              memory: ___
            limits:
              cpu: ___                # YOUR TASK
              memory: ___             # YOUR TASK: memory-OOMKill ceiling — set tightly
          livenessProbe:
            httpGet:
              path: ___               # YOUR TASK: which endpoint from Lab 1? (hint: cheap, not the app root)
              port: ___
            initialDelaySeconds: ___  # YOUR TASK
            periodSeconds: ___
          readinessProbe:
            httpGet:
              path: ___               # YOUR TASK
              port: ___
            periodSeconds: ___        # YOUR TASK: fail fast — readiness gates traffic
```

**Почему важен каждый пропуск (прочитайте, прежде чем заполнять):**

- **`apiVersion` / `kind`** — Deployment живёт в `apps/v1` (*единственная* GA-группа и версия для него). `Pod` — это `v1`, `Service` — это `v1`. Выучите таблицу: к Lab 13 будете писать её руками, не глядя.
- **Треугольник меток.** `metadata.labels`, `spec.selector.matchLabels` и `spec.template.metadata.labels` — три *отдельных* словаря, и во всех нужна одна и та же пара ключ/значение. Селектор **не меняется после создания**: ошиблись в ключе — делайте `kubectl delete` и apply заново.
- **`imagePullPolicy`.** `Always` тянет образ заново на каждом рестарте и **на k3d падает для локально собранных образов**: в registry их нет. `IfNotPresent` берёт то, что уже лежит в containerd (туда как раз кладёт `k3d image import`). `Never` — вариант для параноиков. Выбирайте сознательно.
- **`resources.requests`.** Их читает **планировщик**, чтобы выбрать ноду. Занизите — шумные соседи отберут у вас ресурсы. Завысите — Pod'ы встанут в `Pending`: ни на одной ноде нет запаса.
- **`resources.limits.memory`.** Выше этой черты ядро убивает контейнер по OOM. Ставьте плотно: утечка должна проявиться рестартом, а не смертью всей ноды в swap. Лимиты **CPU** дают тихий throttling. Здесь оставляйте запас.
- **Probe.** Упавший `livenessProbe` = контейнер перезапущен. Упавший `readinessProbe` = Pod вынут из списка endpoints у Service (без рестарта). Оба на корне `/` приложения — самая частая ловушка в проде (медленная страница = цикл рестартов = простой). Берите `/health` или `/healthz`, который отдаёт 200 и не ходит в БД.
- **`strategy.rollingUpdate.maxUnavailable: 0`** как раз гарантирует нулевой простой в Задании 4. По умолчанию (25%) во время обновления одного из трёх Pod'ов просто нет.

### 2.2 — Service

`ВАША ЗАДАЧА`: напишите `k8s/web-service.yaml`. Выберите **тип Service**, при котором приложение открывается с ноутбука без port-forward (таблица на слайде 9 Lecture 9; на k3d это вариант, удобный для разработки).

```yaml
# k8s/web-service.yaml
apiVersion: ___                       # YOUR TASK
kind: ___                             # YOUR TASK
metadata:
  name: ___                           # YOUR TASK: kube-DNS will resolve THIS name inside the cluster
spec:
  type: ___                           # YOUR TASK: see lecture 9 slide 9 — pick the one that exposes :NodePort on every node
  selector:
    ___: ___                          # MUST match the Deployment pod labels — kube-proxy uses this to build endpoints
  ports:
    - name: ___                       # YOUR TASK: same name you gave the containerPort (e.g. `http`).
                                      #            Lab 16's ServiceMonitor uses this string, not the number.
      port: ___                       # YOUR TASK: the port the Service listens on (cluster-side)
      targetPort: ___                 # YOUR TASK: prefer the port NAME (e.g. `http`) over a number —
                                      #            keeps the Service stable when the container port changes
      protocol: TCP
      # nodePort: ___                 # OPTIONAL: pin it (30000–32767), or let K8s pick
```

**Почему важен каждый пропуск:**

- **`name` у Service** — это имя резолвит kube-DNS внутри кластера. `curl http://web/` из любого другого Pod'а попадает в этот Service. Имя короткое и стабильное: переименовать Service — ловушка, DNS-клиенты кэшируют.
- **`type`.** `ClusterIP` (по умолчанию) виден только внутри. `NodePort` открывает высокий порт на каждой ноде. `LoadBalancer` заказывает облачный балансировщик (на k3d это klipper-lb). Для локальной разработки NodePort работает, когда порт хоста уже проброшен. `LoadBalancer` на k3d тоже работает благодаря klipper. Выберите один и обоснуйте.
- **`port` и `targetPort`.** `port` — куда *клиенты* стучатся на IP Service. `targetPort` — на каком порту слушает **контейнер**. Перепутали — endpoints есть, а связи нет. Не ставьте их равными по привычке: это разные вещи.
- **Именованные порты.** `ports[].name: http` (и `targetPort: http`, который ссылается на `ports[].name` контейнера) — **не украшение**. В Lab 16 поле `ServiceMonitor.spec.endpoints[].port` — это **строка** (*имя* порта), а не число. Голый числовой порт молча не резолвится, и оператор пропускает ваш сервис. Задайте имя сейчас, иначе этот Service придётся переписывать.
- **`selector`.** Должен точно совпасть с метками Pod'а. `kubectl get endpoints web` — главная подсказка при отладке: список пуст, значит селектор не попал ни в один Pod.

### 2.3 — Загрузите образ и примените

**Два пути.** Выбирайте по тому, лежит ли образ из Lab 2 публично на GHCR:

**Путь A — GHCR (рекомендуется).** Ваш `ghcr.io/<you>/devops-info-service:1.0.0` публичный (Lab 2, §7). containerd в k3d тянет его сам. **`k3d image import` не нужен.** Переходите к `kubectl apply`.

**Путь B — только локальная сборка.** Образ есть только в Docker на ноутбуке (`devops-info-service:lab02-multi` или похожий). containerd в k3d про него не знает. Его **нужно** импортировать:

```bash
docker images | grep devops-info-service         # confirm the tag exists locally
k3d image import ___ --cluster devops            # YOUR TASK: paste the exact `<name>:<tag>` from above
```

> 💡 Ноды k3d запускают k3s в собственных контейнерах, у каждого свой containerd. Демон Docker на ноутбуке им не виден. `k3d image import` — мост через этот разрыв (он делает `docker save | docker load` в каждую ноду k3d). Если `kubectl get pods` показывает `ErrImagePull`/`ImagePullBackOff`, а тег точно верный, вы пропустили этот шаг.

Когда образ доступен любым из путей:

```bash
kubectl apply -f k8s/web-deployment.yaml
kubectl apply -f k8s/web-service.yaml

kubectl rollout status deploy/web              # blocks until 3/3 ready
kubectl get pods -l app=web -o wide            # YOUR TASK: confirm pods spread across all 3 nodes
kubectl get endpoints web                      # MUST list 3 pod IPs once readiness passes
```

> 💡 **Лестница отладки, если с первого раза не взлетело:**
> 1. `kubectl get pods` — `ImagePullBackOff` значит кривой registry, креды или `imagePullPolicy`; `CrashLoopBackOff` значит контейнер завершается, смотрите логи.
> 2. `kubectl describe pod <name>` — внизу, в секции **Events**, настоящая причина.
> 3. `kubectl logs <pod-name>` — stdout вашего приложения.
> 4. `kubectl get endpoints web` — пустой список значит, что селектор не совпал с метками (треугольник меток сломан).

### 2.4 — Доказательство работы

**Вставьте в `docs/LAB09.md`:**

- `kubectl get deploy,rs,pods,svc,endpoints -l app=web` — должно быть **3/3** ready и **3 IP в endpoints**
- `kubectl get pods -l app=web -o wide` — в колонке **NODE** должны встретиться все 3 ноды (это доказательство планирования на несколько нод и половина причины, зачем вы ставили `--agents 2`)
- Вывод `kubectl port-forward svc/web 8080:80` плюс `curl -s localhost:8080/health`: Service смотрит на живое приложение
- Выбранный `imagePullPolicy` и одно предложение, зачем он
- Ваши requests/limits и одно предложение, зачем такие

---

## Задание 3 — Выкатите служебный сервис `echo` (2 балла)

Второй сервис. Контракт эндпоинтов читайте в `plumbing/echo/README.md`. Этот код вы **не** собираете и **не** правите.

| Путь | Поведение |
|---|---|
| `GET /ping` | Возвращает `pong\n` — главный smoke-тест в Задании 4 |
| `* /echo` | JSON с `hostname` и счётчиком запросов — по нему видно балансировку |
| `GET /healthz` | Возвращает `ok` (200) — сюда вешайте probe |
| `GET /metrics` | Формат Prometheus (скрейпите это в Lab 16) |

Образ — `ghcr.io/inno-devops-labs/echo:v1`. Слушает **порт 8081**.

### 3.1 — Deployment для echo

`ВАША ЗАДАЧА`: напишите `k8s/echo-deployment.yaml`. Та же форма, что у `web`, но другие метки, другой образ, другой порт и другое число реплик. **2 реплики** не обсуждаются: в Задании 4 Service должен крутить round-robin по **двум** Pod'ам.

```yaml
# k8s/echo-deployment.yaml
apiVersion: ___                       # YOUR TASK
kind: ___                             # YOUR TASK
metadata:
  name: ___                           # YOUR TASK
  labels:
    ___: ___                          # YOUR TASK: must be DIFFERENT from web's label value
spec:
  replicas: ___                       # YOUR TASK: read the prose above — why two?
  selector:
    matchLabels:
      ___: ___
  template:
    metadata:
      labels:
        ___: ___
    spec:
      containers:
        - name: ___
          image: ___                  # YOUR TASK: the published echo image — see plumbing/echo/README.md
          ports:
            - name: ___               # YOUR TASK: name the port (e.g. `http`). Same reason as the web Deployment.
              containerPort: ___      # YOUR TASK: see echo README — what port?
              protocol: TCP
          resources:
            requests: { cpu: ___, memory: ___ }    # YOUR TASK: echo is a tiny Go binary
            limits:   { cpu: ___, memory: ___ }
          readinessProbe:
            httpGet:
              path: ___               # YOUR TASK: echo's readiness endpoint — README has it
              port: ___
            periodSeconds: ___
          livenessProbe:
            httpGet:
              path: ___
              port: ___
            initialDelaySeconds: ___
            periodSeconds: ___
```

> 💡 **Бинарники Go крошечные.** Ставьте `requests.memory` в единицы MiB, а `requests.cpu` примерно 10–50m. Скопируете цифры `web` — зарезервируете слишком много ёмкости кластера, и Pod'ы уйдут в `Pending`.

### 3.2 — Service для echo

`ВАША ЗАДАЧА`: напишите `k8s/echo-service.yaml`. Здесь тип `ClusterIP` (только внутри: его зовёт Pod `web`, с ноутбука ходить не нужно). Межсервисный вызов в §4.1 пойдёт на `http://echo:___/ping`. Выберите `port` Service так, чтобы URL был чистым (соглашение — на слайде 9 Lecture 9).

```yaml
# k8s/echo-service.yaml
apiVersion: ___
kind: ___
metadata:
  name: ___                           # YOUR TASK: the DNS name `web` will use to reach echo
spec:
  type: ___                           # YOUR TASK: internal-only — which type?
  selector:
    ___: ___                          # MUST match echo pod labels (NOT web's)
  ports:
    - name: ___                       # YOUR TASK: same name you gave the containerPort
      port: ___                       # YOUR TASK: HTTP-default port, so `http://echo/ping` works with no `:port`
      targetPort: ___                 # YOUR TASK: prefer the port NAME (string) over the container port number
      protocol: TCP
```

> 💡 **Зачем у Service `port: 80`?** Потому что `http://echo/ping` (без порта) по умолчанию идёт на 80. Поставите `port: 8081` — вызывающим придётся писать `http://echo:8081/ping`. Так работает, но лекция, документация и будущий чарт считают чистую форму. Порт Service и порт контейнера **развязаны**. В этом весь смысл.

### 3.3 — Примените и проверьте

```bash
kubectl apply -f k8s/echo-deployment.yaml
kubectl apply -f k8s/echo-service.yaml

kubectl get pods,svc,endpoints -l app=echo     # 2 pods Ready, 2 endpoint IPs
kubectl rollout status deploy/echo
```

### 3.4 — Доказательство работы

**Вставьте в `docs/LAB09.md`:**

- `kubectl get pods,svc,endpoints -l app=echo` — **2** Pod'а в Ready, у Service **2** endpoints
- Один вывод `kubectl describe svc echo` со строками `Selector` и `Endpoints`
- 1–2 предложения, **почему `requests` у echo меньше, чем у `web`**: аргумент про крошечный бинарник Go

---

## Задание 4 — Сеть между сервисами, масштаб, rolling update (2 балла) ← **главное задание**

### 4.1 — Доказательство kube-DNS — самая важная команда этой лабораторной

`ВАША ЗАДАЧА`: **изнутри Pod'а `web`** вызовите эндпоинт `/ping` у `echo` по его **DNS-имени** и зафиксируйте `pong`. Без port-forward. Без NodePort. Запрос должен пройти путь Pod → DNS Service → kube-proxy → Pod echo, а имя резолвит kube-DNS.

```bash
WEBPOD=$(kubectl get pod -l ___=___ -o jsonpath=___)    # YOUR TASK: get the name of any web pod
                                                        #            (hint: -o jsonpath='{.items[0].metadata.name}')

kubectl exec $WEBPOD -- ___ ___                         # YOUR TASK: hit http://echo/ping
                                                        #            (hint: which tool? — see "If curl is missing" below)
# expected output:
# pong
```

**Зачем это.** Если пришёл `pong`, сразу сработали шесть вещей:
1. **kube-DNS** в `kube-system` резолвил `echo` в ClusterIP этого Service
2. **Селектор** Service совпал с метками Pod'ов echo
3. **kube-proxy** прописал правила iptables/IPVS: IP Service → один из двух IP Pod'ов
4. **readinessProbe** Pod'а echo прошёл, поэтому Pod есть в `endpoints`
5. Внутренняя сеть кластера прокинула запрос через границу нод (скорее всего: `web` и `echo` сели на разные ноды)
6. **Треугольник меток** в манифестах согласован от начала до конца

Сломается хоть одно — получите таймаут или DNS NXDOMAIN. **Сохраните вывод.** Это главный артефакт Lab 9.

<details>
<summary>💡 Если в образе <code>web</code> нет <code>curl</code></summary>

В тонком образе из Lab 2 `curl`, скорее всего, нет. Два варианта. Оба годятся, если вызов идёт **изнутри кластера**:

```bash
# Option A — use python (Lab 1 left you a Python interpreter)
kubectl exec $WEBPOD -- python -c \
  "import urllib.request; print(urllib.request.urlopen('http://echo/ping').read().decode())"

# Option B — throwaway client pod inside the cluster
kubectl run tmp --rm -it --image=curlimages/curl:8.11.0 --restart=Never -- \
  curl -s http://echo/ping
```

Что **не** считается: `curl localhost:8088/ping` после `port-forward svc/echo`. Это доказывает, что до echo достучался ноутбук, а не `web`. Весь смысл — вызов **от Pod к Pod**.

</details>

### 4.2 — Балансировка Service по двум Pod'ам echo

У Service echo два endpoints. Постучите в `/echo` несколько раз. В ответе есть `hostname` ответившего Pod'а, так что видно, как kube-proxy раздаёт трафик round-robin.

```bash
kubectl exec $WEBPOD -- sh -c '___'                     # YOUR TASK: hit http://echo/echo 6+ times in a loop,
                                                        #            extract or print the hostname per call
# expected: hostname rotates between the 2 echo pods (roughly 50/50 over enough calls)
```

> 💡 Round-robin по умолчанию на соединение, а не на запрос. Если hostname каждый раз один и тот же, клиент держит одно соединение. Делайте новый запрос каждый раз (`-H 'Connection: close'` для curl или цикл свежих вызовов `urlopen` в Python).

### 4.3 — Масштабируйте `web` до 5 реплик

`ВАША ЗАДАЧА`: поднимите число реплик. Идите **декларативным** путём (правьте `replicas:` в манифесте и делайте `kubectl apply -f`). Императивный `kubectl scale` сработает, но не переживёт следующий `apply` вашего манифеста. Это ловушка, которая убивает GitOps. В Lab 13 встретите её снова.

```bash
# After editing the manifest:
kubectl apply -f k8s/web-deployment.yaml
kubectl get pods -l app=web -w           # watch new pods come up
kubectl get endpoints web                # 5 pod IPs once readiness passes
```

### 4.4 — Rolling update и откат

Смените тег образа `web` (пересоберите образ из Lab 2 с новым тегом или на время смените `name:` контейнера — годится всё, что меняет хэш шаблона Pod'а). Примените и смотрите:

```bash
kubectl apply -f k8s/web-deployment.yaml
kubectl rollout status deploy/web        # blocks until new ReplicaSet is fully ready
kubectl rollout history deploy/web       # shows both revisions

kubectl rollout undo deploy/web          # back to the previous revision
kubectl rollout status deploy/web
```

При `maxUnavailable: 0` (из §2.1) вывод `kubectl get endpoints web` ни разу не должен упасть ниже 5. **Проверьте это во втором шелле.**

### 4.5 — Доказательство работы

**Вставьте в `docs/LAB09.md`:**

- **Доказательство kube-DNS** — ваша команда `kubectl exec $WEBPOD -- ... http://echo/ping` и её вывод `pong`. **Это главный артефакт.**
- Вывод балансировки — 6 и больше hostname, оба имени Pod'ов echo встречаются примерно поровну
- `kubectl get pods -l app=web -o wide` после масштаба до 5: 5 Pod'ов, разложенных по нодам
- `kubectl rollout history deploy/web` — видны минимум 2 ревизии
- Вывод `kubectl rollout undo` плюс `kubectl rollout status`, который подтверждает, что откат прошёл
- 2–3 предложения про самую полезную находку в `kubectl describe` / `kubectl get events`, пока отлаживали. Какая именно строка `Events:` дала ответ?

---

## Задание 5 — Документация (1 балл)

`ВАША ЗАДАЧА`: напишите `k8s/docs/LAB09.md`. Разделы такие, по порядку:

1. **Архитектура** — диаграмма Mermaid: `User → web Service → 3× web pods` и `web pod → echo Service → 2× echo pods`. На второй стрелке пометьте резолв kube-DNS
2. **Кластер** — ваша команда `k3d cluster create` плюс ответы на три вопроса из §1.1
3. **Манифесты** — короткий абзац на каждый файл: какие метки, какой тип Service, сколько реплик, какие эндпоинты у probe — и **почему** для каждого неочевидного выбора
4. **Межсервисное доказательство** — вывод `pong` из §4.1 и по предложению на каждую из шести вещей, которые сработали
5. **Операции** — доказательства масштаба, rolling update и отката из §4.3–4.4
6. **Трудности и решения** — минимум одна настоящая (не «я только увидел YAML»). Вставьте строку `kubectl describe`, которая вас вытащила

Вставьте **фрагменты** манифестов (не файлы целиком: они уже в коммите) и настоящие выводы CLI из Tasks 1–4.

---

## Бонусное задание — Ingress с TLS через Traefik (2 балла)

Здесь вас меньше ведут за руку. В k3d Traefik уже встроен (это LoadBalancer за портами хоста `:8080`/`:8443`, которые вы пробросили в §1.2). **Контроллер ставить не нужно.** Берите `*.localhost` (сам резолвится в `127.0.0.1`, править `/etc/hosts` не надо).

`ВАША ЗАДАЧА`: направьте `web.localhost` → Service `web` и `echo.localhost` → Service `echo`, оба по HTTPS, через один `Ingress`.

1. **Самоподписанный сертификат и Secret** на оба hostname:
   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout tls.key -out tls.crt \
     -subj "/CN=web.localhost" \
     -addext "subjectAltName=DNS:web.localhost,DNS:echo.localhost"
   kubectl create secret tls ___ --key tls.key --cert tls.crt   # YOUR TASK: pick a Secret name
   ```

2. **`k8s/ingress.yaml`** — форма Ingress:

```yaml
apiVersion: ___                       # YOUR TASK: which group/version owns Ingress? (NOT extensions/v1beta1 — that's gone)
kind: ___
metadata:
  name: ___
spec:
  ingressClassName: ___               # YOUR TASK: k3d's built-in controller name (lecture 9 slide 13)
  tls:
    - hosts: [___, ___]               # YOUR TASK: both hostnames
      secretName: ___                 # YOUR TASK: matches the Secret name above
  rules:
    - host: ___                       # YOUR TASK: web's hostname
      http:
        paths:
          - path: ___                 # YOUR TASK
            pathType: ___             # YOUR TASK: Prefix or Exact?
            backend:
              service:
                name: ___             # YOUR TASK: which Service is this rule routing to?
                port:
                  number: ___         # YOUR TASK: the Service's port (NOT the targetPort).
                                      #            Or use `name: http` to reference the named Service port.
    - host: ___                       # YOUR TASK: echo's hostname
      http:
        paths:
          - path: ___
            pathType: ___
            backend:
              service:
                name: ___
                port:
                  number: ___         # (same options as above — number or name)
```

3. **Проверьте** через порт хоста, проброшенный при создании кластера:
   ```bash
   curl -k https://echo.localhost:8443/ping    # → pong, through Traefik + TLS
   curl -k https://web.localhost:8443/health   # → your app's health JSON
   ```

> 💡 **Маршрутизация по хосту** (один Service на hostname) обходится без middleware, который переписывает путь: каждый бэкенд видит путь запроса как есть. Альтернатива по пути (`/web` → web, `/echo` → echo на одном hostname) требует middleware Traefik, чтобы срезать префикс. Ради двух сервисов это лишнее.

**Доказательства (вставьте в `docs/LAB09.md`):**
- `kubectl get pods -n kube-system | grep traefik` — контроллер в Running
- `kubectl get ingress` — ваш Ingress, оба хоста на месте
- Оба вывода `curl -k https://...localhost:8443/...`, настоящие
- 1–2 предложения: что Ingress даёт сверх NodePort Service из §2.2

---

## Как сдавать

```bash
git switch -c lab09
git add k8s/
git commit -m "feat(lab09): web + echo on k3d — cross-service kube-DNS proven"
git push -u origin lab09
```

Откройте **два** PR:

- `your-fork:lab09` → `course-repo:master` *(на ревью)*
- `your-fork:lab09` → `your-fork:master` *(сольётся в ваш собственный main, когда закончите)*

Чеклист PR:

```text
- [ ] Task 1 done — 3-node k3d cluster on K8s 1.36.1, evidence captured
- [ ] Task 2 done — web Deployment (3 replicas) + Service, label triangle correct, probes on cheap endpoint
- [ ] Task 3 done — echo Deployment (2 replicas) + ClusterIP Service, sized for a tiny Go binary
- [ ] Task 4 done — kubectl exec from inside web pod returns `pong` from echo; LB rotation shown; scaled to 5; rolled out & back
- [ ] Task 5 done — LAB09.md with all 6 sections + Mermaid diagram + the pong capture
- [ ] Bonus done — Ingress over HTTPS via Traefik, both hostnames reachable
```

---

## Критерии приёмки

### Задание 1 — кластер k3d (2 балла)
- ✅ `kubectl version` показывает server `1.36.x`
- ✅ `kubectl get nodes -o wide` показывает **3 ноды**, все `Ready`
- ✅ В команде `k3d cluster create` есть проброс портов LoadBalancer `:80→:8080` и `:443→:8443`
- ✅ Ответы из §1.1 лежат в `docs/LAB09.md`

### Задание 2 — сервис web (3 балла)
- ✅ `web-deployment.yaml` и `web-service.yaml` написаны вручную
- ✅ Треугольник меток согласован (метки Deployment = selector.matchLabels = метки шаблона)
- ✅ **Именованы и порт контейнера, и порт Service** (например `name: http`). Это нужно в Lab 10 и Lab 16
- ✅ 3 реплики в Ready; `kubectl get endpoints web` показывает **3 IP**
- ✅ Заданы и **requests, и limits**. Оба probe смотрят на эндпоинт вроде `/health`, а не на корень приложения
- ✅ Pod'ы разложены по всем 3 нодам (вывод `-o wide`)

### Задание 3 — сервис echo (2 балла)
- ✅ `echo-deployment.yaml` использует `ghcr.io/inno-devops-labs/echo:v1` (своей сборки нет)
- ✅ **2** реплики в Ready; у ClusterIP Service **2** endpoints
- ✅ Probe смотрят на `/healthz:8081` (настоящий эндпоинт готовности echo, см. `plumbing/echo/README.md`)
- ✅ Ресурсы меньше, чем у web (крошечный бинарник Go)

### Задание 4 — межсервисный вызов и операции (2 балла)
- ✅ **`kubectl exec` в Pod web и вызов `http://echo/ping` возвращают `pong`** — доказательство kube-DNS снято
- ✅ В выводе балансировки есть hostname **обоих** Pod'ов echo
- ✅ `web` масштабирован до 5; в списке endpoints 5 адресов; простоя не видно
- ✅ Rolling update и откат показаны выводом `kubectl rollout history`

### Задание 5 — документация (1 балл)
- ✅ В `k8s/docs/LAB09.md` все 6 разделов; есть диаграмма Mermaid; вывод pong — главный артефакт

### Бонус — Ingress и TLS (2 балла)
- ✅ Самоподписанный Secret создан с обоими SAN
- ✅ Ingress использует `ingressClassName: traefik`, оба хоста маршрутизируются
- ✅ Оба вызова `curl -k https://...localhost:8443/...` проходят

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — кластер k3d | **2** | кластер K8s 1.36 на 3 нодах поднят, есть ответы, проброс портов для бонуса |
| **Задание 2** — сервис web | **3** | Deployment и Service вручную, треугольник меток верный, probe на дешёвом эндпоинте, requests и limits, 3 endpoints |
| **Задание 3** — сервис echo | **2** | выданный образ выкачен (2 реплики), ClusterIP Service, probe на `/healthz` |
| **Задание 4** — межсервисный вызов и операции | **2** | `pong` изнутри Pod'а web, ротация LB, масштаб, rollout и откат |
| **Задание 5** — документация | **1** | 6 разделов, Mermaid, вывод pong как главное |
| **Бонус** — Ingress и TLS | **2** | Ingress через Traefik, оба hostname по HTTPS |
| **Итого** | **12** | 10 основных + 2 бонус |

---

## Материалы

<details>
<summary>📚 Документация Kubernetes</summary>

- [Заметки к релизу Kubernetes 1.36 «Haru»](https://kubernetes.io/blog/2026/04/22/kubernetes-v1-36-release/)
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service и сеть](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS для Service и Pod](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Liveness, readiness и startup probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Шпаргалка kubectl](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

<details>
<summary>🛠️ Инструменты</summary>

- [k3d](https://k3d.io/) — k3s в Docker
- [Документация k3s](https://docs.k3s.io/)
- [k9s](https://k9scli.io/) — TUI для kubectl
- [kubectx/kubens](https://github.com/ahmetb/kubectx) — переключатель context и namespace
- [stern](https://github.com/stern/stern) — хвост логов с нескольких Pod'ов

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных пробных прогонов)</summary>

- **Несовпадение label и selector — ошибка №1 в YAML для K8s.** У Deployment метки стоят в *трёх* местах: `metadata.labels`, `spec.selector.matchLabels` и `spec.template.metadata.labels`. `spec.selector` у Service — четвёртое. Все должны совпасть. Симптом: `kubectl get endpoints <svc>` пуст, хотя Pod'ы Running. Селектор **неизменяем** после создания Deployment. Ошиблись — только `kubectl delete deploy/...` и apply заново. `kubectl edit` вас не вытащит.

- **`imagePullPolicy: Always` ломается на k3d для локально собранных образов.** `k3d image import` кладёт образ прямо в containerd кластера. Registry с вашим тегом `:latest` нет. `Always` тянет заново на каждом рестарте и получает `ErrImagePull` / `ImagePullBackOff`. Для локально импортированных образов ставьте `IfNotPresent` (это же значение по умолчанию, если тег не `:latest`, но **на умолчание не рассчитывайте**: объявите явно).

- **NodePort на k3d требует проброса порта хоста при создании кластера.** klipper-lb в k3d открывает порты класса LoadBalancer, которые вы перечислили в `-p`. NodePort из диапазона 30000–32767 с ноутбука не достать, пока вы не (a) добавите `-p "3xxxx:3xxxx@server:0"` при создании кластера или (b) сделаете `kubectl port-forward`. Service типа `LoadBalancer` и `port-forward` работают без лишних шагов. Голый NodePort — нет.

- **`livenessProbe` смотрит на главную страницу приложения.** Когда трафик подскакивает, обработчик тормозит. Liveness отваливается по таймауту, K8s перезапускает контейнер, очередь перетекает на оставшиеся Pod'ы, *они* тормозят под удвоенной нагрузкой. **Каскад рестартов** случается ровно тогда, когда системе нужно стоять. Liveness должен смотреть на **дешёвый сигнал внутри процесса** (`/health` отдаёт 200 и не ходит в БД). Поэтому Lab 1 заставляла вас сделать `/health` отдельно.

- **Нет `readinessProbe` или он медленный — в rolling update появляются дыры.** Без readiness K8s кладёт Pod в endpoints Service в ту же секунду, как контейнер *стартовал*, ещё до того, как приложение реально отвечает. Во время rolling update это несколько секунд `connection refused` на каждой замене. Readiness с `periodSeconds: 3–5` и низким `failureThreshold` не пускает неготовые Pod'ы ни в список endpoints, ни в трафик.

- **Путаница `port` и `targetPort`.** `port` — куда *клиенты* стучатся у Service. `targetPort` — на чём слушает **контейнер**. Частая ошибка: `port: 8081` у Service echo. Так работает, но каждый вызывающий пишет `http://echo:8081/ping`, и будущий чарт, ApplicationSet в Lab 13 и ServiceMonitor в Lab 16 вынуждены отдельно обрабатывать порт. Для HTTP-сервисов ставьте `port: 80`, чтобы работали URL только с DNS.

- **Слишком низкие requests — шумные соседи отбирают ресурсы.** У k3d на ноутбуке память конечная. Если реальный расход `web` — `requests: 64Mi`, а вы поставили `8Mi`, планировщик переупакует ноды, и при любой настоящей нагрузке ваши Pod'ы выселит OOM. Ставьте `requests` по тому, что приложение ест в простое, плюс небольшой запас, а не по минимуму, с которым оно вообще стартует.

- **`kubectl edit` вместо `kubectl apply -f`.** `edit` меняет живой объект напрямую. Манифест в git больше не совпадает с реальностью. Следующий `apply` откатит ваш хотфикс. В мире GitOps (Lab 13 и дальше) это гонка на выкладке. Всегда правьте YAML, делайте `apply -f`, коммитьте.

- **В контейнере нет `curl`/`wget`, и вы хватаетесь не за тот обход.** Не делайте `port-forward`, чтобы «проверить» межсервисные вызовы: это доказывает, что ноутбук достаёт echo, а не что достаёт `web`. Используйте `kubectl run tmp --rm -it --image=curlimages/curl ...` или `kubectl exec $WEBPOD -- python -c "..."`, чтобы вызов шёл **изнутри кластера**.

- **Встроенный Traefik в k3d и мышечная память ingress-nginx.** Туториалы в сети считают, что у вас ingress-nginx (`ingressClassName: nginx`, аннотации `nginx.ingress.kubernetes.io/*`). На k3d это Traefik (`ingressClassName: traefik`, аннотации `traefik.ingress.kubernetes.io/*`). Базовые поля `Ingress` в Kubernetes переносимы. Аннотации — нет.

</details>

<details>
<summary>🔍 Шпаргалка по отладке</summary>

```bash
kubectl describe pod <name>           # bottom Events section = root-cause oracle
kubectl get events --sort-by=.lastTimestamp  # cluster-wide event stream
kubectl logs <pod> --previous         # logs from the previous (crashed) container
kubectl get endpoints <svc>           # empty = selector/label mismatch
kubectl exec -it <pod> -- /bin/sh     # shell into the pod (if it has one)
kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -- bash
                                       # DNS + network sandbox with dig, nslookup, curl, tcpdump
```

</details>

---

## Что дальше

| Lab | Что добавляет к этому стеку |
|---:|---|
| 10 | Переупаковать эти манифесты в чарт **Helm 4**; `echo` становится субчартом |
| 11 | Заменить конфиг открытым текстом на **Secrets** (и познакомиться с OpenBao) |
| 12 | **ConfigMap** и **PVC** — прокидывание конфига и выживание состояния между рестартами Pod'ов |
| 13 | **ArgoCD** и GitOps — `web` и `echo` выкатываются через `ApplicationSet` на 2 окружения |
| 14 | **Argo Rollouts** — canary для Deployment `echo` с постепенным сдвигом трафика |
| 16 | **kube-prometheus-stack** — скрейп `/metrics` у `echo` через `ServiceMonitor` |

---

**Удачи!** ☸️

> **Запомните:** сетевая модель K8s складывается только когда есть **второй** сервис, с которым можно поговорить. Этот `pong`, который вернул `echo` на `curl`, запущенный *внутри* Pod'а `web`, — момент, когда всё из Lecture 9 (Pod, Service, label selector, kube-DNS, endpoints, kube-proxy) встаёт на места. Если из лабораторной сохранится только один вывод, пусть это будет он.
