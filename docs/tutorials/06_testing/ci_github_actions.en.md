# GitHub Actions CI

> **Goal:** every `git push` that changes the application code automatically runs the same tests
> you have just learned to write. On a clean GitHub machine, without you.

This page walks through the repository's **real** workflow —
[`.github/workflows/django-tests.yml`](https://github.com/NikoriakViktor/notes_chat_app/blob/main/.github/workflows/django-tests.yml).
All YAML fragments below are copied from that file, nothing is added.

---

## Where the workflow lives

```
notes_chat_app/                   ← корінь репозиторію
└── .github/
    └── workflows/
        ├── django-tests.yml      ← тести Django (ця сторінка)
        └── docs.yml              ← збірка і публікація цієї книги
```

GitHub Actions reads workflows **only** from the `.github/workflows/` directory at the repository root.

---

## What happens on `git push`

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

Redis is **not** started in CI: without `REDIS_URL`, Channels uses the InMemory channel layer,
which is enough for the consumer tests.

---

## The workflow line by line

### Name and triggers

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

| Key | What it means |
|-----|---------------|
| `name` | The name in the **Actions** tab: `Django Tests` |
| `push` / `pull_request` | Runs on a push to `main`/`master` or a PR into those branches |
| `paths` | Only when the application code, templates, static files, dependencies or the workflow itself change. Changes only in `docs/` do **not** trigger this workflow |
| `workflow_dispatch` | The **Run workflow** button in GitHub UI → Actions |

---

### Job 1: Unit, Integration & Consumer Tests

```yaml
jobs:

  # ── Job 1: Unit + Integration + WebSocket Consumer тести ──────────────────────
  unit-and-integration:
    name: Unit, Integration & Consumer Tests
    runs-on: ubuntu-latest
```

**`runs-on: ubuntu-latest`** — a fresh VM from GitHub. It is destroyed when the job ends; there is no state between runs.

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

**`services`** — Docker containers that GitHub starts next to the job. The service is called `db` here,
but its port is published on `localhost:5432`, so `DATABASE_URL` points at `localhost`.
`--health-cmd pg_isready` + `--health-retries 5` — GitHub waits until PostgreSQL is healthy before it starts the steps.

The only environment variable is `DATABASE_URL`: without it `notes_project/settings.py` stops with
`Exception: DATABASE_URL не встановлено. Запускай через docker compose.`

---

**Steps 1–3 — checkout, Python, pip cache**

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

- `actions/checkout@v4` — clones the repository onto the runner. Without this step the machine has none of your code.
- `actions/setup-python@v5` — installs Python 3.12.
- `actions/cache@v4` — caches `~/.cache/pip`. The key depends on `hashFiles('requirements.txt')`: while `requirements.txt` is unchanged, packages come from the cache.

---

**Steps 4–5 — dependencies and system check**

```yaml
      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Django system check
        run: python manage.py check
```

`manage.py check` catches configuration errors (a wrong `INSTALLED_APPS`, broken URLs) before any test runs.

---

**Step 6 — the tests**

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

Six modules are listed explicitly — without `test_selenium` (it runs in Job 2) and without `test_tasks`.
Together that is **209 tests**: models 25, services 65, forms 22, views 84, consumers 9, api 4.

`-v 2` — every test on its own line. In CI this is handy: you see the name of the failing test right away.

If even one test fails → the command returns exit code 1 → the step fails → the job stops → Job 2 does not start.

---

**Steps 7–8 — coverage and the artifact**

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

The tests run a second time, now under `coverage`. `coverage xml` writes the report to `coverage.xml`,
and `actions/upload-artifact@v4` attaches it to the run as `coverage-xml`.

---

### Job 2: Selenium E2E Tests

```yaml
  selenium-e2e:
    name: Selenium E2E Tests
    runs-on: ubuntu-latest
    needs: unit-and-integration
```

**`needs: unit-and-integration`** — Job 2 starts only after Job 1 succeeds. If the unit tests fail, Selenium never starts.

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

- `selenium/standalone-chrome:latest` — Chrome in a container, driven through WebDriver on port 4444.
- `--shm-size=2gb` — Chrome needs a lot of shared memory; without it, it can crash.
- `--add-host=host.docker.internal:host-gateway` — Chrome inside the container opens `http://host.docker.internal:PORT` and reaches the Django test server on the runner.
- `--health-cmd "curl -f http://localhost:4444/wd/hub/status"` — wait until Selenium is ready to accept sessions.
- `SELENIUM_REMOTE_URL` — `_make_headless_driver()` in `notes_app/tests/test_selenium.py` creates a Remote WebDriver instead of a local Chrome.
- `WEB_HOST=host.docker.internal` — `_DockerLiveServerMixin` binds the test server to `0.0.0.0` and puts `host.docker.internal` into `live_server_url`.

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

There is no separate `migrate` step: `manage.py test` creates the test database and applies migrations itself.

---

## The full workflow (as in the repository)

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

The same six modules under `coverage`, run locally against PostgreSQL 16:

```bash
coverage run manage.py test \
  notes_app.tests.test_models notes_app.tests.test_services notes_app.tests.test_forms \
  notes_app.tests.test_views notes_app.tests.test_consumers notes_app.tests.test_api
coverage report --show-missing
```

Real output (the application modules; the full report also lists migrations, tests and `notes_project/*`):

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

`coverage xml -o coverage.xml` — the XML format for Codecov, SonarQube or viewing in an IDE.

---

## How to view the results

**1. The repository's Actions tab**

```
https://github.com/NikoriakViktor/notes_chat_app/actions
```

The list of all runs. Green means everything passed, red means something failed.

**2. The jobs graph**

Click a specific run — you see two boxes:

```
[Unit, Integration & Consumer Tests] ──needs──► [Selenium E2E Tests]
```

**3. The log of a specific step**

Click a job → a step → expand it. This is what the `-v 2` lines look like (real output for `TagModelTest`):

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

The second line of each test is the first line of its docstring: Django prints it at `-v 2`.

**4. Download the artifact**

Run → the **Artifacts** section at the bottom of the page → `coverage-xml`.

**5. Status on the commit**

Every commit and PR gets a badge: ✅ — the workflow passed, ❌ — it failed, 🟡 — in progress.

---

## The connection: tests ↔ CI

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

## What the lines in the log mean when there is an error

The example below is a **deliberately broken** filter in `selectors` (the repository has no such bug).
The test `test_note_list_excludes_other_user_notes` checks that Alice does not see Bob's notes:

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

## Documentation links

| Topic | Link |
|-------|------|
| GitHub Actions — Quick Start | https://docs.github.com/en/actions/writing-workflows/quickstart |
| Understanding GitHub Actions | https://docs.github.com/en/actions/about-github-actions/understanding-github-actions |
| Events that trigger workflows | https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows |
| Workflow syntax (full reference) | https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions |
| Service containers | https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/about-service-containers |
| Caching dependencies | https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows |
| GitHub-hosted runners | https://docs.github.com/en/actions/using-github-hosted-runners/using-github-hosted-runners/about-github-hosted-runners |
| actions/checkout | https://github.com/marketplace/actions/checkout |
| actions/setup-python | https://github.com/marketplace/actions/setup-up-a-specific-version-of-python |
| actions/cache | https://github.com/marketplace/actions/cache |
| actions/upload-artifact | https://github.com/marketplace/actions/upload-a-build-artifact |
| Coverage.py | https://coverage.readthedocs.io/en/latest/ |
| Django system check | https://docs.djangoproject.com/en/5.2/topics/checks/ |

CI/CD theory in the book: [CI/CD](../../08_testing_and_quality/ci_cd_full.md).

---

## Practice: check that CI works

1. Open `https://github.com/NikoriakViktor/notes_chat_app/actions` (or the Actions tab of your fork).
2. Find the **Django Tests** workflow.
3. Click the latest run.
4. Expand the **Unit, Integration & Consumer Tests** job → the **Run unit, integration and consumer tests** step.
5. Find the lines `Ran 209 tests in ...` and `OK`.
6. Go to the **Selenium E2E Tests** job — watch the `selenium/standalone-chrome` container start.

**On your own:** in your fork, add a deliberate error to any test in `notes_app/tests/` (`self.assertFalse(True)`),
`git push`, and watch CI show the FAIL and the Traceback. Remember `paths`: the change must be under `notes_app/**`,
otherwise the workflow does not start.

---

## Next

- **[Checkpoint](checkpoint.md)** — expected result, checklist, practice task
