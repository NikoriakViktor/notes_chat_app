# Контрольна точка: крок 4

## Перевір, що крок робочий { #check }

У кроці ти будуєш `crispy_notes` (додаток `hello_app`). Ті самі шаблони, форми й context processor живуть у цьому репозиторії — у `templates/` і `notes_app/`, тож звіряйся з ними. Реальний вивід на цьому репозиторії (PostgreSQL 16, `DATABASE_URL` задано):

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

22 тести форм (`TagFormTest`, `NoteFormSecurityTest`, `NoteFormValidationTest`, `GroupCreateFormTest`) + 9 тестів створення й редагування нотатки через view.

Що бачить залогінений користувач (Django test client, `demo_alice` з `seed_demo_data`):

```text
GET /notes/       200   templates: ['notes_app/note_list.html', 'layouts/dashboard.html', 'base.html']
                        у context є sidebar_notebooks: True, sidebar_tags: True
GET /notes/new/   200   поля з class="form-control": True, csrfmiddlewaretoken: True
```

Перший рядок — це і є 3 рівні наслідування: сторінка → layout → `base.html`. Sidebar-дані з'явились без жодного рядка у view — їх додав `notes_app.context_processors.sidebar_context`.

??? note "Структура файлів після кроку 4"

    Так виглядає навчальний проєкт кроку (`hello_app`). У цьому репозиторії той самий розклад — під іменем `notes_app/` і з тими самими глобальними `templates/`.

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
    │   ├── services.py               ← create/update/delete для всіх моделей
    │   ├── selectors.py              ← get_user_notes, get_user_notebooks, get_user_tags...
    │   ├── context_processors.py     ← sidebar_context → sidebar_notebooks, sidebar_tags,
    │   │                                sidebar_todo_count, sidebar_shopping_count (+ guard від помилок БД)
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
    │       ├── note_detail.html          ← з inline ReminderForm
    │       ├── note_form.html            ← Tier 3: {% crispy form %}  ★
    │       ├── note_confirm_delete.html
    │       ├── notebook_list.html, notebook_form.html, tag_form.html
    │       ├── todo_list.html, todo_detail.html, todo_form.html, todo_confirm_delete.html
    │       ├── shopping_list.html, shopping_detail.html, shopping_form.html, shopping_confirm_delete.html
    │       └── share_form.html           ← ShareForm (додати юзера до shared_with)
    │
    └── templates/                    ← ГЛОБАЛЬНІ шаблони (DIRS в settings)
        ├── base.html                      ← Рівень 1: HTML5 + Bootstrap CDN
        ├── layouts/
        │   └── dashboard.html             ← Рівень 2: Sidebar + Topbar
        ├── registration/
        │   ├── login.html
        │   └── register.html
        └── components/
            ├── pagination.html
            ├── empty_state.html
            └── confirm_modal.html
    ```

## Чекліст { #checklist }

- [ ] `GET /notes/` використовує 3 шаблони: `notes_app/note_list.html` → `layouts/dashboard.html` → `base.html` (`response.templates` у тесті або Debug Toolbar → Templates).
- [ ] `{% block body %}` визначено в `base.html`, `{% block content %}` — у `layouts/dashboard.html`; сторінка заповнює тільки `content`.
- [ ] `TEMPLATES["DIRS"] = [BASE_DIR / "templates"]` — глобальні шаблони знаходяться; `APP_DIRS = True` — шаблони додатку теж.
- [ ] `"notes_app.context_processors.sidebar_context"` є в `TEMPLATES["OPTIONS"]["context_processors"]`; у context кожної сторінки є `sidebar_notebooks`, `sidebar_tags`, `sidebar_todo_count`, `sidebar_shopping_count`, `sidebar_groups`.
- [ ] `CRISPY_TEMPLATE_PACK = "bootstrap5"`; `{% crispy form %}` на `/notes/new/` рендерить поля з `class="form-control"` і `csrfmiddlewaretoken`.
- [ ] `TodoItemForm` і `ShopItemForm` мають `helper.form_tag = False` — `<form>` пише шаблон, а не crispy.
- [ ] Active state у sidebar — через `request.resolver_match.url_name` (потрібен `django.template.context_processors.request`).
- [ ] Logout — `<form method="post" action="{% url 'logout' %}">` у `dashboard.html`; `GET /accounts/logout/` → `405`.
- [ ] `python manage.py test notes_app.tests.test_forms` → `Ran 22 tests`, `OK`.
- [ ] Можеш пояснити, чим `{{ block.super }}` відрізняється від повного перевизначення блоку.
- [ ] Можеш пояснити, чому context processor загортає запити до БД у `try/except` і що побачить користувач, якщо таблиці ще немає.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `TemplateDoesNotExist: base.html` | `base.html` не там, де Django шукає | файл у `templates/base.html` + `DIRS = [BASE_DIR / "templates"]`, або в `<app>/templates/base.html` |
| `TemplateDoesNotExist: layouts/dashboard.html` | `DIRS` порожній — шукаються лише папки додатків | `"DIRS": [BASE_DIR / "templates"]` |
| `NoReverseMatch: Reverse for 'note_create' not found` | URL name не зареєстровано або не той namespace | перевір `urls.py` і `app_name` |
| `Forbidden (403) CSRF verification failed` після POST | відсутній `{% csrf_token %}` у формі | `{% csrf_token %}` всередину `<form>` (з `{% crispy form %}` і `form_tag=True` crispy додає його сам) |
| Crispy не застосовує Bootstrap стилі | не вказано `CRISPY_TEMPLATE_PACK` або немає `crispy_bootstrap5` в `INSTALLED_APPS` | `CRISPY_TEMPLATE_PACK = "bootstrap5"` + `"crispy_bootstrap5"` |
| Дві вкладені `<form>`, кнопка inline-форми відправляє не те | crispy генерує `<form>`, а шаблон теж її пише | `self.helper.form_tag = False` |
| Sidebar порожній | context processor не зареєстрований | перевір `TEMPLATES["OPTIONS"]["context_processors"]` |
| `AppRegistryNotReady` | моделі імпортуються до завершення ініціалізації Django (напр. у `settings.py` або `<app>/__init__.py`) | прибери імпорт моделей з рівня модуля в цих файлах; порядок `crispy_forms` / додатку в `INSTALLED_APPS` цю помилку не викликає |
| Active state не спрацьовує | `request` немає в context | додай `django.template.context_processors.request` |
| Debug Toolbar не з'являється | `DEBUG` вимкнено, IP не в `INTERNAL_IPS` або в шаблоні немає `</body>` | `DJANGO_DEBUG=1`, `INTERNAL_IPS = ["127.0.0.1"]`, перевір `<body>` |
| Cards різної висоти | немає `h-100` на `.card` | додай `h-100` до `<div class="card">` |
| Футер посередині сторінки | відсутні Sticky Footer класи | `d-flex flex-column min-vh-100` на `<body>` |

## Практика { #practice }

### Розібраний приклад: звідки на сторінці sidebar { #worked-example }

`note_list` нічого не знає про записники в sidebar — передає в шаблон лише нотатки. Як вони потрапляють на сторінку?

1. **Ланцюг шаблонів.** `notes_app/note_list.html` починається з `{% extends 'layouts/dashboard.html' %}`; `dashboard.html` — з `{% extends 'base.html' %}` і заповнює `{% block body %}` sidebar'ом і topbar'ом, залишаючи `{% block content %}` сторінці. Реальний ланцюг для `GET /notes/`:

    ```text
    ['notes_app/note_list.html', 'layouts/dashboard.html', 'base.html']
    ```

2. **Context processor.** Функція викликається на кожен `render()` з `RequestContext`:

    ```python title="notes_app/context_processors.py (фрагмент)"
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

