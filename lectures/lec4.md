# 📌 Lecture 4 — Инфраструктура как код: Terraform и Pulumi

## 📍 Slide 1 – 🏗️ Добро пожаловать в инфраструктуру как код

* 🐶 **Lecture 1 обещала скот, не питомцев** — сегодня делаем
* 📝 **IaC** = облако живёт в Git, а не во вкладках чьего-то браузера
* 🌍 **Terraform + Pulumi** = два способа, которыми современные команды создают инфраструктуру
* 🎯 К концу: одной командой поднимаете VM и сносите её — и смотрите diff до apply

```mermaid
flowchart LR
  Manual[🖱️ Клики в консоли] -->|IaC| Code[📝 Код в Git]
  Code --> Plan[📋 plan]
  Plan --> Apply[🚀 apply]
  Apply --> Cloud[☁️ Воспроизводимое облако]
```

> 🔗 **Связка с Lab 4:** одну и ту же VM вы соберёте *дважды* — раз на Terraform, раз на Pulumi — и почувствуете разницу сами. Бонус добавляет workflow GitHub Actions, который гоняет `plan` на каждый PR.

---

## 📍 Slide 2 – 🎯 Чему научитесь

| # | Результат |
|---|---------|
| 1 | 🧠 Объяснить, почему серверы-снежинки — риск для бизнеса |
| 2 | 📝 Написать рабочую конфигурацию Terraform: provider, resource, variable, output |
| 3 | 🔄 Уверенно пройти цикл `init → plan → apply → destroy` |
| 4 | 📦 Перенести state с локального диска на удалённый backend с блокировкой — и объяснить зачем |
| 5 | 🐍 Поднять ту же инфраструктуру на Pulumi на Python |
| 6 | ⚖️ Выбрать Terraform, Pulumi или OpenTofu по команде и задаче |

---

## 📍 Slide 3 – 🔧 Стек, зафиксированный на май 2026

| Инструмент | Версия | Заметки |
|------|---------|-------|
| 🌍 **Terraform** | **1.15.3** | Релиз 13 мая 2026; BSL-1.1 с **10 августа 2023** |
| 🌱 **OpenTofu** | **1.12.0** | Релиз 14 мая 2026; MPL-2.0, Linux Foundation; CLI-замена 1:1 |
| 📦 **Pulumi** | **3.243.0** | Май 2026; Apache-2.0 целиком |
| ☁️ **AWS provider** | v5.x | В большинстве примеров Lab 4 |
| ☁️ **GCP provider** | v6.x | Альтернатива для Lab 4 |
| ☁️ **Yandex provider** | v0.115+ | По умолчанию для лаб только под Россию |

> 🔧 **`terraform` можно менять на `tofu` в любой команде на этих слайдах.** HCL тот же. Выбор — по лицензии и по тому, как принято в команде. Лабы принимают оба.

---

## 📍 Slide 4 – ❓ Главный вопрос

* 📊 Отчёт Flexera *State of the Cloud* за 2024: **89%** предприятий сидят в нескольких облаках — и **70%+** говорят о неконтролируемом drift (расхождение задуманного и факта)
* ⏱️ Ручной провижининг VM: **часы или дни** на один сервер, и ещё дни, чтобы копия совпала
* 💥 Большинство аварий вызывают **изменения** — неотслеженное изменение хуже всех

> 💬 *«На стейджинге работало — стейджинг был другим.»* — Каждый инженер эксплуатации, всегда

**🤔 Подумайте:**
* Сможете завтра собрать прод-облако с нуля?
* Если тот, кто его собрал, ушёл сегодня утром — через сколько вы застрянете?

---

## 📍 Slide 5 – 🐶 Питомцы и скот (Bill Baker, Microsoft, 2012)

* 🐶 **Питомцы** — с именем (`web-prod-01`), их выхаживают, незаменимы, заболел — простой
* 🐄 **Скот** — с номером (`web-001..099`), одинаковые, расходные, заболел — заменили
* ☁️ **Cloud-native = мышление скота.** Lecture 1 это уже наметила; сегодня делаем взаправду.

