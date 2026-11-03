# Lab 18 — Воспроизводимые сборки на Nix

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Nix%20%26%20Reproducibility-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![type](https://img.shields.io/badge/type-Exam%20Alternative-purple)

> **Цель:** Соберите сервис из Lab 1 и образ контейнера через Nix flakes и докажите, что две независимые сборки попадают в **один и тот же путь `/nix/store`**. Это воспроизводимость бит в бит, которую `Dockerfile` дать не может.
> **Что сдать:** PR из `lab18`, который добавляет `labs/lab18/` (ваш flake и исходники) и `labs/submission18.md`. В описании PR должен быть путь в store, который вы воспроизвели.

---

## Обзор

`Dockerfile` и `requirements.txt` *выглядят* воспроизводимыми. У них drift (со временем разъезжаются):

- `FROM python:3.13-slim` со временем указывает на другое содержимое: тег изменяемый
- `pip install -r requirements.txt` даёт транзитивным зависимостям двигаться, даже если прямые запинены через `==`
- слои Docker вшивают время сборки, поэтому один и тот же `Dockerfile` на каждой сборке даёт другой SHA образа

Nix чинит это так: сборка — **чистая функция своих входов**. Каждый вход (исходники, зависимости, компилятор, флаги сборки) хешируется в выходной путь `/nix/store/<sha256>-<name>-<version>`. Одни и те же входы → тот же хеш → тот же результат, на любой машине, сегодня и в 2036. Тот же хеш — попадание в кэш `cache.nixos.org`, и пересборка не нужна.

В этой лабораторной вы:

- напишете **flake** руками: внешняя форма, inputs, outputs. И выберете сборщик деривации под то, что пакуете
- запините полное замыкание зависимостей через `flake.lock` (якорь воспроизводимости)
- соберёте образ контейнера через `dockerTools`: нет `FROM`, нет `apt-get`, нет времени в слоях
- **докажете**, что две независимые сборки дают **один и тот же путь в store**. Вот это и есть воспроизводимость бит в бит

> ⚠️ **Границы:** никакого Kubernetes, никакого CI. Один сервис, один flake, два запуска `nix build`, одно сравнение хешей. Вся суть — воспроизводимость. Не умничайте.

**Это лабораторная — альтернатива экзамену.** Чтобы заменить финальный экзамен, сдайте и Lab 17 (Cloudflare Workers), и Lab 18 (Nix) до нужного порога. Дедлайн и минимальный балл — в Lab 17. Сама по себе Lab 18 — **бонусная** лабораторная: **10 баллов** за основные задания (6 + 4) и **2 балла** бонуса = **максимум 12 баллов**.

**Стек (май 2026):** Nix **2.25+** (flakes стабильны с 2.18; установщик Determinate или официальный) · контейнер `nixos/nix` как путь для WSL · nixpkgs `nixos-25.11` (релизная ветка, пин через вход flake) · `dockerTools.buildLayeredImage` · Docker (чтобы сделать `docker load` получившегося образа)

> **Lix:** форк Nix от сообщества (март 2024) — drop-in замена `nix`, вычисление быстрее, управление понятнее. Для этой лабораторной годится и Nix 2.25+, и текущий Lix.

> **Про выводы ниже:** каждый вывод команды, хеш и путь в store — **пример**. У вас будет иначе. Воспроизводимым должно быть другое: **ваши** две сборки дают **один и тот же** путь.

---

## Состояние проекта

**Из прошлых лабораторных у вас уже должно быть:**

- `app_python/` — сервис Flask / FastAPI из Lab 1 с `GET /` и `GET /health`
- `app_python/Dockerfile` из Lab 2 (в задании 2 вы сопоставите его с `dockerTools`)

> Если артефактов Lab 1 и Lab 2 уже нет, хватит Flask с одним маршрутом (`app.py` и `/health`) и `requirements.txt` в одну строку. Оценивают *flake*, не приложение.

**Эта лабораторная добавляет:**

- `labs/lab18/flake.nix` — ваш flake (задание 1)
- `labs/lab18/flake.lock` — lock-файл, который пинит полное замыкание (задание 1, он в коммите)
- `labs/lab18/app/` — исходники деривации
- `labs/submission18.md` — отчёт

---

## Подготовка

Nix ставится в `/nix` в корне системы, и для этого нужен `root`. На WSL2 и на хостах с жёсткими ограничениями самый чистый путь — образ Docker **`nixos/nix`**: Nix уже с включёнными flakes, файловая система хоста не трогается. Эталонная сдача собрана так.

**Путь A — контейнер `nixos/nix` (так лучше на WSL и на закрытых хостах):**

```bash
# Pinned image tag — do NOT use :latest (mutable). 2.34 or any 2.25+ is fine.
docker run --rm -it \
  -v "$PWD":/work -w /work \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --entrypoint /bin/sh \
  nixos/nix:2.34.0

# inside the container — flakes need the experimental feature flag explicit
nix --version
nix --extra-experimental-features 'nix-command flakes' eval --expr '1 + 1'
```

Монтирование `docker.sock` нужно, чтобы в задании 2 команда `docker load < result` сработала из контейнера Nix.

**Путь B — установка на хост (Linux / macOS, нужен sudo):**

```bash
# Determinate Systems installer — flakes on by default, clean uninstall
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
nix --version            # expect 2.25.x or newer
```

<details>
<summary>🐧 Альтернатива: официальный установщик Nix. Берите его, если `install.determinate.systems` недоступен или отваливается по таймауту из вашей сети (flakes придётся включить руками)</summary>

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

Затем допишите в `~/.config/nix/nix.conf` и перезапустите терминал:

```
experimental-features = nix-command flakes
```

</details>

Проверка (любой путь): запустите пакет, не устанавливая его.

```bash
nix run nixpkgs#hello   # downloads + runs `hello`, installs nothing permanent
```

Создайте раскладку:

```
labs/lab18/
├── flake.nix          # YOU WRITE — Task 1
├── flake.lock         # generated by `nix flake update`, committed
├── app/               # your Lab 1 sources (or the minimal stub)
│   ├── app.py
│   └── requirements.txt
└── (submission18.md lives one level up at labs/submission18.md)
```

---

## Задание 1 — Напишите flake, который собирает приложение Lab 1 (6 баллов)

**Цель:** руками напишите `flake.nix` по структурному скелету, соберите через него сервис Lab 1 и закоммитьте `flake.lock`, чтобы замыкание было запинено.

### 1.1 — Внешняя форма flake

Flake — выражение Nix с тремя полями верхнего уровня: `description`, `inputs`, `outputs`. **Что внутри каждого — ваше.** Скелет ниже показывает только **форму**. Выбор, из-за которого сборка становится *вашей* и воспроизводимой, оставлен пустым.

```nix
# labs/lab18/flake.nix
{
  description = ___;                       # YOUR TASK: one short string describing what this flake builds

  inputs = {
    # YOUR TASK: pick a NIXPKGS REVISION AND PIN IT.
    # The lab tests `nixos-25.11` (the Nov-2024 release branch). NOT `nixpkgs-unstable`
    # — unstable means non-reproducible across days. Use a release tag.
    # Format: github:NixOS/nixpkgs/<branch-or-rev>
    nixpkgs.url = ___;
  };

  outputs = { self, nixpkgs }:
    let
      # YOUR TASK: pick the system you're building for.
      #   x86_64-linux  — WSL2 / most Linux laptops / CI
      #   aarch64-linux — Linux on Apple Silicon under Docker / Asahi
      #   aarch64-darwin — Apple Silicon macOS native
      #   x86_64-darwin  — Intel macOS
      system = ___;
      pkgs   = ___;                        # YOUR TASK: legacyPackages for the chosen system
    in {
      packages.${system}.default = ___;    # YOUR TASK: a derivation that builds your app — see 1.2
      # Task 2 will add a second output here: packages.${system}.dockerImg
    };
}
```

**Почему каждая строка — пункт критериев:**

- **`description`** — одно предложение; его видно в `nix flake show` и `nix flake metadata`. Пусть оно будет полезным.
- **`inputs.nixpkgs.url`** — якорь воспроизводимости. Релизный тег (`nixos-25.11`) или конкретный SHA коммита замораживает всё дерево зависимостей. **`nixpkgs-unstable` здесь ошибка:** верхушка ветки едет каждый день. Это самая частая ошибка вида «воспроизводимая сборка, которая не воспроизводится».
- **`system`** — выходы flake свои у каждой системы. Выберите ту, на которой будете показывать. Однокурсники на других системах допишут свою в ваш flake.
- **`pkgs`** — `nixpkgs.legacyPackages.${system}`. Так берут набор пакетов, вычисленный для этой системы. Запомните.
- **`packages.${system}.default`** — то, что собирает `nix build` без `.#attr`. Это ваш сервис.

### 1.2 — Выберите деривацию и заполните тело

В store Nix десятки функций-сборщиков (`mkDerivation`, `writeShellApplication`, `buildPythonApplication`, `buildGoModule`, …). У каждой свои допущения о том, что она пакует. Возьмите ту, что совпадает со стеком Lab 1:

| Сборщик | Когда уместен | Ключевые входы |
|---|---|---|
| `pkgs.writeShellApplication` | Скрипт-обёртка, который вызывает что-то ещё (эталонная сдача использует его) | `name`, `runtimeInputs`, `text` |
| `pkgs.stdenv.mkDerivation` | Общий случай: вы сами даёте `buildPhase` / `installPhase` | `src`, `buildInputs`, `installPhase` |
| `pkgs.python3Packages.buildPythonApplication` | Python-приложение с `pyproject.toml` / `setup.py` | `pname`, `version`, `src`, `propagatedBuildInputs`, `format` |
| `pkgs.buildGoModule` | Программа на Go (бонус Lab 1) | `pname`, `src`, `vendorHash` |

**`ВАША ЗАДАЧА`**: в пустое место `default = ___` из 1.1 вызовите сборщик под своё приложение. Ниже только **форма** (начинка по-прежнему ваша):

```nix
# Shape 1 — writeShellApplication (good if your "app" is a thin shell entrypoint
# that calls into python/curl/whatever from the pinned closure)
packages.${system}.default = pkgs.writeShellApplication {
  name         = ___;                       # YOUR TASK: the binary name produced under result/bin/
  runtimeInputs = with pkgs; [ ___ ];       # YOUR TASK: every command your text uses (python3, curl, …)
  text         = ___;                       # YOUR TASK: the shell body. Single-quoted ''…'' multiline OK
};
```

```nix
# Shape 2 — buildPythonApplication (if you have a real Python package)
packages.${system}.default = pkgs.python3Packages.buildPythonApplication {
  pname   = ___;                            # YOUR TASK
  version = ___;                            # YOUR TASK
  src     = ___;                            # YOUR TASK — usually ./app
  format  = ___;                            # YOUR TASK — "other" if no pyproject/setup
  propagatedBuildInputs = with pkgs.python3Packages; [ ___ ]; # YOUR TASK — flask, etc.
  nativeBuildInputs     = [ pkgs.makeWrapper ];
  installPhase = ___;                       # YOUR TASK — copy app.py → $out/bin, wrapProgram with PYTHONPATH
};
```

```nix
# Shape 3 — buildGoModule (if you did the Lab 1 Go bonus)
packages.${system}.default = pkgs.buildGoModule {
  pname      = ___;                         # YOUR TASK
  version    = ___;                         # YOUR TASK
  src        = ___;                         # YOUR TASK
  vendorHash = ___;                         # YOUR TASK — null if no deps; else a sha256 from lib.fakeHash discovery
};
```

> **Почему не PyPI?** Сборка Nix по умолчанию в **песочнице без сети**. Пакеты берутся из запиненного ревижена nixpkgs (`Flask` → `pkgs.python3Packages.flask`). Поэтому замыкание и воспроизводимо: каждая зависимость сама лежит по фиксированному пути `/nix/store/...`, а не «где-то после `pip resolve`».

<details>
<summary>📚 Где смотреть синтаксис деривации</summary>

- [nix.dev — сборка и запуск приложений на Python](https://nix.dev/tutorials/nixos/building-and-running-python-apps)
- [Руководство nixpkgs по Python](https://nixos.org/manual/nixpkgs/stable/#python)
- [`writeShellApplication`](https://nixos.org/manual/nixpkgs/stable/#trivial-builder-writeShellApplication) — самый лёгкий сборщик, с проверкой shellcheck
- [Nix Pills — Our first derivation](https://nixos.org/guides/nix-pills/our-first-derivation.html)

Выберите **один** сборщик и защитите выбор в `submission18.md`. На проверке снимают баллы, если сборщик не соответствует тому, что реально пакуется.

</details>

### 1.3 — Сгенерируйте `flake.lock` и закоммитьте его

`flake.lock` записывает точный git SHA каждого входа flake. Без него `nixpkgs/nixos-25.11` резолвится в *то, что сегодня на верхушке ветки*: пару недель терпимо, за месяцы тихо уезжает. Именно lock-файл делает «сборку в 2026» той же, что «сборка в 2036».

```bash
cd labs/lab18
git add flake.nix app/                    # flakes only see git-tracked files — see Pitfalls
nix flake update                           # writes flake.lock; pins each input to a specific rev
git add flake.lock                         # MUST be committed
nix build                                  # builds packages.${system}.default
./result/bin/<your-name>                   # runs your app
```

`ВАША ЗАДАЧА`: откройте `flake.lock` и **ответьте в `submission18.md`**:

- Какое поле пинит точный ревижен `nixpkgs`? (Подсказка: это не `ref`. Смотрите `locked.rev` и `locked.narHash`.)
- Зачем коммитить `flake.lock`, если `nixpkgs-24.11` уже запинен в `flake.nix`?

### 1.4 — Что приложить

**Вставьте в `labs/submission18.md`:**

- ваш `flake.nix` целиком (он короткий) и по одной строке на каждый заполненный `___`
- первые 20 строк `flake.lock` (узел `nixpkgs`, где видны `locked.rev` и `locked.narHash`)
- вывод команд:

  ```bash
  nix build .#default
  readlink result                            # /nix/store/<hash>-<name>  ← capture this exact path
  ./result/bin/<your-name>                   # proves the binary runs
  ```

- ответ одним абзацем на вопрос 1.3: что именно пинит lock

---

## Задание 2 — Воспроизводимый образ Docker через `dockerTools` (4 балла)

**Цель:** соберите образ контейнера **из своего flake** через `dockerTools.buildLayeredImage` и сопоставьте его с Dockerfile из Lab 2 (у того хеш меняется на каждой сборке).

### 2.1 — Покажите, что образ Lab 2 не воспроизводится

```bash
docker build -t lab2-app:v1 ./app_python && docker save lab2-app:v1 | sha256sum
docker build -t lab2-app:v2 ./app_python && docker save lab2-app:v2 | sha256sum
```

`ВАША ЗАДАЧА`: сохраните обе суммы SHA256 и убедитесь, что они разные. Они разные: Docker пишет время сборки в слои, а `python:3.13-slim` между двумя сборками может уехать. Обе суммы вставьте в `submission18.md`.

### 2.2 — Добавьте во flake выход `dockerImg`

Добавьте второй выход-пакет во flake из задания 1. Скелет ниже — **форма** `buildLayeredImage`. Каждое поле тела ваше. **Целого готового примера нет:** когда образец перед глазами целиком, это уже перепечатка, а не учёба.

```nix
# Inside the same outputs = { … } block from Task 1, alongside packages.${system}.default
packages.${system}.dockerImg = pkgs.dockerTools.buildLayeredImage {
  name = ___;          # YOUR TASK: the image name (becomes `docker images` REPOSITORY).
                       # Not "latest" — that's a tag, not a name. Pick something self-explanatory.

  tag  = ___;          # YOUR TASK: a SemVer / CalVer / git-SHA tag. NOT "latest" (slide 20 of Lec 2).

  contents = [ ___ ];  # YOUR TASK: what goes inside the image filesystem.
                       # At minimum, your Task-1 package (referenced by `self.packages.${system}.default`
                       # OR a local `let app = … in app` binding). Add pkgs.coreutils / pkgs.cacert
                       # only if your binary needs `ls`/`cat` or HTTPS at runtime.

  config = {
    Cmd          = [ ___ ];          # YOUR TASK: the command that runs at container start. Exec form.
                                     # Point at the binary your Task-1 derivation produced —
                                     # e.g. "${app}/bin/<name>". Shell form would re-introduce the PID-1 trap
                                     # you learned to avoid in Lab 2.
    ExposedPorts = { ___ = {}; };    # YOUR TASK: the port your service listens on (Lab 1 default: 5000/tcp;
                                     # Lab 2's container overrode this via PORT=8080 env var, but a fresh Nix
                                     # build with writeShellApplication honors Lab 1's PORT default).
                                     # The key is a string like "5000/tcp". If you want 8080, also add
                                     # `Env = [ "PORT=8080" ];` to this config block.
  };

  # THE reproducibility trick. Nix gives you a knob most container builders don't:
  # the embedded creation timestamp. Set it to a FIXED epoch instead of "now".
  created = ___;       # YOUR TASK: a fixed RFC-3339 timestamp string. NOT "now" — that re-introduces
                       # non-determinism (every build differs). NOT 1970-01-01T00:00:00Z either
                       # (Docker tooling sometimes rejects the literal Unix epoch). The accepted convention
                       # is `1970-01-01T00:00:01Z` — one second after the epoch.
};
```

> **`buildLayeredImage`** собирает компактный слоёный tarball с адресацией по содержимому: нет базового образа `FROM`, нет `apt-get`, нет `RUN`. Запиненная эпоха `created` убирает последний источник недетерминизма.

<details>
<summary>📚 Где смотреть dockerTools</summary>

- [nix.dev — сборка и запуск образов Docker](https://nix.dev/tutorials/nixos/building-and-running-docker-images.html)
- [Справочник nixpkgs по `dockerTools`](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-dockerTools) — `buildImage`, `buildLayeredImage` и `streamLayeredImage`
- [pkgs.cacert](https://search.nixos.org/packages?show=cacert) — если приложение ходит по HTTPS

</details>

### 2.3 — Соберите, загрузите, запустите

```bash
nix build .#dockerImg                  # → result is an image tarball, not a symlink to a binary
file result                            # confirms it's a gzip'd tar

# From inside the Nix container, the docker.sock mount lets this work; on host, you have docker directly
docker load < result                   # loads <your-name>:<your-tag>
docker run -d -p 5001:5000 --name lab18 <your-name>:<your-tag>      # if you exposed 5000/tcp
# or, if you set Env = [ "PORT=8080" ] AND ExposedPorts = { "8080/tcp" = {}; }:
# docker run -d -p 5001:8080 --name lab18 <your-name>:<your-tag>
curl http://localhost:5001/health      # behaves like the Lab 2 container
```

### 2.4 — Сравните с образом Lab 2

```bash
docker images --format '{{.Repository}}\t{{.Tag}}\t{{.Size}}' | grep -E 'lab2-app|<your-name>'
docker history <your-name>:<your-tag>
```

`ВАША ЗАДАЧА`: прочитайте свой `docker history`. У образа Nix обычно так:

- **хеши слоёв адресуются по содержимому** (одно и то же содержимое → один хеш слоя, на разных сборках и машинах)
- внутри **только замыкание**: нет базы Debian, нет кэша `apt`, часто меньше образа на базе slim
- время в колонке `CREATED` — **та эпоха, которую вы задали**, а не `now`

В `submission18.md` одним абзацем сравните образ Lab 2 и образ `dockerTools`: размер, число слоёв и колонку `CREATED`.

### 2.5 — Что приложить

**Вставьте в `labs/submission18.md`:**

- блок `dockerImg` вашего `flake.nix` (contents и config, которые вы написали)
- два **разных** вывода `docker save | sha256sum` из 2.1
- строка лога `docker load` с именем и тегом загруженного образа
- настоящий `curl` на `/health`: загруженный образ запускается и отвечает
- строки `docker images` и `docker history`, где Lab 2 стоит рядом с образом Nix
- два предложения: *почему* обычный `Dockerfile` не доходит до воспроизводимости бит в бит (трюк с эпохой, drift базового образа, недетерминизм `apt-get`). Сошлитесь на лекцию, слайды 15 и 18.

---

## Бонус — Докажите, что сборка воспроизводится бит в бит (2 балла)

**Цель:** запустите **два** независимых `nix build` и покажите, что оба встают на **один и тот же путь `/nix/store`**. Это заголовок всей лабораторной. Успешная сборка сама по себе ничего не доказывает. Доказывает *совпадение двух сборок*.

В эталонной сдаче доказательство такое: **два запуска `nix build .#default`, второй с `--rebuild`**, оба дают один путь в store, плюс явное равенство хешей. Флаг `--rebuild` заставляет Nix сделать настоящую пересборку (не попадание в кэш) и **проверить, что выход новой сборки бит в бит совпадает с уже лежащим путём в store**. Если хоть что-то разошлось, Nix скажет об этом громко.

`ВАША ЗАДАЧА`: придумайте доказательство и выполните его сами. Команд лабораторная НЕ даёт. Нужно:

1. Один раз соберите `packages.${system}.default` и сохраните получившийся путь в store
2. Заставьте Nix сделать **настоящую полную пересборку** (не попадание в кэш) и сохраните путь, на который она встала
3. Покажите, что пути **совпадают** и что сам Nix это подтверждает на шаге 2 ("identical store path both builds" / "checking outputs of …")
4. То же доказательство для `packages.${system}.dockerImg` (tarball образа): `sha256sum result` двух чистых сборок должен совпасть
5. В `submission18.md` объясните, *почему* это работает: роль хеша входов, почему без `flake.lock` нельзя, почему правильная команда — `--rebuild` (один `nix-store --delete` хрупок)

Подсказки (это не команды):

- Нужный флаг сам форсирует пересборку и проверяет равенство выхода. В `nix build --help` ищите опцию, в описании которой есть "rebuild ... and check that the result is the same."
- `readlink result` даёт путь. `sha256sum result` даёт хеш tarball образа Docker.
- Если пути разошлись, в сборке есть недетерминизм. Частые причины: незаменённый `created = "now"`, `mkDerivation` с незапиненным входом `fetch*`, нет `flake.lock`.

**Вставьте в `labs/submission18.md`:**

- две команды, которые вы запустили (порядок важен; нужные найдёте в `nix build --help`)
- сохранённый `readlink result` с обоих запусков `default` — одна и та же строка
- сохранённый `sha256sum result` с двух чистых пересборок `dockerImg` — один и тот же дайджест
- собственная строка подтверждения Nix (что-то вроде `checking outputs of '/nix/store/<drv>'... IDENTICAL store path both builds`)
- короткий абзац: как `flake.lock` закрывает поломку «у меня работает», с которой не справляются `requirements.txt` (Lab 1) и тег образа Helm (Lab 10)?

> 🏆 **В этом вся лабораторная.** Мало того, что flake собирается. «Воспроизводимый» значит, что flake **дважды** собирает **один и тот же путь в store**.

---

## Как сдавать

```bash
git switch -c lab18
git add labs/lab18/flake.nix labs/lab18/flake.lock labs/lab18/app/
git add labs/submission18.md
git commit -m "feat(lab18): Nix reproducible build with flake + dockerTools"
git push -u origin lab18
```

Откройте **два** PR:

- `your-fork:lab18` → `course-repo:master` *(на ревью)*
- `your-fork:lab18` → `your-fork:master` *(сольётся в ваш main, когда закончите)*

Чеклист PR:

```text
- [ ] Task 1 — flake.nix written by hand, flake.lock committed, app builds
- [ ] Task 2 — dockerImg output added; image loads and runs; comparison to Lab 2 Dockerfile documented
- [ ] Bonus — two independent builds land on the identical store path; sha256sum equality proven for the image
```

В описание PR включите **путь в store**, который вы воспроизвели (`/nix/store/<hash>-<name>`), и **sha256 tarball** с двух сборок `dockerImg`. Эти две строки — артефакт лабораторной.

---

## Критерии приёмки

### Задание 1 — Flake собирает приложение (6 баллов)

- ✅ `labs/lab18/flake.nix` написан по скелету (заполнены `description`, `inputs`, `outputs`)
- ✅ `inputs.nixpkgs.url` запинен на **релизный тег** (например, `nixos-25.11`), **не** на `nixpkgs-unstable`
- ✅ `packages.${system}.default` собирается одним из `writeShellApplication`, `mkDerivation`, `buildPythonApplication` или `buildGoModule`. Выбор обоснован в `submission18.md`
- ✅ `flake.lock` есть, **в коммите**, и в сдаче показано поле `locked.rev`
- ✅ `nix build` даёт рабочий бинарник в `result/bin/<name>`, который выполняет логику сервиса Lab 1

### Задание 2 — образ `dockerTools` (4 балла)

- ✅ во flake добавлен `packages.${system}.dockerImg`
- ✅ в `contents` у `buildLayeredImage` входит пакет из задания 1
- ✅ `config.Cmd` указывает на бинарник в exec-форме
- ✅ задан `config.ExposedPorts`
- ✅ `created` — **фиксированная строка эпохи** (не `"now"`), обычно `"1970-01-01T00:00:01Z"`
- ✅ `nix build .#dockerImg` даёт tarball; `docker load < result` проходит; загруженный образ запускается и отдаёт `/health`
- ✅ два запуска `docker build` для Dockerfile из Lab 2 дают **разные** выводы `docker save | sha256sum`, и они сохранены в `submission18.md`

### Бонус — доказательство бит в бит (2 балла)

- ✅ два независимых запуска `nix build .#default` (второй с настоящей пересборкой) дают **один и тот же** путь `/nix/store/<hash>-<name>`
- ✅ два независимых запуска `nix build .#dockerImg` дают **один и тот же** дайджест `sha256sum result`
- ✅ сохранено собственное подтверждение Nix: "identical store path / checking outputs"
- ✅ в `submission18.md` есть абзац про роль `flake.lock` против `requirements.txt` и тега образа Helm

---

## Критерии оценивания

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Flake, lock и сборка | **6** | Внешняя форма заполнена, nixpkgs запинен на релизный тег, сборщик деривации выбран верно, `flake.lock` в коммите, бинарник запускается |
| **Задание 2** — образ `dockerTools` | **4** | Выход образа добавлен, трюк с эпохой `created` применён, образ загружается и запускается, контраст с Lab 2 задокументирован обеими суммами `sha256sum` |
| **Бонус** — доказательство бит в бит | **2** | Две сборки → один и тот же путь в store и один и тот же sha256 образа; сохранена собственная проверка равенства от Nix |
| **Итого** | **12** | 10 за основное + 2 бонус |

**Как оценивают:**

- **10/10 за основное** — flake собирается, образ загружается, доказательство воспроизводимости чистое, разбор `flake.lock` показывает, что вы понимаете, *зачем* он
- **8–9/10** — сборки работают; заявка на воспроизводимость есть, но не хватает одного из главных артефактов (путь в store **или** sha256 образа)
- **6–7/10** — задание 1 работает; образ в задании 2 собирается, но эпоха `created` не выставлена или контраст с Lab 2 не измерен
- **<6/10** — flake не собирается, нет `flake.lock` или взят `nixpkgs-unstable` (воспроизводимости на самом деле нет)

---

## Материалы

<details>
<summary>📚 Основы Nix</summary>

- [nix.dev](https://nix.dev/) — официальные туториалы
- [Zero to Nix](https://zero-to-nix.com/) — трек для начинающих от Determinate Systems (сначала flakes)
- [Nix Pills](https://nixos.org/guides/nix-pills/) — глубокое погружение для любопытных
- [Поиск пакетов NixOS](https://search.nixos.org/) — найти нужный `pkgs.<name>`

</details>

<details>
<summary>📦 Flakes, dockerTools и доказательство</summary>

- [Flakes — NixOS Wiki](https://wiki.nixos.org/wiki/Flakes)
- [Practical Nix Flakes — Serokell](https://serokell.io/blog/practical-nix-flakes)
- [FlakeHub](https://flakehub.com/) — реестр flakes с SemVer
- [Справочник `dockerTools`](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-dockerTools)
- [`nix build --help`](https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-build.html) — нужный для бонуса флаг живёт здесь
- [Reproducible Builds](https://reproducible-builds.org/) — зачем это нужно за пределами Nix

</details>

<details>
<summary>🐧 Nix в контейнере и Nix на хосте</summary>

- [Образы Docker `nixos/nix`](https://hub.docker.com/r/nixos/nix) — запиненные теги (берите `2.34.0` или любой 2.25+)
- [Determinate Nix](https://determinate.systems/) — установщик на хост, flakes включены по умолчанию
- [Lix](https://lix.systems/) — форк сообщества, drop-in замена `nix`

</details>

<details>
<summary>⚠️ Типичные ловушки (из реальных пробных прогонов)</summary>

- **Flakes видят только файлы, которые уже в git.** Если сделать `nix build` до `git add flake.nix flake.lock app/`, Nix сообщит `path 'flake.nix' does not exist` или посчитает выражение по пустому дереву исходников. **Сначала всегда `git add`**, потом сборка. Это самая частая ошибка Lab 18. (На неё наткнулся эталонный прогон.)
- **`safe.directory` внутри контейнера `nixos/nix`.** Контейнер работает от `root` (uid 0), а примонтированный репозиторий на хосте принадлежит вашему uid. `nix build` вызывает libgit2, и тот отказывается: `repository path '/work/' is not owned by current user`. Починка один раз: `git config --global --add safe.directory /work` внутри контейнера. Это **артефакт Nix в контейнере**, не дефект flake. У Nix на хосте такого нет.
- **`dockerTools.buildLayeredImage` с `created = "now"` ломает воспроизводимость.** Каждая сборка даст другой хеш образа. В документации `"now"` указан как значение по умолчанию для `buildImage`. Именно его и надо переопределить. Берите `"1970-01-01T00:00:01Z"` (секунда после эпохи: буквальную эпоху часть инструментов отвергает).
- **`inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable"`.** Верхушка ветки едет каждый день, и сегодняшняя сборка не совпадёт со сборкой на следующей неделе. Смысл воспроизводимости пропадает. Берите релизный тег (`nixos-25.11`) или пиньте конкретный SHA коммита.
- **`nix flake update` переписывает `flake.lock`.** Не запускайте его между делом перед доказательством воспроизводимости: замыкание уедет у вас из-под ног. Запустите один раз, когда `flake.nix` уже написан, закоммитьте `flake.lock` и до конца лабораторной не трогайте.
- **`writeShellApplication` не прощает незаданные переменные.** Он сам вставляет `set -euo pipefail`. Если скрипт читает `$HOSTNAME`, а вы забыли добавить `runtimeInputs = [ pkgs.inetutils ];` (или задать переменную явно), сборка пройдёт, а скрипт упадёт в рантайме. Чище взять значение, которое уже разрешил Nix.
- **`nix-store --delete`, чтобы «принудить пересборку», хрупок.** Nix вполне может достать закэшированный выход откуда-то ещё. Для доказательства в бонусе берите `nix build --rebuild`: он явно делает нужное (настоящая пересборка и проверка равенства выхода).
- **`format = "other"` в `buildPythonApplication`.** Нужен, когда в исходниках нет `pyproject.toml` / `setup.py`. Без него Nix пытается вызвать `pip install .` и падает. Однофайловым приложениям Flask почти всегда нужен `format = "other"`.
- **`USER` в конфиге `dockerTools` — строка, не число.** `config.User = "1000:1000"` работает. `config.User = 1000` вычисляется нормально, но Docker может ругаться на `docker run`. Используйте форму `"uid:gid"`.

</details>

<details>
<summary>💡 Подсказки</summary>

1. Пути в store адресуются по содержимому: одинаковые входы → одинаковый хеш выхода. Всегда.
2. `nix-shell -p <pkg>` (классический CLI) или `nix shell nixpkgs#<pkg>` (новый CLI) открывает шелл, где этот пакет есть в `$PATH`. Удобно проверить зависимость, прежде чем класть её в деривацию.
3. `nix-collect-garbage -d` освобождает место, занятое старыми сборками.
4. Сборки идут в песочнице **без сети**. Каждую зависимость нужно объявить. Если сборке загадочно нужен интернет, вы забыли вход.
5. Для `nixpkgs` берите **релизную ветку** (`nixos-25.11`, `nixos-24.05`), *не* unstable. Релизный тег и делает осмысленным «собрать сегодня и в 2036».
6. `nix flake show` перечисляет каждый выход flake. Полезно, когда блок `packages.${system}` разрастается.

</details>

---

## Что дальше

Nix **ортогонален** оркестрации, которую вы осваивали весь остальной курс: он решает воспроизводимость, не планирование. Где он встретится снова:

| Область | Что добавляет Nix |
|---|---|
| CI | одна строка `nix build` вместо шагов «поставь N инструментов»; кэш сборки через Cachix |
| Dev shell | `nix develop` даёт каждому одни и те же версии Python/Node/Go — без drift `venv` / `asdf` / `nvm` |
| Сборка образов | образы `dockerTools` встают в те же registry и Helm-чарты, что вы использовали в Lab 10–13 |
| Аудит безопасности | артефакт — само замыкание; SBOM получается по построению |
| NixOS | если захотите ту же чистоту на уровне **ОС** |

---

**Прямо:** контейнеры решили *выкладку*. Nix решает *сборку*. Две разные задачи, два разных инструмента. Единственный способ убедиться, что уезжает ровно то, что вы написали, — собрать дважды и увидеть, что хеши совпали. ❄️
