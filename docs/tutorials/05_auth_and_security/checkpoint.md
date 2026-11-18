# Контрольна точка: крок 5

## Перевір, що крок робочий { #check }

Перевірка — на `notes_chat_app` (цей репозиторій), PostgreSQL 16, демо-дані з `python manage.py seed_demo_data --force` (`demo_alice`, `demo_bob`, `demo_carol`, пароль `demo1234`; `--force` потрібен, бо без `DJANGO_DEBUG=1` сидер відмовляє: `CommandError: Demo data seeding is disabled when DEBUG=False. Use --force to override.`). У Docker те саме робить `docker compose run --rm web python manage.py seed_demo_data`.

Тести автентифікації, IDOR і груп:

```text
$ python manage.py test notes_app.tests.test_views.AuthenticationViewTest notes_app.tests.test_views.NoteDetailViewTest notes_app.tests.test_views.NoteEditViewTest notes_app.tests.test_views.NoteDeleteViewTest notes_app.tests.test_views.GroupViewTest notes_app.tests.test_views.TodoListSharingViewTest
Found 29 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.............................
----------------------------------------------------------------------
Ran 29 tests in 12.703s

OK
Destroying test database for alias 'default'...
```

Що відповідає застосунок (Django test client на тих самих демо-даних):

```text
anon GET /notes/              302 /accounts/login/?next=/notes/
anon GET /groups/             302 /accounts/login/?next=/groups/
GET  /accounts/login/         200
POST /accounts/login/ demo_alice / demo1234   302 /notes/
POST /accounts/login/ demo_alice / wrong      200  Будь ласка, введіть правильні ім'я користувача та пароль. Зауважте, що обидва поля чутливі до регістру.
GET  /accounts/logout/        405
POST /accounts/logout/        302 /accounts/login/
POST /accounts/password_reset/ email=alice@demo.example.com   302 /accounts/password_reset/done/, листів: 1
```

IDOR — `demo_bob` пробує чужу особисту нотатку `demo_alice` (`pk=3`) і спільну нотатку групи `demo_команда_розробки` (`pk=1`), де він учасник:

```text
bob GET  /notes/3/          404
bob GET  /notes/3/edit/     404
bob GET  /notes/3/delete/   404
bob POST /notes/3/delete/   404   нотатка існує: True
bob GET  /notes/1/          200   (спільна нотатка групи — читати можна)
bob GET  /notes/1/edit/     302 /notes/1/   (редагувати чужу не можна — повідомлення і назад до деталі)
```

`pk` у твоїй базі можуть бути іншими: сидер створює об'єкти заново. Візьми `pk` будь-якої нотатки `demo_alice` без групи.

Налаштування безпеки (`python manage.py shell -c "from django.conf import settings; ..."`):

```text
SESSION_COOKIE_HTTPONLY True   X_FRAME_OPTIONS DENY   SESSION_COOKIE_SAMESITE Lax   LOGIN_URL /accounts/login/
alice.password → pbkdf2_sha256$1000000$...
```

### IDOR-тест вручну в браузері

```
1. Залогінься як alice (demo_alice / demo1234)

2. Перейди на /notes/ і запам'ятай pk однієї зі своїх нотаток БЕЗ групи (наприклад pk=3)

3. Відкрий incognito-вікно і залогінься як bob (demo_bob / demo1234)

4. У вікні bob'а вручну введи URL:
   http://localhost/notes/3/edit/     ← pk нотатки alice!
   (через nginx — порт 80; напряму до web — http://localhost:8001/notes/3/edit/)

5. Очікуваний результат:
   ✓ 404 Not Found  — захист від IDOR працює
   ✗ Якщо показується форма редагування — IDOR вразливість!

6. Перевір також:
   http://localhost/notes/3/delete/   → теж 404
```

## Чекліст { #checklist }

**AuthN і сесії**

