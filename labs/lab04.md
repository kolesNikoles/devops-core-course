# Lab 4 — инфраструктура как код (Terraform и Pulumi)

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Infrastructure%20as%20Code-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Terraform%20%7C%20Pulumi-informational)

> **Цель:** поднимите *одну и ту же* маленькую VM дважды — один раз на HCL **Terraform**, один раз на Python **Pulumi** — и прочувствуйте компромиссы руками.
> **Что сдаёте:** PR из `lab04` с каталогами `terraform/`, `pulumi/`, файлом `docs/LAB04.md` и (бонус) `.github/workflows/terraform-ci.yml`.

---

## Обзор

В этой лабораторной вы:
- проходите жизненный цикл **`init → plan → apply → destroy`** (и читаете plan как diff)
- пишете четыре примитива HCL — `provider`, `resource`, `variable`, `output` — и один источник `data`
- собираете ту же инфраструктуру заново на Python в Pulumi (настоящие циклы, настоящие типы, настоящие импорты)
- разбираете **управление state** — что лежит в `terraform.tfstate`, почему это секрет и почему важна блокировка

> ⚠️ **Границы:** одна облачная VM (или замена на docker-provider — см. заметку ниже). Без Kubernetes и без выкладки приложения. В Lab 5 вы зайдёте по SSH на то, что соберёте здесь.

**Облако или docker-provider?** Берите облако с free-tier (так и надо: это и есть навык). Если аккаунт с обязательной картой завести не получается, провайдер `kreuzwerker/docker` гоняет *тот же* жизненный цикл локально, на контейнере. **Ссылка на Lab 5 для тех, кто на docker:** в контейнере нет sshd; Lab 5 возьмёт `ansible_connection: community.docker.docker` (или Ansible с `connection: local` против `docker exec`) вместо SSH. Роли те же, connection plugin другой. Оба пути дают полный балл; запишите, какой выбрали.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**
- Lab 1: `app_python/` (и, возможно, `app_go/`) — сервис, который вы запустите на VM
- Lab 2: multi-stage образ Docker, запушенный в GHCR
- Lab 3: GitHub Actions собирает и пушит на каждом PR

**Эта лабораторная добавляет:**
- `terraform/` — HCL, который вы написали сами; поднимает **одну VM** (или один контейнер)
- `pulumi/` — Python, который собирает *ту же* инфраструктуру
- `docs/LAB04.md` — заметки прогона и сравнение Terraform с Pulumi

К Lab 5 вы зайдёте на этот хост по SSH, и Ansible его настроит. К Lab 9 вы замените его на k3d, но подход IaC останется тем же.

---

## Стек и версии

| Инструмент | Версия | Заметки |
|------|---------|-------|
| **Terraform** | **1.15.3** | BSL-1.1 с **2023-08-10**. Зафиксируйте минимум `>= 1.15.0`. |
| **OpenTofu** | **1.12.0** | MPL-2.0, Linux Foundation. Замена один в один: `tofu` подходит для каждой команды ниже. |
| **Pulumi** | **3.243.0** | Везде Apache-2.0. |
| **Провайдер AWS** | `~> 5.0` | Облачный путь с самой полной документацией. |
| **Провайдер GCP** | `~> 6.0` | Альтернатива. |
| **Провайдер Yandex** | `~> 0.115` | Удобно из России, карта на старте не нужна. |
| **Провайдер Docker** | `kreuzwerker/docker ~> 3.0` | Бесплатная локальная замена, если облачного аккаунта нет. |

> **Заметка про OpenTofu:** перелицензирование на BSL 10 августа 2023 сделало OpenTofu необходимым. Сегодняшний 1.12.0 гоняет тот же HCL против того же реестра. Проверяющий принимает любой из двух CLI.

---

## Подготовка

```bash
terraform -version   # or: tofu -version
pulumi version

# Generate an SSH keypair for the VM if you don't have one
ssh-keygen -t ed25519 -f ~/.ssh/lab04 -C "lab04@$(hostname)"
chmod 600 ~/.ssh/lab04
cat  ~/.ssh/lab04.pub          # this is what your IaC will inject
```

