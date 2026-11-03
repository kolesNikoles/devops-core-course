# Lab 6 — Продвинутый Ansible и непрерывная выкладка

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Ansible%20%26%20CI%2FCD-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Ansible%202.21%20|%20Docker%20Compose%20v2%20|%20GitHub%20Actions-informational)

> **Цель:** превратите playbook из Lab 5 в настоящий deployment pipeline (цепочка автоматической сборки и выкладки). Отрефакторьте через `block`/`rescue`/`always`, соберите Compose-стек шаблоном Jinja2, добавьте wipe с двумя предохранителями и выкатывайте одной кнопкой из GitHub Actions.
> **Что сдать:** PR из ветки `lab06` с переработанными `ansible/roles/{common,docker,web_app}`, файлом `.github/workflows/ansible-deploy.yml` и `ansible/docs/LAB06.md`, где есть живое доказательство (list-tags, сработавший rescue, идемпотентный PLAY RECAP).

---

## Обзор

В Lab 5 вы собрали роли, которые **настраивают** сервер. В Lab 6 те же роли **доставляют** приложение: повторно, безопасно и из CI. Вы:

- переработаете `common` и `docker` через `block`/`rescue`/`always` и осознанную таксономию тегов
- замените `docker run` на Compose-стек из шаблона Jinja2 через `community.docker.docker_compose_v2`
- добавите кнопку чистой переустановки (`web_app_wipe`), закрытую **дважды**, чтобы одна опечатка не снесла прод
- подключите `.github/workflows/ansible-deploy.yml`: на каждый push в `ansible/**` идут lint, выкладка и проверка

> ⚠️ **Границы:** лабораторная целится в одну ВМ (вашу из Lab 4) плюс localhost, чтобы практиковаться без целевой ВМ. Rolling rollout на несколько хостов мы обсуждаем, а доказываем только структурно. Те же приёмы переносятся дальше.

**Что остаётся из Lab 5:** YAML, раскладка ролей, inventory, group_vars, Vault. Заново это **не** разбираем.

---

## Состояние проекта

**У вас уже есть из прошлых лабораторных:**
- DevOps Info Service (Lab 1) в registry (Lab 2/3)
- целевая ВМ с доступом по SSH (Lab 4)
- роли Ansible `common`, `docker`, `app_deploy`, inventory, group_vars и Vault (Lab 5)

**Эта лабораторная добавляет:**
- `ansible/roles/{common,docker}`, переработанные блоками и тегами
- `ansible/roles/web_app/` (переименована из `app_deploy`) с `meta/main.yml`, `templates/docker-compose.yml.j2`, `tasks/main.yml`, `tasks/wipe.yml`, `defaults/main.yml`
- `.github/workflows/ansible-deploy.yml`
- `ansible/docs/LAB06.md` — ваш отчёт с доказательствами

---

## Подготовка

```bash
ansible --version | head -1                  # ansible-core 2.21.x
ansible-galaxy collection install community.docker    # 4.x
ansible-lint --version
docker compose version                       # v2.x (the plugin, NOT docker-compose v1)
```

Переименуйте роль из Lab 5 до старта. Новое имя совпадает с переменной wipe в задаче 3:

```bash
cd ansible/roles && git mv app_deploy web_app
# update every reference: playbook roles:, group_vars keys, docs
```

> ⚠️ **Это больше, чем `mv`.** В Lab 5 роль `app_deploy` запускала один контейнер через `community.docker.docker_container`. Задача 2 заменяет это на выкладку через Compose, поэтому `tasks/main.yml` и `handlers/main.yml` нужно **переписать**, а не подправить. Файл Vault и каркас `defaults/main.yml` роли оставьте, но **переименуйте переменные** под новые ключи Lab 6 (`app_port` → `app_host_port` / `app_internal_port`, `docker_image_tag` → `docker_tag`, `app_container_name` → `app_name`). После того как напишете `defaults/main.yml` в §2.4, пройдитесь `grep` по `group_vars/` и `inventory/` в поисках старых имён.

---

## Задача 1 — Рефакторинг: блоки и теги (2 балла)

В Lab 5 роли `common` и `docker` были линейными списками задач. Перестройте их так, чтобы (a) случайный сбой apt или GPG не убивал прогон, (b) из CLI можно было целиться в одну фазу и (c) шаг enable в `systemd` выполнялся **даже если установка упала**.

### 1.1 Стратегия тегов (три оси)

Возьмите три независимые оси тегов и применяйте их одинаково в обеих ролях. От этого одного решения зависит, полезен `--tags` или это шум.

| Ось | Имена тегов, которые нужно использовать | Зачем |
|------|------------------------|----------|
| Компонент | `common`, `docker`, `web_app` | «Тронуть только одну роль» |
| Действие | `install`, `config`, `deploy`, `wipe` | «Прогнать одну фазу сквозь роли» |
| Предохранитель | `web_app_wipe` | Опасно, только по явному opt-in (задача 3) |

`ВАША ЗАДАЧА`: в `ansible/docs/LAB06.md` напишите таблицу из трёх строк: какие два или три тега получает каждый блок. Пример строки: *«блок install в `roles/docker` → `[docker, docker_install, install]`».* Так вы зафиксируете таксономию **до** того, как начнёте расставлять теги.

### 1.2 Переработайте роль `docker`

**Файл:** `roles/docker/tasks/main.yml`

