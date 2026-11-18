# Checkpoint: step 2

## Check that the step works { #check }

From the project folder, with the venv activated:

```text
$ python manage.py check
System check identified no issues (0 silenced).
$ python manage.py showmigrations hello_app
hello_app
 [X] 0001_initial
```

Then go through the whole CRUD. Here is what the application responds at each step (Django test client, no browser):

```text
GET /                        302 /notes/
GET /notes/                  200   (has row-cols-lg-3 and h-100: the Bootstrap Card Grid)
GET /notes/new/              200   (the form has csrfmiddlewaretoken)
POST /notes/new/ title=""    200   {'title': [{'message': "Це поле обов'язкове.", 'code': 'required'}]}
POST /notes/new/ title="ab"  200   Заголовок має бути мінімум 3 символи.
POST /notes/new/ "Купити молоко"   302 /notes/   notes: 2 -> 3
GET /notes/                  messages: ['Нотатку "Купити молоко" успішно створено!']
GET /notes/  (F5)            messages: []
GET /notes/3/                200
POST /notes/3/edit/          302 /notes/3/   same pk: True, notes: 3
GET /notes/3/delete/         200   (confirmation page, the note still exists)
GET /notes/999/              404
POST without a CSRF token    403
POST /notes/3/delete/        302 /notes/   note exists: False
GET /notes/                  messages: [('warning', 'Нотатку "Купити молоко і хліб" видалено.')]
```

Verified with Django 5.2.19, `django-bootstrap5` and `LANGUAGE_CODE = 'uk'` on the code of `forms.py`, `views.py`, `urls.py` and the templates from the section [ModelForm and CRUD](modelform_and_crud.md). The validation error messages are in Ukrainian because of `LANGUAGE_CODE = 'uk'`; with `en-us` you get `This field is required.`

### What to check in the browser

| URL | What to check |
|-----|---------------|
| `http://localhost:8000/` | Redirect to `/notes/` |
| `http://localhost:8000/notes/` | A list of Bootstrap Cards |
| `http://localhost:8000/notes/new/` | The create form with Bootstrap styles |
| `http://localhost:8000/notes/1/` | Note detail |
| `http://localhost:8000/notes/1/edit/` | The edit form (filled with data) |
| `http://localhost:8000/notes/1/delete/` | The delete confirmation page |
| `http://localhost:8000/admin/` | The admin panel with the custom NoteAdmin |

1. List: the Cards have equal height (`h-100`).
2. Mobile (DevTools → Toggle Device, Ctrl+Shift+M): 1 column, hamburger Navbar.
3. Tablet: 2 columns; desktop: 3 columns.
4. After creating a note → a `success` Flash message with the note's title.
5. F5 after POST → you stay on `/notes/`, no duplicate appears (PRG works).

??? note "All the step's commands in order"

    ```bash
    # 1. Activate venv
    source venv/bin/activate          # Linux/Mac
    # or: venv\Scripts\activate      # Windows

    # 2. Install dependencies
    pip install -r requirements.txt

    # 3. Apply migrations
    python manage.py migrate

    # 4. Create an administrator
    python manage.py createsuperuser

    # 5. Start the server
    python manage.py runserver
    # → http://localhost:8000/notes/

    # If you changed models.py:
    python manage.py makemigrations
    python manage.py migrate
    ```

## Checklist { #checklist }

