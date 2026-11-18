# Контрольна точка: крок 2

## Перевір, що крок робочий { #check }

З папки проєкту, з активованим venv:

```text
$ python manage.py check
System check identified no issues (0 silenced).
$ python manage.py showmigrations hello_app
hello_app
 [X] 0001_initial
```

Потім пройди весь CRUD. Ось що відповідає застосунок на кожен крок (Django test client, без браузера):

```text
GET /                        302 /notes/
GET /notes/                  200   (є row-cols-lg-3 і h-100 — Bootstrap Card Grid)
GET /notes/new/              200   (у формі є csrfmiddlewaretoken)
POST /notes/new/ title=""    200   {'title': [{'message': "Це поле обов'язкове.", 'code': 'required'}]}
POST /notes/new/ title="ab"  200   Заголовок має бути мінімум 3 символи.
POST /notes/new/ "Купити молоко"   302 /notes/   нотаток: 2 -> 3
GET /notes/                  повідомлення: ['Нотатку "Купити молоко" успішно створено!']
GET /notes/  (F5)            повідомлення: []
GET /notes/3/                200
POST /notes/3/edit/          302 /notes/3/   той самий pk: True, нотаток: 3
GET /notes/3/delete/         200   (сторінка підтвердження, нотатка ще існує)
GET /notes/999/              404
POST без CSRF-токена         403
POST /notes/3/delete/        302 /notes/   нотатка існує: False
GET /notes/                  повідомлення: [('warning', 'Нотатку "Купити молоко і хліб" видалено.')]
```

Перевірено з Django 5.2.19, `django-bootstrap5`, `LANGUAGE_CODE = 'uk'` на коді `forms.py`, `views.py`, `urls.py` і шаблонах з розділу [ModelForm і CRUD](modelform_and_crud.md). Тексти помилок валідації — українські, бо `LANGUAGE_CODE = 'uk'`; з `en-us` буде `This field is required.`

### Що перевірити в браузері

| URL | Що перевірити |
|-----|---------------|
| `http://localhost:8000/` | Редирект на `/notes/` |
| `http://localhost:8000/notes/` | Список з Bootstrap Cards |
| `http://localhost:8000/notes/new/` | Форма створення з Bootstrap стилями |
| `http://localhost:8000/notes/1/` | Деталь нотатки |
| `http://localhost:8000/notes/1/edit/` | Форма редагування (заповнена даними) |
| `http://localhost:8000/notes/1/delete/` | Сторінка підтвердження видалення |
| `http://localhost:8000/admin/` | Адмін-панель з кастомним NoteAdmin |

1. Список: Cards рівні по висоті (`h-100`).
2. Мобільний (DevTools → Toggle Device, Ctrl+Shift+M): 1 колонка, Navbar гамбургер.
3. Планшет: 2 колонки; Десктоп: 3 колонки.
4. Після створення → Flash-повідомлення `success` з назвою нотатки.
5. F5 після POST → залишаєшся на `/notes/`, дубліката немає (PRG працює).

??? note "Усі команди кроку по порядку"

    ```bash
    # 1. Активувати venv
    source venv/bin/activate          # Linux/Mac
    # або: venv\Scripts\activate      # Windows

    # 2. Встановити залежності
    pip install -r requirements.txt

    # 3. Застосувати міграції
    python manage.py migrate

    # 4. Створити адміністратора
    python manage.py createsuperuser

    # 5. Запустити сервер
    python manage.py runserver
    # → http://localhost:8000/notes/

    # Якщо змінював models.py:
    python manage.py makemigrations
    python manage.py migrate
    ```

## Чекліст { #checklist }

