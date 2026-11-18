# Checkpoint: step 5

## Check that the step works { #check }

The check runs on `notes_chat_app` (this repository), PostgreSQL 16, with demo data from `python manage.py seed_demo_data --force` (`demo_alice`, `demo_bob`, `demo_carol`, password `demo1234`; `--force` is needed because without `DJANGO_DEBUG=1` the seeder refuses: `CommandError: Demo data seeding is disabled when DEBUG=False. Use --force to override.`). In Docker, `docker compose run --rm web python manage.py seed_demo_data` does the same.

Authentication, IDOR and group tests:

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

How the application responds (Django test client on the same demo data):

```text
anon GET /notes/              302 /accounts/login/?next=/notes/
anon GET /groups/             302 /accounts/login/?next=/groups/
GET  /accounts/login/         200
POST /accounts/login/ demo_alice / demo1234   302 /notes/
POST /accounts/login/ demo_alice / wrong      200  Будь ласка, введіть правильні ім'я користувача та пароль. Зауважте, що обидва поля чутливі до регістру.
GET  /accounts/logout/        405
POST /accounts/logout/        302 /accounts/login/
POST /accounts/password_reset/ email=alice@demo.example.com   302 /accounts/password_reset/done/, emails: 1
```

IDOR: `demo_bob` tries `demo_alice`'s personal note (`pk=3`) and a shared note of the group `demo_команда_розробки` (`pk=1`), where he is a member:

```text
bob GET  /notes/3/          404
bob GET  /notes/3/edit/     404
bob GET  /notes/3/delete/   404
bob POST /notes/3/delete/   404   note exists: True
bob GET  /notes/1/          200   (shared group note: reading is allowed)
bob GET  /notes/1/edit/     302 /notes/1/   (editing someone else's note is not: a message and back to the detail page)
```

The `pk` values in your database may differ: the seeder creates the objects from scratch. Take the `pk` of any `demo_alice` note without a group.

Security settings (`python manage.py shell -c "from django.conf import settings; ..."`):

```text
SESSION_COOKIE_HTTPONLY True   X_FRAME_OPTIONS DENY   SESSION_COOKIE_SAMESITE Lax   LOGIN_URL /accounts/login/
alice.password → pbkdf2_sha256$1000000$...
```

### Manual IDOR test in the browser

```
1. Log in as alice (demo_alice / demo1234)

2. Go to /notes/ and remember the pk of one of your notes WITHOUT a group (for example pk=3)

3. Open an incognito window and log in as bob (demo_bob / demo1234)

4. In bob's window, type the URL by hand:
   http://localhost/notes/3/edit/     ← alice's note pk!
   (through nginx, port 80; directly to web: http://localhost:8001/notes/3/edit/)

5. Expected result:
   ✓ 404 Not Found  — IDOR protection works
   ✗ If the edit form is shown — IDOR vulnerability!

6. Also check:
   http://localhost/notes/3/delete/   → also 404
```

## Checklist { #checklist }

**AuthN and sessions**

- [ ] Anonymous `GET /notes/` → `302` to `/accounts/login/?next=/notes/` (`@login_required`, `LOGIN_URL = /accounts/login/`).
- [ ] `include("django.contrib.auth.urls")` is mounted under `accounts/`; `templates/registration/login.html` and `register.html` exist.
- [ ] Logging in with `demo_alice / demo1234` → `302` to `/notes/`; a wrong password → `200` with a form error.
- [ ] `GET /accounts/logout/` → `405`; logout is only a POST form with `{% csrf_token %}` → `302` to `/accounts/login/`.
- [ ] `{% csrf_token %}` in EVERY form with `method="post"`; `SESSION_COOKIE_HTTPONLY = True`.

**Passwords**

- [ ] `POST /accounts/password_reset/` → `302` to `/accounts/password_reset/done/`; the email with the link `/accounts/reset/<uidb64>/<token>/` is visible in `docker compose logs web` (or in the `runserver` terminal), since `EMAIL_BACKEND` defaults to `console`.
- [ ] `password_reset_confirm.html` checks `validlink`.
- [ ] Registering with the password `123` → `200` and three validator errors (`MinimumLengthValidator`, `CommonPasswordValidator`, `NumericPasswordValidator`); with a strong password → `302` to `/notes/` (auto-login).
- [ ] The password in the database is `pbkdf2_sha256$1000000$salt$hash`, not plain text.

**IDOR and groups**

