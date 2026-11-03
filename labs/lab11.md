# Lab 11 — Секреты Kubernetes и OpenBao

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Secret%20Management-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-OpenBao%202.5%20%7C%20K8s%20Secrets-informational)

> **Цель:** хватит класть пароли в Git. Докажите себе, что Kubernetes `Secret` — это **не** шифрование, затем перенесите секреты в **OpenBao 2.5.0** и закройте их политикой только на чтение (принцип наименьших привилегий).
> **Что сдать:** PR из ветки `lab11`, который добавляет `k8s/secrets/` (политика и пример Secret), `k8s/lab10-app/templates/secret.yaml` в ваш Helm-чарт Lab 10 и `docs/LAB11.md` с **реальными** доказательствами (ваши раскодированные значения, ваш `bao kv get`, ваша запрещённая запись).

---

## Обзор

В этой лабораторной вы потренируетесь:
- Создавать `kind: Secret` и **самому декодировать base64**, чтобы навсегда запомнить: base64 ≠ шифрование
- Поднимать **OpenBao 2.5.0** в dev-режиме и выучить контракт переменных окружения (`BAO_ADDR`, `BAO_TOKEN`), который потом переиспользует любая продакшен-схема
- Писать **HCL-политику только на чтение** и токен, привязанный к ней, а затем доказать политику попыткой записи — и её провалом
- Шаблонизировать `Secret` в чарте Lab 10 **без** коммита настоящего значения (значение приходит из `--set`/`-f` в момент установки)
- Выбрать один из двух продакшен-паттернов инъекции: **OpenBao Agent Injector** или **External Secrets Operator (ESO)**

> ⚠️ **Объём:** только OpenBao в dev-режиме (в памяти, один ключ, сам делает unseal). Продакшен-OpenBao (хранилище Raft, auto-unseal через облачный KMS, audit device, TLS) в эту лабораторную не входит — но политики и роли, которые вы здесь напишете, той же формы.

> 💡 **Пять инцидентов, с которых открылась лекция, — Code Spaces 2014, Uber 2016, Toyota 2022, Dropbox 2022, tj-actions 2025 — и в каждом учётные данные лежали не там.** Не zero-day. Не хитрый 0-click. Просто секрет в коде, каталог `.git/`, который отдавал веб-сервер, угнанный фишингом аккаунт GitHub, скомпрометированный GitHub Action, который сливает `env` в логи. Эта лабораторная учит скучным вещам, которые закрыли бы все пять.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**
- Lab 9: кластер k3d 1.36 (`k3d cluster create devops`), сервисы `web` и `echo` запущены
- Lab 10: Helm-чарт в `k8s/lab10-app/` с `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`

**Эта лабораторная добавляет:**
- `k8s/secrets/app-credentials.yaml` — ваш написанный руками манифест `Secret` (демо base64)
- `k8s/lab10-app/templates/secret.yaml` — шаблонный `Secret` в чарте
- `k8s/secrets/lab11-read.hcl` — политика OpenBao только на чтение
- `docs/LAB11.md` — отчёт к сдаче с захваченным выводом CLI
- *(бонус)* либо `k8s/secrets/injector.yaml` (аннотации Agent Injector), либо `k8s/secrets/eso.yaml` (CRD SecretStore и ExternalSecret)

---

## Подготовка

Нужно (проверьте до старта):

- `kubectl version --client` → 1.36.x (из Lab 9)
- `helm version` → 4.1.x (из Lab 10)
- `k3d cluster list` показывает кластер `devops` из Lab 9 в состоянии Running
- **OpenBao 2.5.0** стоит локально. Ставьте через tap brew (`brew install openbao/tap/bao`) или скачайте tar-архив Linux-бинарника `v2.5.0` с `github.com/openbao/openbao/releases` и распакуйте его в `PATH`.
- `bao version` должен напечатать `OpenBao v2.5.0`

Создайте namespace и ServiceAccount, которые потом переиспользуют задания:

```bash
kubectl create namespace lab11
kubectl create serviceaccount lab11-sa -n lab11
```

> **`bao` — drop-in замена `vault`.** OpenBao сохранил протокол обмена, подкоманды CLI и эндпоинты API. Старая мышечная память на `vault kv put ...` по-прежнему работает против сервера OpenBao. В этой лабораторной мы вызываем `bao`, потому что установленный бинарник — OpenBao, а не HashiCorp Vault.

> 📜 **Один абзац про BSL.** В августе 2023 HashiCorp перевёл Vault на Business Source License 1.1: исходники открыты, но non-compete (запрет конкурировать) не даёт делать коммерческие managed-сервисы, которые спорят с HashiCorp. Linux Foundation форкнул Vault 1.14 как **OpenBao** под MPL-2.0 (настоящий open source). OpenBao 2.0 вышел в GA в марте 2024; 2.5.0 (4 февраля 2026) добавил бесплатные Namespace и горизонтальное масштабирование чтения — то, что раньше сидело за paywall HashiCorp Enterprise. Всё, что вы напишете здесь, переносится на любой из двух серверов; за стандарт берём OpenBao, чтобы манифесты пережили любую будущую смену лицензии.