```mermaid
flowchart LR
  Pets[🐶 Питомцы<br/>вход по SSH, правка руками]
  Cattle[🐄 Скот<br/>в коде, замена за минуты]
  Pets -.->|IaC + неизменяемая инфра| Cattle
```

> 📖 Метафору придумал Bill Baker во внутреннем докладе Microsoft в 2012 году, позже её популяризировал Randy Bias из CloudScaling. Двенадцать лет спустя её всё ещё берёт каждая книга по облачной архитектуре. Работает, потому что это правда.

---

## 📍 Slide 6 – 😱 Снежинки, drift и bus factor

Три отказа, которые убирает IaC:

| Отказ | Как выглядит | Чем платите |
|---------|---------------------|---------------|
| ❄️ **Сервер-снежинка** | «Не трогай web-03, что там — знает только Петя» | Не масштабируется, не повторить, не заменить |
| 📉 **Drift конфигурации** | Стейджинг и прод совпадали 6 месяцев назад — уже нет | Баги только в проде, часы на отладку |
| 🚌 **Bus factor = 1** | Архитектура облака в голове у одного человека | Проект встаёт, когда он в отпуске или уволился |

> 🔥 **Жёстко:** если прод нельзя *удалить и собрать заново* из git-тега, у вас нет IaC — у вас история команд `aws`.

---

## 📍 Slide 7 – 💡 Что такое инфраструктура как код на самом деле

* 📝 **Определение:** инфраструктурой управляют и создают её через *машиночитаемые файлы конфигурации под контролем версий*, а не кликами в консоли и разовыми скриптами.
* 🔄 Облако становится **артефактом сборки**, а не средой, собранной руками
* 📜 Каждое изменение идёт тем же путём, что код приложения: PR → ревью → CI → мерж → выкат

```mermaid
flowchart LR
  Code[📝 .tf / .py в Git] --> CI[🧪 CI: plan]
  CI --> Review[👀 Ревью PR]
  Review --> Merge[🚀 Мерж → apply]
  Merge --> Cloud[☁️ Живая инфраструктура]
  Cloud -->|diff| Code
```

> 📖 *Infrastructure as Code* — Kief Morris (2-е изд., 2020). Главная книга по теме; четыре принципа ниже — оттуда.

---

## 📍 Slide 8 – 📐 Четыре принципа (Morris)

1. ♻️ **Воспроизводимость** — один и тот же код → одна и та же инфраструктура, каждый раз
2. 🔁 **Одноразовость** — снести и собрать заново должно быть рутиной, не подвигом
3. 📜 **Согласованность** — dev, staging и prod различаются только переменными
4. 🔍 **Видимость** — `git log` отвечает: кто, что, когда и зачем поменял

> 💡 **Нет хотя бы одного пункта — у вас автоматизация, не IaC.** Bash-скрипты, которые создают VM, — не IaC. Это рецепт, и прогнать его ещё раз безопасно уже нельзя.

---

## 📍 Slide 9 – 🔀 Декларативное и императивное

```mermaid
flowchart LR
  Decl[📝 Декларативно<br/>'Хочу 3 VM'] --> Tool1[🤖 Инструмент сам решает КАК]
  Imp[🔧 Императивно<br/>'Создай VM 1, потом VM 2, потом VM 3'] --> Tool2[🏃 Каждый шаг задаёте вы]
```

| Аспект | 📝 Декларативное | 🔧 Императивное |
|--------|---------------|---------------|
| Вы пишете | Желаемое конечное состояние | Последовательность шагов |
| Идемпотентность | Встроена — повторный прогон безопасен | Её пишете вы |
| Примеры | Terraform HCL, CloudFormation, Kubernetes YAML | Bash-скрипты, вызовы AWS SDK, Pulumi (в основном) |
| Как ломается | Инструмент не может просчитать хитрую миграцию | Шаг 7 падает на повторном прогоне |

> 🤔 **Поворот Pulumi:** код императивный, но движок Pulumi до обращения к облаку сводит его в *декларативный* граф желаемого состояния. Лучшее из двух миров.

---

