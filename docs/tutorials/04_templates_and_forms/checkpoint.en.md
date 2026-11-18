# Checkpoint: step 4

## Check that the step works { #check }

In this step you build `crispy_notes` (the `hello_app` app). The same templates, forms and context processor live in this repository, in `templates/` and `notes_app/`, so compare your work against them. Real output on this repository (PostgreSQL 16, `DATABASE_URL` set):

```text
$ python manage.py check
System check identified no issues (0 silenced).
$ python manage.py test notes_app.tests.test_forms notes_app.tests.test_views.NoteCreateViewTest notes_app.tests.test_views.NoteEditViewTest
Found 31 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
...............................
----------------------------------------------------------------------
Ran 31 tests in 14.984s

OK
Destroying test database for alias 'default'...
```

22 form tests (`TagFormTest`, `NoteFormSecurityTest`, `NoteFormValidationTest`, `GroupCreateFormTest`) + 9 tests that create and edit a note through the view.

What a logged-in user sees (Django test client, `demo_alice` from `seed_demo_data`):

```text
GET /notes/       200   templates: ['notes_app/note_list.html', 'layouts/dashboard.html', 'base.html']
                        context has sidebar_notebooks: True, sidebar_tags: True
GET /notes/new/   200   fields with class="form-control": True, csrfmiddlewaretoken: True
```

The first line is exactly the 3 levels of inheritance: page → layout → `base.html`. The sidebar data appeared without a single line in the view: `notes_app.context_processors.sidebar_context` added it.

??? note "File structure after step 4"

    This is what the step's tutorial project (`hello_app`) looks like. This repository has the same layout, under the name `notes_app/` and with the same global `templates/`.

    ```
    crispy_notes_project/
    │
    ├── requirements.txt              ← django-crispy-forms, crispy-bootstrap5, debug-toolbar
    │
    ├── hello_project/
    │   ├── settings.py               ← CRISPY_TEMPLATE_PACK, context_processors, MESSAGE_TAGS
    │   └── urls.py                   ← accounts/ (login/logout) + hello_app/ + debug_toolbar
    │
    ├── hello_app/
    │   ├── models.py                 ← Note, Notebook, Tag, TodoList, TodoItem,
    │   │                                ShoppingList, ShopItem, Reminder
    │   ├── services.py               ← create/update/delete for all models
    │   ├── selectors.py              ← get_user_notes, get_user_notebooks, get_user_tags...
    │   ├── context_processors.py     ← sidebar_context → sidebar_notebooks, sidebar_tags,
    │   │                                sidebar_todo_count, sidebar_shopping_count (+ guard against DB errors)
    │   ├── forms.py                  ← ★ NoteForm, NotebookForm, TagForm,
    │   │                                TodoListForm, TodoItemForm (form_tag=False),
    │   │                                ShoppingListForm, ShopItemForm (form_tag=False),
    │   │                                ReminderForm, ShareForm
    │   ├── views.py                  ← note CRUD + todo CRUD + shopping CRUD + reminders + register
    │   ├── urls.py
    │   ├── migrations/
    │   │   ├── 0001_initial.py
    │   │   └── 0002_shoppinglist_shared_with_todolist_shared_with.py  ← M2M shared_with
    │   └── templates/hello_app/
    │       ├── note_list.html            ← extends layouts/dashboard.html
    │       ├── note_detail.html          ← with inline ReminderForm
    │       ├── note_form.html            ← Tier 3: {% crispy form %}  ★
    │       ├── note_confirm_delete.html
    │       ├── notebook_list.html, notebook_form.html, tag_form.html
    │       ├── todo_list.html, todo_detail.html, todo_form.html, todo_confirm_delete.html
    │       ├── shopping_list.html, shopping_detail.html, shopping_form.html, shopping_confirm_delete.html
    │       └── share_form.html           ← ShareForm (add a user to shared_with)
    │
    └── templates/                    ← GLOBAL templates (DIRS in settings)
        ├── base.html                      ← Level 1: HTML5 + Bootstrap CDN
        ├── layouts/
        │   └── dashboard.html             ← Level 2: Sidebar + Topbar
        ├── registration/
        │   ├── login.html
        │   └── register.html
        └── components/
            ├── pagination.html
            ├── empty_state.html
            └── confirm_modal.html
    ```

## Checklist { #checklist }