Выберите **одну** цель и держитесь её в обоих инструментах:

| Цель | Минимальная бесплатная единица | Провайдер |
|--------|---------------------|----------|
| **AWS** | `t3.micro` (12 месяцев бесплатно) | `hashicorp/aws` |
| **GCP** | `e2-micro` (зона always-free) | `hashicorp/google` |
| **Yandex** | 2 vCPU @ 20%, 1 GB | `yandex-cloud/yandex` |
| **DigitalOcean** | студенческий кредит `$200` | `digitalocean/digitalocean` |
| **Docker (локально)** | контейнер | `kreuzwerker/docker` |

> **Никаких учётных данных в коде.** Берите переменные окружения (`AWS_ACCESS_KEY_ID`, `GOOGLE_APPLICATION_CREDENTIALS`, `YC_TOKEN`, …) или общий профиль. В `provider {}` их не вписывайте.

Раскладка каталогов (файлы заполните сами):

```
terraform/
├── main.tf          # provider + resources
├── variables.tf     # inputs
├── outputs.tf       # public IP, ssh command
├── terraform.tfvars # values — GITIGNORED
├── .gitignore
└── README.md

pulumi/
├── __main__.py
├── Pulumi.yaml
├── Pulumi.dev.yaml  # GITIGNORED if it holds secrets
├── requirements.txt
└── .gitignore

docs/LAB04.md
```

---

## Задание 1 — VM на Terraform (4 балла)

**Цель:** доступная VM на вашем HCL, с чистой гигиеной state.

### 1.1 — Напишите строительные блоки HCL

Lecture 4 уже назвала шесть примитивов HCL, которые вы встретите сегодня: `terraform {}`, `provider`, `variable`, `data`, `resource`, `output`. Здесь вы их и пишете.

`ВАША ЗАДАЧА`: соберите `terraform/main.tf`, который покрывает **всё** ниже. Не тащите копию со Stack Overflow: так секреты и попадают в tfstate.

Нужная **форма** HCL (заполните каждый `___` и каждый комментарий `# YOUR TASK:` реальными значениями; готовое решение откуда-то ещё **не** копируйте):

```hcl
# main.tf
terraform {
  required_version = ">= ___"                          # YOUR TASK: pin TF/OpenTofu floor
  required_providers {
    ___ = {                                             # YOUR TASK: provider local name
      source  = "___/___"                               # YOUR TASK: registry source
      version = "~> ___"                                # YOUR TASK: pin a major
    }
  }
}

provider "___" {
  region = var.region    # AWS shape; GCP needs `project` + `region` or `zone`,
                         # Yandex needs `zone`, kreuzwerker/docker takes no region.
                         # Adapt to your chosen provider — see Resources block.
  # ⛔ no inline credentials (access_key/secret_key, JSON SA key body, etc.) — env vars or profile only
}

# YOUR TASK: a data source that returns the latest Ubuntu 24.04 image ID
# (Hint per cloud: aws_ami / google_compute_image / yandex_compute_image).
# Hardcoded AMI IDs go stale within months — that's why this is a data source.
data "___" "ubuntu" {
  # filters / owners / family — YOUR TASK
}

# YOUR TASK: a security group / firewall named "lab4-sg" that allows:
#   - inbound 22/tcp from var.my_ip/32 only        (NEVER 0.0.0.0/0 for SSH)
#   - inbound 80/tcp and 5000/tcp from anywhere   (your app port)
#   - egress all
# Tag it { Name = "lab4-sg", Env = "lab4" }.
resource "___" "lab" {
  # name, description, ingress blocks, egress block, tags — YOUR TASK
}

# YOUR TASK: an SSH key resource — public key from var.ssh_public_key.
resource "___" "lab" {
  # name + public_key — YOUR TASK
}

# YOUR TASK: the VM itself. Required wiring:
#   - ami / image  ← from your data source above
#   - instance_type / machine_type ← var.instance_type
#   - key_name / metadata.ssh-keys ← your key resource
#   - vpc_security_group_ids / network_interface ← your SG/firewall
#   - user_data: a one-liner that runs on first boot and prints
#       "hello from $(hostname)" into /var/log/lab4-boot.log
#   - tags: { Name = "lab4-web", Env = "lab4" }
resource "___" "web" {
  # YOUR TASK
}
```