3. **Захист від падіння.** Запити до БД загорнуті в `try/except Exception`: якщо таблиці ще немає (свіжий проєкт до `migrate`) чи з'єднання впало, sidebar буде порожнім, але сторінка відкриється.

4. **Active state.** `dashboard.html` підсвічує пункт меню через `request.resolver_match.url_name == 'note_list'` — тому потрібен `django.template.context_processors.request`.

### Зміни приклад { #modify }

1. Додай у sidebar пункт «Архів» (`/notes/?archived=1`) і підсвічуй його лише тоді, коли в `request.GET` є `archived`.
2. Перепиши одну форму з ручного Bootstrap HTML на Crispy `FormHelper` + `Layout(Row(Column(...)), Submit(...))`.

### Спробуй самостійно: лічильник «сьогодні» { #try-yourself }

Додай у `sidebar_context` ключ `sidebar_today_count` — кількість нотаток поточного користувача, створених сьогодні (`created_at__date=timezone.localdate()`), і покажи його badge'ем біля «Нотатки».

**Критерії перевірки:** анонімний користувач отримує `0` без запитів до БД; після створення нотатки число зростає на 1; `python manage.py test notes_app.tests.test_views` проходить; Debug Toolbar показує +1 SQL-запит на сторінку, не більше.

## Підсумок { #summary }

