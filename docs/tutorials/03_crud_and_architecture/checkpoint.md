# Контрольна точка: крок 3

## Перевір, що крок робочий { #check }

У цьому кроці ти будуєш `notes_project`. Звірити його можна з кодом цього репозиторію: `notes_app/models.py`, `selectors.py` і `services.py` тут — та сама архітектура, лише в кінцевому стані (з групами та чатом, які додадуть наступні кроки). Нижче — реальний вивід на цьому репозиторії, PostgreSQL 16 (`postgres:16-alpine`, як у `docker-compose.yml`) і `DATABASE_URL=postgres://notes_user:notes_pass@localhost:5432/notes_db`:

```text
$ python manage.py check
System check identified no issues (0 silenced).
$ python manage.py showmigrations notes_app
notes_app
 [X] 0001_initial
 [X] 0002_shoppinglist_shared_with_todolist_shared_with
 [X] 0003_note_group_shoppinglist_group
 [X] 0004_chatmessage
$ python manage.py test notes_app.tests.test_models notes_app.tests.test_services
Found 90 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
..........................................................................................
----------------------------------------------------------------------
Ran 90 tests in 33.692s

OK
Destroying test database for alias 'default'...
```

Час залежить від машини. 90 = 25 тестів моделей (`test_models.py`: `unique_together` тегів, `SET_NULL` при видаленні записника й групи, `__str__`) + 65 тестів сервісів (`test_services.py`: `transaction.atomic()`, ізоляція між користувачами, `F()`-перемикачі).

Що дає селектор замість «наївного» циклу — на демо-даних (`python manage.py seed_demo_data --force`, користувач `demo_alice`), кількість SQL-запитів через `CaptureQueriesContext`:

```text
naive notes: 10 queries: 11      # Note.objects.filter(user=alice) + n.notebook.title у циклі
selector notes: 16 queries: 2    # selectors.get_user_notes(alice) + notebook + tags
```

Селектор повертає більше нотаток (16 — це ще й нотатки груп, у яких є `demo_alice`), але за 2 запити: `select_related('notebook', 'group')` і `prefetch_related('tags')`.

Міграція — це SQL, який можна прочитати до застосування:

```text
$ python manage.py sqlmigrate notes_app 0003
BEGIN;
--
-- Add field group to note
--
ALTER TABLE "notes_app_note" ADD COLUMN "group_id" integer NULL CONSTRAINT "notes_app_note_group_id_cd9a5f53_fk_auth_group_id" REFERENCES "auth_group"("id") DEFERRABLE INITIALLY DEFERRED; ...
```

## Чекліст { #checklist }

**Проектування**

- [ ] Домен описано словами до коду; для «Нотаток» є ER-діаграма в Mermaid з типом кожного зв'язку.
- [ ] Для кожного FK `on_delete` обрано свідомо: `Note.notebook` і `Note.group` — `SET_NULL` (`null=True, blank=True`), `Reminder.note` і `TodoItem.todo_list` — `CASCADE`.
- [ ] FK живе на «Many»-стороні: `Note.notebook`, а не `Notebook.notes`.

**Моделі та міграції**

- [ ] `python manage.py showmigrations notes_app` — усі міграції `[X]`; `python manage.py makemigrations --check --dry-run` → `No changes detected`.
- [ ] Моделі мають `related_name`, `__str__`, `Meta.ordering`; ти знаєш різницю `null=True` (БД) і `blank=True` (форма).
- [ ] Перед `migrate` читаєш SQL: `python manage.py sqlmigrate notes_app 0003`; застосовану міграцію не редагуєш, нові називаєш через `--name`.

**Архітектура і ORM**

- [ ] `selectors.py` — тільки читання; `selectors.get_user_notes(user)` дає 2 SQL-запити замість N+1.
- [ ] `services.py` — усі зміни; `create_note`, `create_notebook`, `update_notebook`, `complete_todo_list` загорнуті в `transaction.atomic()`; `toggle_pin_note` робить `update(is_pinned=~F('is_pinned'))`.
- [ ] `views.py` не містить `Model.objects.*` для бізнес-операцій — викликає selectors і services.
- [ ] `python manage.py test notes_app.tests.test_models notes_app.tests.test_services` → `Ran 90 tests` і `OK`.

**CBV і PostgreSQL**

- [ ] У CBV `LoginRequiredMixin` стоїть першим у MRO, `success_url` — через `reverse_lazy`, `form_valid()` викликає service, а не `super().form_valid()`.
- [ ] PostgreSQL запущено в Docker, `DATABASE_URL` задано змінною середовища; дані з SQLite перенесено через `dumpdata`/`loaddata`.
- [ ] Можеш пояснити READ flow (view → selector → QuerySet) і WRITE flow (view → form → service → `transaction.atomic()`).
- [ ] Можеш пояснити, чому `Q(user=user) | Q(group__in=user_groups)` — це правило доступу, а не просто фільтр.

## Типові помилки { #mistakes }