## 📍 Slide 10 – 🛠️ Ландшафт инструментов IaC (май 2026)

| Инструмент | Лицензия | Язык | Мультиоблако | State |
|------|---------|----------|-------------|-------|
| 🌍 **Terraform 1.15.3** | BSL-1.1 (HashiCorp) | HCL | ✅ 3000+ провайдеров | Local / S3 / TF Cloud |
| 🌱 **OpenTofu 1.12.0** | MPL-2.0 (Linux Foundation) | HCL (совместим с Terraform) | ✅ Те же провайдеры | Local / S3 / OCI |
| 📦 **Pulumi 3.243** | Apache-2.0 | Python, TS, Go, C#, Java, YAML | ✅ AWS, GCP, Azure, K8s, … | Pulumi Cloud / S3 / local |
| ☁️ **AWS CloudFormation** | Проприетарная | YAML / JSON | ❌ Только AWS | Ведёт AWS |
| ☁️ **AWS CDK / GCP CDK** | Apache-2.0 | TS, Python, Java, Go | ❌ Своё облако | Под капотом CloudFormation / Deployment Manager |
| ☸️ **Crossplane** | Apache-2.0 | Kubernetes YAML | ✅ через провайдеры | Kubernetes etcd |

> 🔗 **Лекция про Terraform и Pulumi** — их и берёт Lab 4. OpenTofu появится на слайде 21.

---

## 📍 Slide 11 – 📜 Краткая история

* 📅 **Июль 2014** — Mitchell Hashimoto и Armon Dadgar выпускают **Terraform v0.1** в HashiCorp. Первый серьёзный мультиоблачный инструмент IaC.
* 📅 **Сентябрь 2019** — команда Joe Duffy выпускает **Pulumi v1.0**. Первый инструмент IaC на языках общего назначения.
* 📅 **10 августа 2023** — HashiCorp переводит Terraform с MPL-2.0 на **Business Source License (BSL)**. Вендорам, которые строят продукты поверх Terraform, внезапно нужен платный договор.
* 📅 **Август 2023** — Gruntwork, env0, Spacelift и Terragrunt форкают Terraform 1.5 → **OpenTofu**.
* 📅 **Сентябрь 2023** — OpenTofu **отдают Linux Foundation**: управление без привязки к вендору; первый стабильный релиз 1.6 — январь 2024.
* 📅 **Декабрь 2023** — Linux Foundation анонсирует **OpenBao**, параллельный форк сообщества для Vault (та же история с BSL).
* 📅 **Май 2026 (сейчас)** — Terraform 1.15.3 и OpenTofu 1.12.0 совместимы по CLI; большинство компаний берёт что-то одно, не оба сразу.

> 🔥 Переход на BSL — главное событие десятилетия в IaC. Поэтому лекция держит OpenTofu на равных.

---

## 📍 Slide 12 – 🌍 Workflow Terraform

```mermaid
flowchart LR
  Write[📝 Пишем .tf] --> Init[🔧 init]
  Init --> Plan[📋 plan]
  Plan --> Apply[🚀 apply]
  Apply --> Destroy[💥 destroy]
  Plan -.->|ревью| Plan
```

```bash
terraform init      # ⬇️ download provider binaries to .terraform/
terraform plan      # 🔍 diff between code, state, and reality
terraform apply     # 🚀 make reality match code
terraform destroy   # 💥 delete everything in this state
```

> ⚠️ **`plan` священен.** Читайте его. Каждый раз. Особенно перед `apply` в проде. CI должен публиковать plan в PR — бонус Lab 4 это подключает.

---

## 📍 Slide 13 – 🧱 Строительные блоки HCL