---

## Задание 1 — Kubernetes Secrets и ловушка base64 (2 балла)

### 1.1 — Напишите манифест `kind: Secret` руками

`ВАША ЗАДАЧА`: создайте `k8s/secrets/app-credentials.yaml` с `kind: Secret` для вымышленной базы: имя пользователя `app` и пароль на ваш выбор. Берите **декларативную** форму (YAML-файл, который вы применяете через `kubectl apply`), а не `kubectl create secret`.

Решаете вы:
- Значение `type:` — выберите из `Opaque`, `kubernetes.io/dockerconfigjson` или `kubernetes.io/tls`. Неверный выбор не пройдёт валидацию, так что выбор важен. В `docs/LAB11.md` обоснуйте, почему ваш выбор подходит для обычной пары логин и пароль.
- Два ключа в `data:` — назовите их так, как их реально прочитает приложение (в духе имён переменных окружения).
- Значения в base64 посчитайте сами командой `base64`; в этом задании **не** пользуйтесь коротким путём `stringData:`. Смысл — прочувствовать шаг кодирования руками.

Каркас (заполните маркеры YOUR-TASK):

```yaml
# k8s/secrets/app-credentials.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
  namespace: lab11
type: YOUR-TASK                 # which built-in type fits username+password?
data:
  YOUR-TASK: YOUR-TASK          # echo -n 'app' | base64
  YOUR-TASK: YOUR-TASK          # echo -n '<your password>' | base64
```

> ⚠️ **`echo` и `echo -n`** — `echo "foo" | base64` включает хвостовой перевод строки (`Zm9vCg==`); `echo -n "foo" | base64` — нет (`Zm9v`). Перевод строки молча ломает сравнение паролей в приложении, которое их читает, и это самая частая ошибка base64 на пробных прогонах. Берите `-n`.

Примените манифест и прочитайте его обратно:

```bash
kubectl apply -f k8s/secrets/app-credentials.yaml
kubectl get secret app-credentials -n lab11 -o yaml
```

### 1.2 — Декодируйте сами и докажите тезис

`ВАША ЗАДАЧА`: запустите однострочник, который достаёт **ваш** Secret из кластера и прогоняет поле пароля через `base64 -d`, возвращая открытый текст. Вставьте вывод дословно в `docs/LAB11.md` под заголовком **"Момент «ага» с base64"**.

Подсказка (форма команды — подставьте имя ключа, которое выбрали):

```bash
kubectl get secret app-credentials -n lab11 \
  -o jsonpath='{.data.YOUR-TASK}' | base64 -d ; echo
```

В **2–3 предложениях** в `docs/LAB11.md` ответьте: *зачем API Kubernetes вообще использует здесь base64, если это не про безопасность?* (Подсказка: значения `Secret.data` должны быть валидными строками JSON/YAML; бинарник вроде TLS-ключа транспорт не переживёт.)

### 1.3 — Шифрование etcd at rest своими словами

Напишите в `docs/LAB11.md` абзац, который отвечает на все три вопроса:
- Шифруются ли Kubernetes Secrets at rest по умолчанию? (Подсказка: посмотрите `EncryptionConfiguration`.)
- Что KMS-провайдер в этой конфигурации делает такого, чего не делает провайдер `aescbc`?
- Почему включить `EncryptionConfiguration` необходимо, но **недостаточно** для продакшена — от чего это вас не защищает? (Подсказка: любой, у кого есть RBAC `system:masters`.)

### 1.4 — Доказательства

Вставьте в `docs/LAB11.md`:

- Содержимое `k8s/secrets/app-credentials.yaml` (ваш настоящий файл, ваши настоящие строки base64)
- Вывод `kubectl get secret app-credentials -n lab11 -o yaml | grep -A2 ^data:`
- Однострочник декодирования base64 **и его вывод открытым текстом**: видно, что вы восстановили свой пароль
- Ответ в 2–3 предложениях на вопрос *зачем base64?* и абзац про etcd at rest из 1.3

---

## Задание 2 — Секреты под управлением Helm (3 балла)

### 2.1 — Шаблонный Secret в чарте Lab 10

`ВАША ЗАДАЧА`: добавьте `templates/secret.yaml` в чарт Lab 10 (`k8s/lab10-app/templates/secret.yaml`). Он должен отрендерить `kind: Secret` с именем из хелпера `fullname` чарта, нести стандартные лейблы чарта и брать **два** значения из `.Values.secret` — `dbUsername` и `dbPassword`.

