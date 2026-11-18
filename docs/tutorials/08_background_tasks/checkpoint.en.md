# Checkpoint: step 8

## Check that the step works { #check }

The task tests (from the repository root, with `DATABASE_URL` set; in Docker: `docker compose run --rm web python manage.py test notes_app.tests.test_tasks`):

```text
$ python manage.py test notes_app.tests.test_tasks -v 2
Found 18 test(s).
...
Ran 18 tests in 4.767s

OK
```

Worker and Beat together: `celery -A notes_project worker -B -l info` with `REDIS_URL=redis://localhost:6379/0` (in `docker-compose.yml` these are two separate services, `celery-worker` and `celery-beat`). Log lines a minute after startup (during the check Redis listened on a different port; the standard 6379 is shown here; `:**@` is how Celery masks the password in the URL):

```text
[... INFO/MainProcess] Connected to redis://:**@localhost:6379/1
[... INFO/MainProcess] celery@<host> ready.
[... INFO/Beat] Scheduler: Sending due task send-reminders-every-minute (notes_app.tasks.send_reminder_notifications)
[... INFO/MainProcess] Task notes_app.tasks.send_reminder_notifications[32c009c5-...] received
[... INFO/ForkPoolWorker-15] Task notes_app.tasks.send_reminder_notifications[32c009c5-...] succeeded in 0.02193124301265925s: None
```

Celery connected to Redis **DB 1** (`/1`), even though `REDIS_URL` ends in `/0`: `settings.py` does `_REDIS_URL.replace("/0", "/1")` so that the Celery queue does not mix with the Channels channel layer (DB 0).

A manual run of the task without Celery: create an overdue reminder and call the function directly:

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

The default `EMAIL_BACKEND` is `console`, so the email is printed to the terminal (in Docker: `docker compose logs web` / `celery-worker`). The `Subject` is MIME-encoded: it is `Нагадування: <note title>`. Checked on PostgreSQL 16 + Redis 7 with demo data from `seed_demo_data --force`; the `id` and the count in your database will differ.

The HTTP endpoint for the toasts (Django test client):

```text
anon  GET /reminders/check/   302 /accounts/login/?next=/reminders/check/
alice GET /reminders/check/   200 application/json  keys: ['reminders']
```