```hcl
# ☁️ provider — which cloud and how to authenticate
provider "aws" {
  region = var.region
}

# 📊 variable — input
variable "region" {
  description = "AWS region"
  type        = string
  default     = "eu-central-1"
}

# 🏗️ resource — something we create and own
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"   # Ubuntu 22.04 LTS, eu-central-1
  instance_type = "t3.micro"
  tags = { Name = "lab4-web", Env = "lab" }
}

# 🔍 data — read-only lookup of something we don't own
data "aws_vpc" "default" { default = true }

# 📤 output — value to surface after apply (and for other modules)
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

> 📝 Пять примитивов — `provider`, `resource`, `variable`, `output`, `data` — это **90%** реального кода Terraform. Модуль — просто пачка из них.

---

## 📍 Slide 14 – 🔄 Вывод `plan`: читайте как diff

```text
$ terraform plan

Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t3.micro"
      + public_ip     = (known after apply)
      + tags          = {
          + "Env"  = "lab"
          + "Name" = "lab4-web"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

| Символ | Значение |
|:------:|---------|
| `+` | Создать |
| `-` | Уничтожить (риск потери данных!) |
| `~` | Обновить на месте |
| `-/+` | Уничтожить и создать заново (риск потери данных!) |

> 🚨 **`-/+` на ресурсе базы = данные сейчас умрут.** В проде читайте каждый plan, каждый раз.

---

## 📍 Slide 15 – 📦 State: без этого Terraform не работает

* 🗺️ **State** = JSON-карта от вашего кода (`aws_instance.web`) к реальному объекту в облаке (`i-0abc123…`)
* 🔍 Без state Terraform не знает, что уже есть: каждый apply пытался бы создать всё заново
* ⚠️ В state лежат **чувствительные данные**: пароли, приватные IP, иногда приватные ключи. Обращайтесь с ним как с секретом

```mermaid
flowchart LR
  Code[📝 код .tf] --> TF[🌍 terraform]
  State[📦 terraform.tfstate] --> TF
  Cloud[☁️ Живое облако] --> TF
  TF -->|plan diff| You[👀 Вы]
```

> ⚠️ **Никогда не правьте `terraform.tfstate` руками.** Используйте подкоманды `terraform state` (`mv`, `rm`, `import`).

---

## 📍 Slide 16 – 🌐 Локальный state и удалённый

| | 📁 Локальный (`terraform.tfstate` на диске) | 🌐 Удалённый (S3, GCS, TF Cloud, …) |
|-|----------------------------------------|----------------------------------|
| Соло-прототип | ✅ Нормально | Избыточно |
| Команда от 2 | ❌ Два инженера делают apply → гонка, state битый | ✅ Блокировка выстраивает apply в очередь |
| Восстановление после аварии | ❌ Умер ноутбук → state потерян → облако сирота | ✅ Объектное хранилище с версиями |
| Секреты в state | 🟡 Открытым текстом на диске | ✅ Шифрование at rest (KMS) |
| CI/CD | ❌ У каждого runner свой устаревший state | ✅ Общий канонический state |

**📦 Пример удалённого backend (блокировка на S3 + DynamoDB):**

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-tf-state"
    key            = "lab4/terraform.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "tf-locks"     # 🔒 prevents concurrent applies
    encrypt        = true           # 🔐 SSE-KMS
  }
}
```

> 🔗 **Lab 4 начинает с локального state специально** — чтобы было видно, как ломается одновременный apply двух студентов. Бонус переносит state на S3.

---

## 📍 Slide 17 – 🚨 Ловушки state (где и студенты, и профи наступают на грабли)

1. 🙅 **Коммит `terraform.tfstate` в Git** — там секреты, а конфликты мержа в JSON не разобрать. В `.gitignore` — в первый же день.
2. 🔓 **На удалённом state нет блокировки** — два `apply` одновременно гоняются; state портится; чинить руками.
3. 📉 **Drift** — кто-то кликает в консоли AWS; Terraform не в курсе; следующий `apply` откатит или упадёт.
   * Как поймать: `terraform plan` в ночном job CI — любой ненулевой diff = drift.
4. 🔄 **Осиротевшие ресурсы** — сделали `rm -rf` модуля; Terraform забыл его ресурсы; счета идут вечно.
   * Перед удалением кода выполните `terraform state list`.
5. 📥 **Нужно взять под управление облачные объекты, которые создавали не через Terraform?**

```bash
# Adopt an existing EC2 instance into Terraform state
terraform import aws_instance.web i-0abc123def456