```hcl
# variables.tf
# YOUR TASK: declare 4 variables — region, instance_type, my_ip, ssh_public_key.
# Use real HCL types (string / number / bool / list). Reminder: `int` is NOT a
# valid HCL type — use `number`. Mark ssh_public_key sensitive = true.
```

```hcl
# outputs.tf
output "public_ip" {
  value = ___                                           # YOUR TASK: resource ref
}

output "ssh_command" {
  value = ___                                           # YOUR TASK: copy-paste-ready string
                                                        # e.g. "ssh -i ~/.ssh/lab04 ubuntu@<ip>"
}
```

```gitignore
# terraform/.gitignore
# YOUR TASK: ignore everything that holds state, secrets, or local downloads.
# Hint: tfstate, tfstate.backup, the .terraform/ dir, *.tfvars, *.pem, *.key.
# DO commit .terraform.lock.hcl — it pins provider versions for reproducibility.
```

### 1.2 — Пройдите жизненный цикл и прочитайте plan

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan      # READ THIS. + create / ~ update / - destroy / -/+ replace
terraform apply tfplan
```

`ВАША ЗАДАЧА`: после `apply` **зайдите по SSH и докажите, что user_data отработал**:

```bash
ssh -i ~/.ssh/lab04 ubuntu@<public_ip> "cat /var/log/lab4-boot.log; hostname"
```

(Если вы выбрали замену на docker-provider: вместо SSH — `docker exec`, и `curl` на порт приложения. Эталонная сдача показывает эквивалентный захват.)

### 1.3 — Управление state: где это ломается

Здесь часы теряют и студенты, и опытные. Слайды 15–17 Lecture 4 — теория, здесь практика.

`ВАША ЗАДАЧА`: в `docs/LAB04.md` вставьте вывод `terraform state list` и ответьте **суммарно тремя предложениями**:

1. Что в `terraform.tfstate` делает его секретом? (Откройте файл. В Git его не кладите.)
2. Почему локальный state — катастрофа для команды из двух инженеров?
3. Что делает `.terraform.lock.hcl` и почему коммитят *его*, а tfstate — никогда?

### 1.4 — Доказательство

**Вставьте в `docs/LAB04.md`:**

- вывод `terraform plan` (очищенный: вырежьте ARN и ID аккаунтов)
- строку итога `terraform apply` (`N to add, M to change, K to destroy`)
- вывод `terraform state list`
- сессию SSH, где видны `cat /var/log/lab4-boot.log` и `hostname`
- вывод `find terraform -maxdepth 2 -type f | sort`, из которого видно, что `.tfstate` в дереве **нет**

---

## Задание 2 — VM на Pulumi (4 балла)

**Цель:** соберите *ту же* инфраструктуру на Python в Pulumi, чтобы было с чем сравнить.

### 2.1 — Снесите VM Terraform или оставьте её

`ВАША ЗАДАЧА`: выберите один путь и запишите его:

- **Путь A** (рекомендуется): сначала `terraform destroy`, затем поднимите VM на Pulumi и оставьте её для Lab 5. Вставьте вывод `destroy`.
- **Путь B**: оставьте VM Terraform для Lab 5, поднимите VM на Pulumi и после скриншота выполните `pulumi destroy`. Не оставляйте две VM работать без записи: именно так биллинговый алерт приходит в 3 ночи.

### 2.2 — Напишите программу Pulumi

```python
# pulumi/__main__.py
# YOUR TASK: imports — pulumi + your provider package (pulumi_aws / pulumi_gcp / …)
import ___
import ___ as ___

