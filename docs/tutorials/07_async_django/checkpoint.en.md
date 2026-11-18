# Checkpoint: step 7

## Check that the step works { #check }

**7B — WebSocket chat (`notes_chat_app`, this repository).** Consumer tests, from the repository root, with `DATABASE_URL` set:

```text
$ python manage.py test notes_app.tests.test_consumers -v 2
Found 9 test(s).
...
Ran 9 tests in 9.167s

OK
```

The same tests with `REDIS_URL=redis://localhost:6379/0` (then `settings.py` switches on `RedisChannelLayer`), together with the Celery task tests:

```text
$ REDIS_URL=redis://localhost:6379/0 python manage.py test notes_app.tests.test_consumers notes_app.tests.test_tasks
Found 27 test(s).
...
Ran 27 tests in 14.996s

OK
```

A live ASGI server is `uvicorn notes_project.asgi:application --port 8001` (in Docker, `entrypoint.sh` starts it):

```text
INFO:     ASGI 'lifespan' protocol appears unsupported.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8001 (Press CTRL+C to quit)
```

The first line is not an error: Django does not implement ASGI lifespan, so uvicorn just carries on. HTTP and WebSocket go through the same process (`curl` with the headers `Connection: Upgrade`, `Upgrade: websocket`, `Sec-WebSocket-Version: 13`):

```text
GET /accounts/login/                       200
GET /notes/                                302 → /accounts/login/?next=/notes/
WS  /ws/groups/1/chat/  no cookie           HTTP/1.1 403
WS  /ws/groups/1/chat/  demo_bob (member)   HTTP/1.1 101 Switching Protocols
WS  /ws/groups/2/chat/  demo_bob (not a member) HTTP/1.1 403
```

The uvicorn log for these three handshakes: `"WebSocket /ws/groups/1/chat/" 403`, `[accepted]`, `403`. Checked on PostgreSQL 16 + Redis 7 (`redis:7-alpine`) with demo data from `seed_demo_data --force`; the `sessionid` for `demo_bob` was created through `SessionStore`, as in the tests. `403` is the consumer's refusal (`close()` before `accept()`); the browser sees a closed connection with code `1006`.

**7A — async views (`hello_app`).** This is the step's separate learning project: `async_views.py`, `async_selectors.py`, `async_services.py` and the `/async/notes/` routes do not live in this repository, so they are not checked here. Check them in your own project: `python manage.py check`, your tests with `AsyncClient`, and that `runserver` (:8000) and `uvicorn` (:8001) return the same note.

## Checklist { #checklist }

**7A — Async Views (`hello_app`)**

- [ ] `requirements.txt` contains `uvicorn`, `httpx`, `wsproto`.
- [ ] `asgi.py`: `ProtocolTypeRouter` is configured.
- [ ] `async_selectors.py`: lazy QuerySets (regular `def`) + `async def` only where `aget()` is used.
- [ ] `async_services.py`: `sync_to_async(create_note)`, `adelete()`, `aupdate(F(...))`.
- [ ] `async_views.py`: 5 async views: `async_note_list`, `async_note_detail`, `async_note_create`, `async_note_delete`, `async_note_toggle_pin`; wired up under `/async/notes/`.
- [ ] `tests/test_async_views.py`: async tests through `AsyncClient`.
- [ ] A note created through a sync view is visible through an async view (one database).

**7B — WebSocket chat (`notes_chat_app`)**

