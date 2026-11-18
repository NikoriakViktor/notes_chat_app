# Контрольна точка: крок 7

## Перевір, що крок робочий { #check }

**7B — WebSocket-чат (`notes_chat_app`, цей репозиторій).** Consumer-тести, з кореня репозиторію, `DATABASE_URL` задано:

```text
$ python manage.py test notes_app.tests.test_consumers -v 2
Found 9 test(s).
...
Ran 9 tests in 9.167s

OK
```

Ті самі тести з `REDIS_URL=redis://localhost:6379/0` (тоді `settings.py` вмикає `RedisChannelLayer`), разом з тестами Celery-задач:

```text
$ REDIS_URL=redis://localhost:6379/0 python manage.py test notes_app.tests.test_consumers notes_app.tests.test_tasks
Found 27 test(s).
...
Ran 27 tests in 14.996s

OK
```

Живий ASGI-сервер — `uvicorn notes_project.asgi:application --port 8001` (у Docker його запускає `entrypoint.sh`):

```text
INFO:     ASGI 'lifespan' protocol appears unsupported.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8001 (Press CTRL+C to quit)
```

Перший рядок — не помилка: Django не реалізує ASGI lifespan, uvicorn просто продовжує. HTTP і WebSocket через той самий процес (`curl` з заголовками `Connection: Upgrade`, `Upgrade: websocket`, `Sec-WebSocket-Version: 13`):

```text
GET /accounts/login/                       200
GET /notes/                                302 → /accounts/login/?next=/notes/
WS  /ws/groups/1/chat/  без cookie          HTTP/1.1 403
WS  /ws/groups/1/chat/  demo_bob (учасник)  HTTP/1.1 101 Switching Protocols
WS  /ws/groups/2/chat/  demo_bob (не учасник) HTTP/1.1 403
```

Лог uvicorn для цих трьох handshake'ів: `"WebSocket /ws/groups/1/chat/" 403`, `[accepted]`, `403`. Перевірено на PostgreSQL 16 + Redis 7 (`redis:7-alpine`) з демо-даними `seed_demo_data --force`; `sessionid` для `demo_bob` створено через `SessionStore`, як у тестах. `403` — це відмова consumer'а (`close()` до `accept()`), браузер побачить закрите з'єднання з кодом `1006`.

**7A — async views (`hello_app`).** Це окремий навчальний проєкт кроку: `async_views.py`, `async_selectors.py`, `async_services.py` і маршрути `/async/notes/` у цьому репозиторії не живуть, тож тут їх не перевірено. Перевір на своєму проєкті: `python manage.py check`, твої тести з `AsyncClient`, і що `runserver` (:8000) та `uvicorn` (:8001) віддають ту саму нотатку.

## Чекліст { #checklist }

**7A — Async Views (`hello_app`)**

- [ ] `requirements.txt` містить `uvicorn`, `httpx`, `wsproto`.
- [ ] `asgi.py` — `ProtocolTypeRouter` налаштовано.
- [ ] `async_selectors.py` — lazy QuerySets (звичайні `def`) + `async def` тільки де `aget()`.
- [ ] `async_services.py` — `sync_to_async(create_note)`, `adelete()`, `aupdate(F(...))`.
- [ ] `async_views.py` — 5 async views: `async_note_list`, `async_note_detail`, `async_note_create`, `async_note_delete`, `async_note_toggle_pin`; підключені під `/async/notes/`.
- [ ] `tests/test_async_views.py` — async-тести через `AsyncClient`.
- [ ] Нотатка, створена через sync view, видна через async view (одна база).

**7B — WebSocket чат (`notes_chat_app`)**