# YOUR TASK: read the same three inputs you used in TF, via pulumi.Config().
# pulumi config set myIp $(curl -s ifconfig.me)
# pulumi config set sshPublicKey "$(cat ~/.ssh/lab04.pub)"
# pulumi config set instanceType t3.micro
cfg = ___
my_ip       = ___
ssh_pub     = ___
instance_ty = ___

# YOUR TASK: the same Ubuntu lookup as your TF data source,
# but using the provider's get_<image> function (e.g. aws.ec2.get_ami(...)).
ubuntu = ___

# YOUR TASK: the security group / firewall — same rules as TF.
sg = ___

# YOUR TASK: the key pair — same public key.
key = ___

# YOUR TASK: the instance itself. Pass the same user_data string.
web = ___

# YOUR TASK: two exports — public_ip and a ready-to-paste ssh command.
pulumi.export("public_ip", ___)
pulumi.export("ssh_command", ___)         # hint: .apply(lambda ip: f"ssh ...")
```

```ini
# pulumi/Pulumi.yaml
# YOUR TASK: project name lab04, runtime python, description one line.
```

```text
# pulumi/requirements.txt
# YOUR TASK: pin pulumi >= 3.243 and your provider package
# (pulumi-aws / pulumi-gcp / pulumi-yandex / pulumi-docker).
```

```gitignore
# pulumi/.gitignore
# YOUR TASK: ignore venv/, __pycache__/, and any Pulumi.<stack>.yaml that
# contains plaintext secrets. The stack name itself (Pulumi.dev.yaml) may be
# committed if all values are `pulumi config set --secret`-encrypted.
```

### 2.3 — Пройдите жизненный цикл

```bash
pulumi login --local                             # or `pulumi login` for Pulumi Cloud
pulumi stack init dev
pulumi config set aws:region eu-central-1        # adapt to your provider
pulumi config set myIp $(curl -s ifconfig.me)
pulumi config set --secret sshPublicKey "$(cat ~/.ssh/lab04.pub)"

