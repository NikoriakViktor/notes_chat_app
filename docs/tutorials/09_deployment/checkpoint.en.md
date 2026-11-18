# Checkpoint: step 9

## Check that the step works { #check }

The Compose configuration is read without errors and contains these services (from the repository root):

```text
$ docker compose config --services
db
redis
web
nginx
celery-beat
celery-worker
$ docker compose --profile '*' config --services
db
redis
web
nginx
selenium
selenium-tests
celery-beat
celery-worker
ngrok
```

`selenium` and `selenium-tests` are in the `test` profile, `ngrok` is in the `tunnel` profile: a plain `docker compose up` does not start them.

What the `web` container does at startup is `entrypoint.sh` (here it was run locally with `RUN_COLLECTSTATIC=0`, `PORT=18012`, `UVICORN_RELOAD=0` and `DATABASE_URL` pointing to PostgreSQL 16):

```text
[entrypoint] Applying database migrations...
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, notes_app, sessions
Running migrations:
  No migrations to apply.
[entrypoint] Starting ASGI web server on port 18012...
INFO:     Started server process [21595]
INFO:     Waiting for application startup.
INFO:     ASGI 'lifespan' protocol appears unsupported.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:18012 (Press CTRL+C to quit)
INFO:     127.0.0.1:54024 - "GET /accounts/login/ HTTP/1.1" 200 OK
```

In the container the port is `8001` (the default `PORT`), between migrate and uvicorn there is also `collectstatic --noinput`, and with `CREATE_SUPERUSER=1` / `SEED_DEMO_DATA=1` the admin user and the demo data are created. `GET /accounts/login/` is the same request the `web` healthcheck in `docker-compose.yml` uses to decide that the service is `healthy`.

A production check of the settings with `DJANGO_DEBUG=0` and the dev key from `docker-compose.yml`:

```text
$ python manage.py check --deploy
System check identified some issues:

WARNINGS:
?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting. ...
?: (security.W008) Your SECURE_SSL_REDIRECT setting is not set to True. ...
?: (security.W009) Your SECRET_KEY has less than 50 characters, less than 5 unique characters, or it's prefixed with 'django-insecure-' ...
?: (security.W012) SESSION_COOKIE_SECURE is not set to True. ...
?: (security.W016) You have 'django.middleware.csrf.CsrfViewMiddleware' in your MIDDLEWARE, but you have not set CSRF_COOKIE_SECURE to True. ...

System check identified 5 issues (0 silenced).
```

This is expected for a local stack without HTTPS: each warning is an item of the [production checklist](production_checklist.md). Before a real deployment `W009` must go away (your own `SECRET_KEY` in `.env`), and the rest go away once HTTPS is on.