- [ ] `daphne` — перший у `INSTALLED_APPS` (`['daphne', 'django.contrib.admin', ...]`), `channels` — теж у списку; `ASGI_APPLICATION = 'notes_project.asgi.application'`.
- [ ] Без `REDIS_URL` → `channels.layers.InMemoryChannelLayer`; з `REDIS_URL` → `RedisChannelLayer` з `"socket_timeout": None` і `"health_check_interval": 30`.
- [ ] `notes_project/asgi.py`: `django_asgi_app = get_asgi_application()` стоїть ПЕРЕД `from notes_project.routing import websocket_urlpatterns`.
- [ ] `notes_project/routing.py`: `re_path(r'^ws/groups/(?P<group_pk>\d+)/chat/$', consumers.GroupChatConsumer.as_asgi())`.
- [ ] `GroupChatConsumer(AsyncWebsocketConsumer)`: `check_membership`, `load_history`, `save_message` — `@database_sync_to_async`; `load_history` повертає `list(qs.values(...))`.
- [ ] `disconnect()` викликає `group_discard()` під перевіркою `hasattr(self, 'room_group_name')`.
- [ ] Handshake: анонім → `403`, учасник групи → `101`, не учасник → `403`.
- [ ] `notes_app/static/notes_app/js/group_chat.js` пропускає `author` і `content` через `escapeHtml()`.
- [ ] `nginx/nginx.conf`: `proxy_http_version 1.1` + `Upgrade $http_upgrade` + `Connection $connection_upgrade`.
- [ ] `python manage.py test notes_app.tests.test_consumers` → `Ran 9 tests`, `OK` (і з `REDIS_URL`, і без).
- [ ] Можеш пояснити, чому `load_history` повертає `list(...)`, а не QuerySet.
- [ ] Можеш пояснити, чому учасник іншої групи не отримує повідомлень, хоча обидва підключені до того самого channel layer.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `django.core.exceptions.SynchronousOnlyOperation: You cannot call this from an async context - use a thread or sync_to_async.` | sync ORM-виклик у `async def` consumer'а | винеси запит у метод з `@database_sync_to_async` |
| `SynchronousOnlyOperation` лише «іноді», на історії чату | `@database_sync_to_async` повернув lazy QuerySet, а SQL виконався пізніше в async-коді | `return list(qs.values(...))` всередині обгорнутого методу |
| `django.core.exceptions.AppRegistryNotReady: Apps aren't loaded yet.` при старті uvicorn | імпорт `routing` / `consumers` до `get_asgi_application()` | спочатку `django_asgi_app = get_asgi_application()`, потім імпорти |
| `TimeoutError` / `Timeout reading from` Redis, з'єднання чату рветься після простою | малий `socket_timeout` проти блокуючого читання channel layer | `"socket_timeout": None, "health_check_interval": 30` |
| WS через nginx: `400` або з'єднання одразу закривається | немає заголовків Upgrade у proxy | `proxy_http_version 1.1` + `Upgrade` + `Connection` |
| Uvicorn: `No supported WebSocket library detected`, WS → `404` | не встановлено `wsproto` (або `websockets`) | `pip install -r requirements.txt` |
| WS handshake `403` для залогіненого учасника | cookie `sessionid` не дійшов (інший домен/порт) або consumer не знайшов групу | відкривай чат з того самого origin, що й сайт; перевір `group_pk` |
| `runserver` не обслуговує WebSocket | `daphne` не перший у `INSTALLED_APPS` | перемісти `daphne` на початок списку |
| Consumer «не знаходиться», WS → `404` | `websocket_urlpatterns` не підключені в `asgi.py` | `AuthMiddlewareStack(URLRouter(websocket_urlpatterns))` |

??? note "Приклади виправлень у коді"

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
    # ❌ asgi.py — імпорт ДО get_asgi_application():
    from notes_project.routing import websocket_urlpatterns
    django_asgi_app = get_asgi_application()
    # ✅
    django_asgi_app = get_asgi_application()
    from notes_project.routing import websocket_urlpatterns

    # time.sleep() блокує event loop:
    # ❌
    async def connect(self):
        time.sleep(1)
    # ✅
    async def connect(self):
        await asyncio.sleep(1)

    # Lazy QuerySet у @database_sync_to_async:
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

## Практика { #practice }

### Розібраний приклад: consumer-тест на `TransactionTestCase` { #worked-example }

**Чому `TransactionTestCase`, а не `TestCase`:**

```
TestCase: огортає кожен тест у транзакцію (rollback після тесту)
  → consumer звертається до БД через database_sync_to_async в іншому потоці
  → цей потік не бачить незакомічені дані setUp()
  → setUp-об'єкти (User, Group) «зникають» для consumer

TransactionTestCase: не загортає у транзакцію
  → дані записуються у БД і видимі всім потокам
  → consumer бачить User і Group з setUp()
  → ціна: повне очищення БД між тестами (повільніше)
```

**Хелпер.** Тест не проходить через HTTP і `AuthMiddlewareStack` — `scope` задається вручну тим, що в production туди поклали б middleware і `URLRouter`:

```python title="notes_app/tests/test_consumers.py (фрагмент)"
def _make_communicator(group_pk, user):
    communicator = WebsocketCommunicator(
        GroupChatConsumer.as_asgi(),
        f'/ws/groups/{group_pk}/chat/',
    )
    communicator.scope['user'] = user
    communicator.scope['url_route'] = {'kwargs': {'group_pk': str(group_pk)}}
    return communicator
```