Соберите установку apt-ключа, репозитория и `docker-ce` в один блок. Дайте ему `rescue`, который переживает дёрганое зеркало или сбой загрузки GPG, и `always`, который гарантирует, что сервис включён. Зачистка священна: она должна отработать и при успехе установки, и при провале.

`ВАША ЗАДАЧА`: заполните форму ниже. Подсказки в комментариях. Решения из Lab 5 как есть не вставляйте: вы их перестраиваете.

```yaml
- name: ___                                    # YOUR TASK: name the parent block
  block:
    - name: ___                                # YOUR TASK: add Docker apt GPG key
      ___:                                     # YOUR TASK: pick the right ansible.builtin module
        ___                                    # YOUR TASK: its args

    - name: ___                                # YOUR TASK: add Docker apt repo
      ___:
        ___

    - name: Install docker-ce
      ansible.builtin.apt:
        name: docker-ce
        update_cache: true
  rescue:
    - name: ___                                # YOUR TASK: what's a sensible recovery
                                               #            for a flaky mirror/GPG fetch?
      ___:
        ___
      retries: ___                             # YOUR TASK: pick a small int, justify in docs
      delay: ___
  always:
    - name: ___                                # YOUR TASK: ensure dockerd enabled + started
      ___:
        ___
  become: true
  tags: [___, ___, ___]                        # YOUR TASK: pick three tags from the taxonomy
```

> Тег на `block` наследуется каждой задачей внутри. Каждую задачу заново **не** размечайте. Поднимите `become: true` на блок и не повторяйте его. Если `rescue` тоже падает, хост помечается failed, а `always` всё равно выполняется.

### 1.3 Переработайте роль `common`

**Файл:** `roles/common/tasks/main.yml`

Ту же форму, с такими требованиями:
- Блок ставит базовые пакеты из Lab 5 (curl, gnupg, ca-certificates, …).
- `rescue` запускает `apt-get update --fix-missing`, затем повторяет попытку.
- `always` пишет маленький маркер завершения в `/tmp/lab6-common-ran`, чтобы вы доказали: блок прошёл от начала до конца.
- Теги идут по таксономии. `become: true` поднимите на блок.

`ВАША ЗАДАЧА`: напишите файл. Форма та же, что в 1.2, модули другие (список пакетов через `apt`, для маркера `command` или `shell` — выберите правильный и обоснуйте).

### 1.4 Докажите, что `rescue` реально срабатывает

Легко написать `rescue:` и ни разу его не прогнать. Непроверенный путь rescue хуже, чем его отсутствие: что он не работает, вы узнаете в тот день, когда он понадобится.

`ВАША ЗАДАЧА`: один раз специально сломайте задачу внутри одного из блоков, убедитесь, что rescue выполняется, затем откатите. В раздел с доказательствами положите **вывод задачи rescue**. Одной строки recap `rescued=1` мало.

Два безопасных способа вызвать сбой (выберите один: оба валят задачу *блока*, оба дают `rescue` её поймать, оба чисто откатываются):

1. **Поправьте роль на один прогон.** Направьте `apt_repository:` на `http://does-not-exist.example.com/` (или поставьте URL apt-ключа на 404), запустите, затем откатите через `git checkout roles/<role>/tasks/main.yml`. Плюс: ближе всего к реальному сбою. Ровно тот режим отказа, ради которого rescue и существует.
2. **`--extra-vars` и фиктивная проверка.** Добавьте внутрь блока `ansible.builtin.fail: msg="forced"` под условием `when: simulate_failure | default(false) | bool`. Один раз запустите с `-e simulate_failure=true`, снимите доказательство, запустите снова без него. Плюс: не нужно танцевать с `git checkout`. Триггер живёт в роли, и доказательство можно повторить когда угодно.

> Задачу **не** комментируйте и с `--check` **не** гоняйте. Ни то, ни другое не проходит путь rescue. Задача блока должна реально упасть (`failed=1` на строке задачи до строки rescue).

### 1.5 Проверьте выборочный запуск

```bash
ansible-playbook playbooks/provision.yml --list-tags                # shows the union of all tags
ansible-playbook playbooks/provision.yml --list-tasks --tags docker # tasks the next --tags docker run would run
ansible-playbook playbooks/provision.yml --tags install             # action axis: install-only across roles
ansible-playbook playbooks/provision.yml --skip-tags common         # component axis: everything but common
ansible-playbook playbooks/provision.yml --tags docker --check      # dry-run, component axis
```

> `--list-tags` печатает только объединение. Оно **не** доказывает, что таксономия трёхосевая (компонент / действие / предохранитель). Добавьте `--list-tasks --tags <axis>` с каждой оси и покажите: выборочный фильтр попадает в нужный срез. Объяснение лежит в таблице таксономии из §1.1.

### 1.6 Доказательство работы — вставьте в `docs/LAB06.md`

- Вывод **`--list-tags`** без правок. В объединении должны быть теги всех трёх осей (компонент / действие / предохранитель).
- По одному **выборочному прогону `--tags`** с каждой оси (например `--tags docker`, `--tags install` и сухой прогон `--tags web_app_wipe`). В каждом **большинство задач skipped**, а не «все 12 прошли с tag=docker_install».
- **Вывод пути rescue** из 1.4: настоящее имя задачи и результат, не только `PLAY RECAP`.
- Ваша таблица таксономии из 1.1.

**Разберите (ответ в документации):**
- Как теги наследуются из `block` в его задачи? Теги на импорте роли складываются или перекрывают друг друга?
- Чем особый тег `never` отличается от обычного opt-in тега вроде `web_app_wipe`?
- Что происходит с последующими задачами хоста, если его `rescue` тоже падает?