- [ ] `daphne` is first in `INSTALLED_APPS` (`['daphne', 'django.contrib.admin', ...]`), `channels` is in the list too; `ASGI_APPLICATION = 'notes_project.asgi.application'`.
- [ ] Without `REDIS_URL` → `channels.layers.InMemoryChannelLayer`; with `REDIS_URL` → `RedisChannelLayer` with `"socket_timeout": None` and `"health_check_interval": 30`.
- [ ] `notes_project/asgi.py`: `django_asgi_app = get_asgi_application()` comes BEFORE `from notes_project.routing import websocket_urlpatterns`.
- [ ] `notes_project/routing.py`: `re_path(r'^ws/groups/(?P<group_pk>\d+)/chat/$', consumers.GroupChatConsumer.as_asgi())`.
- [ ] `GroupChatConsumer(AsyncWebsocketConsumer)`: `check_membership`, `load_history`, `save_message` use `@database_sync_to_async`; `load_history` returns `list(qs.values(...))`.
- [ ] `disconnect()` calls `group_discard()` under a `hasattr(self, 'room_group_name')` check.
- [ ] Handshake: anonymous → `403`, group member → `101`, non-member → `403`.
- [ ] `notes_app/static/notes_app/js/group_chat.js` passes `author` and `content` through `escapeHtml()`.
- [ ] `nginx/nginx.conf`: `proxy_http_version 1.1` + `Upgrade $http_upgrade` + `Connection $connection_upgrade`.
- [ ] `python manage.py test notes_app.tests.test_consumers` → `Ran 9 tests`, `OK` (both with and without `REDIS_URL`).
- [ ] You can explain why `load_history` returns `list(...)` and not a QuerySet.
- [ ] You can explain why a member of another group does not receive the messages, even though both are connected to the same channel layer.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `django.core.exceptions.SynchronousOnlyOperation: You cannot call this from an async context - use a thread or sync_to_async.` | a sync ORM call in the consumer's `async def` | move the query into a method with `@database_sync_to_async` |
| `SynchronousOnlyOperation` only "sometimes", on the chat history | `@database_sync_to_async` returned a lazy QuerySet, and the SQL ran later in async code | `return list(qs.values(...))` inside the wrapped method |
| `django.core.exceptions.AppRegistryNotReady: Apps aren't loaded yet.` when uvicorn starts | `routing` / `consumers` imported before `get_asgi_application()` | first `django_asgi_app = get_asgi_application()`, then the imports |
| `TimeoutError` / `Timeout reading from` Redis, the chat connection drops after idle time | a small `socket_timeout` versus the channel layer's blocking read | `"socket_timeout": None, "health_check_interval": 30` |
| WS through nginx: `400`, or the connection closes immediately | no Upgrade headers in the proxy | `proxy_http_version 1.1` + `Upgrade` + `Connection` |
| Uvicorn: `No supported WebSocket library detected`, WS → `404` | `wsproto` (or `websockets`) is not installed | `pip install -r requirements.txt` |
| WS handshake `403` for a logged-in member | the `sessionid` cookie did not arrive (different domain/port), or the consumer did not find the group | open the chat from the same origin as the site; check `group_pk` |
| `runserver` does not serve WebSocket | `daphne` is not first in `INSTALLED_APPS` | move `daphne` to the start of the list |
| The consumer "is not found", WS → `404` | `websocket_urlpatterns` are not wired into `asgi.py` | `AuthMiddlewareStack(URLRouter(websocket_urlpatterns))` |

??? note "Fix examples in code"

    ```python
    # SynchronousOnlyOperation:
    # ❌
    async def connect(self):
        group = Group.objects.get(pk=self.group_pk)
    # ✅
    @database_sync_to_async
    def get_group(self, pk):
        return Group.objects.get(pk=pk)

    # AppRegistryNotReady:
    # ❌ asgi.py: import BEFORE get_asgi_application():
    from notes_project.routing import websocket_urlpatterns
    django_asgi_app = get_asgi_application()
    # ✅
    django_asgi_app = get_asgi_application()
    from notes_project.routing import websocket_urlpatterns

    # time.sleep() blocks the event loop:
    # ❌
    async def connect(self):
        time.sleep(1)
    # ✅
    async def connect(self):
        await asyncio.sleep(1)

    # Lazy QuerySet in @database_sync_to_async:
    # ❌
    @database_sync_to_async
    def load_history(self, group_pk):
        return ChatMessage.objects.filter(group_id=group_pk).values(...)
    # ✅
    @database_sync_to_async
    def load_history(self, group_pk):
        qs = ChatMessage.objects.filter(group_id=group_pk).order_by('-timestamp')[:50]
        messages = list(qs.values('id', 'author__username', 'content', 'timestamp'))
        messages.reverse()
        return messages
    ```

