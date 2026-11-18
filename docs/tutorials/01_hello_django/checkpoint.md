# Контрольна точка: крок 1

## Перевір, що крок робочий { #check }

З папки проєкту `hello_project` (там, де лежить `manage.py`), з активованим venv:

```text
$ python -m django --version
5.2.19
$ python manage.py check
System check identified no issues (0 silenced).
$ python manage.py showmigrations hello_app
hello_app
 [X] 0001_initial
```

Сервер — `python manage.py runserver`, у другому терміналі:

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

Перевірено з Django 5.2.19 і Python 3.12 у чистій папці: `startproject hello_project .`, `startapp hello_app`, код view-функцій, маршрутів і моделі `Note` — з розділів цього кроку; дві нотатки створено через `Note.objects.create(...)` у `python manage.py shell`. Твій HTML у `/notes/` буде іншим — залежно від шаблону. `/admin/` для анонімного користувача — це `302` на сторінку логіну; після входу superuser'ом — адмін-панель. Порядок «Друга, потім Перша» — це `Meta.ordering = ['-created_at']`.

??? note "Усі команди кроку по порядку"

    ```bash
    # 1. Створити і активувати venv
    python -m venv venv
    source venv/bin/activate      # Linux/Mac
    # або: venv\Scripts\activate  # Windows

    # 2. Встановити Django
    pip install -r requirements.txt
    # або: pip install django

    # 3. Створити проєкт (крапка важлива!)
    django-admin startproject hello_project .

    # 4. Застосувати початкові міграції
    python manage.py migrate

    # 5. Створити адміністратора
    python manage.py createsuperuser

    # 6. Перевірити адмін-панель
    python manage.py runserver
    # → http://localhost:8000/admin/

    # 7. Створити додаток
    python manage.py startapp hello_app

    # 8. Зареєструвати hello_app у settings.py
    # (додати 'hello_app' до INSTALLED_APPS)

    # 9. Написати view-функції (views.py) і URL-маршрути (urls.py)

    # 10. Підключити hello_app.urls до hello_project/urls.py (include)

    # 11. Описати модель у models.py
    # 12. Зробити міграцію
    python manage.py makemigrations

    # 13. Застосувати міграцію
    python manage.py migrate

    # 14. Зареєструвати модель в admin.py (@admin.register)

    # 15. Фінальна перевірка
    python manage.py runserver
    # http://localhost:8000/        → Hello, Django!
    # http://localhost:8000/about/  → текст
    # http://localhost:8000/notes/  → список нотаток
    # http://localhost:8000/admin/  → адмін-панель
    ```

## Чекліст { #checklist }