- [ ] `GET /notes/` uses 3 templates: `notes_app/note_list.html` → `layouts/dashboard.html` → `base.html` (`response.templates` in a test, or Debug Toolbar → Templates).
- [ ] `{% block body %}` is defined in `base.html`, `{% block content %}` in `layouts/dashboard.html`; the page fills only `content`.
- [ ] `TEMPLATES["DIRS"] = [BASE_DIR / "templates"]`: global templates are found; `APP_DIRS = True`: app templates are found too.
- [ ] `"notes_app.context_processors.sidebar_context"` is in `TEMPLATES["OPTIONS"]["context_processors"]`; the context of every page has `sidebar_notebooks`, `sidebar_tags`, `sidebar_todo_count`, `sidebar_shopping_count`, `sidebar_groups`.
- [ ] `CRISPY_TEMPLATE_PACK = "bootstrap5"`; `{% crispy form %}` on `/notes/new/` renders fields with `class="form-control"` and `csrfmiddlewaretoken`.
- [ ] `TodoItemForm` and `ShopItemForm` have `helper.form_tag = False`: the template writes the `<form>`, not crispy.
- [ ] The active state in the sidebar comes from `request.resolver_match.url_name` (requires `django.template.context_processors.request`).
- [ ] Logout is `<form method="post" action="{% url 'logout' %}">` in `dashboard.html`; `GET /accounts/logout/` → `405`.
- [ ] `python manage.py test notes_app.tests.test_forms` → `Ran 22 tests`, `OK`.
- [ ] You can explain how `{{ block.super }}` differs from fully overriding a block.
- [ ] You can explain why the context processor wraps its DB queries in `try/except` and what the user sees if the table does not exist yet.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `TemplateDoesNotExist: base.html` | `base.html` is not where Django looks for it | put the file in `templates/base.html` + `DIRS = [BASE_DIR / "templates"]`, or in `<app>/templates/base.html` |
| `TemplateDoesNotExist: layouts/dashboard.html` | `DIRS` is empty, so only app folders are searched | `"DIRS": [BASE_DIR / "templates"]` |
| `NoReverseMatch: Reverse for 'note_create' not found` | the URL name is not registered or the namespace is wrong | check `urls.py` and `app_name` |
| `Forbidden (403) CSRF verification failed` after a POST | the form has no `{% csrf_token %}` | put `{% csrf_token %}` inside the `<form>` (with `{% crispy form %}` and `form_tag=True` crispy adds it itself) |
| Crispy does not apply Bootstrap styles | `CRISPY_TEMPLATE_PACK` is not set or `crispy_bootstrap5` is missing from `INSTALLED_APPS` | `CRISPY_TEMPLATE_PACK = "bootstrap5"` + `"crispy_bootstrap5"` |
| Two nested `<form>`s, the inline form's button submits the wrong thing | crispy generates a `<form>`, and the template writes one too | `self.helper.form_tag = False` |
| The sidebar is empty | the context processor is not registered | check `TEMPLATES["OPTIONS"]["context_processors"]` |
| `AppRegistryNotReady` | models are imported before Django finishes initializing (e.g. in `settings.py` or `<app>/__init__.py`) | remove module-level model imports from these files; the order of `crispy_forms` / the app in `INSTALLED_APPS` does not cause this error |
| The active state does not work | `request` is not in the context | add `django.template.context_processors.request` |
| Debug Toolbar does not appear | `DEBUG` is off, the IP is not in `INTERNAL_IPS`, or the template has no `</body>` | `DJANGO_DEBUG=1`, `INTERNAL_IPS = ["127.0.0.1"]`, check `<body>` |
| Cards have different heights | no `h-100` on `.card` | add `h-100` to `<div class="card">` |
| The footer sits in the middle of the page | Sticky Footer classes are missing | `d-flex flex-column min-vh-100` on `<body>` |

## Practice { #practice }

### Worked example: where the sidebar on the page comes from { #worked-example }

`note_list` knows nothing about the notebooks in the sidebar: it passes only notes to the template. How do they get onto the page?

1. **The template chain.** `notes_app/note_list.html` starts with `{% extends 'layouts/dashboard.html' %}`; `dashboard.html` starts with `{% extends 'base.html' %}` and fills `{% block body %}` with the sidebar and the topbar, leaving `{% block content %}` to the page. The real chain for `GET /notes/`:

    ```text
    ['notes_app/note_list.html', 'layouts/dashboard.html', 'base.html']
    ```