## Practice { #practice }

### Worked example: a consumer test on `TransactionTestCase` { #worked-example }

**Why `TransactionTestCase` and not `TestCase`:**

```
TestCase: wraps each test in a transaction (rollback after the test)
  → the consumer reaches the DB through database_sync_to_async in another thread
  → that thread does not see the uncommitted setUp() data
  → the setUp objects (User, Group) "disappear" for the consumer

TransactionTestCase: does not wrap in a transaction
  → the data is written to the DB and is visible to all threads
  → the consumer sees the User and Group from setUp()
  → the price: a full DB flush between tests (slower)
```

**The helper.** The test does not go through HTTP and `AuthMiddlewareStack`: `scope` is filled in by hand with what the middleware and `URLRouter` would put there in production:

```python title="notes_app/tests/test_consumers.py (fragment)"
def _make_communicator(group_pk, user):
    communicator = WebsocketCommunicator(
        GroupChatConsumer.as_asgi(),
        f'/ws/groups/{group_pk}/chat/',
    )
    communicator.scope['user'] = user
    communicator.scope['url_route'] = {'kwargs': {'group_pk': str(group_pk)}}
    return communicator
```

**Broadcast between two members:**

```python title="notes_app/tests/test_consumers.py (GroupChatConsumerMessagesTest)"
async def test_message_broadcast_to_other_member(self):
    comm_alice = _make_communicator(self.group.pk, self.user1)
    comm_bob   = _make_communicator(self.group.pk, self.user2)

    await comm_alice.connect()
    await comm_bob.connect()

    await comm_alice.send_json_to({'content': 'Привіт від Alice!'})

    # Alice also receives her own message (the broadcast includes the sender)
    msg_alice = await comm_alice.receive_json_from()
    self.assertEqual(msg_alice['type'], 'message')
    self.assertEqual(msg_alice['author'], 'alice')

    msg_bob = await comm_bob.receive_json_from()
    self.assertEqual(msg_bob['content'], 'Привіт від Alice!')

    await comm_alice.disconnect()
    await comm_bob.disconnect()
```

The other 8 tests in the file: `test_member_can_connect`, `test_anonymous_user_rejected`, `test_non_member_rejected`, `test_message_saved_to_database` (after the broadcast, `ChatMessage...acount() == 1`), `test_message_contains_timestamp` (`timestamp`, `message_id`), `test_empty_message_not_broadcast` (`receive_nothing(timeout=0.5)`), `test_history_sent_on_connect` (two messages with `type='history'` in creation order), `test_message_stays_within_group`.

!!! note "asyncSetUp() in Django 5.1+"
    Django 5.1 added `asyncSetUp()` and `asyncTearDown()`. You can write:
    ```python
    async def asyncSetUp(self):
        self.user = await User.objects.acreate_user('testuser', password='pass')
        self.group = await DjangoGroup.objects.acreate(name='Chat')
        await self.group.user_set.aadd(self.user)
    ```
    `notes_chat_app` uses a synchronous `setUp()`: it is simpler, and the async ORM methods (`acreate_user`, `acount`) are used only inside the tests themselves.