- [ ] `python manage.py check` → `System check identified no issues (0 silenced).`
- [ ] `http://localhost:8000/` → `200` і текст `Hello, Django!`.
- [ ] `http://localhost:8000/about/` → `200` і текст `Це моя перша сторінка на Django!`.
- [ ] `http://localhost:8000/notes/` → `200`, HTML-список нотаток з шаблону `hello_app/templates/hello_app/note_list.html` (подвоєна папка — це правильно).
- [ ] `http://localhost:8000/admin/` без логіну → `302` на `/admin/login/?next=/admin/`; після входу superuser'ом видно модель `Note`.
- [ ] `python manage.py showmigrations hello_app` → `[X] 0001_initial`.
- [ ] У `python manage.py shell` працює CRUD: `Note.objects.create(...)`, `filter`, `get(pk=2)`, `.save()`, `.delete()` — і зміна видна в адмін-панелі.
- [ ] `reverse('hello_app:about')` → `'/about/'`.
- [ ] Можеш пояснити різницю між `startproject` і `startapp` і чому `makemigrations` та `migrate` — два окремі кроки.
- [ ] Можеш пояснити, як Django знаходить view за URL (ланцюг URLconf: `hello_project/urls.py` → `include('hello_app.urls')` → `path(...)`) і чим відрізняються superuser / staff / regular user.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `/notes/` → `500`, у терміналі `django.db.utils.OperationalError: no such table: hello_app_note` | модель описано, але `makemigrations` / `migrate` не запускались | `python manage.py makemigrations`, потім `python manage.py migrate` |
| `OperationalError: no such table: auth_user` при `createsuperuser` чи вході в `/admin/` | початкові міграції не застосовано | `python manage.py migrate` |
| `django.core.exceptions.ImproperlyConfigured: Specifying a namespace in include() without providing an app_name is not supported.` | `namespace=` є в `include()`, але `app_name` відсутній у `hello_app/urls.py` | додай `app_name = "hello_app"` у `hello_app/urls.py` |
| `django.urls.exceptions.NoReverseMatch: Reverse for 'about' not found. 'about' is not a valid view function or pattern name.` | ім'я маршруту без namespace | `reverse('hello_app:about')`, `{% url 'hello_app:about' %}` |
| `ModuleNotFoundError: No module named 'django'` | venv не активований або Django не встановлений | активуй venv → `pip install -r requirements.txt` |
| `django-admin: command not found` | те саме: Django не встановлений або venv не активований | активуй venv, потім `pip install django` |
| `AttributeError: 'super' object has no attribute 'dicts'` | Python 3.14 несумісний з Django 5.1 | `pip install django --upgrade` (встановить 5.2+) |
| `Page not found (404)` на `/` | маршрут не зареєстровано | перевір `urls.py` в обох файлах |
| `NameError: name 'Note' is not defined` | `views.py` не імпортує модель | `from .models import Note` у `views.py` |
| Django «не бачить» `hello_app`: моделі немає в адмінці, `makemigrations` пише `No changes detected` | додаток не додано в `INSTALLED_APPS` | додай `'hello_app'` у `settings.py` |
| `ModuleNotFoundError: No module named 'hello_project'` | запускаєш не з кореня проєкту | перейди в папку, де є `manage.py` |
| `DisallowedHost at /` | `ALLOWED_HOSTS` не містить хост | додай хост до `ALLOWED_HOSTS` у `settings.py` |
| `Forbidden (403) CSRF verification failed` | `{% csrf_token %}` відсутній у формі | додай `{% csrf_token %}` всередині `<form>` |
| Сервер не зупиняється | — | `Ctrl + C` у терміналі |

## Практика { #practice }

### Розібраний приклад: маршрут з параметром `/greet/<str:name>/` { #worked-example }

Задача: сторінка `/greet/Olena/` має відповідати `Привіт, Olena!`.

1. **View.** Параметр з URL приходить у функцію як іменований аргумент:

    ```python title="hello_app/views.py (фрагмент)"
    def greet(request, name):
        return HttpResponse(f"Привіт, {name}!")
    ```

2. **Маршрут.** Конвертер `str` бере все до наступного `/`:

    ```python title="hello_app/urls.py (фрагмент)"
    path('greet/<str:name>/', views.greet, name='greet'),
    ```

3. **Результат** (Django test client, той самий проєкт):

    ```text
    /greet/Olena/ 200 Привіт, Olena!
    /greet/ 404
    /greet/a/b/ 404
    reverse('hello_app:greet', args=['Taras']) → /greet/Taras/
    ```

    `/greet/` — `404`, бо порожній рядок `str` не приймає; `/greet/a/b/` — `404`, бо `str` не містить `/` (для шляхів зі слешем є конвертер `path`).

### Зміни приклад { #modify }

1. Заміни `<str:name>` на `<int:age>` і поверни `"Тобі {age} років"`. Що поверне `/greet/abc/`?
2. Додай маршрут `/contact/` з view-функцією `contact()`, яка повертає рядок з твоїм ім'ям та email.

### Спробуй самостійно: модель і shell { #try-yourself }

- Створи модель `Note`, зроби міграцію, відкрий адмін-панель і додай 3 нотатки вручну.
- У Django shell знайди нотатку з `id=2`, зміни її `title`, збережи. Перевір в адмін-панелі.
- (Бонус) Додай поле `is_pinned = models.BooleanField(default=False)` до моделі `Note`, зроби нову міграцію і переконайся, що поле з'явилось в адмін-панелі.

