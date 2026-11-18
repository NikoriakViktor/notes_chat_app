# Контрольна точка: крок 6

## Перевір, що крок робочий { #check }

Ті самі тести, що запускає job `unit-and-integration` у `.github/workflows/django-tests.yml`. З кореня репозиторію, `DATABASE_URL` вказує на PostgreSQL (у Docker — `docker compose run --rm web python manage.py test ...`):

```text
$ python manage.py check
System check identified no issues (0 silenced).
$ python manage.py test notes_app.tests.test_models notes_app.tests.test_services notes_app.tests.test_forms notes_app.tests.test_views notes_app.tests.test_consumers notes_app.tests.test_api
Found 209 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.................................................................................................................................................................................................................
----------------------------------------------------------------------
Ran 209 tests in 84.685s

OK
Destroying test database for alias 'default'...
```

Перевірено з PostgreSQL 16 (`postgres:16-alpine`, той самий образ, що в CI і `docker-compose.yml`), Python 3.12, без `REDIS_URL` (consumer-тести працюють з `InMemoryChannelLayer`). Час залежить від машини. З чого складаються 209 — кожен модуль окремо, `-v 2`:

| Модуль | Тестів | Рівень |
|---|---|---|
| `test_models` | 25 | unit: constraints, `__str__`, `SET_NULL`, валідатори через `full_clean()` |
| `test_services` | 65 | unit: бізнес-логіка, ізоляція користувачів, транзакції |
| `test_forms` | 22 | unit: валідація, queryset-безпека полів |
| `test_views` | 84 | integration: HTTP, redirects, ownership, групи |
| `test_consumers` | 9 | WebSocket: connect / broadcast / history / ізоляція груп |
| `test_api` | 4 | REST API `/api/notes/` |
| `test_tasks` | 18 | Celery-задачі (крок 8; у CI-списку не перелічений) |
| `test_selenium` | 23 | E2E у браузері (окремий job) |

??? note "Навчальний `crispy_notes_project` (кроки 1–6, SQLite): який вивід описує глава"

    Глава кроку будує тести й для окремого навчального проєкту з додатком `hello_app`. Цього проєкту в репозиторії немає, тож вивід нижче тут не перевірявся — це орієнтир з глави, а не результат запуску. Твоя кількість тестів залежить від того, що ти написав.

    ```bash
    $ python manage.py test hello_app.tests -v 1

    Creating test database for alias 'default' ...
    Found 129 test(s).
    ...........................................................................................................................ssssss
    ----------------------------------------------------------------------
    Ran 129 tests in 90.2s

    OK (skipped=6)
    Destroying test database for alias 'default' ...
    ```

    ```
    23  — test_models.py  (unit: constraints, __str__, SET_NULL)
    36  — test_services.py (unit: бізнес-логіка, security, транзакції)
    23  — test_forms.py   (unit: валідація, queryset security)
    41  — test_views.py   (integration: HTTP, ownership, redirects)
     6  — test_selenium.py (E2E: браузер, skip без geckodriver)
    ───
    129 — всього
    ```

Ключ до виводу:

```
.   ← passed
s   ← skipped (тест пропущено умовою skipUnless / skipIf)
F   ← failed (assertion не спрацювала)
E   ← error (виняток у тесті, не assertion)

OK (skipped=N) ← все зелено, N пропущено (очікувано)
```

**Selenium.** У Docker E2E-тести запускає окремий сервіс з профілем `test` (`docker-compose.yml`: `selenium-tests` з `SELENIUM_REMOTE_URL=http://selenium:4444/wd/hub` і `WEB_HOST=selenium-tests`):

```bash
docker compose --profile test run --rm selenium-tests
```