---

## Задача 2 — Переход на Docker Compose (3 балла)

Замените `docker run` из Lab 5 на Compose-стек из шаблона Jinja2 и применяйте его через `community.docker.docker_compose_v2`. Выигрыш — декларативное состояние, а идемпотентность получается сама: повторный прогон с тем же отрендеренным файлом показывает `changed=0`.

### 2.1 Объявите зависимость роли

**Файл:** `roles/web_app/meta/main.yml`

Зависимость в `meta/main.yml` заставляет `web_app` подтянуть `docker` сама, в правильном порядке, ровно один раз на play. Вторую запись `roles:` в playbook добавлять не нужно.

`ВАША ЗАДАЧА`: напишите этот файл. Форма:

```yaml
---
dependencies:
  - role: ___                                  # YOUR TASK: which role must run first?
    vars:
      ___: ["{{ ansible_user }}"]              # YOUR TASK: variable name from your
                                               #            Lab 5 docker role
```

### 2.2 Шаблон файла Compose

**Файл:** `roles/web_app/templates/docker-compose.yml.j2`

**Что должно получиться:** рабочий `docker-compose.yml` для **одного** сервиса, который:
- тянет `{{ docker_image }}:{{ docker_tag }}`,
- публикует `{{ app_host_port }}:{{ app_internal_port }}`,
- прокидывает каждую пару ключ/значение из `{{ app_env }}` (словарь из group_vars) как переменные окружения,
- перезапускается по `{{ restart_policy }}` (по умолчанию `unless-stopped`, нужен фильтр Jinja),
- описывает **healthcheck**, который бьёт в `/health` на `{{ app_internal_port }}`, чтобы `docker compose ps` показывал `(healthy)`.

Нужны **все три** механизма Jinja2:
1. подстановка `{{ var }}`
2. хотя бы один блок `{% if %}` (например, не рендерить `environment:`, когда `app_env` пуст)
3. хотя бы одна цепочка фильтров (например `| default('...')` или `| to_nice_yaml`)

`ВАША ЗАДАЧА`: напишите шаблон. Форма намеренно неполная, заполните тело:

```jinja
{# Compose v2 — NO top-level `version:` field (dropped in 2023) #}
services:
  {{ ___ }}:                                   # YOUR TASK: service name var
    image: ___                                 # YOUR TASK: image:tag, two vars
    container_name: ___
    ports:
      - "___:___"                              # YOUR TASK: host:container, two vars
{% if ___ %}                                   # YOUR TASK: only render env block when app_env non-empty
    environment:
{% for ___, ___ in ___ %}                      # YOUR TASK: iterate the dict
      ___: "___"
{% endfor %}
{% endif %}
    restart: {{ ___ | default('___') }}        # YOUR TASK: var + the default value
    healthcheck:
      test: ___                                # YOUR TASK: curl/wget the in-container /health
      interval: ___                            # YOUR TASK: sensible default
      timeout: ___
      retries: ___
```

> Compose v2 выкинул ключ `version:` верхнего уровня. Если в туториале всё ещё пишут `version: '3.8'`, туториал старше 2023. Не ставьте этот ключ. Проверьте рендер локально командой `ansible all -i localhost, -m template -a "src=... dest=/tmp/out.yml"` до полного play.

### 2.3 Реализуйте задачи выкладки

**Файл:** `roles/web_app/tasks/main.yml`

Форма. Заполните тела:

```yaml
---
- name: Include wipe tasks (gated — Task 3)
  ansible.builtin.include_tasks: wipe.yml
  tags: [web_app, web_app_wipe]

- name: ___                                    # YOUR TASK: name the deploy block
  block:
    - name: Create project directory
      ansible.builtin.file:
        path: "{{ compose_project_dir }}"
        state: directory
        mode: "0755"
      # NOTE: an aborted previous run can leave /tmp/<project> as a FILE — see Common Pitfalls

    - name: Render compose file
      ansible.builtin.template:
        src: ___                               # YOUR TASK: which template?
        dest: "___/docker-compose.yml"         # YOUR TASK: where on the target?
        mode: "0644"

    - name: ___                                # YOUR TASK: name this — "docker compose up -d"
      ___:                                     # YOUR TASK: the FQCN module — NOT the v1 module
                                               #            (Common Pitfalls: v1 vs v2)
        project_src: "{{ compose_project_dir }}"
        state: ___                             # YOUR TASK: which state means "up"?
        pull: ___
        remove_orphans: ___

    - name: Smoke test /health
      ansible.builtin.uri:
        url: "___"                             # YOUR TASK: build the URL from app_host_port
        status_code: 200
      register: hc
      retries: ___                             # YOUR TASK: bounded, not infinite
      delay: ___
      until: ___                               # YOUR TASK: condition on hc
  rescue:
    - name: ___                                # YOUR TASK: surface the failure
                                               #            (debug, fail, slack — your call)
      ___:
  tags: [___, ___, ___]                        # YOUR TASK: three tags from the taxonomy
```

Заметки по окружению (это не пометки `ВАША ЗАДАЧА`, это обвязка):
- Коллекция стоит на **контроллере** (управляющий узел): `ansible-galaxy collection install community.docker`.
- `python3-docker` стоит на **цели** (это делает ваша роль `docker`).
- Повторный прогон того же play без изменений должен показать `changed=0`. Если нет, в шаблоне есть что-то нестабильное: метка времени, случайное значение или обход словаря без сортировки.