**Broadcast між двома учасниками:**

```python title="notes_app/tests/test_consumers.py (GroupChatConsumerMessagesTest)"
async def test_message_broadcast_to_other_member(self):
    comm_alice = _make_communicator(self.group.pk, self.user1)
    comm_bob   = _make_communicator(self.group.pk, self.user2)

    await comm_alice.connect()
    await comm_bob.connect()

    await comm_alice.send_json_to({'content': 'Привіт від Alice!'})

    # Alice також отримує власне повідомлення (broadcast включає відправника)
    msg_alice = await comm_alice.receive_json_from()
    self.assertEqual(msg_alice['type'], 'message')
    self.assertEqual(msg_alice['author'], 'alice')

    msg_bob = await comm_bob.receive_json_from()
    self.assertEqual(msg_bob['content'], 'Привіт від Alice!')

    await comm_alice.disconnect()
    await comm_bob.disconnect()
```

Решта 8 тестів файлу: `test_member_can_connect`, `test_anonymous_user_rejected`, `test_non_member_rejected`, `test_message_saved_to_database` (після broadcast `ChatMessage...acount() == 1`), `test_message_contains_timestamp` (`timestamp`, `message_id`), `test_empty_message_not_broadcast` (`receive_nothing(timeout=0.5)`), `test_history_sent_on_connect` (два повідомлення `type='history'` у порядку створення), `test_message_stays_within_group`.

!!! note "asyncSetUp() у Django 5.1+"
    Django 5.1 додав `asyncSetUp()` і `asyncTearDown()`. Можна писати:
    ```python
    async def asyncSetUp(self):
        self.user = await User.objects.acreate_user('testuser', password='pass')
        self.group = await DjangoGroup.objects.acreate(name='Chat')
        await self.group.user_set.aadd(self.user)
    ```
    `notes_chat_app` використовує синхронний `setUp()`: він простіший, а async-методи ORM (`acreate_user`, `acount`) використовуються лише всередині самих тестів.

??? note "Альтернатива: тест через повний ASGI-стек із session cookie"

    Інший підхід, який був у попередній версії цієї сторінки: тест відкриває з'єднання через `notes_project.asgi.application` (тобто через `AuthMiddlewareStack` і `URLRouter`) і автентифікує користувача справжнім cookie `sessionid`. Так перевіряється ще й middleware-шар. У `notes_app/tests/test_consumers.py` цього репозиторію такого коду немає — назви тестів нижче не збігаються з файлом, і в цій контрольній точці цей варіант не запускався.

    **Базова структура**

    ```python
    # варіант тесту через повний ASGI-стек (не файл цього репозиторію)
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
        """Хелпер: створює session cookie для user."""
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
            # setUp() — синхронний, навіть якщо тести async.
            # ORM-виклики виконуємо напряму: setUp() ніколи не запускається
            # в async-контексті, тому database_sync_to_async не потрібний.
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

    **Тест 1: member може підключитись**

    ```python
        async def test_member_can_connect(self):
            comm = self._communicator(user=self.user)
            connected, _ = await comm.connect()
            self.assertTrue(connected)
            await comm.disconnect()
    ```

    **Тест 2: history надсилається після connect**

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

    **Тест 3: non-member отримує відмову**

    ```python
        async def test_non_member_is_rejected(self):
            # other_user НЕ є членом групи
            comm = self._communicator(user=self.other_user)
            connected, _ = await comm.connect()
            self.assertFalse(connected)
            await comm.disconnect()
    ```

    **Тест 4: anonymous отримує відмову**

    ```python
        async def test_anonymous_user_is_rejected(self):
            # Без cookie — anonymous
            comm = WebsocketCommunicator(
                application,
                f"/ws/groups/{self.group.pk}/chat/",
            )
            connected, _ = await comm.connect()
            self.assertFalse(connected)
            await comm.disconnect()
    ```

    **Тест 5: broadcast між двома юзерами**

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

    **Тест 6: повідомлення зберігається у БД**

    ```python
        async def test_message_saved_to_database(self):
            from notes_app.models import ChatMessage

            comm = self._communicator(user=self.user)
            await comm.connect()

            await comm.send_json_to({'content': 'Тестове повідомлення'})
            await comm.receive_json_from()  # чекаємо broadcast відповіді

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