| Концепція | Де у коді |
|-----------|-----------|
| Template Inheritance | `base.html` → `layouts/dashboard.html` → `note_list.html` |
| `{% block %}` | `{% block content %}{% endblock %}` у батьку + той самий блок у дочірньому |
| `{{ block.super }}` | додає вміст батьківського блоку замість заміни |
| FormHelper | `self.helper = FormHelper()` у `__init__` форми в `forms.py` |
| Layout | `Layout(Fieldset(...), Row(Column(...)), Submit(...))` |
| `{% crispy form %}` | у шаблоні — 1 рядок замість ручної розмітки кожного поля |
| Context Processor | `notes_app/context_processors.py` → `sidebar_context(request)` |
| Реєстрація процесора | `settings.py` → `TEMPLATES` → `OPTIONS` → `context_processors` |
| Active nav | `request.resolver_match.url_name == 'note_list'` |
| Static files | `{% load static %}{% static '...' %}`; `STATICFILES_DIRS = [BASE_DIR / "static"]` |
| `form_tag = False` | `TodoItemForm`, `ShopItemForm` (inline forms) |
| Guard від помилок БД | `context_processors.py` — `try/except` навколо запитів |
| M2M `shared_with` | `TodoList.shared_with`, `ShoppingList.shared_with` + міграція `0002` |
| Django Messages | блок `{% if messages %}` у layout + `MESSAGE_TAGS` у `settings.py` |
| Components | `templates/components/` + `{% include %}` у сторінках |

## Самоперевірка { #self-check }

1. Скільки шаблонів рендерить Django для `GET /notes/` і в якому порядку їх «збирає»?
2. Чому `sidebar_notebooks` доступні в шаблоні, хоча view їх не передає?
3. Навіщо `form_tag = False` для `TodoItemForm`?
4. Чому logout — POST-форма, а не посилання?
5. Що буде зі сторінкою, якщо прибрати `try/except` з `sidebar_context` і запустити проєкт до `migrate`?

??? success "Відповіді"

    1. Три: `notes_app/note_list.html`, `layouts/dashboard.html`, `base.html`. Django починає з дочірнього, бачить `{% extends %}`, підіймається до кореня і підставляє блоки дочірніх шаблонів у батьківські.
    2. `render()` будує `RequestContext`, який викликає всі функції з `context_processors` і додає їхні словники в context.
    3. Форма додавання пункту стоїть усередині сторінки, яка сама пише `<form ... action="...">`. З `form_tag=True` crispy згенерував би ще одну `<form>` — вкладені форми HTML не дозволяє.
    4. Вихід змінює стан (видаляє сесію). GET-посилання можна «натиснути» з чужого сайту картинкою; POST вимагає CSRF-токен. З Django 5.0 `LogoutView` взагалі не приймає GET — перевірено: `GET /accounts/logout/` → `405`.
    5. Кожна сторінка з sidebar падала б з `ProgrammingError` / `OperationalError` (таблиці немає) — навіть сторінка логіну, бо процесор викликається на кожен `render()`.

## Що далі { #next }

[Крок 5. Auth і безпека](../05_auth_and_security/index.md) — Login/Logout, сесії, Password Reset, IDOR, групи.