??? note "Alternative: a test through the full ASGI stack with a session cookie"

    Another approach, the one the previous version of this page used: the test opens the connection through `notes_project.asgi.application` (that is, through `AuthMiddlewareStack` and `URLRouter`) and authenticates the user with a real `sessionid` cookie. This also checks the middleware layer. The `notes_app/tests/test_consumers.py` in this repository has no such code: the test names below do not match the file, and this variant was not run for this checkpoint.

    **Basic structure**

    ```python
    # a test variant through the full ASGI stack (not a file in this repository)
    import json
    from channels.testing import WebsocketCommunicator
    from channels.db import database_sync_to_async
    from django.contrib.auth.models import User, Group
    from django.test import TransactionTestCase, override_settings

    from notes_project.asgi import application

    TEST_CHANNEL_LAYERS = {
        "default": {
            "BACKEND": "channels.layers.InMemoryChannelLayer",
        }
    }


    def _make_session_cookie(user):
        """Helper: creates a session cookie for user."""
        from importlib import import_module
        from django.conf import settings

        engine = import_module(settings.SESSION_ENGINE)
        session = engine.SessionStore()
        if user:
            session[settings.AUTH_USER_SESSION_KEY] = user.pk
            session['_auth_user_backend'] = 'django.contrib.auth.backends.ModelBackend'
            session['_auth_user_hash'] = user.get_session_auth_hash()
        session.save()
        return session.session_key


    @override_settings(CHANNEL_LAYERS=TEST_CHANNEL_LAYERS)
    class GroupChatConsumerTest(TransactionTestCase):

        def setUp(self):
            # setUp() is synchronous, even if the tests are async.
            # We make ORM calls directly: setUp() never runs
            # in an async context, so database_sync_to_async is not needed.
            self.user = User.objects.create_user('testuser', password='testpass123')
            self.other_user = User.objects.create_user('otheruser', password='testpass123')
            self.group = Group.objects.create(name='Test Group')
            self.group.user_set.add(self.user)

        def _communicator(self, user=None, group_pk=None):
            session_key = _make_session_cookie(user)
            pk = group_pk if group_pk is not None else self.group.pk
            return WebsocketCommunicator(
                application,
                f"/ws/groups/{pk}/chat/",
                headers=[(b"cookie", f"sessionid={session_key}".encode())],
            )
    ```

    **Test 1: a member can connect**

    ```python
        async def test_member_can_connect(self):
            comm = self._communicator(user=self.user)
            connected, _ = await comm.connect()
            self.assertTrue(connected)
            await comm.disconnect()
    ```

    **Test 2: history is sent after connect**

    ```python
        async def test_member_receives_history_on_connect(self):
            from notes_app.models import ChatMessage
            await database_sync_to_async(ChatMessage.objects.create)(
                group=self.group, author=self.user, content='Старе повідомлення',
            )

            comm = self._communicator(user=self.user)
            await comm.connect()

            response = await comm.receive_json_from()
            self.assertEqual(response['type'], 'history')
            self.assertEqual(response['content'], 'Старе повідомлення')

            await comm.disconnect()
    ```

    **Test 3: a non-member is rejected**

    ```python
        async def test_non_member_is_rejected(self):
            # other_user is NOT a member of the group
            comm = self._communicator(user=self.other_user)
            connected, _ = await comm.connect()
            self.assertFalse(connected)
            await comm.disconnect()
    ```

    **Test 4: an anonymous user is rejected**

    ```python
        async def test_anonymous_user_is_rejected(self):
            # No cookie means anonymous
            comm = WebsocketCommunicator(
                application,
                f"/ws/groups/{self.group.pk}/chat/",
            )
            connected, _ = await comm.connect()
            self.assertFalse(connected)
            await comm.disconnect()
    ```

    **Test 5: broadcast between two users**

    ```python
        async def test_message_broadcast_to_all_members(self):
            await database_sync_to_async(self.group.user_set.add)(self.other_user)

            comm1 = self._communicator(user=self.user)
            comm2 = self._communicator(user=self.other_user)

            await comm1.connect()
            await comm2.connect()

            await comm1.send_json_to({'content': 'Привіт від testuser'})

            response1 = await comm1.receive_json_from()
            response2 = await comm2.receive_json_from()

            self.assertEqual(response1['type'], 'message')
            self.assertEqual(response1['content'], 'Привіт від testuser')

            self.assertEqual(response2['type'], 'message')
            self.assertEqual(response2['content'], 'Привіт від testuser')
            self.assertEqual(response2['author'], 'testuser')

            await comm1.disconnect()
            await comm2.disconnect()
    ```

    **Test 6: the message is saved to the DB**

    ```python
        async def test_message_saved_to_database(self):
            from notes_app.models import ChatMessage

            comm = self._communicator(user=self.user)
            await comm.connect()

            await comm.send_json_to({'content': 'Тестове повідомлення'})
            await comm.receive_json_from()  # wait for the broadcast response

            count = await database_sync_to_async(
                ChatMessage.objects.filter(
                    group=self.group,
                    author=self.user,
                    content='Тестове повідомлення',
                ).count
            )()
            self.assertEqual(count, 1)

            await comm.disconnect()
    ```