- [ ] Personal objects: `get_object_or_404(Notebook, pk=pk, user=request.user)`; someone else's `pk` → `404`, not `403`.
- [ ] Shared notes: the Q filter `Q(user=user) | Q(group__in=user_groups)` + a `note.user != request.user` check before editing; a group member can read (`200`) but not edit (`302` to the detail page).
- [ ] `Note.group` and `ShoppingList.group` use `on_delete=SET_NULL`: deleting a group turns its notes into personal ones instead of deleting them.
- [ ] `services.create_group(name=..., creator=...)` creates the group and immediately adds the author as the first member; `selectors.get_user_groups` uses `annotate(member_count=Count('user'))`.

**Security settings**

- [ ] `X_FRAME_OPTIONS = "DENY"`, `SESSION_COOKIE_SAMESITE = "Lax"`; in production `DEBUG = False`, `ALLOWED_HOSTS` lists only real domains, the HTTPS settings are ready to be switched on.
- [ ] You can explain why someone else's object gives `404` and not `403`.
- [ ] You can explain the difference between AuthN ("who are you?") and AuthZ ("what are you allowed to do?") using `/notes/1/edit/` for `demo_bob` as the example.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `Forbidden (403) CSRF verification failed. Request aborted.` after submitting | the form has no `{% csrf_token %}` | `{% csrf_token %}` right after `<form method="post">` |
| The «Вийти» (Log out) link gives `405 Method Not Allowed` | `LogoutView` accepts only POST (GET was removed in Django 5.0) | `<form method="post" action="{% url 'logout' %}">{% csrf_token %}<button>Вийти</button></form>` |
| `request.user.is_authenticated` → `False` even though you logged in | `SessionMiddleware` / `AuthenticationMiddleware` is missing, or Auth comes before Session | in `MIDDLEWARE`: `SessionMiddleware` first, then `AuthenticationMiddleware` |
| The password reset email "never arrives" | the `console` dev backend writes the email to the log, not to a mailbox | `docker compose logs -f web` (in Docker) or the `runserver` terminal; look for `Subject: Скидання пароля на ...` |
| Another user opens `/notes/<pk>/edit/` of someone else's note | the view uses `get_object_or_404(Note, pk=pk)` without an owner check | personal: `get_object_or_404(Model, pk=pk, user=request.user)`; shared: Q filter + `note.user != request.user` |
| Members' notes disappeared after the group was deleted | `Note.group = ForeignKey(Group, on_delete=CASCADE)` | `on_delete=models.SET_NULL, null=True, blank=True` + a new migration |
| `seed_demo_data` → `CommandError: Demo data seeding is disabled when DEBUG=False. Use --force to override.` | the seeder protects the production database | `DJANGO_DEBUG=1` or `--force` (on a dev database only) |
| `403 CSRF verification failed` when logging in through an ngrok domain | the domain is not in `CSRF_TRUSTED_ORIGINS` | the `NGROK_DOMAIN` variable (settings adds it to `CSRF_TRUSTED_ORIGINS`) or `CSRF_TRUSTED_ORIGINS=https://...` |

??? note "Fix examples in code"

    **IDOR for shared objects**

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

    **Middleware order**

    ```python
    MIDDLEWARE = [
        ...
        'django.contrib.sessions.middleware.SessionMiddleware',     # ← Session first
        ...
        'django.contrib.auth.middleware.AuthenticationMiddleware',  # ← then Auth
        ...
    ]
    ```

    **A group without cascading deletion of notes**

    ```python
    group = models.ForeignKey(
        Group,
        on_delete=models.SET_NULL,   # ← SET_NULL, not CASCADE!
        null=True,
        blank=True,
        related_name='notes',
    )
    # After this, create a new migration: python manage.py makemigrations
    ```

## Practice { #practice }

### Worked example: why `demo_bob` can see but not edit { #worked-example }

`demo_bob` is a member of the group `demo_команда_розробки`. The note `pk=1` belongs to `demo_alice` and is tagged with this group. How does `note_edit` decide what to do?

1. **Find it or 404.** The queryset is limited to what the user is allowed to see:

    ```python title="notes_app/views.py (note_edit, excerpt)"
    user_groups = request.user.groups.all()
    note = get_object_or_404(
        Note.objects.filter(Q(user=request.user) | Q(group__in=user_groups)),
        pk=pk,
    )
    ```

    For alice's personal note (`pk=3`) bob matches neither condition → `404`. The fact that the note exists is not revealed.

2. **Reading ≠ changing.** The note `pk=1` is found (bob is in the group), but:

    ```python
    if note.user != request.user:
        messages.error(request, 'Ти не можеш редагувати нотатку іншого користувача.')
        return redirect('notes_app:note_detail', pk=pk)
    ```