The tests are the same set as in CI (`Found 209 test(s)` … `OK`, see [step 6](../06_testing/checkpoint.md#check)).

The full stack `docker compose up --build`, nginx on `http://localhost`, ngrok and `docker compose --profile test run --rm selenium-tests` were **not run** for this checkpoint: on the machine used for the check, ports 80 and 8001 were taken by another stack, and the `notes-chat-app` project containers could not be restarted. Check the checklist items below that concern a running stack on your own machine.

## Checklist { #checklist }

**Starting the stack**

- [ ] `.env` is created from `.env.example` (`cp .env.example .env`), and the `SECRET_KEY` is your own.
- [ ] `docker compose config --services` → `db`, `redis`, `web`, `nginx`, `celery-beat`, `celery-worker`.
- [ ] `docker compose up --build` finishes without errors; in `docker compose ps`, `db`, `redis`, `web`, `nginx`, `celery-worker` are `(healthy)`, and `celery-beat` is `Up`.
- [ ] `docker compose logs web` shows `[entrypoint] Applying database migrations...` → `Collecting static files...` → `Starting ASGI web server on port 8001...`.
- [ ] `http://localhost` (nginx, port 80) and `http://localhost:8001` (`web` directly) open the login page; static files (`/static/...`) are served by nginx from the `staticfiles` volume.
- [ ] Logging in as `demo_alice / demo1234` (with `SEED_DEMO_DATA=1` or after `docker compose run --rm web python manage.py seed_demo_data`) succeeds, without a `403`.

**WebSocket and background tasks**

- [ ] The group chat works through nginx: in DevTools `ws://localhost/ws/groups/<pk>/chat/` → `101`; two tabs with different users see the broadcast.
- [ ] `docker compose logs celery-beat` → `Sending due task send-reminders-every-minute`.
- [ ] `docker compose logs nginx`: an access log without `502`.

**ngrok (the `tunnel` profile)**

- [ ] `NGROK_AUTHTOKEN` and `NGROK_DOMAIN` are set in `.env`; `docker compose --profile tunnel up -d ngrok`.
- [ ] `http://localhost:4040` (`NGROK_UI_PORT`) is the ngrok dashboard; logging in through `https://<NGROK_DOMAIN>` works without a CSRF `403`.

**Tests and production**

- [ ] Unit / integration / consumer tests:

    ```bash
    docker compose run --rm web python manage.py test \
      notes_app.tests.test_models \
      notes_app.tests.test_services \
      notes_app.tests.test_forms \
      notes_app.tests.test_views \
      notes_app.tests.test_consumers \
      notes_app.tests.test_api \
      -v 2
    ```

- [ ] Selenium E2E (the `test` profile, Chrome in the `selenium` container, smoke tests through nginx):

    ```bash
    docker compose --profile test run --rm selenium-tests
    ```

- [ ] `docker compose exec web python manage.py check --deploy` is reviewed; for production `W004`–`W016` are closed.
- [ ] You can explain why `nginx` starts only after `web: condition: service_healthy`.
- [ ] You can explain how a `wss://<domain>/ws/...` request passes through ngrok → nginx → uvicorn → `GroupChatConsumer`.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `Exception: DATABASE_URL не встановлено. Запускай через docker compose.` | Django is started outside Compose without the variable | `docker compose run --rm web ...` or set `DATABASE_URL` |
| `Bind for 0.0.0.0:80 failed: port is already allocated` | port 80 / 8001 / 4444 is taken by another process | `NGINX_PORT=8080`, `WEB_PORT=8011`, `SELENIUM_PORT=...` in `.env` |
| `dependency failed to start: container ... is unhealthy` | `web` does not respond on `/accounts/login/` (the migrations failed, an error in settings) | `docker compose logs web`: the first error after `[entrypoint]` |
| `502 Bad Gateway` from nginx | `web` is still starting or has crashed | `docker compose ps`, `docker compose logs web` |
| A page without styles through nginx | `collectstatic` did not run (`RUN_COLLECTSTATIC=0`) or the `staticfiles` volume is empty | `docker compose run --rm web python manage.py collectstatic --noinput` |
| A WebSocket through nginx closes immediately | `proxy_http_version 1.1` / `Upgrade` / `Connection` are missing | `map $http_upgrade $connection_upgrade` + the headers in `location /` |
| `Forbidden (403) CSRF verification failed` through ngrok | the domain is not in `CSRF_TRUSTED_ORIGINS` | `NGROK_DOMAIN` in `.env` (settings adds `https://<domain>`) |
| `CommandError: Demo data seeding is disabled when DEBUG=False. Use --force to override.` | the seeder protects production | `SEED_DEMO_DATA=1` (the entrypoint passes `--force` itself) or `--force` by hand on a dev database |
| The chat works in one tab, but not across processes | no `REDIS_URL` → `InMemoryChannelLayer` | Compose already sets `REDIS_URL=redis://redis:6379/0`; check `docker compose config` |

## Practice { #practice }

### Worked example: the stack startup chain { #worked-example }

`docker compose up` does not start everything at once: the order is set by `depends_on` + `healthcheck`:

1. **`db` and `redis`** start first; `db` becomes `healthy` after `pg_isready -U $POSTGRES_USER -d $POSTGRES_DB`, `redis` after `redis-cli ping`.
2. **`web`** waits for `db: condition: service_healthy` and `redis: condition: service_healthy`, then runs `entrypoint.sh`: `migrate --noinput` → `collectstatic --noinput` → (`CREATE_SUPERUSER`) → (`SEED_DEMO_DATA`) → `python -m uvicorn notes_project.asgi:application --host 0.0.0.0 --port 8001 --proxy-headers --forwarded-allow-ips '*'`. It is `healthy` when `urlopen('http://127.0.0.1:8001/accounts/login/')` returns a response.
3. **`nginx`** waits for `web: condition: service_healthy`, so there is no `502` on the first request; `upstream django { server web:8001; }`.
4. **`celery-worker`** and **`celery-beat`** wait for `db` and `redis`.

The local `entrypoint.sh` run from the "Check" section is step 2 without `collectstatic`.

### Modify the example { #modify }

1. Change `NGINX_PORT=8080` in `.env` and make sure the site opens on `http://localhost:8080`, and that `docker compose config` shows `published: "8080"`.
2. Set `UVICORN_RELOAD=0` and check in `docker compose logs web` that the line about `--reload` is gone (that is how it should be in production).

### Try it yourself: close `check --deploy` { #try-yourself }

Prepare a production `.env` and the settings so that `python manage.py check --deploy` with `DJANGO_DEBUG=0` does not show `W009`, and `W004`, `W008`, `W012`, `W016` are switched on by environment variables only under HTTPS.

**Success criteria:** the local stack without HTTPS works as before; with the production variables `check --deploy` → `System check identified no issues (0 silenced).`; the 209 tests of the CI set pass.

## Summary { #summary }

| Concept | Where in the code |
|-----------|-----------|
| Docker Compose chain | `docker-compose.yml`: `depends_on` + `healthcheck` |
| PostgreSQL connection | `notes_project/settings.py`: parsing `DATABASE_URL` |
| Redis channel layer | `settings.py`: `CHANNEL_LAYERS` with `socket_timeout=None`, `health_check_interval=30` |
| nginx WebSocket config | `nginx/nginx.conf`: `map $http_upgrade` + `proxy_http_version 1.1` |
| Static volume | the `staticfiles` volume in `docker-compose.yml` + nginx `location /static/ { alias /staticfiles/; }` |
| CSRF + ngrok | `settings.py`: `CSRF_TRUSTED_ORIGINS` from `NGROK_DOMAIN` |
| `entrypoint.sh` chain | migrate → collectstatic → (superuser) → (seed) → uvicorn |
| Consumer lifecycle | `notes_app/consumers.py`: `connect` / `receive` / `chat_message` / `disconnect` |
| `database_sync_to_async` | `consumers.py`: `check_membership`, `load_history`, `save_message` |
| XSS protection in JS | `notes_app/static/notes_app/js/group_chat.js`: `escapeHtml()` |
| IDOR protection | `notes_app/views.py`: personal objects use `get_object_or_404(Model, pk=pk, user=request.user)`, notes use a Q filter + an owner check |
| Group Q filter | `notes_app/selectors.py`: `Q(user=user) \| Q(group__in=user_groups)` |
| Seed data | `notes_app/management/commands/seed_demo_data.py` |
| Docker Selenium | `notes_app/tests/test_selenium.py`: `_DockerLiveServerMixin` + `SELENIUM_REMOTE_URL`; the `selenium-tests` service |
| Consumer tests | `notes_app/tests/test_consumers.py`: `WebsocketCommunicator` |
| `daphne` first | `settings.py` `INSTALLED_APPS`: `daphne` before `django.*` and `channels` |

You have gone the whole route from the first Django view to a production-ready stack:

```
01 → First Django view (URL → View → Template)
02 → First model (ORM, forms, PRG)
03 → CRUD and business logic (selectors/services, N+1, transactions)
04 → Templates and Bootstrap (3-level inheritance, Crispy Forms)
05 → Authentication and security (login, register, IDOR, groups)
06 → Testing (test pyramid, TestCase, WebsocketCommunicator, Selenium, CI)
07 → Async and WebSocket (ASGI, Channels, Consumer, channel layer)
08 → Background tasks (Celery worker and Beat, reminders, browser toast)
09 → Deployment (Docker Compose, nginx, ngrok, production checklist)
```

You know how to:

- package a Django application into a Docker Compose stack with 6 base services (+ `selenium`, `selenium-tests`, `ngrok` in profiles);
- set up nginx as a reverse proxy with WebSocket support;
- solve the CSRF problem when working through ngrok;
- set up a Redis channel layer for broadcast between processes;
- run Selenium E2E tests in Docker through Remote WebDriver;
- prepare the application for production: a security checklist, secret management.

## Self-check { #self-check }

1. Why does `nginx` not get a `502` on the first request after `docker compose up`?
2. What does `entrypoint.sh` do if a migration fails?
3. Why `--proxy-headers --forwarded-allow-ips '*'` in the uvicorn command?
4. Why are `selenium` and `ngrok` in profiles and not in the main stack?
5. Which of the 5 `check --deploy` warnings can you close without HTTPS?

??? success "Answers"

    1. `nginx` depends on `web: condition: service_healthy`, and `web` becomes `healthy` only when `/accounts/login/` responds, that is, after the migrations and the uvicorn startup.
    2. `set -eu` at the top: any command with a non-zero exit code stops the script, uvicorn does not start, the healthcheck does not pass, and `nginx` is not started. The cause is in `docker compose logs web`.
    3. The requests come from nginx (and ngrok), so without these options Django would see the nginx IP and `http` instead of the real client IP and `https`. `--proxy-headers` takes `X-Forwarded-For` / `X-Forwarded-Proto`.
    4. They are needed only for E2E tests and a public tunnel. In the main stack they would take ports 4444 / 4040 and resources for nothing, and ngrok also requires a token.
    5. Only `W009`: a long random `SECRET_KEY`. `HSTS`, `SSL_REDIRECT` and `Secure` cookies without HTTPS would break the site: the browser would not send the cookies over `http`.

## What's next { #next }

[Notes Chat App: project overview](../../12_final_project/README.md): the architecture, models, tests and deployment of the final application in one picture.