- [ ] `GET /` → `302` to `/notes/`.
- [ ] `POST /notes/new/` with an empty `title` → `200`, the form with the error `Це поле обов'язкове.`; with `title="ab"` → `Заголовок має бути мінімум 3 символи.` (`clean_title`).
- [ ] A valid `POST /notes/new/` → `302` to `/notes/`, the number of notes +1, a `success` Flash message; after F5 the message disappears and there is no duplicate.
- [ ] `POST /notes/<pk>/edit/` → `302` to `/notes/<pk>/`, the `pk` has not changed, the number of notes is the same (UPDATE, not INSERT).
- [ ] `GET /notes/<pk>/delete/` only shows the confirmation; only `POST` deletes → `302` to `/notes/` and a `warning` message.
- [ ] `GET /notes/999/` → `404`, not `500` (`get_object_or_404`).
- [ ] `POST` without `{% csrf_token %}` → `403`.
- [ ] The templates inherit `base.html` via `{% extends %}`; Bootstrap is included via CDN in `base.html`; Sticky Footer: `body.d-flex.flex-column.min-vh-100` + `main.flex-grow-1`.
- [ ] You can explain what PRG is and why F5 after PRG is safe.
- [ ] You can explain why `NoteForm(request.POST, instance=note)` does an UPDATE, while `NoteForm(request.POST)` does an INSERT.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `TemplateDoesNotExist: base.html` | `base.html` is not where Django looks for it | the file must be at `hello_app/templates/base.html` (not in `hello_app/templates/hello_app/`) |
| `NoReverseMatch: Reverse for 'note_create' not found` | the URL name is not registered or the namespace is wrong | check `hello_app/urls.py` and `app_name = 'hello_app'` |
| `NoReverseMatch` for `'note_list'` in a template | the namespace is not specified | `{% url 'hello_app:note_list' %}` (with a colon) |
| `Forbidden (403) CSRF verification failed` after POST | `{% csrf_token %}` is missing from the form | add `{% csrf_token %}` inside the `<form>` |
| Bootstrap styles are not applied to the fields | `class='form-control'` is not added to the widget | check `NoteForm.Meta.widgets` in `forms.py` |
| `IntegrityError` on save | the object was saved bypassing `form.is_valid()` | save only after `form.is_valid()` |
| A duplicate after F5 | there is no redirect after POST | PRG: always `return redirect(...)` after `form.save()` |
| Editing creates a new note | `note_edit` has no `instance=note` | `NoteForm(request.POST, instance=note)` |
| Cards of different heights | no `h-100` on `.card` | add `h-100` to `<div class="card">` |
| The footer is in the middle of the page | no Flexbox on body | `d-flex flex-column min-vh-100` on `<body>` + `flex-grow-1` on `<main>` |
| The buttons on a card cannot be clicked (there is a `stretched-link`) | `stretched-link` covers the buttons | add `position-relative z-1` to the `btn-group` |
| Flash messages do not appear | `MESSAGE_TAGS` is not configured or `base.html` has no messages block | check `settings.py` → `MESSAGE_TAGS` and `base.html` → `{% if messages %}` |
| `ImproperlyConfigured: Specifying a namespace in include() without providing an app_name` | `app_name` is missing from `urls.py` | add `app_name = 'hello_app'` |

## Practice { #practice }

### Worked example: the `priority` field { #worked-example }

Task: notes get a priority — Low / Medium / High.

1. **Model.** `choices` restricts the values and gives you `get_priority_display()`:

    ```python title="hello_app/models.py (excerpt)"
    PRIORITY_CHOICES = [(1, 'Низький'), (2, 'Середній'), (3, 'Високий')]
    priority = models.PositiveSmallIntegerField(
        choices=PRIORITY_CHOICES, default=2, verbose_name='Пріоритет'
    )
    ```

2. **Migration.**

    ```text
    $ python manage.py makemigrations hello_app
    Migrations for 'hello_app':
      hello_app/migrations/0002_note_priority.py
        + Add field priority to note
    $ python manage.py migrate hello_app
      Applying hello_app.0002_note_priority... OK
    ```

    `default=2` is needed because the table already has rows: without it, `makemigrations` will ask which value to give the existing notes.

3. **Form.** `fields = ['title', 'content', 'priority']` — ModelForm builds the `<select>` itself.

4. **Result:**

    ```text
    GET /notes/new/                   the form has <select name="priority">
    POST title="Пріоритетна" priority=3   302 → get_priority_display() = 'Високий'
    POST title="Погана" priority=9        200 → Зробить коректний вибір, 9 немає серед варіантів вибору.
    ```

    The value `9` was rejected by the form — it never reached the database. The error text is Django's standard `uk` translation (exactly like that, with «Зробить»).