### 2.4 Переменные

**Файл:** `roles/web_app/defaults/main.yml`

`ВАША ЗАДАЧА`: объявите значения по умолчанию. Нужные ключи: `app_name`, `docker_image`, `docker_tag`, `app_host_port`, `app_internal_port`, `compose_project_dir`, `restart_policy`, `app_env`. Возьмите разумные значения. **Не** ставьте `docker_tag` в `latest` для прода: запиньте его в `group_vars` для каждого окружения. Настоящие секреты держите в файле Vault из Lab 5 (`group_vars/.../vault.yml`) и оттуда подмешайте их в `app_env`.

> ⚠️ **Заодно перенесите имена из Lab 5.** В Lab 5 были `app_port`, `app_container_name`, `docker_image_tag`. Они сталкиваются с ключами Lab 6 или просто путают: `app_host_port` / `app_internal_port`, `app_name`, `docker_tag`. После `defaults/main.yml` выполните `grep -RIn 'app_port\|app_container_name\|docker_image_tag' ansible/` и переименуйте всё, что ещё сидит на старых ключах (group_vars, host_vars, vault). Если оставить оба варианта, шаблоны ломаются молча.

### 2.5 Проверьте

```bash
ansible-playbook playbooks/deploy.yml --tags deploy            # run #1: changed
ansible-playbook playbooks/deploy.yml --tags deploy            # run #2: ok (changed=0)
# On the target VM. Substitute YOUR values from group_vars/all.yml (not Jinja —
# Jinja is only rendered inside Ansible-managed files, not in your shell).
docker compose -f "<compose_project_dir>/docker-compose.yml" ps   # (healthy)
curl -fsS "http://<VM-IP>:<app_host_port>/health" | jq .
# Or run the same checks via Ansible, where {{...}} IS rendered:
# ansible all -m shell -a 'docker compose -f {{ compose_project_dir }}/docker-compose.yml ps'
```

### 2.6 Доказательство работы — вставьте в `docs/LAB06.md`

- **Отрендеренный** `docker-compose.yml` (после шаблона), хотя бы первые 20 строк.
- Оба PLAY RECAP рядом: прогон №1 `changed≥1`, прогон №2 `changed=0`. **Это доказательство идемпотентности.** Если второй прогон всё ещё показывает `changed=2`, шаблон каждый раз рендерится по-другому. Отсортируйте ключи словаря и уберите метки времени.
- `docker compose ps` со статусом `(healthy)`.
- Вывод `curl /health` с вашим настоящим hostname и uptime.

**Разберите:**
- `restart: always` против `unless-stopped`. Когда разница реально даёт о себе знать?
- Почему `python3-docker` ставится на **цель**, а не на контроллер?
- Почему один шаблон Compose лучше, чем N задач `docker run`?

---

## Задача 3 — Wipe с двумя предохранителями (2 балла)

Чистая переустановка иногда единственный безопасный откат: битый том, ротированный секрет, сломанный кэш. Кнопка wipe (полная зачистка) нужна, **и** её нужно закрыть, чтобы сонная опечатка в `--tags` в два часа ночи не снесла прод.

### 3.1 Два предохранителя

Два **независимых** предохранителя: проверка переменной в `when:` **и** opt-in тег. Чтобы wipe пошёл, должны совпасть оба.

| Вызов | Wipe? | Почему |
|------------|-------|-----|
| `deploy.yml` | Нет | переменная false, тег не запрошен |
| `deploy.yml --tags web_app_wipe` | Нет | `when` блокирует (переменная всё ещё false) |
| `deploy.yml -e web_app_wipe=true` | Да — сначала wipe, **потом** выкладка | чистая переустановка |
| `deploy.yml -e web_app_wipe=true --tags web_app_wipe` | Да — **только** wipe | задачи выкладки пропущены по тегу |

> Зачем два предохранителя, а не один особый тег `never`? `never` — один предохранитель. Кто передал `--tags web_app_wipe` (а с `never` — кто передал `--tags never`), тот и запускает wipe. Второй, независимый предохранитель (переменная) значит: опечатка или ошибка CI только в **одном** измерении безвредна, второй предохранитель всё ещё держит. Это защита в глубину, а не одна хитрость в одном месте.

### 3.2 Реализация

**Файл:** `roles/web_app/tasks/wipe.yml`

Форма:

```yaml
---
- name: Wipe web application
  block:
    - name: ___                                # YOUR TASK: name it — "compose down"
      ___:                                     # YOUR TASK: which module?
        project_src: "{{ compose_project_dir }}"
        state: ___                             # YOUR TASK: which state removes it?
      failed_when: ___                         # YOUR TASK: tolerate the already-clean case
                                               #            (hint: `false` is one answer; a list
                                               #             of conditions is the better one)

    - name: ___                                # YOUR TASK: remove the project dir
      ___:
        path: "___"
        state: ___

    - name: ___                                # YOUR TASK: small completion debug
      ___:
        msg: "___"
  when: ___ | bool                             # YOUR TASK: gate #1 — variable
                                               #            (why is `| bool` mandatory?
                                               #             see Common Pitfalls)
  tags: [___, ___]                             # YOUR TASK: gate #2 — opt-in tag
```

**Файл:** `roles/web_app/defaults/main.yml` — добавьте:

```yaml
web_app_wipe: ___                              # YOUR TASK: safe default — what should it be?
```

