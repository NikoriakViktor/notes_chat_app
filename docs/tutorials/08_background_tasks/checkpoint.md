# Контрольна точка: крок 8

## Перевір, що крок робочий { #check }

Тести задач (з кореня репозиторію, `DATABASE_URL` задано; у Docker — `docker compose run --rm web python manage.py test notes_app.tests.test_tasks`):

```text
$ python manage.py test notes_app.tests.test_tasks -v 2
Found 18 test(s).
...
Ran 18 tests in 4.767s

OK
```

Worker і Beat разом — `celery -A notes_project worker -B -l info` з `REDIS_URL=redis://localhost:6379/0` (у `docker-compose.yml` це два окремі сервіси, `celery-worker` і `celery-beat`). Рядки з логу через хвилину після старту (у перевірці Redis слухав інший порт; тут показано стандартний 6379, `:**@` — так Celery маскує пароль у URL):

```text
[... INFO/MainProcess] Connected to redis://:**@localhost:6379/1
[... INFO/MainProcess] celery@<host> ready.
[... INFO/Beat] Scheduler: Sending due task send-reminders-every-minute (notes_app.tasks.send_reminder_notifications)
[... INFO/MainProcess] Task notes_app.tasks.send_reminder_notifications[32c009c5-...] received
[... INFO/ForkPoolWorker-15] Task notes_app.tasks.send_reminder_notifications[32c009c5-...] succeeded in 0.02193124301265925s: None
```

Celery підключився до Redis **DB 1** (`/1`), хоча `REDIS_URL` закінчується на `/0`: `settings.py` робить `_REDIS_URL.replace("/0", "/1")`, щоб черга Celery не змішувалась із channel layer Channels (DB 0).

Ручний прогін задачі без Celery — створити прострочене нагадування і викликати функцію напряму:

```text
$ python manage.py shell -c "...Reminder.objects.create(note=n, remind_at=timezone.now() - timedelta(minutes=2), message='Ручний тест!')..."
Created reminder id: 13
$ python manage.py shell -c "from notes_app.tasks import send_reminder_notifications; send_reminder_notifications()"
Content-Type: text/plain; charset="utf-8"
...
Subject: =?utf-8?b?0J3QsNCz0LDQtNGD0LLQsNC90L3Rjzog...?=
From: noreply@notes.local
To: alice@demo.example.com
...
Ручний тест!
-------------------------------------------------------------------------------
$ python manage.py shell -c "from notes_app.models import Reminder; print('Sent:', Reminder.objects.filter(is_sent=True).count())"
Sent: 1
```

`EMAIL_BACKEND` за замовчуванням — `console`, тож лист друкується в термінал (у Docker — `docker compose logs web` / `celery-worker`). `Subject` закодовано MIME: це `Нагадування: <назва нотатки>`. Перевірено на PostgreSQL 16 + Redis 7 з демо-даними `seed_demo_data --force`; `id` і кількість у твоїй базі будуть іншими.

HTTP-ендпоінт для toast'ів (Django test client):

```text
anon  GET /reminders/check/   302 /accounts/login/?next=/reminders/check/
alice GET /reminders/check/   200 application/json  ключі: ['reminders']
```