pulumi preview                                   # ≈ terraform plan
pulumi up --yes
# ... ssh in and verify ...
pulumi destroy --yes                             # if Path B above, or before Lab 5 if Path A
```

### 2.4 — Доказательство

**Вставьте в `docs/LAB04.md`:**

- вывод `pulumi preview` (очищенный)
- строку итога `pulumi up`
- сессию SSH на VM Pulumi (или эквивалент на docker-стенде)
- доказательство `terraform destroy` (путь A) или доказательство `pulumi destroy` (путь B)

### 2.5 — Сравнение: ради этого вы делаете оба

`ВАША ЗАДАЧА`: в `docs/LAB04.md` напишите по 3–5 предложений на каждый пункт:

1. **Читаемость** — какой файл через неделю было легче пробежать глазами?
2. **Логика и циклы** — представьте ту же VM × `["dev","staging","prod"]`. Какой инструмент для этого оказался естественнее?
3. **Отладка** — когда упали `apply` или `up`, чьё сообщение об ошибке быстрее вывело на исправление?
4. **State и секреты** — куда каждый инструмент кладёт state? Кто шифрует секреты по умолчанию?
5. **Когда вы выберете каждый** — по одному конкретному сценарию на инструмент, из вашего опыта на этой неделе.

---

## Задание 3 — документация (2 балла)

В `docs/LAB04.md` по порядку должно быть:

1. **Провайдер и цель** — какое облако (или docker-стенд), какой регион или зона, размер инстанса и заметка о стоимости в одну строку (должно выйти `$0`).
2. **Реализация Terraform** — какая версия, раскладка файлов, ответ из трёх предложений про state из п. 1.3, очищенный вывод жизненного цикла, доказательство SSH.
3. **Реализация Pulumi** — язык и версия, вывод жизненного цикла, доказательство SSH.
4. **Terraform и Pulumi** — сравнение из пяти пунктов из п. 2.5.
5. **Подготовка к Lab 5 и уборка** — какая VM (если какая-то) остаётся жить, или как Lab 5 поднимет новую из вашего IaC. Плюс доказательство `destroy` для всего, что вы снесли.

> Весь вывод терминала очищайте: никаких полных ID аккаунтов, ключей доступа и API-токенов. Всё, что восстановили по памяти, а не скопировали из терминала, помечайте `(illustrative)`.

---

## Бонус — CI/CD для IaC и удалённый state (2 балла)

### Часть 1 — проверка в GitHub Actions (1 балл)

`ВАША ЗАДАЧА`: создайте `.github/workflows/terraform-ci.yml`, который запускается на PR, затрагивающих `terraform/**`, и гоняет `fmt -check -recursive`, `init -backend=false`, `validate` и `tflint`. Только официальные action: `hashicorp/setup-terraform` (или `opentofu/setup-opentofu`) и `terraform-linters/setup-tflint`. Долгоживущих облачных кредов нет: `-backend=false` не поднимает state.

Каркас — заполните пропуски:

```yaml
# .github/workflows/terraform-ci.yml
name: terraform-ci
on:
  pull_request:
    paths:
      - 'terraform/**'
      - '.github/workflows/terraform-ci.yml'

jobs:
  validate:
    runs-on: ubuntu-24.04
    defaults:
      run: { working-directory: terraform }
    steps:
      - uses: actions/checkout@v4
      - uses: ___                                       # YOUR TASK: setup-terraform action + version pin
        with:
          terraform_version: "___"                      # YOUR TASK: the version from the table above
      # YOUR TASK: 4 steps — fmt -check -recursive, init -backend=false, validate, tflint.
      # tflint needs its own setup action; add a minimal terraform/.tflint.hcl too.
```

**Докажите:**
- Откройте PR, который трогает `terraform/`. Workflow запускается и остаётся зелёным.
- Сверху запушьте коммит, который ломает `fmt`. Workflow становится **красным** на шаге `fmt -check`. Скриншот или вставьте падающий лог.
- Троньте файл вне `terraform/**`. Workflow **не** запускается.

### Часть 2 — Переезд на удалённый state с блокировкой (1 балл)

`ВАША ЗАДАЧА`:

1. Создайте бакет для state: S3 с **версионированием и шифрованием**, GCS с версионированием или Yandex Object Storage. Руками или крошечным отдельным конфигом Terraform. Запишите, как именно.
2. Добавьте блок `backend "s3"` или `backend "gcs"` в конфиг `terraform/`:
   - Для S3 ставьте `use_lockfile = true` (нативная блокировка state в TF 1.10+: отдельная таблица DynamoDB больше не нужна).
   - Для GCS блокировка встроена и включается сама.
   - **Всегда** ставьте `encrypt = true`.
3. Выполните `terraform init -migrate-state` и убедитесь, что локальный `terraform.tfstate` уехал в бакет.
4. Снова запустите `terraform plan`. Он должен сообщить **No changes** (state переехал чисто, а не был импортирован заново).
5. В `docs/LAB04.md` за 2–3 предложения объясните, **почему блокировка не даёт двум инженерам испортить state** и как выглядит поломка без неё.

---

## Как сдавать

```bash
git switch -c lab04
git add terraform/ pulumi/ docs/LAB04.md
git add .github/workflows/terraform-ci.yml      # bonus only
git commit -m "feat(lab04): IaC with Terraform + Pulumi"
git push -u origin lab04
```

Откройте **два** PR:

- `your-fork:lab04` → `course-repo:master` *(на ревью)*
- `your-fork:lab04` → `your-fork:master`

Чеклист PR:

```text
- [ ] terraform/ provisions a VM end-to-end; SSH proven
- [ ] pulumi/ recreates the same infra; SSH proven
- [ ] No *.tfstate, *.tfvars, .terraform/, or Pulumi secret stack file in the diff
- [ ] .terraform.lock.hcl IS committed
- [ ] docs/LAB04.md has all 5 sections + the TF-vs-Pulumi comparison
- [ ] At most ONE VM left running, and it's named in docs (for Lab 5)
- [ ] Bonus (optional): terraform-ci.yml proven green+red; remote state migrated
```

---

## Критерии приёмки

### Задание 1 — Terraform (4 балла)
- ✅ Провайдер выбран, аутентификация через env или профиль (кредов в коде нет)
- ✅ источник `data` для образа Ubuntu (не захардкоженный AMI или ID)
- ✅ security group или firewall пускает **SSH 22 только на ваш IP/32**
- ✅ на месте все шесть примитивов: `terraform{}`, `provider`, `variable`, `resource`, `data`, `output`
- ✅ `user_data` отрабатывает при первой загрузке и пишет в `/var/log/lab4-boot.log`
- ✅ сессия SSH в `docs/LAB04.md` показывает строку boot-лога и hostname
- ✅ `.gitignore` верный: tfstate игнорируется, `.terraform.lock.hcl` закоммичен
- ✅ ответ из трёх предложений про управление state есть

### Задание 2 — Pulumi (4 балла)
- ✅ та же инфраструктура собрана заново на Python в Pulumi (или на TS, Go, Java — Python рекомендуется)
- ✅ `pulumi preview` и `pulumi up` проходят; SSH доказан
- ✅ живой остаётся максимум один из двух VM, и это записано
- ✅ сравнение Terraform и Pulumi из пяти пунктов написано

### Задание 3 — документация (2 балла)
- ✅ в `docs/LAB04.md` есть все 5 разделов
- ✅ выводы очищены, доказательство уборки приложено

### Бонус (2 балла)
- ✅ `terraform-ci.yml` гоняет `fmt`, `init`, `validate` и `tflint`, ограничен путями `terraform/**`, зелёный и красный прогоны доказаны — **1 балл**
- ✅ backend удалённый, зашифрованный и **с блокировкой**; миграция подтверждена; зачем нужна блокировка, объяснено — **1 балл**

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — VM Terraform | **4** | Рабочая инфраструктура, data source, ограниченный SSH, переменные и output, чистая гигиена state |
| **Задание 2** — VM Pulumi | **4** | Та же инфраструктура в Pulumi, SSH доказан, сравнение написано |
| **Задание 3** — документация | **2** | Все 5 разделов, очищенный вывод, доказательство уборки |
| **Бонус** — CI для IaC | **1** | Workflow с фильтром путей: fmt, validate, lint; доказаны зелёный и красный прогоны |
| **Бонус** — удалённый state | **1** | Зашифрованный backend с блокировкой, миграция доказана, обоснование написано |
| **Итого** | **10 + 2** | 10 основных + 2 бонуса |

**Жёсткие условия:** только free-tier · никаких секретов и state в Git · доказательство SSH обязательно · для Lab 5 остаётся максимум одна VM.

---

## Материалы

<details>
<summary>📚 Terraform / OpenTofu</summary>

- [Документация Terraform](https://developer.hashicorp.com/terraform/docs) · [Registry](https://registry.terraform.io/)
- [OpenTofu](https://opentofu.org) — форк-замена; гайд по миграции
- [Backend S3](https://developer.hashicorp.com/terraform/language/settings/backends/s3) · [Блокировка state](https://developer.hashicorp.com/terraform/language/state/locking) · [`terraform import`](https://developer.hashicorp.com/terraform/cli/import)
- [tflint](https://github.com/terraform-linters/tflint)
- *Terraform: Up & Running* — Brikman (4-е изд., 2024 — там есть и OpenTofu)

</details>

<details>
<summary>📦 Pulumi</summary>

- [Документация Pulumi](https://www.pulumi.com/docs/) · [Registry](https://www.pulumi.com/registry/)
- [Pulumi и Terraform](https://www.pulumi.com/docs/concepts/vs/terraform/) · [Секреты](https://www.pulumi.com/docs/concepts/secrets/)

</details>

<details>
<summary>☁️ Краткие справки по провайдерам</summary>

- **AWS** — [документация провайдера](https://registry.terraform.io/providers/hashicorp/aws/latest/docs); аутентификация через `AWS_ACCESS_KEY_ID` / `~/.aws/credentials`. Owner ID Ubuntu: `099720109477`.
- **GCP** — [документация провайдера](https://registry.terraform.io/providers/hashicorp/google/latest/docs); аутентификация через `gcloud auth application-default login`. Включите Compute Engine API. SSH-ключ в metadata инстанса: `metadata = { ssh-keys = "ubuntu:${var.ssh_public_key}" }`.
- **Yandex** — [документация провайдера](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs); JSON-ключ сервисного аккаунта. Форма free-tier: `standard-v3`, `core_fraction = 20`, 1 GB RAM.
- **Стенд Docker** — `kreuzwerker/docker ~> 3.0`; `docker_image` и `docker_container`. SSH нет; `docker exec` и curl на опубликованный порт.

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных пробных прогонов)</summary>

- **`type = int` в HCL невалиден.** Типы переменных: `string`, `number`, `bool`, `list`, `map`, `set`, `object`, `tuple`, `any`. `int` уронит `terraform validate` с сообщением "Invalid type specification". Берите `number`.
- **Закоммиченный `terraform.tfstate`.** В нём ID ресурсов, приватные IP, иногда секреты открытым текстом, а конфликт слияния JSON *не восстановить*. Первая строка `.gitignore`.
- **Забытый `.terraform.lock.hcl`.** **Его как раз коммитят.** Он фиксирует хеши провайдеров, чтобы `init` коллеги разрешился в тот же патч `aws ~> 5.x`, что и у вас. Без файла двое инженеров могут сделать apply с чуть разными версиями провайдера и получить разную инфраструктуру.
- **Креды в `provider {}`.** Code Spaces (2014) удалили за 12 часов ровно из-за этого. Переменные окружения или общий профиль, в коде их нет. В `.tfvars` тоже не кладите: этот файл студенты коммитят случайно.
- **Egress с default-deny на новой security group AWS.** У ресурса Terraform `aws_security_group` *нет неявного egress*, в отличие от консоли AWS. Блок `egress` напишите сами, иначе VM не достанет интернет для `apt`, user_data зависнет, а `apply` отрапортует успех на сломанном хосте.
- **Расчёт на VPC по умолчанию.** На аккаунтах AWS в первый день он есть, новые организации удаляют его по требованиям compliance. Либо найдите его через `data "aws_vpc" "default" { default = true }`, либо создайте свой. Не рассчитывайте, что он на месте.
- **SSH настежь.** `0.0.0.0/0` на порту 22, и через несколько часов VM уже в ботнете. Только `var.my_ip/32`. (Свой IP: `curl -s ifconfig.me`.)
- **Захардкоженный AMI ID.** `ami-0c55b159cbfafe1f0` *один раз* был AMI Ubuntu 22.04 в eu-central-1. AMI меняются, когда Canonical выкатывает патчи; захардкоженные ID через месяцы отвечают 404. Фильтруйте `data "aws_ami"` по шаблону имени.
- **Два одновременных `apply` на локальном state.** JSON state записывается наполовину, Terraform его не разбирает, а восстановление — правка руками, которая сама запрещена. Бонус существует именно из-за этого.

</details>

---

## Что дальше

| Lab | Что она делает с вашей VM из Lab 4 |
|---:|---|
| 5 | Роли Ansible заходят по SSH и ставят Docker и приложение из Lab 1–3 |
| 6 | Блоки Ansible `blocks`/`rescue`, теги, выкладка Compose по тому же пути SSH |
| 9 | k3d заменяет модель VM, собранной руками: тот же подход IaC, другая единица |
| 13 | ArgoCD забирает выкладки; кластер по-прежнему описывает ваш IaC |

> **Запомните:** если с чистого листа не выходит `git clone && terraform apply`, это не IaC, а история кликов в консоли. Никаких секретов в коде, никакого state в Git. Пять правил со слайда 22 Lecture 4 оплачены живыми деньгами (слайд 23).
