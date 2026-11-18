# GitHub Actions CI

> **Мета:** кожен `git push`, який змінює код застосунку, автоматично запускає ті самі тести,
> які ти щойно навчився писати. На чистій машині GitHub, без твоєї участі.

Ця сторінка розбирає **реальний** workflow репозиторію —
[`.github/workflows/django-tests.yml`](https://github.com/NikoriakViktor/notes_chat_app/blob/main/.github/workflows/django-tests.yml).
Усі фрагменти YAML нижче скопійовані з цього файлу, нічого не додано.

---

## Де живе workflow

```
notes_chat_app/                   ← корінь репозиторію
└── .github/
    └── workflows/
        ├── django-tests.yml      ← тести Django (ця сторінка)
        └── docs.yml              ← збірка і публікація цієї книги
```

GitHub Actions читає workflow **тільки** з директорії `.github/workflows/` у корені репозиторію.

---

## Що відбувається при `git push`

```
Ти:  git push origin main   (змінено notes_app/, notes_project/, templates/, static/,
         │                    requirements.txt або сам workflow)
         ▼
GitHub:  подія push → workflow "Django Tests"
         │
         ▼
Runner:  нова чиста VM ubuntu-latest
         │
         ├── Job 1: unit-and-integration — "Unit, Integration & Consumer Tests"
         │     PostgreSQL service (postgres:16-alpine)
         │     checkout → Python 3.12 → pip cache → pip install
         │     → manage.py check
         │     → manage.py test models/services/forms/views/consumers/api
         │     → coverage report + артефакт coverage-xml
         │     Результат: ✅ або ❌
         │
         └── Job 2: selenium-e2e — "Selenium E2E Tests" (тільки якщо Job 1 ✅)
               PostgreSQL service + selenium/standalone-chrome service
               checkout → Python 3.12 → pip cache → pip install
               → manage.py test notes_app.tests.test_selenium
               Результат: ✅ або ❌
         │
         ▼
GitHub:  зелений/червоний статус на коміті або PR
```

Redis у CI **не** запускається: без `REDIS_URL` Channels використовує InMemory channel layer,
і тестам consumer-ів цього достатньо.

---

## Workflow рядок за рядком

### Назва і тригери

```yaml
name: Django Tests

# ── Тригери ───────────────────────────────────────────────────────────────────
on:
  push:
    branches: [ main, master ]
    paths:
      - 'notes_app/**'
      - 'notes_project/**'
      - 'templates/**'
      - 'static/**'
      - 'requirements.txt'
      - '.github/workflows/django-tests.yml'
  pull_request:
    branches: [ main, master ]
    paths:
      - 'notes_app/**'
      - 'notes_project/**'
      - 'templates/**'
      - 'static/**'
      - 'requirements.txt'
      - '.github/workflows/django-tests.yml'
  workflow_dispatch:   # кнопка "Run workflow" в GitHub UI → Actions
```

| Ключ | Що означає |
|------|-----------|
| `name` | Назва у вкладці **Actions**: `Django Tests` |
| `push` / `pull_request` | Запуск при push у `main`/`master` або PR у ці гілки |
| `paths` | Тільки якщо змінено код застосунку, шаблони, статику, залежності або сам workflow. Зміни лише в `docs/` цей workflow **не** запускають |
| `workflow_dispatch` | Кнопка **Run workflow** у GitHub UI → Actions |

---

### Job 1: Unit, Integration & Consumer Tests

```yaml
jobs:

  # ── Job 1: Unit + Integration + WebSocket Consumer тести ──────────────────────
  unit-and-integration:
    name: Unit, Integration & Consumer Tests
    runs-on: ubuntu-latest
```

**`runs-on: ubuntu-latest`** — свіжа VM від GitHub. Після завершення вона знищується, між запусками немає стану.

```yaml
    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: notes_db
          POSTGRES_USER: notes_user
          POSTGRES_PASSWORD: notes_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgres://notes_user:notes_pass@localhost:5432/notes_db
```

**`services`** — Docker-контейнери, які GitHub запускає поруч із job. Сервіс тут називається `db`,
але порт прокинуто на `localhost:5432`, тому `DATABASE_URL` дивиться на `localhost`.
`--health-cmd pg_isready` + `--health-retries 5` — GitHub чекає, поки PostgreSQL стане healthy, і лише тоді починає steps.

Єдина змінна середовища — `DATABASE_URL`: без неї `notes_project/settings.py` зупиняється з
`Exception: DATABASE_URL не встановлено. Запускай через docker compose.`

---

**Steps 1–3 — checkout, Python, кеш pip**

```yaml
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
```

- `actions/checkout@v4` — клонує репозиторій на runner. Без цього кроку на машині немає жодного файлу з твоїм кодом.
- `actions/setup-python@v5` — встановлює Python 3.12.
- `actions/cache@v4` — кешує `~/.cache/pip`. Ключ залежить від `hashFiles('requirements.txt')`: поки `requirements.txt` не змінився, пакети беруться з кешу.

---

**Steps 4–5 — залежності і system check**

```yaml
      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Django system check
        run: python manage.py check
```

`manage.py check` ловить помилки конфігурації (неправильний `INSTALLED_APPS`, зламані URL-и) ще до запуску тестів.

---

**Step 6 — тести**

```yaml
      - name: Run unit, integration and consumer tests
        run: |
          python manage.py test \
            notes_app.tests.test_models \
            notes_app.tests.test_services \
            notes_app.tests.test_forms \
            notes_app.tests.test_views \
            notes_app.tests.test_consumers \
            notes_app.tests.test_api \
            -v 2
```

Явно перераховані шість модулів — без `test_selenium` (він у Job 2) і без `test_tasks`.
Разом це **209 тестів**: models 25, services 65, forms 22, views 84, consumers 9, api 4.

`-v 2` — кожен тест окремим рядком. У CI це зручно: одразу видно, яке ім'я тесту впало.

Якщо хоча б один тест падає → команда повертає exit code 1 → step провалений → job зупиняється → Job 2 не стартує.

---

**Steps 7–8 — coverage і артефакт**

```yaml
      - name: Generate coverage report
        run: |
          coverage run manage.py test \
            notes_app.tests.test_models \
            notes_app.tests.test_services \
            notes_app.tests.test_forms \
            notes_app.tests.test_views \
            notes_app.tests.test_consumers \
            notes_app.tests.test_api
          coverage report --show-missing
          coverage xml -o coverage.xml

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-xml
          path: coverage.xml
```

Тести проганяються вдруге, вже під `coverage`. `coverage xml` зберігає звіт у `coverage.xml`,
а `actions/upload-artifact@v4` прикріплює його до run під іменем `coverage-xml`.

---

### Job 2: Selenium E2E Tests

```yaml
  selenium-e2e:
    name: Selenium E2E Tests
    runs-on: ubuntu-latest
    needs: unit-and-integration
```

**`needs: unit-and-integration`** — Job 2 стартує тільки після успіху Job 1. Якщо unit-тести впали, Selenium навіть не запускається.

```yaml
    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: notes_db
          POSTGRES_USER: notes_user
          POSTGRES_PASSWORD: notes_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

      selenium:
        image: selenium/standalone-chrome:latest
        ports:
          - 4444:4444
        options: >-
          --add-host=host.docker.internal:host-gateway
          --shm-size=2gb
          --health-cmd "curl -f http://localhost:4444/wd/hub/status"
          --health-interval 5s
          --health-timeout 10s
          --health-retries 5

    env:
      DATABASE_URL: postgres://notes_user:notes_pass@localhost:5432/notes_db
      SELENIUM_REMOTE_URL: http://localhost:4444/wd/hub
      WEB_HOST: host.docker.internal
```

- `selenium/standalone-chrome:latest` — Chrome у контейнері, керується через WebDriver на порту 4444.
- `--shm-size=2gb` — Chrome потребує багато shared memory, без цього він може падати.
- `--add-host=host.docker.internal:host-gateway` — Chrome у контейнері відкриває `http://host.docker.internal:PORT` і бачить Django test server на runner.
- `--health-cmd "curl -f http://localhost:4444/wd/hub/status"` — чекаємо, поки Selenium готовий приймати сесії.
- `SELENIUM_REMOTE_URL` — `_make_headless_driver()` у `notes_app/tests/test_selenium.py` створює Remote WebDriver замість локального Chrome.
- `WEB_HOST=host.docker.internal` — `_DockerLiveServerMixin` прив'язує test server до `0.0.0.0` і підставляє `host.docker.internal` у `live_server_url`.

```yaml
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install dependencies
        run: pip install -r requirements.txt

      # SELENIUM_REMOTE_URL → _make_headless_driver() → Remote WebDriver → selenium контейнер
      # WEB_HOST=host.docker.internal → Django test server bind 0.0.0.0, URL = host.docker.internal
      - name: Run Selenium E2E tests
        run: python manage.py test notes_app.tests.test_selenium -v 2
```

Окремого кроку `migrate` немає: `manage.py test` сам створює тестову БД і застосовує міграції.

---

## Повний workflow (як у репозиторії)

```yaml
# GitHub Actions CI — Django Tests (notes_chat_app)

# ЯК ПЕРЕГЛЯНУТИ РЕЗУЛЬТАТИ:
#   https://github.com/NikoriakViktor/notes_chat_app/actions

name: Django Tests

# ── Тригери ───────────────────────────────────────────────────────────────────
on:
  push:
    branches: [ main, master ]
    paths:
      - 'notes_app/**'
      - 'notes_project/**'
      - 'templates/**'
      - 'static/**'
      - 'requirements.txt'
      - '.github/workflows/django-tests.yml'
  pull_request:
    branches: [ main, master ]
    paths:
      - 'notes_app/**'
      - 'notes_project/**'
      - 'templates/**'
      - 'static/**'
      - 'requirements.txt'
      - '.github/workflows/django-tests.yml'
  workflow_dispatch:   # кнопка "Run workflow" в GitHub UI → Actions

# ── Jobs ──────────────────────────────────────────────────────────────────────
jobs:

  # ── Job 1: Unit + Integration + WebSocket Consumer тести ──────────────────────
  unit-and-integration:
    name: Unit, Integration & Consumer Tests
    runs-on: ubuntu-latest

    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: notes_db
          POSTGRES_USER: notes_user
          POSTGRES_PASSWORD: notes_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgres://notes_user:notes_pass@localhost:5432/notes_db

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Django system check
        run: python manage.py check

      - name: Run unit, integration and consumer tests
        run: |
          python manage.py test \
            notes_app.tests.test_models \
            notes_app.tests.test_services \
            notes_app.tests.test_forms \
            notes_app.tests.test_views \
            notes_app.tests.test_consumers \
            notes_app.tests.test_api \
            -v 2

      - name: Generate coverage report
        run: |
          coverage run manage.py test \
            notes_app.tests.test_models \
            notes_app.tests.test_services \
            notes_app.tests.test_forms \
            notes_app.tests.test_views \
            notes_app.tests.test_consumers \
            notes_app.tests.test_api
          coverage report --show-missing
          coverage xml -o coverage.xml

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-xml
          path: coverage.xml


  # ── Job 2: Selenium E2E тести ─────────────────────────────────────────────────
  #
  # Selenium через selenium/standalone-chrome контейнер (як у docker-compose.yml).
  #
  # Мережева схема:
  #   runner (Django test server на 0.0.0.0:PORT)
  #   ↕  через host.docker.internal (bridge gateway)
  #   selenium container → SELENIUM_REMOTE_URL = http://localhost:4444/wd/hub
  #
  # --add-host=host.docker.internal:host-gateway:
  #   Додає запис у /etc/hosts selenium-контейнера.
  #   host.docker.internal → IP Docker gateway (= IP runner-машини).
  #   Chrome всередині контейнера відкриває http://host.docker.internal:PORT
  #   і бачить Django test server, що слухає на 0.0.0.0.
  #
  # WEB_HOST=host.docker.internal:
  #   _DockerLiveServerMixin в test_selenium.py бачить WEB_HOST і:
  #   1. Прив'язує Django test server до 0.0.0.0 (не тільки 127.0.0.1)
  #   2. Замінює 0.0.0.0 на host.docker.internal у live_server_url
  #   Тож Selenium отримує URL http://host.docker.internal:PORT — досяжний із контейнера.
  selenium-e2e:
    name: Selenium E2E Tests
    runs-on: ubuntu-latest
    needs: unit-and-integration

    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: notes_db
          POSTGRES_USER: notes_user
          POSTGRES_PASSWORD: notes_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

      selenium:
        image: selenium/standalone-chrome:latest
        ports:
          - 4444:4444
        options: >-
          --add-host=host.docker.internal:host-gateway
          --shm-size=2gb
          --health-cmd "curl -f http://localhost:4444/wd/hub/status"
          --health-interval 5s
          --health-timeout 10s
          --health-retries 5

    env:
      DATABASE_URL: postgres://notes_user:notes_pass@localhost:5432/notes_db
      SELENIUM_REMOTE_URL: http://localhost:4444/wd/hub
      WEB_HOST: host.docker.internal

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install dependencies
        run: pip install -r requirements.txt

      # SELENIUM_REMOTE_URL → _make_headless_driver() → Remote WebDriver → selenium контейнер
      # WEB_HOST=host.docker.internal → Django test server bind 0.0.0.0, URL = host.docker.internal
      - name: Run Selenium E2E tests
        run: python manage.py test notes_app.tests.test_selenium -v 2
```

---

## Coverage report

Ті самі шість модулів під `coverage`, запущені локально проти PostgreSQL 16:

```bash
coverage run manage.py test \
  notes_app.tests.test_models notes_app.tests.test_services notes_app.tests.test_forms \
  notes_app.tests.test_views notes_app.tests.test_consumers notes_app.tests.test_api
coverage report --show-missing
```

Реальний вивід (рядки модулів застосунку; повний звіт містить також міграції, тести і `notes_project/*`):

```
Found 209 test(s).
...
Ran 209 tests in 115.805s

OK
```

```
Name                                                                         Stmts   Miss  Cover   Missing
----------------------------------------------------------------------------------------------------------
notes_app/api.py                                                                37      0   100%
notes_app/consumers.py                                                          53      4    92%   173-174, 233-234
notes_app/forms.py                                                             141      2    99%   249, 324
notes_app/models.py                                                            138      6    96%   32, 143, 165, 181, 204, 272
notes_app/selectors.py                                                          58      8    86%   30, 54, 121, 142-143, 156-158
notes_app/services.py                                                          162      2    99%   40-41
notes_app/views.py                                                             435     97    78%   73-74, 148-149, 187-188, ...
----------------------------------------------------------------------------------------------------------
TOTAL                                                                         2308    135    94%
```

`coverage xml -o coverage.xml` — XML-формат для Codecov, SonarQube або перегляду в IDE.

---

## Як переглянути результати

**1. Вкладка Actions репозиторію**

```
https://github.com/NikoriakViktor/notes_chat_app/actions
```

Список усіх запусків. Зелений — усе пройшло, червоний — щось впало.

**2. Граф jobs**

Клікни на конкретний run — побачиш два прямокутники:

```
[Unit, Integration & Consumer Tests] ──needs──► [Selenium E2E Tests]
```

**3. Лог конкретного step**

Клікни на job → на step → розгорни. Так виглядають рядки `-v 2` (реальний вивід для `TagModelTest`):

```
Found 4 test(s).
...
test_str_returns_hash_name (notes_app.tests.test_models.TagModelTest.test_str_returns_hash_name)
ЩО ПЕРЕВІРЯЄМО: __str__ тегу повертає '#python', а не просто 'python'. ... ok
test_unique_together_same_user_same_name_raises (notes_app.tests.test_models.TagModelTest.test_unique_together_same_user_same_name_raises)
ЩО ПЕРЕВІРЯЄМО: не можна створити два теги з однаковою назвою ... ok
...
----------------------------------------------------------------------
Ran 4 tests in 2.539s

OK
```

Другий рядок кожного тесту — перший рядок його docstring: Django виводить його при `-v 2`.

**4. Завантажити артефакт**

Run → розділ **Artifacts** унизу сторінки → `coverage-xml`.

**5. Статус на коміті**

Кожен коміт і PR отримує значок: ✅ — workflow пройшов, ❌ — впав, 🟡 — виконується.

---

## Зв'язок: тести ↔ CI

```
Ти пишеш:                  CI запускає:                         Ти отримуєш:

test_models.py    ─────►   Job 1 unit-and-integration    ──►  ✅ або ❌
test_services.py  ─────►                                 ──►  Traceback прямо в GitHub
test_forms.py     ─────►                                 ──►  артефакт coverage-xml
test_views.py     ─────►
test_consumers.py ─────►
test_api.py       ─────►

test_selenium.py  ─────►   Job 2 selenium-e2e            ──►  ✅ або ❌ після Job 1
                           (needs: unit-and-integration)       Chrome у Docker-контейнері

test_tasks.py     ─────►   у CI не запускається — тільки локально
```

---

## Що означають рядки у лозі при помилці

Приклад нижче — **навмисно зламаний** фільтр у `selectors` (такого бага в репозиторії немає).
Тест `test_note_list_excludes_other_user_notes` перевіряє, що Alice не бачить нотаток Bob'а:

```
FAIL: test_note_list_excludes_other_user_notes (notes_app.tests.test_views.NoteListViewTest.test_note_list_excludes_other_user_notes)
      ↑ Назва тесту, клас і модуль → одразу зрозуміло ЩО зламалось і де шукати

Traceback (most recent call last):
  File ".../notes_app/tests/test_views.py", line ..., in test_note_list_excludes_other_user_notes
    self.assertNotContains(response, 'Bob Secret Note')
AssertionError: ...
      ↑ Очікували, що рядка НЕМАЄ, але він є → нотатки Bob'а видимі Alice
      → баг у фільтрі по user у selectors

Process completed with exit code 1
      ↑ test runner повернув 1 → GitHub Actions вважає step проваленим
```

---

## Посилання на документацію

| Тема | Посилання |
|------|-----------|
| GitHub Actions — Quick Start | https://docs.github.com/en/actions/writing-workflows/quickstart |
| Understanding GitHub Actions | https://docs.github.com/en/actions/about-github-actions/understanding-github-actions |
| Events that trigger workflows | https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows |
| Workflow syntax (повний довідник) | https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions |
| Service containers | https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/about-service-containers |
| Caching dependencies | https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows |
| GitHub-hosted runners | https://docs.github.com/en/actions/using-github-hosted-runners/using-github-hosted-runners/about-github-hosted-runners |
| actions/checkout | https://github.com/marketplace/actions/checkout |
| actions/setup-python | https://github.com/marketplace/actions/setup-up-a-specific-version-of-python |
| actions/cache | https://github.com/marketplace/actions/cache |
| actions/upload-artifact | https://github.com/marketplace/actions/upload-a-build-artifact |
| Coverage.py | https://coverage.readthedocs.io/en/latest/ |
| Django system check | https://docs.djangoproject.com/en/5.2/topics/checks/ |

Теорія CI/CD у книзі: [CI/CD](../../08_testing_and_quality/ci_cd_full.md).

---

## Практика: перевір, що CI працює

1. Відкрий `https://github.com/NikoriakViktor/notes_chat_app/actions` (або Actions свого форку).
2. Знайди workflow **Django Tests**.
3. Клікни на останній run.
4. Розгорни Job **Unit, Integration & Consumer Tests** → step **Run unit, integration and consumer tests**.
5. Знайди рядки `Ran 209 tests in ...` і `OK`.
6. Перейди до Job **Selenium E2E Tests** — подивись, як піднімається контейнер `selenium/standalone-chrome`.

**Самостійно:** у своєму форку додай навмисну помилку в будь-який тест (`self.assertFalse(True)`) у `notes_app/tests/`,
зроби `git push` і подивись, як CI показує FAIL і Traceback. Пам'ятай про `paths`: зміна має бути в `notes_app/**`,
інакше workflow не запуститься.

---

## Далі

- **[Checkpoint](checkpoint.md)** — очікуваний результат, чеклист, практичне завдання