- [ ] Анонімний `GET /notes/` → `302` на `/accounts/login/?next=/notes/` (`@login_required`, `LOGIN_URL = /accounts/login/`).
- [ ] `include("django.contrib.auth.urls")` підключено під `accounts/`; є `templates/registration/login.html` і `register.html`.
- [ ] Вхід `demo_alice / demo1234` → `302` на `/notes/`; неправильний пароль → `200` з помилкою форми.
- [ ] `GET /accounts/logout/` → `405`; вихід — тільки POST-форма з `{% csrf_token %}` → `302` на `/accounts/login/`.
- [ ] `{% csrf_token %}` у КОЖНІЙ формі з `method="post"`; `SESSION_COOKIE_HTTPONLY = True`.

**Паролі**

- [ ] `POST /accounts/password_reset/` → `302` на `/accounts/password_reset/done/`; лист з посиланням `/accounts/reset/<uidb64>/<token>/` видно в `docker compose logs web` (або в терміналі `runserver`) — `EMAIL_BACKEND` за замовчуванням `console`.
- [ ] `password_reset_confirm.html` перевіряє `validlink`.
- [ ] Реєстрація з паролем `123` → `200` і три помилки валідаторів (`MinimumLengthValidator`, `CommonPasswordValidator`, `NumericPasswordValidator`); з надійним паролем → `302` на `/notes/` (auto-login).
- [ ] Пароль у базі — `pbkdf2_sha256$1000000$salt$hash`, не відкритий текст.

**IDOR і групи**

- [ ] Особисті об'єкти: `get_object_or_404(Notebook, pk=pk, user=request.user)`; чужий `pk` → `404`, не `403`.
- [ ] Спільні нотатки: Q-фільтр `Q(user=user) | Q(group__in=user_groups)` + перевірка `note.user != request.user` перед редагуванням; учасник групи читає (`200`), але не редагує (`302` на деталь).
- [ ] `Note.group` і `ShoppingList.group` — `on_delete=SET_NULL`: видалення групи робить нотатки особистими, а не видаляє їх.
- [ ] `services.create_group(name=..., creator=...)` створює групу й одразу додає автора першим учасником; `selectors.get_user_groups` використовує `annotate(member_count=Count('user'))`.

**Security settings**

- [ ] `X_FRAME_OPTIONS = "DENY"`, `SESSION_COOKIE_SAMESITE = "Lax"`; у production `DEBUG = False`, `ALLOWED_HOSTS` — лише реальні домени, HTTPS-налаштування готові до увімкнення.
- [ ] Можеш пояснити, чому для чужого об'єкта `404`, а не `403`.
- [ ] Можеш пояснити різницю AuthN («хто ти?») і AuthZ («що тобі можна?») на прикладі `/notes/1/edit/` для `demo_bob`.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `Forbidden (403) CSRF verification failed. Request aborted.` після сабміту | у формі немає `{% csrf_token %}` | `{% csrf_token %}` одразу після `<form method="post">` |
| Посилання «Вийти» дає `405 Method Not Allowed` | `LogoutView` приймає тільки POST (GET прибрано в Django 5.0) | `<form method="post" action="{% url 'logout' %}">{% csrf_token %}<button>Вийти</button></form>` |
| `request.user.is_authenticated` → `False`, хоча вхід був | немає `SessionMiddleware` / `AuthenticationMiddleware` або Auth стоїть перед Session | у `MIDDLEWARE`: спочатку `SessionMiddleware`, потім `AuthenticationMiddleware` |
| Лист для скидання пароля «не приходить» | dev-бекенд `console` пише лист у лог, а не на пошту | `docker compose logs -f web` (у Docker) або термінал `runserver`; шукай `Subject: Скидання пароля на ...` |
| Інший користувач відкриває `/notes/<pk>/edit/` чужої нотатки | view бере `get_object_or_404(Note, pk=pk)` без перевірки власника | особисті: `get_object_or_404(Model, pk=pk, user=request.user)`; спільні: Q-фільтр + `note.user != request.user` |
| Після видалення групи зникли нотатки учасників | `Note.group = ForeignKey(Group, on_delete=CASCADE)` | `on_delete=models.SET_NULL, null=True, blank=True` + нова міграція |
| `seed_demo_data` → `CommandError: Demo data seeding is disabled when DEBUG=False. Use --force to override.` | сидер захищає production-базу | `DJANGO_DEBUG=1` або `--force` (лише на dev-базі) |
| `403 CSRF verification failed` при вході через ngrok-домен | домену немає в `CSRF_TRUSTED_ORIGINS` | змінна `NGROK_DOMAIN` (settings додає її в `CSRF_TRUSTED_ORIGINS`) або `CSRF_TRUSTED_ORIGINS=https://...` |