Browser toast у справжньому браузері тут перевірено Selenium-тестами `SeleniumReminderToastTest` (4 тести — `ok` у локальному прогоні з Remote Chrome, див. [крок 6](../06_testing/checkpoint.md#check)).

## Чекліст { #checklist }

- [ ] `notes_project/celery.py` містить `app = Celery('notes_project')`, `config_from_object('django.conf:settings', namespace='CELERY')`, `autodiscover_tasks()`; `notes_project/__init__.py` імпортує `celery_app`.
- [ ] `CELERY_BROKER_URL` і `CELERY_RESULT_BACKEND` — Redis DB 1; channel layer — DB 0 (лог worker'а: `Connected to redis://:**@.../1`).
- [ ] `CELERY_BEAT_SCHEDULE` = `{'send-reminders-every-minute': {'task': 'notes_app.tasks.send_reminder_notifications', 'schedule': 60.0}}`; у лозі Beat — `Sending due task send-reminders-every-minute`.
- [ ] `celery-worker` і `celery-beat` — окремі сервіси в `docker-compose.yml`; `celery-beat` — рівно один екземпляр.
- [ ] Задачі — `@shared_task` (не `@app.task`) у `notes_app/tasks.py`.
- [ ] `send_reminder_notifications` бере прострочені нагадування з `select_related('note__user')` і зберігає `save(update_fields=['is_sent'])`; ручний прогін робить `is_sent=True`.
- [ ] Повторення — `relativedelta(days=1 / weeks=1 / months=1)` у `_schedule_next`.
- [ ] `GET /reminders/check/` для залогіненого → `200`, `application/json`, `{"reminders": [...]}`; для аноніма → `302` на логін (`@login_required`).
- [ ] `reminders.js` опитує ендпоінт одразу і далі кожні `POLL_INTERVAL = 60_000` мс; текст іде через `escapeHtml()`; `sessionStorage` не показує той самий toast двічі за сесію; контейнер `#reminder-toast-container` у `base.html` має `aria-live="polite"`.
- [ ] `python manage.py test notes_app.tests.test_tasks` → `Ran 18 tests`, `OK`; тести використовують `@override_settings(EMAIL_BACKEND='django.core.mail.backends.locmem.EmailBackend')` і перевіряють `mail.outbox`.
- [ ] Можеш пояснити, чому довга задача не може виконуватись усередині view.
- [ ] Можеш пояснити, що станеться, якщо запустити два `celery-beat`.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `consumer: Cannot connect to redis://:**@localhost:6379/1: Error 111 connecting to localhost:6379. Connection refused..` | Redis не запущено або `REDIS_URL` вказує не туди (без `REDIS_URL` брокер — `redis://localhost:6379/1`) | `docker compose up -d redis`; у Docker `REDIS_URL=redis://redis:6379/0` |
| Задача «зникає»: `Received unregistered task of type 'notes_app.tasks...'` | worker не знає задачу: немає `autodiscover_tasks()` або імпорту `celery_app` у `notes_project/__init__.py` | перевір `celery.py` і `__init__.py`, перезапусти worker |
| Beat пише `Sending due task`, але листів немає | працює лише Beat, worker не запущено | `docker compose up -d celery-worker` |
| Кожне нагадування приходить двічі | запущено два `celery-beat` | один екземпляр Beat; для кількох — `django-celery-beat` з блокуванням у БД |
| Лист не приходить на пошту | `EMAIL_BACKEND` = `console` | дивись `docker compose logs celery-worker`; для реальної пошти — SMTP-змінні `EMAIL_*` |
| Toast не з'являється | користувач не залогінений (`302`), нагадування старше години або вже показане в цій сесії | DevTools → Network → `/reminders/check/` → JSON; очисти `sessionStorage` |
| Місячне повторення 31 січня → 1 березня | `timedelta(days=30)` замість `relativedelta(months=1)` | `relativedelta` |

## Практика { #practice }

### Розібраний приклад: що робить `send_reminder_notifications` за один прохід { #worked-example }

Ручний прогін з розділу «Перевір» — це рівно те, що робить Beat кожні 60 секунд:

```python title="notes_app/tasks.py (фрагмент)"
now = timezone.now()
pending = (
    Reminder.objects
    .select_related('note__user')
    .filter(remind_at__lte=now, is_sent=False)
)

for reminder in pending:
    user = reminder.note.user
    if user.email:
        send_mail(
            subject=f'Нагадування: {reminder.note.title}',
            ...
        )
    reminder.is_sent = True
    reminder.save(update_fields=['is_sent'])
    _schedule_next(reminder)
```

1. `select_related('note__user')` — нотатка й власник приходять одним JOIN: без нього кожна ітерація робила б ще 2 запити.
2. `is_sent=False` у фільтрі + `is_sent = True` після відправки — наступний прохід Beat цього нагадування вже не візьме.
3. `update_fields=['is_sent']` — `UPDATE` одного стовпця, а не всього рядка.
4. `_schedule_next` для `daily` / `weekly` / `monthly` створює наступне нагадування.

Чому `relativedelta`, а не `timedelta`:

```text
>>> datetime(2024, 1, 31) + relativedelta(months=1)
2024-02-29 00:00:00
>>> datetime(2024, 1, 31) + timedelta(days=30)
2024-03-01 00:00:00
```

### Зміни приклад { #modify }

1. Додай `repeat_pattern = 'yearly'` з `relativedelta(years=1)` — і в `REPEAT_CHOICES` моделі (потрібна міграція), і в `delta_map`.
2. Поверни з задачі `{"sent": n}` замість `None` і подивись, як змінився рядок `succeeded in ...s: ...` у лозі worker'а.

### Спробуй самостійно { #try-yourself }

1. **Перевірка воркера:** `docker compose up -d`, потім `docker compose logs -f celery-beat` — чекай `Sending due task send-reminders-every-minute`.
2. **Browser toast:** увійди у браузері, створи нагадування в минулому (як у розділі «Перевір»), відкрий `/notes/` — toast з'являється внизу праворуч після першого запиту до `/reminders/check/`. DevTools → Network → перевір JSON.
3. **Масштабування (дослідницьке):** що станеться з двома `celery-beat`? Як захиститися від дублювання? Підказка: `django-celery-beat`.

**Критерії перевірки:** нове нагадування в минулому отримує `is_sent=True` протягом хвилини після старту стеку; лист видно в логах; toast показується один раз за сесію; 18 тестів `test_tasks` проходять.

## Підсумок { #summary }

| Поняття | Що запам'ятати |
|---|---|
| Request → Response | view має відповісти швидко; довга робота — у фон |
| Celery | Broker (Redis DB 1) → Worker виконує задачі; Result Backend зберігає результат |
| Beat | розклад `CELERY_BEAT_SCHEDULE`; один екземпляр |
| `@shared_task` | задача не прив'язана до конкретного `app` |
| Redis DB 0 / DB 1 | Channels і Celery не змішують ключі |
| `select_related('note__user')` | один JOIN замість запитів у циклі |
| `update_fields` | оновлення одного стовпця |
| `relativedelta` | календарні місяці, а не «30 днів» |
| Polling | `reminders.js` → `GET /reminders/check/` кожну хвилину; WebSocket тут зайвий |
| Toast UI | Bootstrap Toast, `aria-live`, `escapeHtml()`, `sessionStorage` |
| Тести | `@override_settings(EMAIL_BACKEND=locmem)` + `mail.outbox` |

## Самоперевірка { #self-check }

1. Чому не можна надсилати нагадування прямо з view?
2. Навіщо Celery окрема Redis DB?
3. Що станеться, якщо прибрати `is_sent=False` з фільтра?
4. Чому для toast'ів — polling, а не WebSocket?
5. Як тест перевіряє лист, не надсилаючи його?

??? success "Відповіді"

    1. View живе, поки триває запит: нагадування має спрацювати в певний час, навіть коли користувача немає на сайті. До того ж SMTP може відповідати секундами — запит висів би.
    2. Channels і Celery пишуть у Redis власні ключі й списки. Окрема DB (`/1`) не дає їм перетинатися, і `FLUSHDB` однієї не зачепить іншу.
    3. Кожен прохід Beat (щохвилини) знову брав би вже надіслані нагадування — лист приходив би щохвилини.
    4. Нагадування приходять рідко, точність «до хвилини» достатня, а HTTP-запит раз на хвилину простіший за постійне з'єднання, consumer і channel layer.
    5. `locmem.EmailBackend` складає листи в `django.core.mail.outbox`; тест перевіряє `len(mail.outbox)`, `to`, `subject`.

## Що далі { #next }

[Крок 9. Deployment](../09_deployment/index.md) — Docker Compose стек, nginx, ngrok, production checklist.
