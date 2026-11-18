# Checkpoint: step 1

## Check that the step works { #check }

From the `hello_project` project folder (the one that contains `manage.py`), with the venv activated:

```text
$ python -m django --version
5.2.19
$ python manage.py check
System check identified no issues (0 silenced).
$ python manage.py showmigrations hello_app
hello_app
 [X] 0001_initial
```

Start the server with `python manage.py runserver`, then in a second terminal:

```text
$ curl -s http://localhost:8000/
Hello, Django!
$ curl -s http://localhost:8000/about/
Це моя перша сторінка на Django!
$ curl -s -o /dev/null -w "%{http_code} -> %{redirect_url}\n" http://localhost:8000/admin/
302 -> http://localhost:8000/admin/login/?next=/admin/
$ curl -s http://localhost:8000/notes/
<h1>Мої нотатки</h1>
<ul>

  <li>Друга нотатка</li>

  <li>Перша нотатка</li>

</ul>
```

Verified with Django 5.2.19 and Python 3.12 in a clean folder: `startproject hello_project .`, `startapp hello_app`, and the code of the view functions, routes and the `Note` model taken from the sections of this step; two notes were created with `Note.objects.create(...)` in `python manage.py shell`. Your HTML at `/notes/` will differ depending on your template. For an anonymous user, `/admin/` returns a `302` to the login page; after you log in as a superuser, you see the admin panel. The order "second, then first" comes from `Meta.ordering = ['-created_at']`.

??? note "All the step's commands in order"

    ```bash
    # 1. Create and activate venv
    python -m venv venv
    source venv/bin/activate      # Linux/Mac
    # or: venv\Scripts\activate  # Windows

    # 2. Install Django
    pip install -r requirements.txt
    # or: pip install django

    # 3. Create the project (the dot matters!)
    django-admin startproject hello_project .

    # 4. Apply the initial migrations
    python manage.py migrate

    # 5. Create an administrator
    python manage.py createsuperuser

    # 6. Check the admin panel
    python manage.py runserver
    # → http://localhost:8000/admin/

    # 7. Create an app
    python manage.py startapp hello_app

    # 8. Register hello_app in settings.py
    # (add 'hello_app' to INSTALLED_APPS)

    # 9. Write view functions (views.py) and URL routes (urls.py)

    # 10. Connect hello_app.urls to hello_project/urls.py (include)

    # 11. Describe the model in models.py
    # 12. Create a migration
    python manage.py makemigrations

    # 13. Apply the migration
    python manage.py migrate

    # 14. Register the model in admin.py (@admin.register)

    # 15. Final check
    python manage.py runserver
    # http://localhost:8000/        → Hello, Django!
    # http://localhost:8000/about/  → text
    # http://localhost:8000/notes/  → list of notes
    # http://localhost:8000/admin/  → admin panel
    ```

## Checklist { #checklist }