??? note "Приклади виправлень у коді"

    **IDOR для спільних об'єктів**

    ```python
    user_groups = request.user.groups.all()
    note = get_object_or_404(
        Note.objects.filter(Q(user=request.user) | Q(group__in=user_groups)),
        pk=pk,
    )
    if note.user != request.user:
        messages.error(request, 'Ти не можеш редагувати нотатку іншого користувача.')
        return redirect('notes_app:note_detail', pk=pk)
    ```

    **Порядок middleware**

    ```python
    MIDDLEWARE = [
        ...
        'django.contrib.sessions.middleware.SessionMiddleware',     # ← спочатку Session
        ...
        'django.contrib.auth.middleware.AuthenticationMiddleware',  # ← потім Auth
        ...
    ]
    ```

    **Група без каскадного видалення нотаток**

    ```python
    group = models.ForeignKey(
        Group,
        on_delete=models.SET_NULL,   # ← SET_NULL, не CASCADE!
        null=True,
        blank=True,
        related_name='notes',
    )
    # Після цього — нова міграція: python manage.py makemigrations
    ```

## Практика { #practice }

### Розібраний приклад: чому `demo_bob` бачить, але не редагує { #worked-example }

`demo_bob` — учасник групи `demo_команда_розробки`. Нотатка `pk=1` належить `demo_alice` і позначена цією групою. Як `note_edit` вирішує, що робити?

1. **Знайти чи 404.** Queryset обмежено тим, що користувач має право бачити:

    ```python title="notes_app/views.py (note_edit, фрагмент)"
    user_groups = request.user.groups.all()
    note = get_object_or_404(
        Note.objects.filter(Q(user=request.user) | Q(group__in=user_groups)),
        pk=pk,
    )
    ```

    Для особистої нотатки alice (`pk=3`) bob не проходить жодну з умов → `404`. Факт існування нотатки не розкрито.

2. **Читати ≠ змінювати.** Нотатку `pk=1` знайдено (bob у групі), але:

    ```python
    if note.user != request.user:
        messages.error(request, 'Ти не можеш редагувати нотатку іншого користувача.')
        return redirect('notes_app:note_detail', pk=pk)
    ```

3. **Результат** (виміряно):

    ```text
    bob GET /notes/3/edit/   404
    bob GET /notes/1/        200
    bob GET /notes/1/edit/   302 /notes/1/
    ```

### Зміни приклад { #modify }

1. Дозволь учасникам групи редагувати спільні нотатки, але не видаляти: зміни перевірку лише в `note_edit`. Який тест у `NoteEditViewTest` має впасти — і чи правильно, що впаде?
2. Тимчасово заміни в `note_edit` queryset на `get_object_or_404(Note, pk=pk)` (без Q-фільтра), повтори IDOR-тест — і переконайся, що bob тепер отримує не `404`, а `302`. Поверни захист.

### Спробуй самостійно { #try-yourself }

1. **Базове:** запусти `docker compose up --build`, зареєструй нового користувача, протестуй Password Reset — знайди лист у `docker compose logs web` і зміни пароль.
2. **Середнє:** створи групу «Моя команда», додай `demo_bob`. Створи нотатку з цією групою, залогінься як `demo_bob` і переконайся, що нотатка є в `/notes/`.
3. **Дослідницьке:** DevTools → Application → Cookies після входу: у `sessionid` має бути `HttpOnly ✓`. У Console `document.cookie` не містить `sessionid`.

