# Контрольна точка: крок 9

## Перевір, що крок робочий { #check }

Конфігурація Compose читається без помилок і містить такі сервіси (з кореня репозиторію):

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

`selenium` і `selenium-tests` — профіль `test`, `ngrok` — профіль `tunnel`: звичайний `docker compose up` їх не запускає.

Що робить контейнер `web` при старті — `entrypoint.sh` (тут запущено локально з `RUN_COLLECTSTATIC=0`, `PORT=18012`, `UVICORN_RELOAD=0` і `DATABASE_URL` на PostgreSQL 16):

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

У контейнері порт — `8001` (`PORT` за замовчуванням), між migrate і uvicorn ще `collectstatic --noinput`, а з `CREATE_SUPERUSER=1` / `SEED_DEMO_DATA=1` — створення адміністратора й демо-даних. `GET /accounts/login/` — той самий запит, яким healthcheck `web` у `docker-compose.yml` вирішує, що сервіс `healthy`.

Production-перевірка налаштувань з `DJANGO_DEBUG=0` і dev-ключем з `docker-compose.yml`:

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

Це очікувано для локального стеку без HTTPS: кожне попередження — пункт [production checklist](production_checklist.md). Перед справжнім деплоєм `W009` має зникнути (власний `SECRET_KEY` у `.env`), а решта — після увімкнення HTTPS.