# Or, with Terraform 1.5+, declare it once and let TF import on apply
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}
```

> 🔥 **Поиск drift — половина пользы IaC.** Скот, который ушёл с ранчо, всё ещё скот — пока кто-нибудь его не переименует.

---

## 📍 Slide 18 – 📦 Pulumi: инфраструктура на любимом языке

* 🐍 Пишите инфраструктуру на **Python, TypeScript, Go, C#, Java** или YAML
* 🔧 Циклы, условия, функции, классы, юнит-тесты — привычные инструменты разработчика работают
* 📦 State лежит в Pulumi Cloud (бесплатно одному, платно команде) или у себя (S3, Azure Blob)
* 🔐 Секреты в state **шифруются по умолчанию** (галочку включать не надо)

```python
# 🐍 pulumi/__main__.py — same VM as the Terraform example
import pulumi
import pulumi_aws as aws

web = aws.ec2.Instance("web",
    ami="ami-0c55b159cbfafe1f0",
    instance_type="t3.micro",
    tags={"Name": "lab4-web", "Env": "lab"})

pulumi.export("public_ip", web.public_ip)
```

```bash
pulumi up        # equivalent to terraform plan + apply
pulumi preview   # plan only
pulumi destroy   # tear down
```

> 🔗 **Lab 4 Task 2** переписывает вашу VM с Terraform на Pulumi Python — разницу почувствуете за 30 минут.

---

## 📍 Slide 19 – 🧠 Зачем писать инфраструктуру кодом

```python
# 🐍 Loops, conditionals, and reuse — natural in Python
for env in ["dev", "staging", "prod"]:
    aws.ec2.Instance(f"web-{env}",
        ami="ami-0c55b159cbfafe1f0",
        instance_type="t3.micro" if env != "prod" else "t3.large",
        tags={"Env": env})
```

```hcl
# 🌍 The same thing in HCL — works, but the syntax is less natural
locals {
  envs = { dev = "t3.micro", staging = "t3.micro", prod = "t3.large" }
}
resource "aws_instance" "web" {
  for_each      = local.envs
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value
  tags          = { Env = each.key }
}
```

> 💡 **Инструмент выбирает команда.** Python-команды с сильной разработкой часто берут Pulumi. Платформенные команды на десятках провайдеров, которым нужна экосистема HashiCorp, обычно берут Terraform или OpenTofu. **Lab 4 заставляет сделать оба варианта, чтобы вы выбрали сами.**

---

## 📍 Slide 20 – ⚖️ Terraform, Pulumi и OpenTofu

| Аспект | 🌍 Terraform 1.15.3 | 🌱 OpenTofu 1.12.0 | 📦 Pulumi 3.243 |
|--------|--------------------|------------------|------------------|
| 📜 Лицензия | BSL-1.1 | MPL-2.0 | Apache-2.0 |
| 🏢 Кто ведёт | HashiCorp (IBM, 2025) | Linux Foundation | Pulumi Corp |
| 📝 Язык | HCL | HCL (совместим с Terraform) | Python, TS, Go, C#, Java, YAML |
| 🧠 Парадигма | Декларативная | Декларативная | Императивный код → декларативный граф |
| 🔄 Логика | `for_each`, `count`, `dynamic` | То же + доработки `for_each` | Целый язык программирования |
| 🧪 Юнит-тесты | Terratest (Go) | То же | Нативно (pytest, jest, go test) |
| 📦 State по умолчанию | Local / S3 / TF Cloud | Local / S3 / OCI | Pulumi Cloud / S3 |
| 🔐 Секреты в state | Открытый текст (если backend не шифрует) | Открытый текст + своё шифрование с 1.7+ | Шифрование по умолчанию |
| 🌐 Сообщество | Самое большое (миллионы людей) | Быстро растёт (за ним LF) | Меньше, но своё |

> 🎯 **В курсе по умолчанию:** Lab 4 принимает любой из трёх. Берите то, чем команда пользуется вживую.

---

## 📍 Slide 21 – 🌱 OpenTofu: зачем вам этот форк

* 📅 **10 августа 2023** — HashiCorp меняет лицензию Terraform с MPL-2.0 (разрешительная) на BSL-1.1 (исходники открыты, коммерция ограничена)
* 😡 Компании, чьи продукты *зависят* от Terraform (Spacelift, env0, Gruntwork), получают вопрос выживания
* 🍴 **Август 2023:** они форкают Terraform 1.5 и публикуют его как **OpenTofu** под MPL-2.0
* 🏛️ **Сентябрь 2023:** OpenTofu **отдают Linux Foundation** → управление без вендора
* 🏛️ **Декабрь 2023:** Linux Foundation анонсирует **OpenBao** — тот же приём с форком, теперь на Vault
* ✅ **Сегодня:** OpenTofu 1.12.0 — замена Terraform 1.15.3 один в один: тот же HCL, те же провайдеры, тот же реестр (почти)
* 💼 **2025:** IBM покупает HashiCorp; Terraform остаётся на BSL — форк навсегда

```bash
# Identical workflows
terraform init && terraform plan && terraform apply
tofu init && tofu plan && tofu apply