Цей шлях у контрольній точці не запускався. Запуск поза Compose-мережею — власний контейнер `selenium/standalone-chrome`, `SELENIUM_REMOTE_URL=http://localhost:<port>/wd/hub`, `WEB_HOST=host.docker.internal`, як у CI — у WSL2 дав `Found 23 test(s)`, 16 `ok`, 3 `skipped` (`SeleniumDockerStackSmokeTest`: `external Docker Selenium smoke tests require SELENIUM_EXTERNAL_BASE_URL`) і 4 `ERROR`: `SeleniumNoteFlowTest.test_create_note_via_form` (`net::ERR_CONNECTION_REFUSED`) та 3 тести `SeleniumWebSocketChatTest` (`TimeoutException`). Тобто зелений результат Selenium тут не підтверджено; орієнтир — job `selenium-e2e` у [GitHub Actions](https://github.com/NikoriakViktor/notes_chat_app/actions).

## Чекліст { #checklist }

**Структура**

- [ ] `notes_app/tests/__init__.py` існує (без нього `manage.py test` не знайде модулі).
- [ ] Кожен тест-метод починається з `test_`; `setUp()` дочірнього класу, що перевизначає батьківський, викликає `super().setUp()` (`BaseServiceTest`, `BaseViewTest`, `BaseFormTest`).
- [ ] CI-набір (6 модулів) → `Found 209 test(s)` і `OK`.

**Unit (models, services, forms)**

- [ ] Валідатори перевіряються через `full_clean()`, а не `.objects.create()` (`test_priority_above_4_raises_validation_error`).
- [ ] Після `delete()` пов'язаного об'єкта — `refresh_from_db()` перед перевіркою (`test_note_group_becomes_null_when_group_deleted`).
- [ ] Security: чужий тег не приєднується (`test_create_note_ignores_other_users_tags`), чужий записник не пропускає форма (`NoteFormSecurityTest`).

**Integration (views)**

- [ ] Використовується `force_login()`, а не `login()`.
- [ ] Кожен захищений URL перевіряється на redirect для anonymous (`AuthenticationViewTest`, 8 тестів `test_*_redirects_anonymous`).
- [ ] Не власник отримує `404`, не `403` (`NoteDetailViewTest.test_non_owner_gets_404`, `NoteEditViewTest.test_non_owner_cannot_edit_gets_404`).
- [ ] IDOR через форму: `NoteCreateViewTest.test_form_rejects_other_users_notebook_idor_prevention`.

**Consumers**

- [ ] `TransactionTestCase`, а не `TestCase`; `communicator.scope['user']` і `scope['url_route']` задані до `connect()`; `await communicator.disconnect()` у кожному тесті.
- [ ] `python manage.py test notes_app.tests.test_consumers` → `Ran 9 tests`, `OK`.

**Selenium і CI**

- [ ] Класи E2E мають `@unittest.skipUnless(SELENIUM_AVAILABLE, ...)`; `_DockerLiveServerMixin` підставляє `WEB_HOST` у `live_server_url`; `_make_headless_driver()` бере `SELENIUM_REMOTE_URL`.
- [ ] Workflow — `.github/workflows/django-tests.yml` у корені; Job 1 `unit-and-integration` (PostgreSQL service + `health-cmd pg_isready`); Job 2 `selenium-e2e` з `needs: unit-and-integration` і service `selenium/standalone-chrome`; `coverage.xml` зберігається як artifact `coverage-xml`.
- [ ] Можеш пояснити, чому consumer-тести не можуть бути `TestCase`.
- [ ] Можеш пояснити, чим `force_login()` кращий за `login()` у view-тестах і що він НЕ перевіряє.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `Exception: DATABASE_URL не встановлено. Запускай через docker compose.` | `notes_project/settings.py` без `DATABASE_URL` не стартує | `docker compose run --rm web python manage.py test ...` або `DATABASE_URL=postgres://notes_user:notes_pass@localhost:5432/notes_db` |
| `Ran 0 tests` / `NO TESTS RAN` | немає `tests/__init__.py` або метод не починається з `test_` | створи порожній `__init__.py`, перейменуй метод |
| `AttributeError: 'XTest' object has no attribute 'alice'` | `setUp()` дочірнього класу не викликає `super().setUp()` | `super().setUp()` першим рядком |
| Тест валідатора зелений, хоча значення некоректне | `.objects.create()` не викликає валідатори | `obj.full_clean()` і `assertRaises(ValidationError)` |
| Після `group.delete()` у тесті `note.group` все ще не `None` | об'єкт у пам'яті не оновився | `note.refresh_from_db()` |
| Consumer не бачить користувача / групу з `setUp()` | `TestCase` тримає дані в незакоміченій транзакції, consumer працює в іншому потоці | `TransactionTestCase` |
| `connected` → `False` для учасника групи | у `scope` немає `user` або `url_route` | `communicator.scope['user'] = user`, `scope['url_route'] = {'kwargs': {'group_pk': str(pk)}}` |
| `selenium.common.exceptions.WebDriverException: Message: unknown error: net::ERR_CONNECTION_REFUSED` | браузер у контейнері не досягає test server (його `localhost` ≠ твій) | `WEB_HOST` = ім'я, за яким контейнер бачить test server (`selenium-tests` у Compose, `host.docker.internal` у CI) |
| `skipped 'external Docker Selenium smoke tests require SELENIUM_EXTERNAL_BASE_URL'` | smoke-тести стеку потребують запущений nginx | очікувано поза `selenium-tests`; у Compose змінну задано (`http://nginx`) |

## Практика { #practice }

### Розібраний приклад: тест «Bob не бачить нотатку Alice» { #worked-example }

Що має бути правдою: інший користувач отримує `404` на деталь чужої особистої нотатки. Шар — integration: потрібен view, URL і шаблон, але не браузер.

```python title="notes_app/tests/test_views.py (за зразком NoteDetailViewTest)"
def test_non_owner_gets_404(self):
    self.client.force_login(self.bob)
    response = self.client.get(
        reverse('notes_app:note_detail', args=[self.alice_note.pk])
    )
    self.assertEqual(response.status_code, 404)
```

- `force_login()` — без форми логіну й хешування пароля: тест перевіряє доступ, а не вхід.
- `reverse('notes_app:note_detail', ...)` — тест не ламається, якщо зміниться URL-шаблон.
- `404`, а не `403`, — та сама вимога, що в [кроці 5](../05_auth_and_security/checkpoint.md#self-check).

Запуск одного класу:

```text
$ python manage.py test notes_app.tests.test_views.NoteDetailViewTest
```

### Зміни приклад { #modify }

1. Напиши той самий тест для `note_edit` і `note_delete` (GET і POST). Скільки нових тестів додалося і чи проходять вони?
2. Додай перевірку, що після `POST` на видалення чужої нотатки вона залишилась у базі: `self.assertTrue(Note.objects.filter(pk=...).exists())`.

### Спробуй самостійно { #try-yourself }

**Завдання 1 — CASCADE (базовий).** Видалення `TodoList` видаляє всі його `TodoItem`:

```python
def test_todo_items_deleted_when_todo_list_deleted(self):
    todo_list = TodoList.objects.create(user=self.alice, title='Shopping')
    TodoItem.objects.create(todo_list=todo_list, text='Milk')
    TodoItem.objects.create(todo_list=todo_list, text='Bread')

    todo_list.delete()

    self.assertEqual(TodoItem.objects.count(), ___)  # що тут?
```

**Завдання 2 — неіснуюча група (просунутий).** Consumer відхиляє з'єднання до групи `pk=999`:

```python
async def test_nonexistent_group_rejects_connection(self):
    communicator = _make_communicator(999, self.user)
    connected, _ = await communicator.connect()
    self.assertFalse(connected)
```

**Критерії перевірки:** нові тести проходять разом з рештою (CI-набір більше не 209 — порахуй, скільки стало); навмисно зламаний код (`on_delete=SET_NULL` замість `CASCADE` у `TodoItem.todo_list`) робить тест 1 червоним.

## Підсумок { #summary }

| Концепція | Де дивитись у коді |
|-----------|-------------------|
| Arrange / Act / Assert | будь-який метод `test_*` |
| `setUp` і базові класи | `BaseServiceTest`, `BaseViewTest`, `BaseFormTest` |
| `assertRaises` | `test_models.py` — `test_unique_together_same_user_same_name_raises` |
| `full_clean()` vs `save()` | `test_models.py` — `test_priority_above_4_raises_validation_error` |
| `refresh_from_db()` | `test_models.py` — `test_note_group_becomes_null_when_group_deleted` |
| Persistence test | `test_services.py` — `test_toggle_pin_note_persisted_in_db` |
| Mass Assignment | `test_services.py` — `test_create_note_ignores_other_users_tags` |
| Транзакційна логіка | `test_services.py` — `test_create_notebook_default_unsets_previous_default` |
| Ізоляція між юзерами | `test_services.py` — `test_create_notebook_default_only_affects_same_user` |
| Form queryset security | `test_forms.py` — `NoteFormSecurityTest` |
| `force_login` | `BaseViewTest` |
| `@login_required` | `AuthenticationViewTest` — redirect для anonymous |
| `assertContains` / `assertNotContains` | `NoteListViewTest` — `test_note_list_shows_own_notes`, `test_note_list_excludes_other_user_notes` |
| Ownership: 404 | `NoteDetailViewTest.test_non_owner_gets_404` |
| Group access | `NoteDetailViewTest.test_group_member_can_view_note` |
| IDOR через форму | `NoteCreateViewTest.test_form_rejects_other_users_notebook_idor_prevention` |
| Помилки форми | `NoteEditViewTest.test_invalid_edit_shows_form_errors` |
| `assertRedirects` | `NoteDeleteViewTest.test_delete_redirects_to_note_list` |
| `TransactionTestCase` + `WebsocketCommunicator` | `test_consumers.py` — `_make_communicator()`, `connect()`, `receive_json_from()`, `disconnect()` |
| Broadcast | `GroupChatConsumerMessagesTest.test_message_broadcast_to_other_member` |
| Ізоляція груп | `GroupChatMultipleGroupsTest.test_message_stays_within_group` |
| `StaticLiveServerTestCase` | `SeleniumLoginFlowTest`, `SeleniumNoteFlowTest` |
| Docker live server | `_DockerLiveServerMixin` — `WEB_HOST` → `live_server_url` |
| Session cookie trick | `_login_via_cookie()` — обходить форму логіну |
| Remote WebDriver | `_make_headless_driver()` → `SELENIUM_REMOTE_URL` |
| `implicitly_wait` | чекати елемент до N секунд |

## Самоперевірка { #self-check }

1. Чому `test_models` і `test_services` — unit, а `test_views` — integration, хоча всі ходять у базу?
2. Навіщо `TransactionTestCase` у consumer-тестах і яка його ціна?
3. Чому тест валідатора через `.objects.create()` може бути зеленим для некоректних даних?
4. Чому Selenium-тести винесені в окремий CI-job з `needs:`?
5. Що означає `OK (skipped=3)` — тести пройшли?

??? success "Відповіді"

    1. Рівень визначає, скільки шарів проходить тест, а не наявність бази. Unit кличе одну функцію (модель, сервіс, форму); integration проходить URL → middleware → view → шаблон через Test Client.
    2. `TestCase` загортає тест у транзакцію з rollback; consumer працює через `database_sync_to_async` в іншому потоці й незакомічених даних `setUp()` не бачить. `TransactionTestCase` комітить реально, а потім очищає таблиці — повільніше.
    3. `create()` викликає лише `save()`; валідатори полів працюють у `full_clean()` (його викликають форми). База обмеження на кшталт «пріоритет ≤ 4» не знає.
    4. Вони повільні й потребують браузер. Якщо базова логіка вже червона, E2E нічого нового не скажуть, лише витратять хвилини.
    5. Ті, що запустились, пройшли; 3 не запускались зовсім (тут — smoke-тести без `SELENIUM_EXTERNAL_BASE_URL`). Пропущений тест нічого не доводить — перевір причину skip.

## Що далі { #next }

[Крок 7. Async Django](../07_async_django/index.md) — ASGI, Django Channels, `GroupChatConsumer`, channel layer.