Жёсткие правила:
- Берите `stringData:` (чтобы Helm не кодировал дважды); API Kubernetes кодирует в base64 на своей стороне.
- В `values.yaml` задайте оба поля **строками-заглушками** (`"PLACEHOLDER_USER"`, `"PLACEHOLDER_PASS"`). Настоящие значения приходят из `--set` или из `-f override.yaml` в момент установки — *никогда* не коммитьте их.

Каркас (заполните маркеры YOUR-TASK):

```yaml
# k8s/lab10-app/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: YOUR-TASK              # use the fullname helper from your _helpers.tpl
  labels:
    YOUR-TASK                  # use the labels helper from your _helpers.tpl
type: YOUR-TASK                # same choice as Task 1.1
stringData:
  DB_USERNAME: YOUR-TASK       # quote a value pulled from .Values.secret.*
  DB_PASSWORD: YOUR-TASK
```

И в `values.yaml`:

```yaml
secret:
  dbUsername: "PLACEHOLDER_USER"
  dbPassword: "PLACEHOLDER_PASS"   # override at install time, NEVER commit real values
```

### 2.2 — Подключите его в Deployment

`ВАША ЗАДАЧА`: отредактируйте существующий шаблон Deployment в чарте (`templates/deployment.yaml`), чтобы контейнер `web` читал оба ключа как переменные окружения. Используйте паттерн `envFrom` + `secretRef` (чище, чем `secretKeyRef` на каждый ключ, когда весь Secret нужен одному потребителю).

Каркас (заполните маркер YOUR-TASK):

```yaml
containers:
  - name: web
    image: # ... your Lab 10 image ...
    envFrom:
      - secretRef:
          name: YOUR-TASK      # same helper-rendered name as in Task 2.1
```

### 2.3 — Установите с настоящим значением вне репозитория

`ВАША ЗАДАЧА`: установите (или обновите) чарт, передав настоящий пароль через `--set` **или** через `-f local-secrets.yaml`, который лежит в `.gitignore`. Затем проверьте в запущенном Pod.

> ⚠️ **Ловушка `--set` и логов CI.** `helm upgrade --install ... --set secret.dbPassword='hunter2'` на ноутбуке нормально. В CI пароль оказывается в логе workflow, если переменная не обёрнута как секрет (`${{ secrets.DB_PASSWORD }}` в GHA) **и** вы не вызываете `helm upgrade ... --set secret.dbPassword="$DB_PASSWORD"` без echo. В `docs/LAB11.md` напишите, какой паттерн вы бы взяли в настоящем GitHub Action.

> 🔮 **Заглядывая в Lab 13: `--set` ArgoCD НЕ переживает.** На следующей неделе ArgoCD будет синхронизировать этот же чарт из Git и учтёт только `helm.valueFiles` / `helm.valuesObject` на CR `Application` — у GitOps-reconcile **нет ручки `--set`**. Поэтому бонус ставит **OpenBao Agent Injector** или **ESO**: они доставляют секрет в обход Git, и чарт в Git может вечно ехать с заглушками. Если бонус пропустите, обходной путь в Lab 13 — `helm.valuesObject`, который ссылается на Secret под внешним управлением. Настоящего значения в Git быть не должно.

Команды проверки (для примера — ваш вывод будет другим):

```bash
helm upgrade --install lab10-app k8s/lab10-app -n lab11 \
  --set secret.dbPassword='YOUR-PASSWORD'         # YOUR-TASK: pick a value

# Find the actual web Deployment name — your fullname helper decides it; don't
# guess it. The `app.kubernetes.io/name=lab10-app` label matches both web and echo
# (they share the chart), so filter on the `-web` suffix in the name to get only web:
WEB=$(kubectl get deploy -n lab11 -l app.kubernetes.io/name=lab10-app \
       -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep -- '-web$' | head -1)
kubectl exec -n lab11 deploy/"$WEB" -- env | grep '^DB_'
# DB_USERNAME=app
# DB_PASSWORD=YOUR-PASSWORD                       # ← present in pod env
kubectl describe pod -n lab11 -l app.kubernetes.io/name=lab10-app | grep -A1 -i secret
# (the Secret name appears; the value does NOT — describe redacts envFrom values)
```

### 2.4 — Доказательства

Вставьте в `docs/LAB11.md`:

- Ваш `templates/secret.yaml` и нужный фрагмент `values.yaml` (только заглушки — *никакого настоящего пароля*)
- Команду `helm upgrade ... --set`, которую вы реально запускали (пароль можно замазать до `***`)
- Вывод `kubectl exec ... env | grep ^DB_`: значения попали в контейнер (пароль здесь тоже можно замазать до `***`)
- Ответ в 2–3 предложениях: «как бы я сделал это в CI»

---

## Задание 3 — Интеграция с OpenBao (3 балла)