### Зміни приклад { #modify }

1. Додай у `test_empty_message_not_broadcast` перевірку повідомлення довжиною 2001 символ — consumer відкидає `len(content) > 2000`.
2. Додай у `connect()` логування перед `accept()` і подивись його в `docker compose logs -f web`:

    ```python
    import logging
    logger = logging.getLogger(__name__)

    async def connect(self):
        ...
        logger.info("User %s connected to group %s", self.user.username, self.group_pk)
        await self.accept()
    ```

### Спробуй самостійно { #try-yourself }

1. **Спостереження:** `docker compose up -d`, відкрий чат групи, DevTools → Network → WS: знайди `ws://.../ws/groups/<pk>/chat/`, надішли повідомлення і переглянь frames.
2. **Broadcast:** два браузери під `demo_alice` і `demo_bob` (пароль `demo1234`) у групі `demo_команда_розробки` — повідомлення з'являється в іншому вікні без перезавантаження.
3. **Авторизація:** підключись до групи, членом якої не є (`demo_bob` → `demo_сімейний_чат`). Що показує DevTools і який `event.code` отримує `onclose`? (Handshake відхилено до `accept()` → `1006`.)
4. **Розуміння коду:** що станеться, якщо в `load_history()` прибрати `list()`?

**Критерії перевірки:** у DevTools видно `101 Switching Protocols` для своєї групи і відмову для чужої; повідомлення зберігаються (`ChatMessage` у адмінці); 9 consumer-тестів проходять і з `REDIS_URL`, і без.

## Підсумок { #summary }

| Поняття | Що запам'ятати |
|---|---|
| ASGI | один процес обслуговує HTTP і WebSocket; `uvicorn notes_project.asgi:application` |
| `ProtocolTypeRouter` | `"http"` → Django, `"websocket"` → `AuthMiddlewareStack(URLRouter(...))` |
| Порядок імпортів | `get_asgi_application()` до імпорту моделей / routing |
| `AuthMiddlewareStack` | читає `sessionid` з handshake → `scope["user"]` |
| `GroupChatConsumer` | `connect` (перевірка членства → `accept`/`close`), `receive`, `chat_message`, `disconnect` |
| Channel layer | InMemory без `REDIS_URL` (один процес), Redis з ним (кілька процесів) |
| `database_sync_to_async` | кожен ORM-виклик з async-коду; повертати вже обчислені дані |
| Безпека | `403` на handshake для аноніма й не учасника; `escapeHtml()` у JS |
| nginx | `proxy_http_version 1.1` + `Upgrade` + `Connection` |
| Тести | `TransactionTestCase` + `WebsocketCommunicator`, `scope` вручну |

## Самоперевірка { #self-check }

1. Чому `get_asgi_application()` має бути викликаний до `from notes_project.routing import ...`?
2. Що повертає handshake для користувача, який не є учасником групи, і на якому етапі consumer це вирішує?
3. Навіщо Redis channel layer, якщо з InMemory тести теж зелені?
4. Чому `load_history` повертає `list(qs.values(...))`?
5. Що дає `daphne` першим у `INSTALLED_APPS`?

??? success "Відповіді"

    1. `routing` імпортує `consumers`, а ті — моделі. Моделі можна імпортувати лише після `django.setup()`, який робить `get_asgi_application()`. Інакше — `AppRegistryNotReady`.
    2. `HTTP/1.1 403` (перевірено для `demo_bob` → група 2). У `connect()`: `check_membership()` повертає `False` → `close()` до `accept()`.
    3. InMemory живе в пам'яті одного процесу. У Docker / production працює кілька процесів (воркери uvicorn, кілька контейнерів) — повідомлення з одного має дійти до клієнтів іншого. Це вміє лише спільний брокер.
    4. QuerySet лінивий: без `list()` SQL виконався б пізніше, вже в async-коді поза потоком `database_sync_to_async` → `SynchronousOnlyOperation`. `values()` до того ж дає прості словники замість моделей.
    5. Daphne підміняє команду `runserver` на ASGI-версію, яка обслуговує й WebSocket. Підміна працює, лише якщо `daphne` стоїть перед `django.contrib.staticfiles`, що теж перевизначає `runserver`.

## Що далі { #next }

[Крок 8. Celery та фонові задачі](../08_background_tasks/index.md) — Celery worker і Beat, нагадування, browser toast.

Теорія до цього кроку в книзі: [Async and Realtime](../../09_async_and_realtime/README.md).