2. **The context processor.** The function is called on every `render()` with a `RequestContext`:

    ```python title="notes_app/context_processors.py (excerpt)"
    def sidebar_context(request):
        if not request.user.is_authenticated:
            return {
                'sidebar_notebooks': [],
                'sidebar_tags': [],
                'sidebar_todo_count': 0,
                'sidebar_shopping_count': 0,
                'sidebar_groups': [],
            }
        ...
        return {
            'sidebar_notebooks': notebooks,
            'sidebar_tags': tags,
            'sidebar_todo_count': todo_count,
            'sidebar_shopping_count': shopping_count,
            'sidebar_groups': sidebar_groups,
        }
    ```

3. **Protection against crashes.** The DB queries are wrapped in `try/except Exception`: if the table does not exist yet (a fresh project before `migrate`) or the connection dropped, the sidebar is empty, but the page still opens.

4. **Active state.** `dashboard.html` highlights a menu item with `request.resolver_match.url_name == 'note_list'`, which is why `django.template.context_processors.request` is required.

### Modify the example { #modify }

1. Add an «Архів» (Archive) item to the sidebar (`/notes/?archived=1`) and highlight it only when `request.GET` contains `archived`.
2. Rewrite one form from hand-written Bootstrap HTML to Crispy `FormHelper` + `Layout(Row(Column(...)), Submit(...))`.

### Try it yourself: a "today" counter { #try-yourself }

Add a `sidebar_today_count` key to `sidebar_context`: the number of the current user's notes created today (`created_at__date=timezone.localdate()`), and show it as a badge next to «Нотатки» (Notes).

**Success criteria:** an anonymous user gets `0` without any DB queries; after you create a note the number goes up by 1; `python manage.py test notes_app.tests.test_views` passes; Debug Toolbar shows +1 SQL query per page, no more.

## Summary { #summary }

| Concept | Where in the code |
|-----------|-----------|
| Template Inheritance | `base.html` → `layouts/dashboard.html` → `note_list.html` |
| `{% block %}` | `{% block content %}{% endblock %}` in the parent + the same block in the child |
| `{{ block.super }}` | adds the parent block's content instead of replacing it |
| FormHelper | `self.helper = FormHelper()` in the form's `__init__` in `forms.py` |
| Layout | `Layout(Fieldset(...), Row(Column(...)), Submit(...))` |
| `{% crispy form %}` | in the template: 1 line instead of hand-written markup for every field |
| Context Processor | `notes_app/context_processors.py` → `sidebar_context(request)` |
| Registering the processor | `settings.py` → `TEMPLATES` → `OPTIONS` → `context_processors` |
| Active nav | `request.resolver_match.url_name == 'note_list'` |
| Static files | `{% load static %}{% static '...' %}`; `STATICFILES_DIRS = [BASE_DIR / "static"]` |
| `form_tag = False` | `TodoItemForm`, `ShopItemForm` (inline forms) |
| Guard against DB errors | `context_processors.py`: `try/except` around the queries |
| M2M `shared_with` | `TodoList.shared_with`, `ShoppingList.shared_with` + migration `0002` |
| Django Messages | the `{% if messages %}` block in the layout + `MESSAGE_TAGS` in `settings.py` |
| Components | `templates/components/` + `{% include %}` in pages |

## Self-check { #self-check }

1. How many templates does Django render for `GET /notes/`, and in what order does it "assemble" them?
2. Why are `sidebar_notebooks` available in the template even though the view does not pass them?
3. Why does `TodoItemForm` need `form_tag = False`?
4. Why is logout a POST form and not a link?
5. What happens to the page if you remove the `try/except` from `sidebar_context` and start the project before `migrate`?

??? success "Answers"

    1. Three: `notes_app/note_list.html`, `layouts/dashboard.html`, `base.html`. Django starts with the child, sees `{% extends %}`, climbs up to the root and puts the child templates' blocks into the parents.
    2. `render()` builds a `RequestContext`, which calls every function from `context_processors` and adds their dictionaries to the context.
    3. The add-item form sits inside a page that writes its own `<form ... action="...">`. With `form_tag=True` crispy would generate one more `<form>`, and HTML does not allow nested forms.
    4. Logging out changes state (it deletes the session). A GET link can be "clicked" from another site with an image; a POST requires a CSRF token. Since Django 5.0 `LogoutView` does not accept GET at all. Verified: `GET /accounts/logout/` → `405`.
    5. Every page with the sidebar would crash with `ProgrammingError` / `OperationalError` (the table does not exist), even the login page, because the processor is called on every `render()`.

## What's next { #next }

[Step 5. Auth and security](../05_auth_and_security/index.md): Login/Logout, sessions, Password Reset, IDOR, groups.