3. **Result** (measured):

    ```text
    bob GET /notes/3/edit/   404
    bob GET /notes/1/        200
    bob GET /notes/1/edit/   302 /notes/1/
    ```

### Modify the example { #modify }

1. Allow group members to edit shared notes but not delete them: change the check in `note_edit` only. Which test in `NoteEditViewTest` should fail, and is it right that it fails?
2. Temporarily replace the queryset in `note_edit` with `get_object_or_404(Note, pk=pk)` (no Q filter), repeat the IDOR test, and make sure bob now gets `302` instead of `404`. Then restore the protection.

### Try it yourself { #try-yourself }

1. **Basic:** run `docker compose up --build`, register a new user, test Password Reset: find the email in `docker compose logs web` and change the password.
2. **Intermediate:** create a group «My team» and add `demo_bob`. Create a note with this group, log in as `demo_bob` and make sure the note is in `/notes/`.
3. **Exploratory:** DevTools → Application → Cookies after logging in: `sessionid` must have `HttpOnly ✓`. In the Console, `document.cookie` does not contain `sessionid`.

**Success criteria:** the IDOR test gives `404` for other users' personal objects; a group member sees the shared note; the link from the email opens the new password form, and opening it again gives `validlink` = `False`; the 29 tests from the "Check" section pass.

## Summary { #summary }

| Concept | Where in the code |
|-----------|-----------|
| AuthN vs AuthZ | `views.py`: `@login_required` + access Q filter + `note.user != request.user` for editing |
| Middleware chain | `settings.py`: `MIDDLEWARE`, order Session → Auth |
| Session flow | `settings.py`: `SESSION_COOKIE_*` + `django.contrib.sessions` |
| Login/Logout | `templates/registration/login.html` + `accounts/` → `django.contrib.auth.urls` |
| POST logout | `templates/layouts/dashboard.html`: `<form method="post" action="{% url 'logout' %}">` |
| Password Reset | `templates/registration/password_reset_*.html` (5 templates) |
| DEV email | `EMAIL_BACKEND` = `console` → `docker compose logs -f web`, link `/accounts/reset/...` |
| Password Change | `templates/registration/password_change_*.html` |
| IDOR protection | personal objects: `get_object_or_404(Model, pk=pk, user=...)`; shared: Q filter + owner check |
| 404, not 403 | do not reveal that someone else's object exists |
| Group FK SET_NULL | `models.py`: `Note.group`, `ShoppingList.group` |
| Q filter | `selectors.py`: `Q(user=user) \| Q(group__in=user_groups)` |
| N+1 protection | `selectors.py`: `select_related('notebook', 'group')`, `prefetch_related('tags')` |
| Group CRUD | `services.py`: `create_group`, `add_user_to_group`, `remove_user_from_group`, `delete_group` |
| 3-action POST | `views.py` `group_detail`: `action='add'/'remove'/'leave'` |
| SET_NULL on group deletion | `notes_app/templates/notes_app/group_confirm_delete.html`: explains that the notes will become personal |
| Security settings | `settings.py`: `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SAMESITE`, `X_FRAME_OPTIONS`, `DEBUG` from the `DJANGO_DEBUG` variable |
| ngrok CSRF | `settings.py`: `CSRF_TRUSTED_ORIGINS` from `NGROK_DOMAIN` |
| PBKDF2 passwords | `auth_user.password`: `pbkdf2_sha256$1000000$salt$hash` |

## Self-check { #self-check }

1. Why does someone else's note give `404` and not `403`?
2. What does `@login_required` check, and what does it NOT check?
3. Why does `GET /accounts/logout/` return `405`?
4. What happens to notes tagged with a group if the group is deleted?
5. Why does `sessionid` need `HttpOnly`?

??? success "Answers"

    1. `403` says "the object exists, but it is not yours": an attacker iterating over `pk` values learns which ids exist. `404` is the same for "does not exist" and "not yours".
    2. Only AuthN: that the user is logged in. Whether they have the right to a specific object (AuthZ) is decided by the queryset filtered by `user` / groups.
    3. Logging out changes state, so since Django 5.0 `LogoutView` accepts only POST with a CSRF token. A GET logout could be triggered from someone else's page.
    4. `on_delete=SET_NULL`: `group_id` becomes `NULL`, and the notes stay with their owners as personal notes. The group's chat messages (`ChatMessage.group`, `CASCADE`) are deleted.
    5. JavaScript on the page (including code injected through XSS) cannot read the cookie via `document.cookie`, so it cannot steal the session.

## What's next { #next }

[Step 6. Testing](../06_testing/index.md): the test pyramid: models, services, forms, views, consumers, Selenium and CI.