- [ ] `GET /` → `302` на `/notes/`.
- [ ] `POST /notes/new/` з порожнім `title` → `200`, форма з помилкою `Це поле обов'язкове.`; з `title="ab"` → `Заголовок має бути мінімум 3 символи.` (`clean_title`).
- [ ] Правильний `POST /notes/new/` → `302` на `/notes/`, кількість нотаток +1, Flash-повідомлення `success`; після F5 повідомлення зникає, дубліката немає.
- [ ] `POST /notes/<pk>/edit/` → `302` на `/notes/<pk>/`, `pk` не змінився, кількість нотаток та сама (UPDATE, а не INSERT).
- [ ] `GET /notes/<pk>/delete/` лише показує підтвердження; видаляє тільки `POST` → `302` на `/notes/` і повідомлення `warning`.
- [ ] `GET /notes/999/` → `404`, а не `500` (`get_object_or_404`).
- [ ] `POST` без `{% csrf_token %}` → `403`.
- [ ] Шаблони наслідують `base.html` через `{% extends %}`; Bootstrap підключений через CDN у `base.html`; Sticky Footer: `body.d-flex.flex-column.min-vh-100` + `main.flex-grow-1`.
- [ ] Можеш пояснити, що таке PRG і чому F5 після PRG безпечний.
- [ ] Можеш пояснити, чому `NoteForm(request.POST, instance=note)` робить UPDATE, а `NoteForm(request.POST)` — INSERT.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `TemplateDoesNotExist: base.html` | `base.html` не там, де Django шукає | файл має бути в `hello_app/templates/base.html` (не в `hello_app/templates/hello_app/`) |
| `NoReverseMatch: Reverse for 'note_create' not found` | URL name не зареєстровано або не той namespace | перевір `hello_app/urls.py` і `app_name = 'hello_app'` |
| `NoReverseMatch` для `'note_list'` у шаблоні | не вказано namespace | `{% url 'hello_app:note_list' %}` (з двокрапкою) |
| `Forbidden (403) CSRF verification failed` після POST | відсутній `{% csrf_token %}` у формі | додай `{% csrf_token %}` всередину `<form>` |
| Bootstrap стилі не застосовуються до полів | `class='form-control'` не додано до widget | перевір `NoteForm.Meta.widgets` у `forms.py` |
| `IntegrityError` при збереженні | об'єкт збережено в обхід `form.is_valid()` | зберігай тільки після `form.is_valid()` |
| Дублікат після F5 | немає redirect після POST | PRG: завжди `return redirect(...)` після `form.save()` |
| Редагування створює нову нотатку | у `note_edit` немає `instance=note` | `NoteForm(request.POST, instance=note)` |
| Cards різної висоти | немає `h-100` на `.card` | додай `h-100` до `<div class="card">` |
| Футер посередині сторінки | немає Flexbox на body | `d-flex flex-column min-vh-100` на `<body>` + `flex-grow-1` на `<main>` |
| Кнопки на картці не клікаються (є `stretched-link`) | `stretched-link` перекриває кнопки | додай `position-relative z-1` на `btn-group` |
| Flash-повідомлення не з'являються | `MESSAGE_TAGS` не налаштовано або в `base.html` немає блоку | перевір `settings.py` → `MESSAGE_TAGS` і `base.html` → `{% if messages %}` |
| `ImproperlyConfigured: Specifying a namespace in include() without providing an app_name` | `app_name` відсутній у `urls.py` | додай `app_name = 'hello_app'` |

## Практика { #practice }

### Розібраний приклад: поле `priority` { #worked-example }

Задача: у нотатки з'являється пріоритет — Низький / Середній / Високий.

1. **Модель.** `choices` обмежує значення і дає `get_priority_display()`:

    ```python title="hello_app/models.py (фрагмент)"
    PRIORITY_CHOICES = [(1, 'Низький'), (2, 'Середній'), (3, 'Високий')]
    priority = models.PositiveSmallIntegerField(
        choices=PRIORITY_CHOICES, default=2, verbose_name='Пріоритет'
    )
    ```

2. **Міграція.**

    ```text
    $ python manage.py makemigrations hello_app
    Migrations for 'hello_app':
      hello_app/migrations/0002_note_priority.py
        + Add field priority to note
    $ python manage.py migrate hello_app
      Applying hello_app.0002_note_priority... OK
    ```

    `default=2` потрібен, бо в таблиці вже є рядки: без нього `makemigrations` спитає, яке значення поставити наявним нотаткам.

3. **Форма.** `fields = ['title', 'content', 'priority']` — ModelForm сам робить `<select>`.

4. **Результат:**

    ```text
    GET /notes/new/                   у формі є <select name="priority">
    POST title="Пріоритетна" priority=3   302 → get_priority_display() = 'Високий'
    POST title="Погана" priority=9        200 → Зробить коректний вибір, 9 немає серед варіантів вибору.
    ```

    Значення `9` відсіяла форма — до бази воно не дійшло. Текст помилки — стандартний переклад Django для `uk` (саме так, з «Зробить»).

