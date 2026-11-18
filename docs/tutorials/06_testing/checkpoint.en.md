# Checkpoint: step 6

## Check that the step works { #check }

These are the same tests that the `unit-and-integration` job in `.github/workflows/django-tests.yml` runs. From the repository root, with `DATABASE_URL` pointing to PostgreSQL (in Docker: `docker compose run --rm web python manage.py test ...`):

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

Verified with PostgreSQL 16 (`postgres:16-alpine`, the same image as in CI and `docker-compose.yml`), Python 3.12, without `REDIS_URL` (the consumer tests work with `InMemoryChannelLayer`). The time depends on the machine. What the 209 are made of, each module separately, `-v 2`:

| Module | Tests | Level |
|---|---|---|
| `test_models` | 25 | unit: constraints, `__str__`, `SET_NULL`, validators via `full_clean()` |
| `test_services` | 65 | unit: business logic, user isolation, transactions |
| `test_forms` | 22 | unit: validation, queryset security of fields |
| `test_views` | 84 | integration: HTTP, redirects, ownership, groups |
| `test_consumers` | 9 | WebSocket: connect / broadcast / history / group isolation |
| `test_api` | 4 | REST API `/api/notes/` |
| `test_tasks` | 18 | Celery tasks (step 8; not in the CI list) |
| `test_selenium` | 23 | E2E in the browser (a separate job) |

??? note "The tutorial `crispy_notes_project` (steps 1–6, SQLite): the output the chapter describes"

    The step's chapter also builds tests for a separate tutorial project with the `hello_app` app. That project is not in the repository, so the output below was not verified here: it is a reference from the chapter, not the result of a run. Your number of tests depends on what you wrote.

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
    36  — test_services.py (unit: business logic, security, transactions)
    23  — test_forms.py   (unit: validation, queryset security)
    41  — test_views.py   (integration: HTTP, ownership, redirects)
     6  — test_selenium.py (E2E: browser, skipped without geckodriver)
    ───
    129 — total
    ```

How to read the output:

```
.   ← passed
s   ← skipped (the test was skipped by a skipUnless / skipIf condition)
F   ← failed (an assertion did not hold)
E   ← error (an exception in the test, not an assertion)

OK (skipped=N) ← all green, N skipped (expected)
```

**Selenium.** In Docker the E2E tests are run by a separate service with the `test` profile (`docker-compose.yml`: `selenium-tests` with `SELENIUM_REMOTE_URL=http://selenium:4444/wd/hub` and `WEB_HOST=selenium-tests`):

```bash
docker compose --profile test run --rm selenium-tests
```