| Симптом | Причина | Що зробити |
|---|---|---|
| `hello_app.Page.notebook: (fields.E320) Field specifies on_delete=SET_NULL, but cannot be null.` | `SET_NULL` без `null=True` | `ForeignKey(..., on_delete=models.SET_NULL, null=True, blank=True)` |
| `ImproperlyConfigured: The included URLconf 'hello_project.urls' does not appear to have any patterns in it. ... probably caused by a circular import.` | `success_url = reverse(...)` в атрибуті класу виконується під час імпорту `views.py`, коли `urls.py` ще не завантажено | `reverse_lazy('hello_app:note_list')` |
| На сторінці списку десятки однакових `SELECT` у Debug Toolbar | N+1: `note.notebook` / `note.tags` у циклі без `select_related` / `prefetch_related` | читати через селектор з `select_related('notebook')` і `prefetch_related('tags')` |
| Нотатка створюється, а теги/записник — ні (або навпаки) | частина операції впала посеред запису | групова зміна — в `transaction.atomic()` у сервісі |
| Нотатка створена без перевірок сервісу (теги чужого користувача прив'язались) | `form.save()` або `super().form_valid()` у view обходить `services` | у view — `services.create_note(user=request.user, ...)` |
| `OperationalError: could not connect to server` / `Connection refused` на 5432 | PostgreSQL не запущено або інший порт у `DATABASE_URL` | запусти контейнер, перевір `DATABASE_URL` |
| `No changes detected`, хоча модель змінено | додаток не в `INSTALLED_APPS` або зміна не в тому `models.py` | перевір `INSTALLED_APPS` і шлях до файлу |

??? note "Приклади виправлень у коді"

    **Починати з `models.py`**

    ```
    ❌ НЕПРАВИЛЬНО: одразу пишемо class Note(models.Model)
    ✅ ПРАВИЛЬНО: спочатку описуємо домен словами, потім ER-діаграма, потім код
    ```

    **QuerySet у View**

    ```python
    # ❌ НЕПРАВИЛЬНО:
    def note_list(request):
        notes = Note.objects.filter(user=request.user).select_related(...)
        return render(...)

    # ✅ ПРАВИЛЬНО:
    def note_list(request):
        notes = selectors.get_user_notes(request.user)
        return render(...)
    ```

    **Пряма мутація у View**

    ```python
    # ❌ НЕПРАВИЛЬНО:
    def note_create(request):
        if form.is_valid():
            note = form.save()  # обходить services!

    # ✅ ПРАВИЛЬНО:
    def note_create(request):
        if form.is_valid():
            note = services.create_note(user=request.user, ...)
    ```

    **N+1 при зверненні до FK**

    ```python
    # ❌ НЕПРАВИЛЬНО:
    notes = Note.objects.all()
    for note in notes:
        print(note.notebook.title)  # N+1!

    # ✅ ПРАВИЛЬНО:
    notes = Note.objects.select_related('notebook').all()
    for note in notes:
        print(note.notebook.title)  # 1 запит!
    ```

    **SET_NULL без null=True**

    ```python
    # ❌ НЕПРАВИЛЬНО:
    notebook = models.ForeignKey(Notebook, on_delete=models.SET_NULL)
    # python manage.py check: fields.E320

    # ✅ ПРАВИЛЬНО:
    notebook = models.ForeignKey(
        Notebook, on_delete=models.SET_NULL,
        null=True, blank=True,
    )
    ```

    **`super().form_valid()` у CBV**

    ```python
    # ❌ НЕПРАВИЛЬНО у CBV з services:
    def form_valid(self, form):
        super().form_valid(form)  # там form.save() — обходить services!

    # ✅ ПРАВИЛЬНО:
    def form_valid(self, form):
        note = services.create_note(user=self.request.user, ...)
        return redirect('hello_app:note_detail', pk=note.pk)
    ```

    **`reverse()` замість `reverse_lazy()` у class attribute**

    ```python
    # ❌ НЕПРАВИЛЬНО: виконується при завантаженні класу
    class NoteDeleteView(DeleteView):
        success_url = reverse('hello_app:note_list')  # ImproperlyConfigured (circular import)

    # ✅ ПРАВИЛЬНО: виконується ліниво при запиті
    class NoteDeleteView(DeleteView):
        success_url = reverse_lazy('hello_app:note_list')
    ```

## Практика { #practice }

### Розібраний приклад: прибрати N+1 зі списку нотаток { #worked-example }

Задача: сторінка списку показує для кожної нотатки назву записника і теги. Скільки SQL-запитів це коштує?

1. **Наївно.** `for n in Note.objects.filter(user=alice): n.notebook.title` — один запит на список і ще один на кожен записник. Для 10 нотаток `demo_alice` — **11** запитів (виміряно `CaptureQueriesContext`).
2. **Селектор.** Те, що вже є в `notes_app/selectors.py`:

    ```python title="notes_app/selectors.py (фрагмент)"
    def get_user_notes(user, *, archived=False, notebook=None, tag=None, search=None):
        user_groups = user.groups.all()
        qs = Note.objects.filter(
            Q(user=user) | Q(group__in=user_groups), is_archived=archived
        ).select_related('notebook', 'group').prefetch_related('tags')
        ...
        return qs.order_by('-is_pinned', '-priority', '-updated_at')
    ```

    `select_related` — JOIN в тому самому запиті (FK «один»); `prefetch_related` — один додатковий запит на всі теги (M2M «багато»).
3. **Результат:** 16 нотаток (власні + групові), назва записника й теги кожної — **2** запити. Кількість запитів більше не залежить від кількості нотаток.

### Зміни приклад { #modify }

1. Додай у `get_user_notes` параметр `pinned_only=False` і умову `qs.filter(is_pinned=True)`. Перевір, що запитів так само 2.
2. Прибери `prefetch_related('tags')` і порахуй запити ще раз — скільки їх тепер і чому?

### Спробуй самостійно: «Бібліотека» { #library }

1. Намалюй ER-діаграму для системи «Бібліотека»: `Book`, `Author`, `Genre`, `Review`. Вкажи типи зв'язків і `on_delete` для кожного FK.
2. Реалізуй моделі з `related_name`, `__str__`, `ordering`.
3. Напиши `get_books_by_genre(genre)` у `selectors.py` з `select_related('author').prefetch_related('genres')`.
4. Напиши `create_book(*, title, author_id, genre_ids)` у `services.py` з `transaction.atomic()`.
5. У Django shell зміни `author` у книзі й перевір, що `updated_at` оновився.
6. Перепиши `BookListView` і `BookCreateView` з FBV на CBV з `LoginRequiredMixin` і `UserQuerySetMixin`.
7. Встанови `django-debug-toolbar` і перевір кількість запитів на сторінці списку книг.

**Критерії перевірки:** `python manage.py check` без помилок; `makemigrations --check --dry-run` → `No changes detected`; сторінка списку книг робить сталу кількість запитів (2–3) незалежно від кількості книг; помилка посеред `create_book` не залишає «напівкнигу» без жанрів.

## Підсумок { #summary }

| Поняття | Що запам'ятати |
|---|---|
| Методологія | Домен → Сутності → Атрибути → Зв'язки → Нормалізація → `on_delete` → Django модель |
| Зв'язки | `OneToOneField` (`UserProfile.user`), `ForeignKey` (`Note.notebook`), `ManyToManyField` (`Note.tags`) |
| `on_delete` | `CASCADE` — дочірнє зникає разом; `SET_NULL` — лишається без батька, потрібен `null=True` |
| Міграції | `makemigrations` → `sqlmigrate` → `migrate`; застосовану не редагуй |
| Selectors | тільки читання, `select_related` / `prefetch_related` проти N+1 |
| Services | усі зміни; `transaction.atomic()` для групи операцій, `F()` для атомарних оновлень |
| `Q` | OR-умови: `Q(user=user) \| Q(group__in=user_groups)` |
| `select_for_update()` | блокування рядка проти гонок |
| `annotate()` | обчислення в SQL замість Python |
| CBV | `.as_view()`, `LoginRequiredMixin` першим, `reverse_lazy`, `form_valid()` → service |
| PostgreSQL | Docker + `DATABASE_URL`; перенос з SQLite — `dumpdata` / `loaddata` |

## Самоперевірка { #self-check }

1. Чому `Note.notebook` — `SET_NULL`, а `Reminder.note` — `CASCADE`?
2. Скільки SQL-запитів зробить список із 100 нотаток з `select_related('notebook')` і `prefetch_related('tags')`?
3. Що станеться, якщо `create_note` впаде після `INSERT` нотатки, але до прив'язки тегів, — з `transaction.atomic()` і без?
4. Чому `success_url = reverse(...)` ламає старт проєкту, а `reverse_lazy` — ні?
5. Навіщо `sqlmigrate`, якщо `migrate` і так усе зробить?

??? success "Відповіді"

    1. Записник — лише «папка»: видалив папку — нотатки мають лишитися (стають «без записника»). Нагадування без нотатки не має сенсу — воно зникає разом з нею.
    2. Два, як і для 16: один запит з JOIN на записники й один на всі теги. Кількість не залежить від N.
    3. З `atomic()` — ROLLBACK: нотатки немає, база в попередньому стані. Без нього — нотатка без тегів, тобто напівзаписаний об'єкт.
    4. `reverse()` виконується під час визначення класу, тобто під час імпорту `views.py`, коли кореневий `urls.py` ще імпортує цей самий модуль — круговий імпорт. `reverse_lazy` відкладає обчислення до першого запиту.
    5. Щоб побачити SQL до того, як він змінить базу: який стовпець, `NULL` чи ні, яке обмеження. Особливо на production-базі з даними.

## Що далі { #next }

[Крок 4. Templates і Forms](../04_templates_and_forms/index.md) — 3-рівнева Template Inheritance, Crispy Forms, Context Processor для Sidebar.