Это главное задание. Вы поднимете OpenBao, выучите **контракт переменных окружения**, который потом переиспользует любая продакшен-схема, напишете **HCL-политику только на чтение** и докажете её: попробуете записать — и не сможете.

### 3.1 — Поднимите dev-сервер

`ВАША ЗАДАЧА`: запустите dev-сервер OpenBao с **запоминающимся** ID корневого токена (им вы будете входить). Затем экспортируйте две переменные окружения, от которых зависит каждая команда `bao`.

Каркас (заполните маркеры YOUR-TASK — и да, **смысл** пропусков в том, чтобы вы вбили контракт в голову):

```bash
# Start the dev server in the background. -dev = unsealed + in-memory.
# Redirect output to a log file so server lines don't tangle with your prompt.
bao server -dev -dev-root-token-id=___ >/tmp/bao.log 2>&1 &   # YOUR-TASK: pick a token id (e.g. 'devroot')

# Two env vars EVERY bao command needs:
export BAO_ADDR=___                              # YOUR-TASK: the dev server URL (default port is 8200)
export BAO_TOKEN=___                             # YOUR-TASK: must match the -dev-root-token-id above

# Give the listener ~1s to come up before the first call, or poll bao status.
sleep 1
bao status                                       # Sealed false, Storage Type inmem, Version 2.5.0
```

> 🧹 **Как остановить dev-сервер.** Это последняя фоновая задача в вашей оболочке — `kill %1` (или `kill $(pgrep -f 'bao server -dev')`), когда закончите. Перезапуск стирает все секреты (INMEM); в этом весь смысл.

> 💡 **Контракт:** каждый клиент `bao` (или `vault`) — ваш CLI, ваши приложения, провайдер ESO, Agent injector — аутентифицируется парой `BAO_ADDR` и токена. В dev-режиме токен корневой. В продакшене токен приходит из **auth method** (JWT ServiceAccount Kubernetes, AppRole, OIDC), живёт недолго и ограничен политикой. Имена переменных те же.

> ⚠️ **Dev-сервер — это INMEM.** Каждый положенный секрет исчезает, когда процесс перезапускается. Так и задумано: лабораторная идёт быстро, а модель безопасности читается без двусмысленности.

### 3.2 — Положите секрет на путь KV-v2 и прочитайте его

`ВАША ЗАДАЧА`: включите secrets engine KV-v2 на пути, который выберете, затем запишите секрет минимум с **двумя** парами ключ–значение на пути, который выберете, затем прочитайте одно из полей обратно.

Каркас:

```bash
# YOUR TASK lines below:
#   -path=:  pick the mount path (convention: 'secret')
#   put arg: the mount path you chose + a logical name (e.g. secret/lab11/db)
#   first  field=value: a username (use a literal — never a real password)
#   second field=value: a password (literal, not from env)
#   get:     read back one field by name

bao secrets enable -path=___ kv-v2

bao kv put ___/lab11/db \
  ___=app \
  ___=hunter2

bao kv get -field=___ ___/lab11/db
```

> ⚠️ **Ловушка путей KV-v1 и KV-v2.** `bao kv put secret/foo k=v` пишет *логический* путь `secret/foo`, но **API**-путь в KV-v2 — `secret/data/foo` (вставлен сегмент `data/`). Политики и Agent injector ссылаются на API-путь, поэтому правило HCL должно говорить `path "secret/data/lab11/db"`, а не `path "secret/lab11/db"`. Забыть это — способ номер один, которым «правильная» политика запрещает чтение.

> ⚠️ **`bao kv put` и `bao write` на одном data-пути.** KV-v2 версионирует данные под `data/`; `bao write secret/data/foo data='{"k":"v"}'` работает, но JSON у него другой формы, чем у `bao kv put`. Для KV-v2 держитесь `bao kv put` / `bao kv get`; `bao write` — для всего остального (auth backend, политики, роли).

### 3.3 — Напишите политику только на чтение

`ВАША ЗАДАЧА`: создайте `k8s/secrets/lab11-read.hcl` с одной станзой, которая даёт **read** ровно на data-путь вашего секрета и **ничего больше**. Ни `list`, ни `update`, ни `delete`, ни glob — весь смысл в наименьших привилегиях.

Каркас (заполните маркеры YOUR-TASK):

```hcl
# k8s/secrets/lab11-read.hcl
path "___" {                                     # YOUR-TASK: the API path to the secret you wrote in 3.2
  capabilities = [___]                           # YOUR-TASK: a single-element list, the minimum verb
}
```

Примените её и выпустите токен, привязанный к ней:

```bash
bao policy write lab11-read k8s/secrets/lab11-read.hcl
APP_TOKEN=$(bao token create -policy=lab11-read -ttl=1h -field=token)
echo "$APP_TOKEN"      # save for the next step
```

### 3.4 — Докажите политику, пытаясь её сломать