This path was not run for this checkpoint. A run outside the Compose network (your own `selenium/standalone-chrome` container, `SELENIUM_REMOTE_URL=http://localhost:<port>/wd/hub`, `WEB_HOST=host.docker.internal`, as in CI) on WSL2 gave `Found 23 test(s)`, 16 `ok`, 3 `skipped` (`SeleniumDockerStackSmokeTest`: `external Docker Selenium smoke tests require SELENIUM_EXTERNAL_BASE_URL`) and 4 `ERROR`: `SeleniumNoteFlowTest.test_create_note_via_form` (`net::ERR_CONNECTION_REFUSED`) and 3 `SeleniumWebSocketChatTest` tests (`TimeoutException`). So a green Selenium result is not confirmed here; the reference is the `selenium-e2e` job in [GitHub Actions](https://github.com/NikoriakViktor/notes_chat_app/actions).

## Checklist { #checklist }

**Structure**

- [ ] `notes_app/tests/__init__.py` exists (without it `manage.py test` will not find the modules).
- [ ] Every test method starts with `test_`; a child class's `setUp()` that overrides the parent's calls `super().setUp()` (`BaseServiceTest`, `BaseViewTest`, `BaseFormTest`).
- [ ] The CI set (6 modules) → `Found 209 test(s)` and `OK`.

**Unit (models, services, forms)**

- [ ] Validators are tested via `full_clean()`, not `.objects.create()` (`test_priority_above_4_raises_validation_error`).
- [ ] After `delete()` of a related object, `refresh_from_db()` comes before the check (`test_note_group_becomes_null_when_group_deleted`).
- [ ] Security: someone else's tag is not attached (`test_create_note_ignores_other_users_tags`), the form rejects someone else's notebook (`NoteFormSecurityTest`).

**Integration (views)**

- [ ] `force_login()` is used, not `login()`.
- [ ] Every protected URL is checked for a redirect for anonymous users (`AuthenticationViewTest`, 8 `test_*_redirects_anonymous` tests).
- [ ] A non-owner gets `404`, not `403` (`NoteDetailViewTest.test_non_owner_gets_404`, `NoteEditViewTest.test_non_owner_cannot_edit_gets_404`).
- [ ] IDOR through a form: `NoteCreateViewTest.test_form_rejects_other_users_notebook_idor_prevention`.

**Consumers**

- [ ] `TransactionTestCase`, not `TestCase`; `communicator.scope['user']` and `scope['url_route']` are set before `connect()`; `await communicator.disconnect()` in every test.
- [ ] `python manage.py test notes_app.tests.test_consumers` → `Ran 9 tests`, `OK`.

**Selenium and CI**

- [ ] The E2E classes have `@unittest.skipUnless(SELENIUM_AVAILABLE, ...)`; `_DockerLiveServerMixin` puts `WEB_HOST` into `live_server_url`; `_make_headless_driver()` takes `SELENIUM_REMOTE_URL`.
- [ ] The workflow is `.github/workflows/django-tests.yml` at the root; Job 1 `unit-and-integration` (PostgreSQL service + `health-cmd pg_isready`); Job 2 `selenium-e2e` with `needs: unit-and-integration` and the `selenium/standalone-chrome` service; `coverage.xml` is saved as the `coverage-xml` artifact.
- [ ] You can explain why consumer tests cannot be `TestCase`.
- [ ] You can explain why `force_login()` is better than `login()` in view tests and what it does NOT check.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `Exception: DATABASE_URL не встановлено. Запускай через docker compose.` | `notes_project/settings.py` does not start without `DATABASE_URL` | `docker compose run --rm web python manage.py test ...` or `DATABASE_URL=postgres://notes_user:notes_pass@localhost:5432/notes_db` |
| `Ran 0 tests` / `NO TESTS RAN` | there is no `tests/__init__.py`, or the method does not start with `test_` | create an empty `__init__.py`, rename the method |
| `AttributeError: 'XTest' object has no attribute 'alice'` | the child class's `setUp()` does not call `super().setUp()` | `super().setUp()` as the first line |
| A validator test is green even though the value is invalid | `.objects.create()` does not run validators | `obj.full_clean()` and `assertRaises(ValidationError)` |
| After `group.delete()` in a test, `note.group` is still not `None` | the in-memory object was not refreshed | `note.refresh_from_db()` |
| The consumer does not see the user / group from `setUp()` | `TestCase` keeps data in an uncommitted transaction, and the consumer runs in another thread | `TransactionTestCase` |
| `connected` → `False` for a group member | `scope` has no `user` or `url_route` | `communicator.scope['user'] = user`, `scope['url_route'] = {'kwargs': {'group_pk': str(pk)}}` |
| `selenium.common.exceptions.WebDriverException: Message: unknown error: net::ERR_CONNECTION_REFUSED` | the browser in the container cannot reach the test server (its `localhost` ≠ yours) | `WEB_HOST` = the name under which the container sees the test server (`selenium-tests` in Compose, `host.docker.internal` in CI) |
| `skipped 'external Docker Selenium smoke tests require SELENIUM_EXTERNAL_BASE_URL'` | the stack smoke tests need a running nginx | expected outside `selenium-tests`; in Compose the variable is set (`http://nginx`) |

## Practice { #practice }

### Worked example: the "Bob does not see Alice's note" test { #worked-example }

What must be true: another user gets `404` on the detail page of someone else's personal note. The layer is integration: you need the view, the URL and the template, but not a browser.

```python title="notes_app/tests/test_views.py (modeled on NoteDetailViewTest)"
def test_non_owner_gets_404(self):
    self.client.force_login(self.bob)
    response = self.client.get(
        reverse('notes_app:note_detail', args=[self.alice_note.pk])
    )
    self.assertEqual(response.status_code, 404)
```

- `force_login()` skips the login form and password hashing: the test checks access, not logging in.
- `reverse('notes_app:note_detail', ...)`: the test does not break if the URL pattern changes.
- `404`, not `403`: the same requirement as in [step 5](../05_auth_and_security/checkpoint.md#self-check).

Running one class:

```text
$ python manage.py test notes_app.tests.test_views.NoteDetailViewTest
```

### Modify the example { #modify }

1. Write the same test for `note_edit` and `note_delete` (GET and POST). How many new tests were added, and do they pass?
2. Add a check that after a `POST` deleting someone else's note, the note is still in the database: `self.assertTrue(Note.objects.filter(pk=...).exists())`.

### Try it yourself { #try-yourself }

**Task 1: CASCADE (basic).** Deleting a `TodoList` deletes all of its `TodoItem`s:

```python
def test_todo_items_deleted_when_todo_list_deleted(self):
    todo_list = TodoList.objects.create(user=self.alice, title='Shopping')
    TodoItem.objects.create(todo_list=todo_list, text='Milk')
    TodoItem.objects.create(todo_list=todo_list, text='Bread')

    todo_list.delete()

    self.assertEqual(TodoItem.objects.count(), ___)  # what goes here?
```

**Task 2: a nonexistent group (advanced).** The consumer rejects a connection to the group `pk=999`:

```python
async def test_nonexistent_group_rejects_connection(self):
    communicator = _make_communicator(999, self.user)
    connected, _ = await communicator.connect()
    self.assertFalse(connected)
```

**Success criteria:** the new tests pass together with the rest (the CI set is no longer 209: count how many there are now); deliberately broken code (`on_delete=SET_NULL` instead of `CASCADE` in `TodoItem.todo_list`) turns test 1 red.

## Summary { #summary }

| Concept | Where to look in the code |
|-----------|-------------------|
| Arrange / Act / Assert | any `test_*` method |
| `setUp` and base classes | `BaseServiceTest`, `BaseViewTest`, `BaseFormTest` |
| `assertRaises` | `test_models.py`: `test_unique_together_same_user_same_name_raises` |
| `full_clean()` vs `save()` | `test_models.py`: `test_priority_above_4_raises_validation_error` |
| `refresh_from_db()` | `test_models.py`: `test_note_group_becomes_null_when_group_deleted` |
| Persistence test | `test_services.py`: `test_toggle_pin_note_persisted_in_db` |
| Mass Assignment | `test_services.py`: `test_create_note_ignores_other_users_tags` |
| Transactional logic | `test_services.py`: `test_create_notebook_default_unsets_previous_default` |
| Isolation between users | `test_services.py`: `test_create_notebook_default_only_affects_same_user` |
| Form queryset security | `test_forms.py`: `NoteFormSecurityTest` |
| `force_login` | `BaseViewTest` |
| `@login_required` | `AuthenticationViewTest`: redirect for anonymous users |
| `assertContains` / `assertNotContains` | `NoteListViewTest`: `test_note_list_shows_own_notes`, `test_note_list_excludes_other_user_notes` |
| Ownership: 404 | `NoteDetailViewTest.test_non_owner_gets_404` |
| Group access | `NoteDetailViewTest.test_group_member_can_view_note` |
| IDOR through a form | `NoteCreateViewTest.test_form_rejects_other_users_notebook_idor_prevention` |
| Form errors | `NoteEditViewTest.test_invalid_edit_shows_form_errors` |
| `assertRedirects` | `NoteDeleteViewTest.test_delete_redirects_to_note_list` |
| `TransactionTestCase` + `WebsocketCommunicator` | `test_consumers.py`: `_make_communicator()`, `connect()`, `receive_json_from()`, `disconnect()` |
| Broadcast | `GroupChatConsumerMessagesTest.test_message_broadcast_to_other_member` |
| Group isolation | `GroupChatMultipleGroupsTest.test_message_stays_within_group` |
| `StaticLiveServerTestCase` | `SeleniumLoginFlowTest`, `SeleniumNoteFlowTest` |
| Docker live server | `_DockerLiveServerMixin`: `WEB_HOST` → `live_server_url` |
| Session cookie trick | `_login_via_cookie()`: bypasses the login form |
| Remote WebDriver | `_make_headless_driver()` → `SELENIUM_REMOTE_URL` |
| `implicitly_wait` | wait for an element for up to N seconds |

## Self-check { #self-check }

1. Why are `test_models` and `test_services` unit tests and `test_views` integration tests, even though they all hit the database?
2. Why do consumer tests need `TransactionTestCase`, and what does it cost?
3. Why can a validator test via `.objects.create()` be green for invalid data?
4. Why are the Selenium tests moved into a separate CI job with `needs:`?
5. What does `OK (skipped=3)` mean: did the tests pass?

??? success "Answers"

    1. The level is defined by how many layers the test goes through, not by whether a database is involved. A unit test calls one function (a model, a service, a form); an integration test goes through URL → middleware → view → template via the Test Client.
    2. `TestCase` wraps the test in a transaction with a rollback; the consumer runs through `database_sync_to_async` in another thread and does not see the uncommitted data from `setUp()`. `TransactionTestCase` really commits and then truncates the tables, which is slower.
    3. `create()` only calls `save()`; field validators run in `full_clean()` (which forms call). The database does not know constraints like "priority ≤ 4".
    4. They are slow and need a browser. If the basic logic is already red, E2E tests will tell you nothing new and will only waste minutes.
    5. The ones that ran passed; 3 did not run at all (here, the smoke tests without `SELENIUM_EXTERNAL_BASE_URL`). A skipped test proves nothing: check the reason for the skip.

## What's next { #next }

[Step 7. Async Django](../07_async_django/index.md): ASGI, Django Channels, `GroupChatConsumer`, channel layer.