# Yes, you can copy a Terraform repo and run `tofu` against it
```

> 🔥 В 2026 разумный ответ такой: *скорее всего* OpenTofu для новых проектов, Terraform — если уже платите HashiCorp за поддержку. Выбирайте сознательно.

---

## 📍 Slide 22 – 🔐 Безопасность: не станьте заголовком

**Пять правил. Выучите.**

1. 🚫 **Никогда не коммитьте учётные данные.** Ни в `.tf`, ни в `.tfvars`, ни в блоках `provider {}`.

```hcl
# ❌ NEVER
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# ✅ Let the provider read env vars or AWS profile
provider "aws" {
  region = var.region
}
```

2. 🙈 **Никогда не коммитьте `terraform.tfstate` и `*.tfstate.backup`** — там уже расшифрованные секреты. В `.gitignore` — первой строкой.
3. 🌐 **Удалённый backend: шифрование и блокировка** — S3 + SSE-KMS + блокировка DynamoDB, GCS с версиями или Pulumi Cloud.
4. 🔑 **В CI в облако ходят через OIDC, не через долгоживущие ключи.** GitHub Actions → AWS / GCP через `aws-actions/configure-aws-credentials` или `google-github-actions/auth` — секретов в репозитории нет. Никогда.
5. 🔍 **Сканируйте код IaC.** `tfsec`, `checkov`, `terrascan` — встройте их в CI. Ловят публичные бакеты S3, дыры без шифрования, wildcard в IAM.

> 🔗 **Связка с курсом:** бонус Lab 4 добавляет на каждый PR шаг `terraform plan` и `checkov`. Электив DevSecOps копает глубже.

---

## 📍 Slide 23 – 💥 Реальные инциденты: зачем эти правила

* 🏦 **Code Spaces (2014)** — учётные данные в репозитории → захват консоли AWS → атакующий удалил S3 + EC2 + RDS → компания *буквально закрылась* за 12 часов. Хрестоматия: не коммитьте учётные данные.
* 🚗 **Uber (2016)** — ключи AWS захардкожены в приватном репозитории GitHub → утащили записи 57 млн пользователей → мировое на $148 млн. Приватный ≠ безопасный.
* 🏭 **2024 SAP Concur** — внутренний state Terraform в незашифрованном бакете S3 годами сливал PII сотрудников.
* 🐍 **2025 SnakeKeylogger / Codefinger** — атакующие нашли слитые токены Pulumi Cloud в публичных репозиториях, перечислили бакеты S3 и вымогали выкуп через SSE-C.
* 📉 **Knight Capital (2012, до IaC)** — ручной выкат на 7 серверов из 8, на восьмом остался старый код → минус $440 млн за 45 минут. Руками = drift = катастрофа.

> 💡 **Правила со слайда 22 остановили бы каждый из этих случаев.** За правила уже заплатили живыми деньгами.

---

## 📍 Slide 24 – 🎯 Главное

1. 🏗️ **Инфраструктура как код = облако живёт в Git** — воспроизводимо, одноразово, согласованно, видно
2. 🐄 **Скот, не питомцы** — имя сервера — роль и номер, не кличка
3. 🔄 **Читайте `plan`** — каждый раз, особенно перед apply в проде
4. 📦 **Удалённый state с блокировкой и шифрованием с первого дня** — в любой команде
5. 🌍 **Terraform/OpenTofu** — большие экосистемы и декларативная простота; **Pulumi** — команды, которые живут в коде, и сложная логика
6. 🌱 **OpenTofu есть, потому что лицензия важна** — знайте историю, выбирайте сами
7. 🔐 **Секретов в коде нет. State не в Git. Для CI — OIDC.** Три правила, без торга.

> 💡 **Если с нуля не проходит `git clone && terraform apply`, это не IaC, а проклятая таблица кликов.**

---

## 📍 Slide 25 – 🧠 Сдвиг мышления

| 😰 Раньше | 🚀 Как в IaC |
|--------|--------------|
| 🙅 «Зайду по SSH и починю» | 📝 «Меняю код, plan, apply» |
| 🚫 «Не трогай web-03» | 💪 «Снеси web-03, TF соберёт заново» |
| 👉 «Кто это поднял?» | 📜 ответит `git blame` |
| 😨 «Руками быстрее» | ⚡ «Руками быстрее *один раз*. IaC быстрее всегда.» |
| 💻 «Работает в моём облаке» | 🌍 «Работает в любом аккаунте, где гоняют этот код» |
| 🤷 «Потом задокументируем» | 📝 «Код И ЕСТЬ документация» |

> ❓ В какой колонке команда, в которой вы хотите работать?

---

## 📍 Slide 26 – 🚀 Что дальше

**📚 Следующая лекция: *Управление конфигурацией с Ansible.*** Terraform поднимает VM; Ansible настраивает то, что работает *внутри*. Они дополняют друг друга.

* 🔧 Идемпотентные playbook по SSH
* 📦 Роли, inventory и Ansible Galaxy
* 🤖 Поставить Docker на VM, которые поднимете в Lab 4

**🔬 Lab 4 на этой неделе:** поднимите облачную VM и на Terraform, *и* на Pulumi, сравните ощущения и (бонус) встройте `terraform plan` в проверку PR в GitHub Actions. VM не гасите — **Lab 5 зайдёт на неё по SSH**.

```mermaid
flowchart LR
  Lab4[🏗️ Lab 4: TF + Pulumi VM] --> Lab5[🔧 Lab 5: Ansible настраивает VM]
  Lab5 --> Lab9[☸️ Lab 9: K8s на похожих VM]
  Lab9 --> Future[🚀 Labs 13+: GitOps сверху]