### Modify the example { #modify }

1. Show the priority on the note detail page: `{{ note.get_priority_display }}`.
2. Sort the list so that important notes come first: `Note.objects.order_by('-priority', '-created_at')`.

### Try it yourself: a priority Badge { #try-yourself }

In `note_list.html`, show a Bootstrap Badge with the priority on each card: `bg-secondary` for 1, `bg-warning` for 2, `bg-danger` for 3.

**Success criteria:** after you create 3 notes with different priorities, the list shows three different colours; `python manage.py check` reports no errors; `showmigrations hello_app` → `[X] 0002_note_priority`.

## Summary { #summary }

| Concept | Where in the code |
|-----------|-----------|
| ModelForm | `hello_app/forms.py` → `class NoteForm(forms.ModelForm)` |
| Field validation | `NoteForm.clean_title()` → `ValidationError` |
| PRG pattern | `note_create`: `return redirect(...)` after `form.save()` |
| UPDATE instead of INSERT | `NoteForm(request.POST, instance=note)` |
| Messages | `messages.success(request, ...)` + `{% if messages %}` in `base.html` |
| Template Inheritance | `{% extends 'base.html' %}` + `{% block content %}` |
| CRUD views | `note_list`, `note_detail`, `note_create`, `note_edit`, `note_delete` |
| 404 instead of 500 | `get_object_or_404(Note, pk=pk)` |
| Bootstrap Grid | `row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4` |
| Sticky Footer | `body.d-flex.flex-column.min-vh-100` + `main.flex-grow-1` |
| Protecting DELETE | POST, not GET + `{% csrf_token %}` |

What changed in the files:

| File | What you did |
|------|-------------|
| `hello_project/settings.py` | `django_bootstrap5` in INSTALLED_APPS, `MESSAGE_TAGS` |
| `hello_app/forms.py` | `NoteForm(ModelForm)` — NEW |
| `hello_app/views.py` | 6 views: `index`, `note_list`, `note_detail`, `note_create`, `note_edit`, `note_delete` |
| `hello_app/urls.py` | 6 URL routes |
| `hello_app/admin.py` | `NoteAdmin` with `list_display`, `search_fields`, `@admin.display` |
| `hello_app/templates/base.html` | Bootstrap CDN, Navbar, Messages, Footer — NEW |
| `hello_app/templates/hello_app/note_list.html` | Bootstrap Card Grid — REWRITTEN |
| `hello_app/templates/hello_app/note_detail.html` | Detail + Breadcrumb — NEW |
| `hello_app/templates/hello_app/note_form.html` | Bootstrap Form — NEW |
| `hello_app/templates/hello_app/note_confirm_delete.html` | Deletion — NEW |

## Self-check { #self-check }

1. What does `POST /notes/new/` with a validation error return, and why not a redirect?
2. Why is deletion done only via POST?
3. Where does `form.errors['title']` come from when `title="ab"`?
4. Why does the Flash message disappear after F5?
5. What changes if you remove `instance=note` from `note_edit`?

??? success "Answers"

    1. `200` with the same form: it already contains the entered data and the errors. A redirect would lose them. Redirect only after a successful save.
    2. Anyone and anything can make a GET request: a link, an `<img src>`, browser prefetching. POST requires a form with a CSRF token — without it you get `403` (verified).
    3. First the field's standard validators run (`required`, `max_length`), then `clean_title()`: it strips whitespace and raises `ValidationError` if there are fewer than 3 characters.
    4. A message is stored until it is read for the first time: the template loop `{% for message in messages %}` "consumes" it. The second `GET /notes/` gets an empty list (verified: `[]`).
    5. A form without `instance` means a new note: `form.save()` will do an INSERT, and instead of an update you get a copy.

## What's next { #next }

[Step 3. CRUD and architecture](../03_crud_and_architecture/index.md) — `notes_project`: Selectors, Services, ER diagrams, PostgreSQL, CBV.