**Критерії перевірки:** `showmigrations hello_app` показує дві застосовані міграції; `python manage.py check` без помилок; зміна з shell видна в `/admin/hello_app/note/`.

## Підсумок { #summary }

| Концепція | Де у коді |
|---|---|
| URL маршрут | `hello_app/urls.py` → `path('about/', views.about)` |
| View-функція | `hello_app/views.py` → `def about(request): return HttpResponse(...)` |
| Підключення app URLs | `hello_project/urls.py` → `include('hello_app.urls')` |
| Namespace | `app_name = 'hello_app'` + `namespace='hello_app'` |
| Зворотний URL | `reverse('hello_app:about')` або `{% url 'hello_app:about' %}` |
| Path converter | `path('notes/<int:pk>/', ...)` |
| Шаблон | `hello_app/templates/hello_app/note_list.html` |
| Контекст | `render(request, 'шаблон.html', {'notes': queryset})` |
| Модель → БД | `makemigrations` + `migrate` |
| ORM CRUD | `create`, `filter`, `get`, `.save()`, `.delete()` |
| Адмін реєстрація | `@admin.register(Note)` у `admin.py` |
| MVT | Model → View → Template |

Що змінилось у файлах:

| Файл | Що ти зробив |
|---|---|
| `hello_project/settings.py` | Додав `'hello_app'` в `INSTALLED_APPS` |
| `hello_app/models.py` | Написав модель `Note` (CharField, TextField, DateTimeField) |
| `hello_app/admin.py` | Зареєстрував `NoteAdmin` через `@admin.register` |
| `hello_app/migrations/0001_initial.py` | Автоматично згенеровано `makemigrations` |
| `hello_app/views.py` | Написав функції `index()`, `about()`, `note_list()` |
| `hello_app/urls.py` | Створив файл з `app_name` і маршрутами `/, /about/, /notes/` |
| `hello_project/urls.py` | Підключив маршрути через `include()` |
| `hello_app/templates/hello_app/note_list.html` | Перший HTML-шаблон |

## Самоперевірка { #self-check }

1. Чим проєкт (`startproject`) відрізняється від додатку (`startapp`)?
2. Навіщо дві команди — `makemigrations` і `migrate`?
3. Чому `/notes/` падає з `no such table`, а `/` і `/about/` працюють?
4. Що станеться, якщо прибрати `app_name` з `hello_app/urls.py`, залишивши `namespace=` в `include()`?
5. Чому шаблон лежить у `hello_app/templates/hello_app/`, а не просто в `hello_app/templates/`?
6. Що приймає `render()` і що він повертає?

??? success "Відповіді"

    1. Проєкт — пакет налаштувань (`settings.py`, кореневий `urls.py`, `wsgi.py`/`asgi.py`) і `manage.py`; додаток — модуль з власними моделями, views і URL, який підключається в `INSTALLED_APPS`. Один проєкт — багато додатків.
    2. `makemigrations` порівнює моделі з попередніми міграціями і пише новий файл міграції (план змін). `migrate` виконує ці плани на базі. Між ними файл можна прочитати (`sqlmigrate`) і закомітити в git.
    3. `/` і `/about/` повертають `HttpResponse` без звернення до бази. `note_list` робить `Note.objects.all()` — це `SELECT` з таблиці `hello_app_note`, якої ще немає.
    4. Проєкт не стартує: `ImproperlyConfigured: Specifying a namespace in include() without providing an app_name is not supported.` (перевірено `python manage.py check`).
    5. Django збирає `templates/` з усіх додатків в один простір імен. Подвоєна папка робить ім'я `hello_app/note_list.html` унікальним — інакше два додатки з `note_list.html` конфліктували б.
    6. `request`, шлях до шаблону і словник `context`; повертає `HttpResponse` з відрендереним HTML (статус `200`).

## Що далі { #next }

[Крок 2. Bootstrap Notes](../02_first_model/index.md) — Bootstrap 5, ModelForm, повноцінний CRUD з PRG-паттерном.