```

**👋 До встречи на Lecture 5.**

> 🐄 От снежинок к скоту — по одному `terraform apply`.

---

## 📚 Материалы и что почитать дальше

**📕 Книги:**
* *Terraform: Up & Running* — Yevgeniy Brikman, O'Reilly (3-е изд., 2022; 4-е изд. уже с OpenTofu, конец 2025)
* *Infrastructure as Code* — Kief Morris, O'Reilly (2-е изд., 2020). Главная книга по теме.
* *Pulumi: Up & Running* — Adam Gordon Bell, O'Reilly (2024).
* *The Phoenix Project* — Kim, Behr, Spafford. До сих пор лучше всех разыгрывает проблемы, которые закрывает IaC.

**🔗 Ссылки:**
* 🌐 [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform) — официальная документация Terraform (1.10)
* 🌐 [opentofu.org](https://opentofu.org) — документация OpenTofu и гайд по переезду
* 🌐 [pulumi.com/docs](https://www.pulumi.com/docs/) — официальная документация Pulumi (3.140)
* 🌐 [registry.terraform.io](https://registry.terraform.io/) — провайдеры и модули сообщества (годится и для OpenTofu)
* 🌐 [github.com/aquasecurity/tfsec](https://github.com/aquasecurity/tfsec) — сканер безопасности для IaC

**🎓 Квиз:** квиз после лекции идёт в окно таблицы лидеров за недели 4–6.