Строка `include_tasks: wipe.yml` из задачи 2.3 стоит **до** блока выкладки, поэтому `-e web_app_wipe=true` даёт сначала wipe, потом выкладку, за один прогон.

### 3.3 Проверьте все четыре строки

`ВАША ЗАДАЧА`: прогоните каждый из четырёх вызовов в §3.1 и сохраните вывод. Для каждого докажите то, что утверждаете:
- Строка 1 (просто `deploy.yml`). Поищите задачи wipe в recap или в выводе `--list-tasks`. Они должны быть skipped.
- Строка 2 (только `--tags web_app_wipe`). Задачи wipe **входят** в play, но `when:` их пропускает. В recap стоит `skipped`, не `ok`.
- Строка 3 (`-e web_app_wipe=true`). `docker ps` до (работает), `docker ps` после wipe внутри прогона (пропал), после выкладки (снова работает, свежий ID контейнера).
- Строка 4 (`-e ... --tags web_app_wipe`). Wipe выполняется, задачи выкладки пропущены по тегу, `docker ps` в конце пустой.

### 3.4 Доказательство работы — вставьте в `docs/LAB06.md`

- Четыре снимка терминала, по одному на строку, каждый подписан вызовом.
- `docker ps --filter name=<app>` сразу до и сразу после прогона «только wipe».
- Одна строка, **почему `| bool` обязателен** в `when:`. (Если застряли, спойлер в разделе «Типичные ловушки».)

---

## Задача 4 — CI/CD на GitHub Actions (3 балла)

Автоматизируйте всё: push в `main`, затем lint, выкладка и проверка. Где цель это умеет, предпочитайте **короткоживущие учётные данные OIDC** долгоживущему SSH-ключу.

### 4.1 Каркас workflow

**Файл:** `.github/workflows/ansible-deploy.yml`

Форма. Заполните тела:

```yaml
name: Ansible Deploy

on:
  push:
    branches: [main]
    paths:
      - '___'                                  # YOUR TASK: trigger on ansible/** changes
      - '!___'                                 # YOUR TASK: exclude docs-only changes
      - '___'                                  # YOUR TASK: self-test on workflow file edits
  workflow_dispatch:                           # manual deploys

permissions:
  contents: read
  id-token: ___                                # YOUR TASK: which value enables OIDC?

jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install '___' ___             # YOUR TASK: pin ansible-core to 2.21.* + ansible-lint
      - run: ___                               # YOUR TASK: lint the ansible/ tree

  deploy:
    needs: ___                                 # YOUR TASK: depend on lint
    runs-on: ubuntu-24.04
    environment: ___                           # YOUR TASK: GH Environment name
                                               #            (gives you approval + per-env secrets)
    steps:
      - uses: actions/checkout@v4
      # YOUR TASK: pick ONE auth tier from §4.2 and run the playbook + smoke test
```

> Зафиксируйте `ansible-core==2.21.*`, чтобы CI совпал с локальным lockfile. У ansible-core **нет метки LTS**. Зафиксируйте тот минор, который стыкуется с вашей коллекцией `community.docker` 4.x.

### 4.2 Выберите уровень аутентификации

> ⚠️ **Сначала простая ошибка:** автоматически подставляемый `secrets.GITHUB_TOKEN` аутентифицирует вас в **GitHub** (API, GHCR). Это **не** SSH-ключ, войти им на ВМ нельзя. Обычный SSH на ВМ всегда требует *чего-то ещё*: ключа или федеративного механизма, который в итоге выдаёт ключ или сессию.

> ⚠️ **OIDC для SSH на ВМ, честно.** Выигрыш OIDC в том, что в GitHub нет долгоживущего секрета. Но ВМ на том конце должна принимать федеративную личность. Это AWS SSM Session Manager / EC2 Instance Connect, GCP IAP TCP-tunnel или короткоживущий SSH-сертификат, который выдало облако (например Teleport, HashiCorp Boundary, Smallstep). **Голый `ssh user@ip` на собранную руками ВМ с `authorized_keys` OIDC не понимает.** Берите OIDC, только если облако из Lab 4 умеет один из этих путей. Иначе честный ответ — уровень 2 (секрет Environment и deploy key). Лабораторная оценивает **обоснование выбора**, а не OIDC для красоты.

| Уровень | Механизм | Компромисс |
|------|-----------|-----------|
| Секрет репозитория как env | `secrets.SSH_KEY` → `~/.ssh/id_ed25519` | Просто. Долгоживущий ключ — это новый «пароль в исходниках» |
| GitHub Environment | Секреты на окружение (deploy key на каждое) и ручное подтверждение | Нужно для прода. Небольшой операционный наклад |
| **OIDC + облачный IAM** | Короткоживущий федеративный токен, дальше SSH через облако (SSM / IAP / сертификат) | В CI нет статического ключа. Цель должна уметь федеративный путь SSH |

`ВАША ЗАДАЧА`: выберите один уровень и реализуйте его. Обоснуйте выбор в документации в 2–3 предложениях, со ссылкой на компромиссы выше. Каркасы:

**OIDC (рекомендуется):**

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ___                  # YOUR TASK: an IAM role ARN trusting GH OIDC
          aws-region: ___
      - name: Deploy
        run: |
          ___                                  # YOUR TASK: ansible-playbook against your inventory