**Критерії перевірки:** IDOR-тест дає `404` для чужих особистих об'єктів; учасник групи бачить спільну нотатку; посилання з листа відкриває форму нового пароля, а повторне — `validlink` = `False`; 29 тестів з розділу «Перевір» проходять.

## Підсумок { #summary }

| Концепція | Де у коді |
|-----------|-----------|
| AuthN vs AuthZ | `views.py` — `@login_required` + Q-фільтр доступу + `note.user != request.user` для редагування |
| Middleware chain | `settings.py` — `MIDDLEWARE`, порядок Session → Auth |
| Session flow | `settings.py` — `SESSION_COOKIE_*` + `django.contrib.sessions` |
| Login/Logout | `templates/registration/login.html` + `accounts/` → `django.contrib.auth.urls` |
| POST logout | `templates/layouts/dashboard.html` — `<form method="post" action="{% url 'logout' %}">` |
| Password Reset | `templates/registration/password_reset_*.html` (5 шаблонів) |
| DEV email | `EMAIL_BACKEND` = `console` → `docker compose logs -f web`, посилання `/accounts/reset/...` |
| Password Change | `templates/registration/password_change_*.html` |
| IDOR захист | особисті об'єкти: `get_object_or_404(Model, pk=pk, user=...)`; спільні: Q-фільтр + перевірка власника |
| 404, не 403 | не розкривати факт існування чужого об'єкта |
| Group FK SET_NULL | `models.py` — `Note.group`, `ShoppingList.group` |
| Q-фільтр | `selectors.py` — `Q(user=user) \| Q(group__in=user_groups)` |
| N+1 захист | `selectors.py` — `select_related('notebook', 'group')`, `prefetch_related('tags')` |
| Group CRUD | `services.py` — `create_group`, `add_user_to_group`, `remove_user_from_group`, `delete_group` |
| 3-action POST | `views.py` `group_detail` — `action='add'/'remove'/'leave'` |
| SET_NULL при видаленні групи | `notes_app/templates/notes_app/group_confirm_delete.html` — пояснює, що нотатки стануть особистими |
| Security settings | `settings.py` — `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SAMESITE`, `X_FRAME_OPTIONS`, `DEBUG` зі змінної `DJANGO_DEBUG` |
| ngrok CSRF | `settings.py` — `CSRF_TRUSTED_ORIGINS` з `NGROK_DOMAIN` |
| PBKDF2 паролі | `auth_user.password` — `pbkdf2_sha256$1000000$salt$hash` |

## Самоперевірка { #self-check }

1. Чому для чужої нотатки — `404`, а не `403`?
2. Що перевіряє `@login_required` і чого він НЕ перевіряє?
3. Чому `GET /accounts/logout/` повертає `405`?
4. Що станеться з нотатками, позначеними групою, якщо групу видалити?
5. Навіщо `HttpOnly` на `sessionid`?

??? success "Відповіді"

    1. `403` каже «об'єкт є, але не твій» — атакувальник, перебираючи `pk`, дізнається, які id існують. `404` однаковий для «немає» і «не твоє».
    2. Тільки AuthN — що користувач залогінений. Чи має він право на конкретний об'єкт (AuthZ), вирішує queryset із фільтром за `user` / групами.
    3. Вихід змінює стан, тому з Django 5.0 `LogoutView` приймає лише POST з CSRF-токеном. GET-вихід можна було викликати з чужої сторінки.
    4. `on_delete=SET_NULL`: `group_id` стає `NULL`, нотатки лишаються у власників як особисті. Повідомлення чату групи (`ChatMessage.group`, `CASCADE`) видаляються.
    5. JavaScript на сторінці (зокрема впроваджений через XSS) не може прочитати cookie через `document.cookie`, тож не вкраде сесію.

## Що далі { #next }

[Крок 6. Тестування](../06_testing/index.md) — піраміда тестів: models, services, forms, views, consumers, Selenium і CI.