### Modify the example { #modify }

1. Add to `test_empty_message_not_broadcast` a check for a message that is 2001 characters long: the consumer rejects `len(content) > 2000`.
2. Add logging to `connect()` before `accept()` and watch it in `docker compose logs -f web`:

    ```python
    import logging
    logger = logging.getLogger(__name__)

    async def connect(self):
        ...
        logger.info("User %s connected to group %s", self.user.username, self.group_pk)
        await self.accept()
    ```

### Try it yourself { #try-yourself }

1. **Observation:** `docker compose up -d`, open a group chat, DevTools → Network → WS: find `ws://.../ws/groups/<pk>/chat/`, send a message and inspect the frames.
2. **Broadcast:** two browsers as `demo_alice` and `demo_bob` (password `demo1234`) in the group `demo_команда_розробки`: the message appears in the other window without a reload.
3. **Authorization:** connect to a group you are not a member of (`demo_bob` → `demo_сімейний_чат`). What does DevTools show, and which `event.code` does `onclose` get? (The handshake is rejected before `accept()` → `1006`.)
4. **Understanding the code:** what happens if you remove `list()` in `load_history()`?

**Success criteria:** DevTools shows `101 Switching Protocols` for your own group and a refusal for someone else's; messages are saved (`ChatMessage` in the admin); the 9 consumer tests pass both with and without `REDIS_URL`.

## Summary { #summary }

| Concept | What to remember |
|---|---|
| ASGI | one process serves HTTP and WebSocket; `uvicorn notes_project.asgi:application` |
| `ProtocolTypeRouter` | `"http"` → Django, `"websocket"` → `AuthMiddlewareStack(URLRouter(...))` |
| Import order | `get_asgi_application()` before importing models / routing |
| `AuthMiddlewareStack` | reads `sessionid` from the handshake → `scope["user"]` |
| `GroupChatConsumer` | `connect` (membership check → `accept`/`close`), `receive`, `chat_message`, `disconnect` |
| Channel layer | InMemory without `REDIS_URL` (one process), Redis with it (several processes) |
| `database_sync_to_async` | every ORM call from async code; return already evaluated data |
| Security | `403` on the handshake for anonymous users and non-members; `escapeHtml()` in JS |
| nginx | `proxy_http_version 1.1` + `Upgrade` + `Connection` |
| Tests | `TransactionTestCase` + `WebsocketCommunicator`, `scope` by hand |

## Self-check { #self-check }

1. Why must `get_asgi_application()` be called before `from notes_project.routing import ...`?
2. What does the handshake return for a user who is not a member of the group, and at which stage does the consumer decide this?
3. Why do you need the Redis channel layer if the tests are green with InMemory too?
4. Why does `load_history` return `list(qs.values(...))`?
5. What does putting `daphne` first in `INSTALLED_APPS` give you?

??? success "Answers"

    1. `routing` imports `consumers`, and those import the models. Models can be imported only after `django.setup()`, which `get_asgi_application()` performs. Otherwise you get `AppRegistryNotReady`.
    2. `HTTP/1.1 403` (checked for `demo_bob` → group 2). In `connect()`: `check_membership()` returns `False` → `close()` before `accept()`.
    3. InMemory lives in the memory of a single process. In Docker / production several processes run (uvicorn workers, several containers), and a message from one has to reach the clients of another. Only a shared broker can do that.
    4. A QuerySet is lazy: without `list()` the SQL would run later, already in async code outside the `database_sync_to_async` thread → `SynchronousOnlyOperation`. On top of that, `values()` gives plain dictionaries instead of model instances.
    5. Daphne replaces the `runserver` command with an ASGI version that also serves WebSocket. The replacement works only if `daphne` comes before `django.contrib.staticfiles`, which also overrides `runserver`.

## What's next { #next }

[Step 8. Celery and background tasks](../08_background_tasks/index.md): the Celery worker and Beat, reminders, a browser toast.

The theory for this step in the book: [Async and Realtime](../../09_async_and_realtime/README.md).