```

**SSH в секрете (запасной путь):**

```yaml
      - env:
          VAULT:  ${{ secrets.VAULT_PASS }}
          SSH:    ${{ secrets.SSH_KEY }}
          VM_IP:  ${{ secrets.VM_IP }}        # secret: target VM IP or DNS name from your Lab 4 / inventory
        run: |
          mkdir -p ~/.ssh && chmod 700 ~/.ssh
          install -m 600 /dev/stdin ~/.ssh/id_ed25519 <<< "$SSH"
          install -m 600 /dev/stdin /tmp/vault        <<< "$VAULT"
          ssh-keyscan -H "$VM_IP" >> ~/.ssh/known_hosts    # avoids host-key prompts
          ___                                  # YOUR TASK: ansible-playbook with --vault-password-file
```

> Никогда не печатайте секрет в лог (не `run: echo "$SSH"` для отладки: кладите его в файл через перенаправленный stdin, как `install` выше). Self-hosted runner в сети цели убирает SSH-ключ совсем, но это граница доверия. На PR из форков его **никогда** не гоняйте.

### 4.3 Проверяйте в CI, а не только код возврата 0

Код возврата 0 у `ansible-playbook` значит только, что playbook прошёл без ошибок. Это **не** значит, что приложение отвечает. Добавьте шаг smoke-теста.

`ВАША ЗАДАЧА`: напишите шаг, который опрашивает `/health` **на выкатанной ВМ** (не `localhost` на runner: это просто curl до runner GitHub) до N раз. N ограничьте сами. На первом 200 выходите с 0. Если сервис так и не поднялся, выходите с 1. Пишите обычным shell, без стороннего action. Подсказка: `curl -fsS http://<VM-IP>:<app_host_port>/health` возвращает ненулевой код на ответе не из 2xx. Состыкуйте его с коротким циклом повторов. Адрес ВМ берите из того же источника, что и playbook (переменная inventory, переменная GH или `terraform output -json`).

Добавьте бейдж статуса в `README.md` репозитория:

```markdown
[![Ansible Deploy](https://github.com/<you>/<repo>/actions/workflows/ansible-deploy.yml/badge.svg)](https://github.com/<you>/<repo>/actions/workflows/ansible-deploy.yml)
```

### 4.4 Стратегия выкладки

`serial:` на play ограничивает, сколько хостов выкатываются **в один момент**. Это **не** значит «выкатить на 50% хостов и остановиться». При 10 хостах и `serial: "50%"` Ansible делает **две пачки по 5**. Новая версия в итоге доезжает до каждого хоста. Рычаг крутит *параллельность*, не *охват*. Для настоящего canary, который останавливается посмотреть, используйте список: `serial: [1, 5, "100%"]`. Первая пачка из 1, потом 5, потом остальные.

```yaml
- name: Deploy web tier
  hosts: webservers
  serial: "___"                                # YOUR TASK: pick one — explain the meaning
                                               #            in your docs (rolling vs canary vs all)
  max_fail_percentage: ___                     # YOUR TASK: pick a small int, justify
  roles: [web_app]
```

| Стратегия | Типичный `serial:` | Когда |
|----------|-------------------|------|
| Rolling | `1` или `25%` | Без простоя за балансировщиком |
| Canary | `1` (первая пачка, потом смотрим) | Проверить на одном хосте, прежде чем идти дальше |
| Все сразу | не указывайте `serial:` | Только dev/staging |

В документации: какую стратегию вы выбрали для прода и почему?

### 4.5 Доказательство работы — вставьте в `docs/LAB06.md`

- Скриншот (или вставленный блок лога) зелёного прогона workflow.
- Строки лога, где **ansible-lint проходит** (не skipped и не сплошные warnings).
- Вывод шага smoke-теста (curl 200).
- URL зелёного бейджа статуса.
- 2–3 предложения, почему вы выбрали такой `serial:` и такой уровень аутентификации.

**Разберите:**
- По безопасности: долгоживущий SSH-ключ в секрете репозитория против OIDC и облачного IAM. Какой радиус поражения, если утечёт каждое?
- Как добавить job отката, который стартует, если шаг проверки упал? Набросайте YAML.
- Почему self-hosted runner никогда не должен выполнять job из PR форков?

---

## Бонус — Продвижение между окружениями (2 балла)

> Одна бонусная задача (2 балла). Проведите **один неизменяемый образ** по пути `staging` → `prod` через inventory на каждое окружение и ручное подтверждение. Тот же артефакт, без пересборки и без смены тега.

Опирайтесь на workflow из задачи 4. Суть в том, что один и тот же образ (запинен SHA или SemVer, **не** `:latest`) сам едет на staging, а на прод — только после того, как человек подтвердит.

`ВАША ЗАДАЧА`:

1. **Два inventory** с запиненным `docker_tag` на окружение (staging впереди, prod отстаёт). Оба тега должны указывать на **один и тот же `docker_image`**, который есть в вашем registry. Продвижение значит «тот же образ, более новое окружение», и никогда не пересборку:

   
```
   ansible/inventories/
     staging/hosts.ini   group_vars/all.yml   docker_tag: ___   # YOUR TASK: the newer SemVer or :sha-abc123 — what staging is testing
     prod/hosts.ini      group_vars/all.yml   docker_tag: ___   # YOUR TASK: the SemVer / SHA that already passed staging — what prod is running
   ```

   Файлы Vault на каждое окружение. **Не** ветвите playbook по `inventory_hostname == 'prod'`. Ветвите только по **переменным**.