Тести — той самий набір, що в CI (`Found 209 test(s)` … `OK`, див. [крок 6](../06_testing/checkpoint.md#check)).

Повний стек `docker compose up --build`, nginx на `http://localhost`, ngrok і `docker compose --profile test run --rm selenium-tests` у цій контрольній точці **не запускались**: на машині перевірки порти 80 і 8001 зайняті іншим стеком, а контейнери проєкту `notes-chat-app` не можна було перезапускати. Пункти чекліста нижче, що стосуються запущеного стеку, перевір у себе.

## Чекліст { #checklist }

**Запуск стеку**

- [ ] `.env` створено з `.env.example` (`cp .env.example .env`), `SECRET_KEY` — власний.
- [ ] `docker compose config --services` → `db`, `redis`, `web`, `nginx`, `celery-beat`, `celery-worker`.
- [ ] `docker compose up --build` завершується без помилок; `docker compose ps` — `db`, `redis`, `web`, `nginx`, `celery-worker` мають `(healthy)`, `celery-beat` — `Up`.
- [ ] `docker compose logs web` показує `[entrypoint] Applying database migrations...` → `Collecting static files...` → `Starting ASGI web server on port 8001...`.
- [ ] `http://localhost` (nginx, порт 80) і `http://localhost:8001` (напряму `web`) відкривають сторінку логіну; статика (`/static/...`) віддається nginx з volume `staticfiles`.
- [ ] Логін `demo_alice / demo1234` (з `SEED_DEMO_DATA=1` або після `docker compose run --rm web python manage.py seed_demo_data`) — успішний, без `403`.

**WebSocket і фонові задачі**

- [ ] Груповий чат працює через nginx: у DevTools `ws://localhost/ws/groups/<pk>/chat/` → `101`; дві вкладки різних користувачів бачать broadcast.
- [ ] `docker compose logs celery-beat` → `Sending due task send-reminders-every-minute`.
- [ ] `docker compose logs nginx` — access log без `502`.

**ngrok (профіль `tunnel`)**

- [ ] `NGROK_AUTHTOKEN` і `NGROK_DOMAIN` задані в `.env`; `docker compose --profile tunnel up -d ngrok`.
- [ ] `http://localhost:4040` (`NGROK_UI_PORT`) — ngrok dashboard; логін через `https://<NGROK_DOMAIN>` — без CSRF `403`.

**Тести й production**

- [ ] Unit / integration / consumer тести:

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

- [ ] Selenium E2E (профіль `test`, Chrome у контейнері `selenium`, smoke-тести через nginx):

    ```bash
    docker compose --profile test run --rm selenium-tests
    ```

- [ ] `docker compose exec web python manage.py check --deploy` переглянуто; для production `W004`–`W016` закрито.
- [ ] Можеш пояснити, чому `nginx` стартує лише після `web: condition: service_healthy`.
- [ ] Можеш пояснити, як запит `wss://<домен>/ws/...` проходить ngrok → nginx → uvicorn → `GroupChatConsumer`.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `Exception: DATABASE_URL не встановлено. Запускай через docker compose.` | Django запущено поза Compose без змінної | `docker compose run --rm web ...` або задай `DATABASE_URL` |
| `Bind for 0.0.0.0:80 failed: port is already allocated` | порт 80 / 8001 / 4444 зайнятий іншим процесом | `NGINX_PORT=8080`, `WEB_PORT=8011`, `SELENIUM_PORT=...` у `.env` |
| `dependency failed to start: container ... is unhealthy` | `web` не відповідає на `/accounts/login/` (міграції впали, помилка в settings) | `docker compose logs web` — перша помилка після `[entrypoint]` |
| `502 Bad Gateway` від nginx | `web` ще стартує або впав | `docker compose ps`, `docker compose logs web` |
| Сторінка без стилів через nginx | `collectstatic` не відпрацював (`RUN_COLLECTSTATIC=0`) або volume `staticfiles` порожній | `docker compose run --rm web python manage.py collectstatic --noinput` |
| WebSocket через nginx одразу закривається | немає `proxy_http_version 1.1` / `Upgrade` / `Connection` | `map $http_upgrade $connection_upgrade` + заголовки в `location /` |
| `Forbidden (403) CSRF verification failed` через ngrok | домену немає в `CSRF_TRUSTED_ORIGINS` | `NGROK_DOMAIN` у `.env` (settings додає `https://<домен>`) |
| `CommandError: Demo data seeding is disabled when DEBUG=False. Use --force to override.` | сидер захищає production | `SEED_DEMO_DATA=1` (entrypoint сам передає `--force`) або `--force` вручну на dev-базі |
| Чат працює в одній вкладці, але не між процесами | немає `REDIS_URL` → `InMemoryChannelLayer` | у Compose `REDIS_URL=redis://redis:6379/0` уже задано — перевір `docker compose config` |

## Практика { #practice }

### Розібраний приклад: ланцюг старту стеку { #worked-example }

`docker compose up` не запускає все одночасно — порядок задають `depends_on` + `healthcheck`:

1. **`db` і `redis`** стартують першими; `db` стає `healthy` після `pg_isready -U $POSTGRES_USER -d $POSTGRES_DB`, `redis` — після `redis-cli ping`.
2. **`web`** чекає `db: condition: service_healthy` і `redis: condition: service_healthy`, потім `entrypoint.sh`: `migrate --noinput` → `collectstatic --noinput` → (`CREATE_SUPERUSER`) → (`SEED_DEMO_DATA`) → `python -m uvicorn notes_project.asgi:application --host 0.0.0.0 --port 8001 --proxy-headers --forwarded-allow-ips '*'`. `healthy` — коли `urlopen('http://127.0.0.1:8001/accounts/login/')` повертає відповідь.
3. **`nginx`** чекає `web: condition: service_healthy` — тому `502` на першому запиті не буває; `upstream django { server web:8001; }`.
4. **`celery-worker`** і **`celery-beat`** чекають `db` і `redis`.

Локальний прогін `entrypoint.sh` з розділу «Перевір» — це крок 2 без `collectstatic`.

### Зміни приклад { #modify }

1. Зміни `NGINX_PORT=8080` у `.env` і переконайся, що сайт відкривається на `http://localhost:8080`, а `docker compose config` показує `published: "8080"`.
2. Постав `UVICORN_RELOAD=0` і подивись у `docker compose logs web`, що рядка про `--reload` більше немає (для production так і має бути).

### Спробуй самостійно: закрий `check --deploy` { #try-yourself }

Підготуй production-`.env` і налаштування так, щоб `python manage.py check --deploy` з `DJANGO_DEBUG=0` не показував `W009`, а `W004`, `W008`, `W012`, `W016` вмикались змінними середовища лише за HTTPS.

**Критерії перевірки:** локальний стек без HTTPS працює як раніше; з production-змінними `check --deploy` → `System check identified no issues (0 silenced).`; 209 тестів CI-набору проходять.

## Підсумок { #summary }

| Концепція | Де у коді |
|-----------|-----------|
| Docker Compose ланцюг | `docker-compose.yml` — `depends_on` + `healthcheck` |
| PostgreSQL підключення | `notes_project/settings.py` — парсинг `DATABASE_URL` |
| Redis channel layer | `settings.py` — `CHANNEL_LAYERS` з `socket_timeout=None`, `health_check_interval=30` |
| nginx WebSocket config | `nginx/nginx.conf` — `map $http_upgrade` + `proxy_http_version 1.1` |
| Static volume | volume `staticfiles` у `docker-compose.yml` + nginx `location /static/ { alias /staticfiles/; }` |
| CSRF + ngrok | `settings.py` — `CSRF_TRUSTED_ORIGINS` з `NGROK_DOMAIN` |
| `entrypoint.sh` ланцюг | migrate → collectstatic → (superuser) → (seed) → uvicorn |
| Consumer lifecycle | `notes_app/consumers.py` — `connect` / `receive` / `chat_message` / `disconnect` |
| `database_sync_to_async` | `consumers.py` — `check_membership`, `load_history`, `save_message` |
| XSS захист у JS | `notes_app/static/notes_app/js/group_chat.js` — `escapeHtml()` |
| IDOR захист | `notes_app/views.py` — особисті об'єкти `get_object_or_404(Model, pk=pk, user=request.user)`, нотатки — Q-фільтр + перевірка власника |
| Q-фільтр групи | `notes_app/selectors.py` — `Q(user=user) \| Q(group__in=user_groups)` |
| Seed data | `notes_app/management/commands/seed_demo_data.py` |
| Docker Selenium | `notes_app/tests/test_selenium.py` — `_DockerLiveServerMixin` + `SELENIUM_REMOTE_URL`; сервіс `selenium-tests` |
| Consumer тести | `notes_app/tests/test_consumers.py` — `WebsocketCommunicator` |
| `daphne` перший | `settings.py` `INSTALLED_APPS` — `daphne` перед `django.*` і `channels` |

Ти пройшов увесь маршрут від першого Django view до production-ready стеку:

```
01 → Перший Django view (URL → View → Template)
02 → Перша модель (ORM, форми, PRG)
03 → CRUD та бізнес-логіка (selectors/services, N+1, транзакції)
04 → Шаблони та Bootstrap (3-рівневе наслідування, Crispy Forms)
05 → Автентифікація і безпека (login, register, IDOR, групи)
06 → Тестування (піраміда тестів, TestCase, WebsocketCommunicator, Selenium, CI)
07 → Async та WebSocket (ASGI, Channels, Consumer, channel layer)
08 → Фонові задачі (Celery worker і Beat, нагадування, browser toast)
09 → Deployment (Docker Compose, nginx, ngrok, production checklist)
```

Ти знаєш, як:

- запакувати Django-застосунок у Docker Compose стек з 6 базовими сервісами (+ `selenium`, `selenium-tests`, `ngrok` у профілях);
- налаштувати nginx як reverse proxy з підтримкою WebSocket;
- вирішити проблему CSRF при роботі через ngrok;
- налагодити Redis channel layer для broadcast між процесами;
- запустити Selenium E2E тести в Docker через Remote WebDriver;
- підготувати застосунок до production: security checklist, secret management.

## Самоперевірка { #self-check }

1. Чому `nginx` не отримує `502` на першому запиті після `docker compose up`?
2. Що зробить `entrypoint.sh`, якщо міграція впаде?
3. Навіщо `--proxy-headers --forwarded-allow-ips '*'` в команді uvicorn?
4. Чому `selenium` і `ngrok` у профілях, а не в основному стеку?
5. Які з 5 попереджень `check --deploy` можна закрити без HTTPS?

??? success "Відповіді"

    1. `nginx` залежить від `web: condition: service_healthy`, а `web` стає `healthy` лише коли `/accounts/login/` відповідає — тобто після міграцій і старту uvicorn.
    2. `set -eu` на початку: будь-яка команда з ненульовим кодом зупиняє скрипт, uvicorn не стартує, healthcheck не проходить, `nginx` не запускається. Причина — у `docker compose logs web`.
    3. Запити приходять від nginx (і ngrok), тож без них Django бачив би IP nginx і `http` замість справжнього IP клієнта й `https`. `--proxy-headers` бере `X-Forwarded-For` / `X-Forwarded-Proto`.
    4. Вони потрібні лише для E2E-тестів і публічного тунелю. В основному стеку вони займали б порти 4444 / 4040 і ресурси без потреби, а ngrok ще й вимагає токен.
    5. Лише `W009` — довгий випадковий `SECRET_KEY`. `HSTS`, `SSL_REDIRECT` і `Secure`-cookie без HTTPS зламають сайт: браузер не надсилатиме cookie по `http`.

## Що далі { #next }

[Notes Chat App: огляд проєкту](../../12_final_project/README.md) — архітектура, моделі, тести й деплоймент фінального застосунку однією картиною.