### Зміни приклад { #modify }

1. Покажи пріоритет на деталі нотатки: `{{ note.get_priority_display }}`.
2. Відсортуй список так, щоб спочатку йшли важливі: `Note.objects.order_by('-priority', '-created_at')`.

### Спробуй самостійно: Badge з пріоритетом { #try-yourself }

У `note_list.html` покажи Bootstrap Badge з пріоритетом на кожній картці: `bg-secondary` для 1, `bg-warning` для 2, `bg-danger` для 3.

**Критерії перевірки:** після створення 3 нотаток з різним пріоритетом список показує три різні кольори; `python manage.py check` без помилок; `showmigrations hello_app` → `[X] 0002_note_priority`.

## Підсумок { #summary }

| Концепція | Де у коді |
|-----------|-----------|
| ModelForm | `hello_app/forms.py` → `class NoteForm(forms.ModelForm)` |
| Валідація поля | `NoteForm.clean_title()` → `ValidationError` |
| PRG-паттерн | `note_create`: `return redirect(...)` після `form.save()` |
| UPDATE замість INSERT | `NoteForm(request.POST, instance=note)` |
| Messages | `messages.success(request, ...)` + `{% if messages %}` у `base.html` |
| Template Inheritance | `{% extends 'base.html' %}` + `{% block content %}` |
| CRUD views | `note_list`, `note_detail`, `note_create`, `note_edit`, `note_delete` |
| 404 замість 500 | `get_object_or_404(Note, pk=pk)` |
| Bootstrap Grid | `row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4` |
| Sticky Footer | `body.d-flex.flex-column.min-vh-100` + `main.flex-grow-1` |
| Захист DELETE | POST, не GET + `{% csrf_token %}` |

Що змінилось у файлах:

| Файл | Що ти зробив |
|------|-------------|
| `hello_project/settings.py` | `django_bootstrap5` у INSTALLED_APPS, `MESSAGE_TAGS` |
| `hello_app/forms.py` | `NoteForm(ModelForm)` — НОВИЙ |
| `hello_app/views.py` | 6 views: `index`, `note_list`, `note_detail`, `note_create`, `note_edit`, `note_delete` |
| `hello_app/urls.py` | 6 URL маршрутів |
| `hello_app/admin.py` | `NoteAdmin` з `list_display`, `search_fields`, `@admin.display` |
| `hello_app/templates/base.html` | Bootstrap CDN, Navbar, Messages, Footer — НОВИЙ |
| `hello_app/templates/hello_app/note_list.html` | Bootstrap Card Grid — ПЕРЕПИСАНИЙ |
| `hello_app/templates/hello_app/note_detail.html` | Деталь + Breadcrumb — НОВИЙ |
| `hello_app/templates/hello_app/note_form.html` | Bootstrap Form — НОВИЙ |
| `hello_app/templates/hello_app/note_confirm_delete.html` | Видалення — НОВИЙ |

## Самоперевірка { #self-check }

1. Що повертає `POST /notes/new/` з помилкою валідації і чому не redirect?
2. Чому видалення — тільки через POST?
3. Звідки береться `form.errors['title']` при `title="ab"`?
4. Чому Flash-повідомлення зникає після F5?
5. Що зміниться, якщо в `note_edit` прибрати `instance=note`?

??? success "Відповіді"

    1. `200` з тією самою формою: в ній уже є введені дані й помилки. Redirect їх загубив би. Redirect — тільки після успішного збереження.
    2. GET-запит може зробити будь-хто й будь-що: посилання, `<img src>`, префетч браузера. POST вимагає форму з CSRF-токеном — без нього `403` (перевірено).
    3. Спершу стандартні валідатори поля (`required`, `max_length`), потім `clean_title()`: він обрізає пробіли й кидає `ValidationError`, якщо менше 3 символів.
    4. Повідомлення зберігається до першого читання: шаблон `{% for message in messages %}` його «споживає». Другий `GET /notes/` отримує порожній список (перевірено: `[]`).
    5. Форма без `instance` — це нова нотатка: `form.save()` зробить INSERT, і замість оновлення з'явиться копія.

## Що далі { #next }

[Крок 3. CRUD і архітектура](../03_crud_and_architecture/index.md) — `notes_project`: Selectors, Services, ER-діаграми, PostgreSQL, CBV.