- [ ] `python manage.py check` → `System check identified no issues (0 silenced).`
- [ ] `http://localhost:8000/` → `200` and the text `Hello, Django!`.
- [ ] `http://localhost:8000/about/` → `200` and the text `Це моя перша сторінка на Django!`.
- [ ] `http://localhost:8000/notes/` → `200`, an HTML list of notes from the template `hello_app/templates/hello_app/note_list.html` (the doubled folder is correct).
- [ ] `http://localhost:8000/admin/` without logging in → `302` to `/admin/login/?next=/admin/`; after you log in as a superuser, the `Note` model is visible.
- [ ] `python manage.py showmigrations hello_app` → `[X] 0001_initial`.
- [ ] CRUD works in `python manage.py shell`: `Note.objects.create(...)`, `filter`, `get(pk=2)`, `.save()`, `.delete()` — and the change is visible in the admin panel.
- [ ] `reverse('hello_app:about')` → `'/about/'`.
- [ ] You can explain the difference between `startproject` and `startapp`, and why `makemigrations` and `migrate` are two separate steps.
- [ ] You can explain how Django finds a view by URL (the URLconf chain: `hello_project/urls.py` → `include('hello_app.urls')` → `path(...)`) and how superuser / staff / regular user differ.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `/notes/` → `500`, and the terminal shows `django.db.utils.OperationalError: no such table: hello_app_note` | the model is described, but `makemigrations` / `migrate` were not run | `python manage.py makemigrations`, then `python manage.py migrate` |
| `OperationalError: no such table: auth_user` on `createsuperuser` or when logging in to `/admin/` | the initial migrations were not applied | `python manage.py migrate` |
| `django.core.exceptions.ImproperlyConfigured: Specifying a namespace in include() without providing an app_name is not supported.` | `namespace=` is set in `include()`, but `app_name` is missing from `hello_app/urls.py` | add `app_name = "hello_app"` to `hello_app/urls.py` |
| `django.urls.exceptions.NoReverseMatch: Reverse for 'about' not found. 'about' is not a valid view function or pattern name.` | the route name is used without the namespace | `reverse('hello_app:about')`, `{% url 'hello_app:about' %}` |
| `ModuleNotFoundError: No module named 'django'` | the venv is not activated or Django is not installed | activate the venv → `pip install -r requirements.txt` |
| `django-admin: command not found` | the same: Django is not installed or the venv is not activated | activate the venv, then `pip install django` |
| `AttributeError: 'super' object has no attribute 'dicts'` | Python 3.14 is incompatible with Django 5.1 | `pip install django --upgrade` (installs 5.2+) |
| `Page not found (404)` at `/` | the route is not registered | check `urls.py` in both files |
| `NameError: name 'Note' is not defined` | `views.py` does not import the model | `from .models import Note` in `views.py` |
| Django "does not see" `hello_app`: the model is missing from the admin, `makemigrations` says `No changes detected` | the app is not added to `INSTALLED_APPS` | add `'hello_app'` to `settings.py` |
| `ModuleNotFoundError: No module named 'hello_project'` | you are not running from the project root | go to the folder that contains `manage.py` |
| `DisallowedHost at /` | `ALLOWED_HOSTS` does not contain the host | add the host to `ALLOWED_HOSTS` in `settings.py` |
| `Forbidden (403) CSRF verification failed` | `{% csrf_token %}` is missing from the form | add `{% csrf_token %}` inside the `<form>` |
| The server does not stop | — | `Ctrl + C` in the terminal |

## Practice { #practice }

### Worked example: a route with a parameter `/greet/<str:name>/` { #worked-example }

Task: the page `/greet/Olena/` must respond with `Привіт, Olena!`.

1. **View.** The parameter from the URL arrives in the function as a keyword argument:

    ```python title="hello_app/views.py (excerpt)"
    from django.utils.html import escape

    def greet(request, name):
        return HttpResponse(f"Привіт, {escape(name)}!")
    ```

2. **Route.** The `str` converter takes everything up to the next `/`:

    ```python title="hello_app/urls.py (excerpt)"
    path('greet/<str:name>/', views.greet, name='greet'),
    ```

3. **Result** (Django test client, the same project):

    ```text
    /greet/Olena/ 200 Привіт, Olena!
    /greet/%3Cimg%20src=x%20onerror=alert(1)%3E/ 200 Привіт, &lt;img src=x onerror=alert(1)&gt;!
    /greet/ 404
    /greet/a/b/ 404
    reverse('hello_app:greet', args=['Taras']) → /greet/Taras/
    ```

    `/greet/` gives `404` because `str` does not accept an empty string; `/greet/a/b/` gives `404` because `str` does not match `/` (for paths with slashes there is the `path` converter).

    !!! warning "Why `escape(name)`"
        `name` comes from the URL, so anyone can write it. `HttpResponse` returns the string as HTML (`text/html`) and escapes nothing. Without `escape()` the same URL would return `Привіт, <img src=x onerror=alert(1)>!` and the browser would run someone else's JavaScript (reflected XSS). `escape()` turns `<` and `>` into `&lt;` and `&gt;`, so the browser shows them as text. Django templates escape variables on their own, so in step 2, where the response is built with `render()`, `escape()` is no longer needed.

### Modify the example { #modify }

1. Replace `<str:name>` with `<int:age>` and return `"Тобі {age} років"`. What will `/greet/abc/` return?
2. Add a `/contact/` route with a `contact()` view function that returns a string with your name and email.