2. **Workflow продвижения.** Расширьте `ansible-deploy.yml` (или добавьте `ansible-promote.yml`):
   - `deploy-staging` сам запускается на push в `main` против `inventories/staging`.
   - У `deploy-prod` стоит `needs: deploy-staging`, он использует `environment: production` с **обязательным ревьюером** и целится в `inventories/prod`.
   - В оба едет **один и тот же** `docker_image`. По окружениям отличается только `docker_tag`.
   - На проде используются `serial:` и `max_fail_percentage:` из задачи 4.4.

   Каркас:

   
```yaml
   deploy-prod:
     needs: ___                                # YOUR TASK
     environment: ___                          # YOUR TASK: env name with required reviewer
     runs-on: ubuntu-24.04
     steps:
       - uses: actions/checkout@v4
       # YOUR TASK: assume creds (auth tier from Task 4.2), then:
       #   - run deploy.yml -i ansible/inventories/prod
       #   - smoke-test /health on a prod host
   ```

3. **Проверьте** от начала до конца:
   - Выкладка на staging сама стартует на push.
   - Job прода **встаёт** на подтверждение (скриншот UI GitHub, где виден предохранитель).
   - Оба окружения отдают ожидаемые (разные) теги через `/health` или поле `?version=`.

**Доказательство:** два файла `group_vars/all.yml` с разными запиненными тегами, скриншот предохранителя подтверждения на проде и вывод `curl`, который показывает: каждое окружение крутит свою запиненную версию.

---

## Как сдавать

```bash
git switch -c lab06
git add ansible/ .github/workflows/ansible-deploy.yml ansible/docs/LAB06.md
git commit -m "feat(lab06): advanced ansible — blocks, tags, compose, CI"
git push -u origin lab06
```

Убедитесь, что `.vault_pass` и любой незашифрованный секрет **не** в индексе. Зашифрованные файлы Vault коммитить можно.

Откройте **два** PR:
- `your-fork:lab06` → `course-repo:master` *(на ревью)*
- `your-fork:lab06` → `your-fork:master` *(сольётся в ваш main, когда закончите)*

Чеклист PR:

```text
- [ ] Task 1 — roles refactored, --list-tags + selective --tags captured, rescue fired
- [ ] Task 2 — Compose templated, idempotent (changed=0 on run #2), /health 200
- [ ] Task 3 — all four wipe rows verified, | bool justified
- [ ] Task 4 — workflow green, ansible-lint, smoke test, badge
- [ ] Bonus — two-env promotion behind required reviewer (optional)
```

---

## Критерии приёмки

### Основные задачи (10 баллов)

**Блоки и теги (2 балла):**
- ✅ `docker` и `common` переработаны через `block`/`rescue`/`always`
- ✅ Трёхосевая таксономия тегов стоит на каждом блоке
- ✅ Сняты вывод `--list-tags` и один выборочный прогон `--tags`
- ✅ Путь rescue явно **сработал** (вывод задачи, не только `rescued=1`)

**Docker Compose (3 балла):**
- ✅ `app_deploy` переименована в `web_app`, все ссылки обновлены
- ✅ `meta/main.yml` объявляет зависимость `docker`
- ✅ Шаблон Jinja2 использует подстановку, **и** `{% if %}`, **и** хотя бы один фильтр
- ✅ Выкладка через `community.docker.docker_compose_v2`; оба прогона сняты (changed → 0)
- ✅ Приложение доступно на `/health`; контейнер показывает `(healthy)`

**Логика wipe (2 балла):**
- ✅ Закрыто и `when: web_app_wipe | bool`, и тегом `web_app_wipe`
- ✅ Подключено **до** блока выкладки (чистая переустановка работает за один прогон)
- ✅ Сняты все четыре строки вызовов

**CI/CD (3 балла):**
- ✅ Workflow гоняет `ansible-lint`, затем выкладку
- ✅ Фильтры путей настроены (документация исключена, workflow проверяет сам себя)
- ✅ Аутентификация через OIDC или секрет GitHub Environment (секрет не печатается)
- ✅ Шаг smoke-теста и зелёный бейдж
- ✅ `serial:` выбран и обоснован

### Бонус (2 балла)
- ✅ Отдельные inventory `staging` / `prod` с запиненным `docker_tag` на окружение
- ✅ Продвигается один неизменяемый образ (без пересборки, без `:latest`)
- ✅ Прод за Environment с обязательным ревьюером
- ✅ Rolling rollout с `serial:` и `max_fail_percentage:`
- ✅ Доказательство, что оба окружения отдают свои запиненные версии

---

## Критерии оценивания

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — Блоки и теги | **2** | block/rescue/always; теги по трём осям; rescue сработал, есть вывод |
| **Задача 2** — Docker Compose | **3** | переименованная роль и зависимость в meta; Jinja2 (переменные, if, фильтр); идемпотентно |
| **Задача 3** — Логика wipe | **2** | два предохранителя; все четыре строки проверены |
| **Задача 4** — CI/CD | **3** | lint, затем выкладка, затем проверка; аутентификация OIDC или Environment; serial обоснован |
| **Бонус** — Продвижение между окружениями | **2** | неизменяемый образ; предохранитель обязательного ревьюера; оба окружения доказаны |
| **Итого** | **12** | 10 основных + 2 бонус |

**Шкала оценки:**
- **10/10:** Всё работает. Rescue, wipe и идемпотентность доказаны. CI проверяет. Документация чёткая.
- **8–9/10:** Всё работает. Мелкие дыры в доказательствах или в документации.
- **6–7/10:** Основная выкладка работает. Слабая стратегия тегов, нет доказательства rescue или тонкая проверка в CI.
- **<6/10:** Нет выкладки через Compose, wipe без предохранителя или workflow не работает.