`ВАША ЗАДАЧА`: войдите новым токеном (задайте `BAO_TOKEN=$APP_TOKEN`), затем выполните **две** команды и сохраните оба вывода:

1. `bao kv get -field=...`, который **проходит** (политика разрешает read).
2. `bao kv put ...` на тот же путь, который **падает** с 403 (политика запись не разрешает).

Провал второй команды и есть доказательство. Если она проходит, политика выдала слишком много — вернитесь к 3.3.

Каркас:

```bash
BAO_TOKEN=$APP_TOKEN bao kv get -field=___ ___/lab11/db    # YOUR-TASK: should print the value
BAO_TOKEN=$APP_TOKEN bao kv put ___/lab11/db ___=evil      # YOUR-TASK: should fail with 403
# Error writing data to secret/data/lab11/db: ...permission denied
```

Верните корневой токен перед любой уборкой: `export BAO_TOKEN=<your-dev-root-token-id>`.

### 3.5 — Доказательства

Вставьте в `docs/LAB11.md`:

- `bao version` (должна показать 2.5.0)
- Точную строку `bao server -dev -dev-root-token-id=...` и строки `export BAO_ADDR`/`BAO_TOKEN` (контракт переменных — токен можно замазать)
- `bao status`, где видно `Sealed false`, `Storage Type inmem`, `Version 2.5.0`
- Подтверждение создания из `bao kv put ...` и `bao kv get -field=...`, который вернул ваше значение
- Содержимое `k8s/secrets/lab11-read.hcl` (ваша настоящая политика)
- **Два** вывода из 3.4 рядом: чтение проходит, запись падает с `permission denied`. **Это главное доказательство.**

---

## Задание 4 — Документация (2 балла)

`ВАША ЗАДАЧА`: доведите `docs/LAB11.md` до этих разделов, в таком порядке:

1. **Момент «ага» с base64** — ваш раскодированный открытый текст (1.2) и абзац «зачем тогда base64?» (1.3)
2. **Шифрование etcd at rest** — абзац из 1.3: KMS против `aescbc` и от чего `EncryptionConfiguration` не защищает
3. **Секрет под управлением Helm** — фрагмент чарта, команда установки (замазанная), доказательство env в Pod (2.4)
4. **Workflow OpenBao** — контракт переменных, put/get KV-v2, политика только на чтение и доказательство запрещённой записи (3.5)
5. **История Vault → OpenBao** — один абзац: BSL, август 2023, форк Linux Foundation, зачем OpenBao в этом курсе
6. **Разбор безопасности** — когда хватает нативного Secret Kubernetes, а когда нужен OpenBao; что dev-режим от вас прячет и что всплывает в продакшене (нет Raft, нет audit device, нет auto-unseal, нет TLS); 2–3 предложения про альтернативы SOPS / Sealed Secrets и когда они уместны
7. **Продакшен-чеклист** — пять пунктов, которые вы реально положили бы в документ по харденингу прода (роли с наименьшими привилегиями, audit device и т.д.)

---

## Бонусное задание — выберите ОДНО (2 балла)

Выберите **A** или **B**. Оба на 2 балла. **Не делайте оба.** Здесь нарочно меньше подсказок — вы это заслужили.

### Вариант A — OpenBao Agent Injector

Поставьте Agent Injector через Helm-чарт OpenBao, повесьте аннотации на Pod и докажите, что секрет оказывается на диске внутри Pod, а приложение про OpenBao не знает.

**Обязательные артефакты** (в `k8s/secrets/injector.yaml`):

1. **Аннотации** на шаблоне Pod, которые запускают инъекцию. Имена ниже — контракт: заполните значения; имена с хвостом `-<file>` шаблонизируете *вы* (кусок с именем файла становится именем файла в `/vault/secrets/`):

   ```yaml
   # k8s/secrets/injector.yaml — annotations: section of your pod template
   vault.hashicorp.com/agent-inject: "true"
   vault.hashicorp.com/role: YOUR-TASK                          # the OpenBao role you create below
   vault.hashicorp.com/agent-inject-secret-YOUR-TASK: YOUR-TASK # filename : the KV API path (mind the data/ segment)
   vault.hashicorp.com/agent-inject-template-YOUR-TASK: |       # render the secret as a .env-style file (>=2 keys)
     YOUR-TASK
   ```

2. `serviceAccountName: lab11-sa` в spec Pod — это ServiceAccount, чей JWT проверяет OpenBao.

**Обязательная конфигурация OpenBao** (запускайте внутри Pod сервера openbao):

- `bao auth enable kubernetes` и `bao write auth/kubernetes/config kubernetes_host=...`
- `bao write auth/kubernetes/role/lab11 ...`, который привязывает `lab11-sa`, namespace `lab11`, политику `lab11-read` и разумный `ttl`.