### Try it yourself: the model and the shell { #try-yourself }

- Create the `Note` model, make a migration, open the admin panel and add 3 notes by hand.
- In the Django shell, find the note with `id=2`, change its `title` and save it. Check it in the admin panel.
- (Bonus) Add the field `is_pinned = models.BooleanField(default=False)` to the `Note` model, make a new migration and make sure the field has appeared in the admin panel.

**Success criteria:** `showmigrations hello_app` shows two applied migrations; `python manage.py check` reports no errors; the change made in the shell is visible at `/admin/hello_app/note/`.

## Summary { #summary }

| Concept | Where in the code |
|---|---|
| URL route | `hello_app/urls.py` → `path('about/', views.about)` |
| View function | `hello_app/views.py` → `def about(request): return HttpResponse(...)` |
| Connecting app URLs | `hello_project/urls.py` → `include('hello_app.urls')` |
| Namespace | `app_name = 'hello_app'` + `namespace='hello_app'` |
| Reverse URL | `reverse('hello_app:about')` or `{% url 'hello_app:about' %}` |
| Path converter | `path('notes/<int:pk>/', ...)` |
| Template | `hello_app/templates/hello_app/note_list.html` |
| Context | `render(request, 'шаблон.html', {'notes': queryset})` |
| Model → DB | `makemigrations` + `migrate` |
| ORM CRUD | `create`, `filter`, `get`, `.save()`, `.delete()` |
| Admin registration | `@admin.register(Note)` in `admin.py` |
| MVT | Model → View → Template |

What changed in the files:

| File | What you did |
|---|---|
| `hello_project/settings.py` | Added `'hello_app'` to `INSTALLED_APPS` |
| `hello_app/models.py` | Wrote the `Note` model (CharField, TextField, DateTimeField) |
| `hello_app/admin.py` | Registered `NoteAdmin` with `@admin.register` |
| `hello_app/migrations/0001_initial.py` | Generated automatically by `makemigrations` |
| `hello_app/views.py` | Wrote the functions `index()`, `about()`, `note_list()` |
| `hello_app/urls.py` | Created the file with `app_name` and the routes `/, /about/, /notes/` |
| `hello_project/urls.py` | Connected the routes with `include()` |
| `hello_app/templates/hello_app/note_list.html` | Your first HTML template |

## Self-check { #self-check }

1. How is a project (`startproject`) different from an app (`startapp`)?
2. Why are there two commands, `makemigrations` and `migrate`?
3. Why does `/notes/` fail with `no such table` while `/` and `/about/` work?
4. What happens if you remove `app_name` from `hello_app/urls.py` but keep `namespace=` in `include()`?
5. Why does the template live in `hello_app/templates/hello_app/` and not just in `hello_app/templates/`?
6. What does `render()` take and what does it return?

??? success "Answers"

    1. A project is a package of settings (`settings.py`, the root `urls.py`, `wsgi.py`/`asgi.py`) plus `manage.py`; an app is a module with its own models, views and URLs that you plug in through `INSTALLED_APPS`. One project, many apps.
    2. `makemigrations` compares the models with the previous migrations and writes a new migration file (a plan of changes). `migrate` runs these plans against the database. In between, you can read the file (`sqlmigrate`) and commit it to git.
    3. `/` and `/about/` return an `HttpResponse` without touching the database. `note_list` runs `Note.objects.all()`, which is a `SELECT` from the `hello_app_note` table, and that table does not exist yet.
    4. The project does not start: `ImproperlyConfigured: Specifying a namespace in include() without providing an app_name is not supported.` (verified with `python manage.py check`).
    5. Django collects `templates/` from all apps into one namespace. The doubled folder makes the name `hello_app/note_list.html` unique; otherwise two apps with a `note_list.html` would conflict.
    6. `request`, the template path and a `context` dict; it returns an `HttpResponse` with the rendered HTML (status `200`).

## What's next { #next }

[Step 2. Bootstrap Notes](../02_first_model/index.md) — Bootstrap 5, ModelForm, full CRUD with the PRG pattern.