---

## Материалы

<details>
<summary>📚 Документация</summary>

- [Блоки Ansible (обработка ошибок)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
- [Теги Ansible](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html)
- [Зависимости ролей](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#using-role-dependencies)
- [`community.docker.docker_compose_v2`](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
- [Справка по файлу Compose](https://docs.docker.com/reference/compose-file/)
- [Шаблоны Jinja2](https://jinja.palletsprojects.com/)
- [Фильтры `paths` в GHA](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-filters)
- [GitHub OIDC для облачных выкладок](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Environments и обязательные ревьюеры](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

</details>

<details>
<summary>⚠️ Типичные ловушки (с реальных сухих прогонов)</summary>

- **Застывший `/tmp/<project>` в виде файла блокирует `state: directory`.** Прерванный прошлый прогон может оставить путь проекта файлом (или симлинком на файл). Задача `file: state=directory` падает с *"file exists, refusing to overwrite"*. Как починить: один раз `rm -rf <dir>`, затем повторите прогон. Или явно поставьте `file: state=absent` перед созданием (безопасно, только если wipe не зависит от содержимого).
- **Имена модулей Compose v2 и v1.** `community.docker.docker_compose` (v1, устарел) и `community.docker.docker_compose_v2` (текущий) отличаются одним подчёркиванием. v1 вызывает мёртвый Python CLI `docker-compose` и на свежей Ubuntu 24.04 падает с *"docker-compose: command not found"*. **Всегда берите `docker_compose_v2`.**
- **FQCN против голого имени модуля.** `ansible-lint` помечает `apt:` / `file:` / `template:` как `fqcn[action-core]`. Пишите `ansible.builtin.apt`, `ansible.builtin.file`, `ansible.builtin.template`. Иначе CI завалит PR.
- **Что значит `serial: "50%"`.** `serial` ограничивает **одновременные** хосты, не общее число. При 10 хостах и `serial: "50%"` Ansible делает **две пачки по 5**. Новая версия в итоге доезжает до каждого. Это **не** «выкатить на 50% и остановиться». Для настоящего canary берите `serial: [1, 5, "100%"]` (список: первая пачка из 1, потом 5, потом остальные).
- **`-e web_app_wipe=true` — это СТРОКА.** Ansible передаёт `--extra-vars key=value` как Python `str`, не `bool`. Без `| bool` условие `when: web_app_wipe` смотрит на истинность **строки**, а и `"true"`, и `"false"` непустые, поэтому обе проходят. Предохранитель, который должен беречь прод, срабатывает и на `-e web_app_wipe=false`. Фильтр `| bool` приводит `"true"` / `"false"` / `"yes"` / `"no"` / `"1"` / `"0"` к настоящим булевым. Забудете — предохранитель сломан **в самую опасную сторону**: wipe идёт, когда вы написали «false».
- **`| default('latest')` на проде.** Удобно для dev, смертельно на проде: повторная выкладка с `docker_tag=latest` молча подхватывает всё, что с тех пор запушили в registry. Зафиксируйте тег по окружениям в `group_vars`. Бонусная задача делает это явным.
- **`become: true` повторён на каждой задаче.** Поднимите его на `block`. Тег на блоке размечает и каждую задачу внутри. Меньше шума, меньше багов с drift.
- **Lab 7 снова берёт порт 8080.** Compose-стек следующей лабораторной тоже описывает сервис `app` на порту хоста 8080. Если стек Lab 6 ещё работает, когда вы стартуете Lab 7, второй `docker compose up -d` падает с *"port is already allocated"*. Выберите здесь непересекающийся `app_host_port` (например 8084) или сделайте `docker compose down` для Lab 6 перед стартом Lab 7. Lab 7 ждёт метку контейнера `app=devops-python`. Если держите оба живыми, поставьте эту метку и на контейнер этой лабораторной, чтобы правило relabel в Lab 7 его увидело.

</details>

<details>
<summary>🛠️ Инструменты, которые стоит знать</summary>

- `ansible-lint` — крутится в CI. Сначала прогоните локально, чтобы не гонять круги.
- `ansible-playbook --list-tasks --list-tags --list-hosts` — три флага, без выполнения. Удобно для проверки перед стартом.
- `ansible-doc -t filter -l` — все фильтры Jinja2, которые идут с ansible-core.
- `jq` — в паре с выводом `community.docker.docker_compose_v2`, который вы сняли через `register:`.

</details>

---

## Что дальше

Приложение выкатили. Следующие лабораторные его смотрят и измеряют:

| Lab | Что добавляет к этому сервису |
|---:|---|
| 7 | Логи Loki + Alloy. Стек выкатывается теми же приёмами, что в этой лабораторной |
| 8 | Prometheus `/metrics`, инструментированные эндпоинты, PromQL по методу RED |
| 9 | Kubernetes (k3d): переезд с Compose на K8s, `/health` остаётся |
| 10 | Helm-чарты версии 4 для шаблонных выкладок в K8s |
| 13 | ArgoCD: декларативный GitOps, который тянет изменения сам |
| 14 | Argo Rollouts: canary и blue-green для того же сервиса |

---

**Удачи.** Блок даёт границу ошибки, тег — скальпель, Compose — декларативное состояние, CI — кнопку. Двойной предохранитель не даёт кнопке сработать самой.