**Доказательства:**

- `kubectl get pods -n openbao`: Pod injector в состоянии Running
- `kubectl describe pod -n lab11 <your-pod>`: видны **внедрённые init-контейнер и sidecar** (`vault-agent-init` и `vault-agent`), которых вы туда НЕ клали
- `kubectl exec -n lab11 <your-pod> -c <app-container> -- cat /vault/secrets/db`: виден отрендеренный файл (значение можно замазать — важно само *наличие* файла)

> Injector — это **MutatingAdmissionWebhook**. Он правит podspec в момент admission. Поэтому в YAML у вас *один* контейнер, а у запущенного Pod — *три*: два добавил webhook. Тот же механизм Istio использует для sidecar envoy.

### Вариант B — External Secrets Operator (ESO)

Поставьте ESO через Helm, направьте `SecretStore` на ваш сервер OpenBao, и `ExternalSecret` породит **нативный Secret Kubernetes**, который приложение читает через `envFrom: secretRef`. Приложение не знает, что ESO существует.

**Обязательные артефакты** (в `k8s/secrets/eso.yaml`):

1. CRD `SecretStore` (`apiVersion: external-secrets.io/v1`) с именем `openbao` в namespace `lab11` и блоком `provider.vault` (провайдер `vault` в ESO совместим с OpenBao). Аутентификация — `kubernetes`, роль, которую вы создаёте в OpenBao, и `serviceAccountRef: {name: lab11-sa}`.
2. CRD `ExternalSecret`, который порождает нативный Secret `db-creds`.

Каркас (заполните маркеры YOUR-TASK — имена полей CRD и есть контракт):

```yaml
# k8s/secrets/eso.yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata: {name: openbao, namespace: lab11}
spec:
  provider:
    vault:
      server: YOUR-TASK                 # the in-cluster URL of the OpenBao server (Service DNS + port)
      path: YOUR-TASK                   # the KV mount path you chose in Task 3.2
      version: YOUR-TASK                # KV engine version
      auth:
        kubernetes:
          mountPath: kubernetes
          role: YOUR-TASK               # the OpenBao role bound to lab11-sa
          serviceAccountRef: {name: lab11-sa}
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: db, namespace: lab11}
spec:
  refreshInterval: YOUR-TASK            # production default vs demo: pick deliberately
  secretStoreRef: {name: openbao, kind: SecretStore}
  target: {name: YOUR-TASK}             # the native K8s Secret ESO will produce (consumed by your app)
  data:
    - secretKey: YOUR-TASK              # the key as it appears IN the produced K8s Secret
      remoteRef: {key: YOUR-TASK, property: YOUR-TASK}  # the OpenBao logical path + the field name in it
    # YOUR-TASK: add a second key — username — with the same shape
```

**Обязательная конфигурация OpenBao:** тот же Kubernetes auth и та же привязка роли, что в варианте A (это тот же auth method).

**Обязательная проводка приложения:** поправьте чарт Lab 10, чтобы он читал `db-creds` через `envFrom: secretRef`. Приложение теперь читает *нативный Secret Kubernetes* — без sidecar, без аннотаций, без клиента OpenBao в Pod. Синхронизацию ESO делает в обход.

**Доказательства:**

- `kubectl get pods -n external-secrets`: контроллер ESO в состоянии Running
- `kubectl get externalsecret db -n lab11`: статус `STATUS: SecretSynced`
- `kubectl get secret db-creds -n lab11`: Secret, который породил ESO, существует
- `kubectl exec -n lab11 deploy/lab10-app-web -- env | grep ^DB_`: значения попали в приложение

> ESO **опрашивает** OpenBao каждые `refreshInterval`. Для демо уменьшите интервал (например, `30s`), но продакшен-значение по умолчанию — `1h`: ротация в OpenBao доезжает не сразу. Этот компромисс — главное отличие от Agent Injector (тот обновляется по lease токена).

**Документация бонуса:** допишите в `docs/LAB11.md` раздел с манифестом, который вы использовали, захватами доказательств выше и **одним абзацем**, почему вы выбрали A или B под тот тип приложения, с которым хотите это запускать (ориентир: «динамические креды БД с TTL 15 минут» → Agent Injector; «каждый микросервис в монорепозитории на 50 Pod читает один и тот же небольшой набор секретов» → ESO).

---

## Как сдавать

```bash
git switch -c lab11
git add k8s/secrets/ k8s/lab10-app/templates/secret.yaml \
        k8s/lab10-app/values.yaml docs/LAB11.md
# also stage k8s/secrets/injector.yaml (Option A) OR k8s/secrets/eso.yaml (Option B) if you did the bonus
git commit -m "feat(lab11): secrets + OpenBao (read-only policy, base64 demo)"
git push -u origin lab11
```

Откройте **два** PR:

- `your-fork:lab11` → `course-repo:master` *(на ревью)*
- `your-fork:lab11` → `your-fork:master`

Чеклист PR:

```text
- [ ] Task 1 — app-credentials.yaml + base64 decode proof in docs/LAB11.md
- [ ] Task 2 — templates/secret.yaml in chart, placeholders only in values.yaml, env proof
- [ ] Task 3 — bao 2.5.0 dev server, BAO_ADDR/BAO_TOKEN exported, KV put/get, read-only policy, denied-write proof
- [ ] Task 4 — docs/LAB11.md has all 7 sections
- [ ] Bonus (optional) — Agent Injector OR ESO with proof captures
- [ ] No real secret values in any committed file (grep your diff before pushing)
```

---

## Критерии приёмки

### Задание 1 (2 балла)
- ✅ `k8s/secrets/app-credentials.yaml` — ваш собственный, написанный руками `kind: Secret` с осознанным выбором `type:` и двумя ключами `data:`, закодированными в base64
- ✅ В `docs/LAB11.md` есть однострочник декодирования base64 **вместе с вашим выводом открытым текстом**
- ✅ Ответ «Зачем base64?» и абзац про etcd at rest есть, и они верные

### Задание 2 (3 балла)
- ✅ `k8s/lab10-app/templates/secret.yaml` есть, использует хелперы имени и лейблов чарта и читает из `.Values.secret.*`
- ✅ `values.yaml` везёт **только строки-заглушки** — никакого настоящего пароля
- ✅ Deployment читает Secret через `envFrom: secretRef`
- ✅ Переменные в Pod подтверждены через `kubectl exec ... env | grep ^DB_`; `kubectl describe` значения НЕ показывает
- ✅ `docs/LAB11.md` описывает паттерн CI, которым вы не дадите `--set` слить пароль

### Задание 3 (3 балла)
- ✅ `bao version` показывает 2.5.0
- ✅ Был запущен `bao server -dev -dev-root-token-id=...`; экспортированы `BAO_ADDR` и `BAO_TOKEN`
- ✅ Включён KV-v2, секрет записан с ≥ 2 ключами, прочитан обратно через `bao kv get -field=...`
- ✅ `k8s/secrets/lab11-read.hcl` даёт **только read** на точный data-путь (без glob, без лишних capabilities)
- ✅ Доказательство запрещённой записи (`bao kv put ...` → `permission denied`) лежит в `docs/LAB11.md`

### Задание 4 (2 балла)
- ✅ В `docs/LAB11.md` все семь разделов (1.2 и 1.3, etcd at rest, Helm, OpenBao, история Vault→OpenBao, разбор безопасности, продакшен-чеклист)

### Бонус (2 балла) — только один вариант
- ✅ **A:** Agent Injector задеплоен; внедрённые init и sidecar видны в `kubectl describe pod`; файл `/vault/secrets/<file>` есть в контейнере приложения; свой шаблон рендерит `.env` с несколькими ключами
- ✅ **B:** ESO установлен; у `ExternalSecret` статус `SecretSynced`; нативный Secret `db-creds` есть; в env Pod приложения лежат значения

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Secrets Kubernetes и ловушка base64 | **2** | Secret написан руками, открытый текст раскодирован, base64 и шифрование разобраны, etcd at rest понят |
| **Задание 2** — Secret под управлением Helm | **3** | Шаблонный Secret, значения-заглушки, проводка envFrom, настоящее значение подано при установке, паттерн CI описан |
| **Задание 3** — интеграция с OpenBao | **3** | Dev-сервер поднят, контракт переменных усвоен, put/get KV-v2, политика только на чтение держит (доказательство запрещённой записи) |
| **Задание 4** — документация | **2** | Все семь разделов на месте, разбор безопасности настоящий, история BSL и OpenBao верная |
| **Бонус** — Injector или ESO | **2** | Рабочая инъекция (A) или синхронизация (B), с манифестами и доказательствами |
| **Итого** | **12** | 10 основных + 2 бонус |

**Градации:**
- **10/10:** все четыре основных задания сделаны, доказательство запрещённой записи настоящее, настоящие секреты не закоммичены, разбор безопасности показывает, что вы правда поняли
- **8–9/10:** все задания сделаны, в документации мелкие дыры или не хватает одного захвата
- **6–7/10:** задания 1 и 2 крепкие, OpenBao поднят, но политика слишком широкая (нет доказательства запрещённой записи) или путь KV неверный
- **<6/10:** OpenBao не запущен, нет демо base64 или в коммите лежат настоящие секреты

---

## Материалы

<details>
<summary>📚 Документация</summary>

- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — концепции и таблица `type:`
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) — EncryptionConfiguration, KMS-провайдер, ротация ключей
- [OpenBao docs](https://openbao.org/docs/) — CLI `bao`, KV-v2, auth method, политики
- [OpenBao 2.5.0 release notes](https://openbao.org/community/release-notes/2-5-0/) — 4 февраля 2026
- [OpenBao Kubernetes auth](https://openbao.org/docs/auth/kubernetes/) — TokenReview API, привязка роли
- [Vault Agent annotations (OpenBao-compatible)](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations) — справочник аннотаций
- [External Secrets Operator](https://external-secrets.io/) — CRD и список провайдеров
- [ESO Vault/OpenBao provider](https://external-secrets.io/latest/provider/hashicorp-vault/) — форма конфигурации

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных пробных прогонов)</summary>

- **Base64 — это кодирование, не шифрование.** `kubectl get secret -o yaml | grep -A1 data:` и следом `| base64 -d` возвращает открытый текст **без ключа**. Это факт номер один, который каждый джун понимает неправильно. Если из лабораторной вы унесёте одну вещь, пусть будет эта.
- **`echo` добавляет хвостовой перевод строки.** `echo "foo" | base64` ≠ `echo -n "foo" | base64`. Перевод строки молча делает пароль внутри приложения невалидным. Берите `-n`.
- **Шифрованию etcd at rest нужен KMS, а не один `aescbc`.** Провайдер `aescbc` с локальным ключом хранит ключ на том же диске, что и данные, которые он «защищает»: ключ лежит рядом с замком. KMS-провайдеры (AWS KMS, GCP KMS, OpenBao Transit) держат ключ вне кластера.
- **Dev-сервер OpenBao — это INMEM.** Перезапустите процесс — и секретов нет. В этом смысл: dev-режим нужен, чтобы учить API, а не чтобы что-то хранить. Прочитайте в лекции слайд про антипаттерн «dev-режим в проде».
- **Разница путей KV-v1 и KV-v2.** `bao kv put secret/foo k=v` пишет *логический* путь; политики и Agent injector должны ссылаться на **API**-путь (`secret/data/foo`, со вставленным `data/`), иначе чтение молча запрещается. Та же ловушка живёт в поле `remoteRef.key` у ESO.
- **`bao kv put` и `bao write`.** Оба трогают один путь на KV-v2, но JSON у них разной формы; для KV-v2 держитесь `kv put`/`kv get`, чтобы не словить несовпадение формы.
- **Helm `--set` утекает в логи CI.** `helm upgrade ... --set db.password='hunter2'` в логе GitHub Action — это будущие Code Spaces и tj-actions. Оберните как `${{ secrets.* }}` и ссылайтесь через переменную окружения; лучше заведите настоящий менеджер секретов и пусть значение доставит ESO или Agent Injector.
- **`automountServiceAccountToken: true` по умолчанию.** Каждый Pod получает токен ServiceAccount, нужен он ему или нет. Для Pod, которые не ходят в API Kubernetes и в OpenBao, поставьте `false`: радиус поражения станет меньше.
- **`vault.hashicorp.com/role: admin` — это новый `--privileged`.** Одна роль на приложение на окружение. Наименьшие привилегии — весь смысл политики, которую вы написали в 3.3.
- **Корневой токен dev-режима остаётся в истории оболочки.** `export BAO_TOKEN=root` переживает сессию в `~/.bash_history`. Берите уникальное значение на каждую dev-сессию и сбрасывайте его (`unset BAO_TOKEN`), когда закончите.

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- [gitleaks](https://github.com/gitleaks/gitleaks) — сканер для pre-commit и CI; ловит те самые человеческие моменты
- [trufflehog](https://github.com/trufflesecurity/trufflehog) — детектор глубже, находит высокоэнтропийные строки в истории git
- [SOPS](https://getsops.io/) — шифрование на уровне файла через KMS/age/PGP; дружит с GitOps
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) — шифрование публичным ключом на каждый кластер; CRD и контроллер
- [k9s](https://k9scli.io/) — терминальный UI; `:secret` показывает все секреты сразу

</details>

---

## Что дальше

| Lab | Что добавляет |
|---:|---|
| 12 | ConfigMap и PVC — несекретный конфиг, и постоянное состояние переживает удаление Pod |
| 13 | ArgoCD и GitOps — защищённый чарт выкатывается через Application/ApplicationSet |
| 14 | canary в Argo Rollouts; секреты ротируются под progressive delivery |
| 15 | StatefulSet — стабильное имя, PVC на каждый Pod и секреты на реплику |
| 16 | стек kube-prometheus — скрейп `/sys/metrics` у OpenBao |

**Удачи.** 🔐

> **Запомните:** момент, когда вы декодировали свой Secret командой `base64 -d` и получили открытый текст, — это момент, когда стало ясно, почему каждую утечку из списка инцидентов лекции можно было не допустить. Держите это чувство. Берите его на каждое ревью PR, где кто-то «только на сейчас» кладёт настоящее значение в `values.yaml`.