The browser toast in a real browser is checked here by the `SeleniumReminderToastTest` Selenium tests (4 tests, `ok` in a local run with Remote Chrome, see [step 6](../06_testing/checkpoint.md#check)).

## Checklist { #checklist }

- [ ] `notes_project/celery.py` contains `app = Celery('notes_project')`, `config_from_object('django.conf:settings', namespace='CELERY')`, `autodiscover_tasks()`; `notes_project/__init__.py` imports `celery_app`.
- [ ] `CELERY_BROKER_URL` and `CELERY_RESULT_BACKEND` are Redis DB 1; the channel layer is DB 0 (the worker log: `Connected to redis://:**@.../1`).
- [ ] `CELERY_BEAT_SCHEDULE` = `{'send-reminders-every-minute': {'task': 'notes_app.tasks.send_reminder_notifications', 'schedule': 60.0}}`; the Beat log shows `Sending due task send-reminders-every-minute`.
- [ ] `celery-worker` and `celery-beat` are separate services in `docker-compose.yml`; `celery-beat` runs as exactly one instance.
- [ ] The tasks are `@shared_task` (not `@app.task`) in `notes_app/tasks.py`.
- [ ] `send_reminder_notifications` takes the overdue reminders with `select_related('note__user')` and saves with `save(update_fields=['is_sent'])`; a manual run sets `is_sent=True`.
- [ ] Repeats: `relativedelta(days=1 / weeks=1 / months=1)` in `_schedule_next`.
- [ ] `GET /reminders/check/` for a logged-in user → `200`, `application/json`, `{"reminders": [...]}`; for an anonymous user → `302` to the login page (`@login_required`).
- [ ] `reminders.js` polls the endpoint right away and then every `POLL_INTERVAL = 60_000` ms; the text goes through `escapeHtml()`; `sessionStorage` does not show the same toast twice per session; the `#reminder-toast-container` container in `base.html` has `aria-live="polite"`.
- [ ] `python manage.py test notes_app.tests.test_tasks` → `Ran 18 tests`, `OK`; the tests use `@override_settings(EMAIL_BACKEND='django.core.mail.backends.locmem.EmailBackend')` and check `mail.outbox`.
- [ ] You can explain why a long task cannot run inside a view.
- [ ] You can explain what happens if you run two `celery-beat` instances.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `consumer: Cannot connect to redis://:**@localhost:6379/1: Error 111 connecting to localhost:6379. Connection refused..` | Redis is not running, or `REDIS_URL` points to the wrong place (without `REDIS_URL` the broker is `redis://localhost:6379/1`) | `docker compose up -d redis`; in Docker, `REDIS_URL=redis://redis:6379/0` |
| The task "disappears": `Received unregistered task of type 'notes_app.tasks...'` | the worker does not know the task: `autodiscover_tasks()` or the `celery_app` import in `notes_project/__init__.py` is missing | check `celery.py` and `__init__.py`, restart the worker |
| Beat writes `Sending due task`, but there are no emails | only Beat is running, the worker is not started | `docker compose up -d celery-worker` |
| Every reminder arrives twice | two `celery-beat` instances are running | one Beat instance; for several, `django-celery-beat` with locking in the DB |
| The email does not arrive in the mailbox | `EMAIL_BACKEND` = `console` | look at `docker compose logs celery-worker`; for real mail, the SMTP `EMAIL_*` variables |
| The toast does not appear | the user is not logged in (`302`), the reminder is older than an hour, or it was already shown in this session | DevTools → Network → `/reminders/check/` → JSON; clear `sessionStorage` |
| A monthly repeat goes from January 31 → March 1 | `timedelta(days=30)` instead of `relativedelta(months=1)` | `relativedelta` |

## Practice { #practice }

### Worked example: what `send_reminder_notifications` does in one pass { #worked-example }

The manual run from the "Check" section is exactly what Beat does every 60 seconds:

```python title="notes_app/tasks.py (fragment)"
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

1. `select_related('note__user')`: the note and its owner arrive in one JOIN; without it every iteration would make 2 more queries.
2. `is_sent=False` in the filter + `is_sent = True` after sending: the next Beat pass will no longer pick up this reminder.
3. `update_fields=['is_sent']`: an `UPDATE` of one column, not of the whole row.
4. `_schedule_next` creates the next reminder for `daily` / `weekly` / `monthly`.

Why `relativedelta` and not `timedelta`:

```text
>>> datetime(2024, 1, 31) + relativedelta(months=1)
2024-02-29 00:00:00
>>> datetime(2024, 1, 31) + timedelta(days=30)
2024-03-01 00:00:00
```

### Modify the example { #modify }

1. Add `repeat_pattern = 'yearly'` with `relativedelta(years=1)`, both in the model's `REPEAT_CHOICES` (a migration is needed) and in `delta_map`.
2. Return `{"sent": n}` from the task instead of `None` and see how the `succeeded in ...s: ...` line in the worker log changes.

### Try it yourself { #try-yourself }

1. **Checking the worker:** `docker compose up -d`, then `docker compose logs -f celery-beat`; wait for `Sending due task send-reminders-every-minute`.
2. **Browser toast:** log in in the browser, create a reminder in the past (as in the "Check" section), open `/notes/`: the toast appears at the bottom right after the first request to `/reminders/check/`. DevTools → Network → check the JSON.
3. **Scaling (research):** what happens with two `celery-beat` instances? How do you protect against duplication? Hint: `django-celery-beat`.

**Success criteria:** a new reminder in the past gets `is_sent=True` within a minute after the stack starts; the email is visible in the logs; the toast is shown once per session; the 18 `test_tasks` tests pass.

## Summary { #summary }

| Concept | What to remember |
|---|---|
| Request → Response | a view must respond quickly; long work goes to the background |
| Celery | Broker (Redis DB 1) → the Worker runs the tasks; the Result Backend stores the result |
| Beat | the `CELERY_BEAT_SCHEDULE` schedule; one instance |
| `@shared_task` | the task is not tied to a specific `app` |
| Redis DB 0 / DB 1 | Channels and Celery do not mix their keys |
| `select_related('note__user')` | one JOIN instead of queries in a loop |
| `update_fields` | an update of one column |
| `relativedelta` | calendar months, not "30 days" |
| Polling | `reminders.js` → `GET /reminders/check/` every minute; WebSocket is unnecessary here |
| Toast UI | Bootstrap Toast, `aria-live`, `escapeHtml()`, `sessionStorage` |
| Tests | `@override_settings(EMAIL_BACKEND=locmem)` + `mail.outbox` |

## Self-check { #self-check }

1. Why can't you send reminders directly from a view?
2. Why does Celery need a separate Redis DB?
3. What happens if you remove `is_sent=False` from the filter?
4. Why polling for the toasts, and not WebSocket?
5. How does a test check an email without sending it?

??? success "Answers"

    1. A view lives only as long as the request: a reminder has to fire at a given time, even when the user is not on the site. Besides, SMTP can take seconds to respond, and the request would hang.
    2. Channels and Celery write their own keys and lists to Redis. A separate DB (`/1`) keeps them from overlapping, and a `FLUSHDB` of one does not touch the other.
    3. Every Beat pass (every minute) would pick up the already sent reminders again, and the email would arrive every minute.
    4. Reminders come rarely, "to the minute" precision is enough, and one HTTP request a minute is simpler than a permanent connection, a consumer and a channel layer.
    5. `locmem.EmailBackend` puts the emails into `django.core.mail.outbox`; the test checks `len(mail.outbox)`, `to`, `subject`.

## What's next { #next }

[Step 9. Deployment](../09_deployment/index.md): the Docker Compose stack, nginx, ngrok, the production checklist.
