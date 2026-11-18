# Checkpoint: step 3

## Check that the step works { #check }

In this step you build `notes_project`. You can compare it with the code of this repository: `notes_app/models.py`, `selectors.py` and `services.py` here follow the same architecture, only in its final state (with groups and chat, which the next steps add). Below is real output on this repository, PostgreSQL 16 (`postgres:16-alpine`, as in `docker-compose.yml`) and `DATABASE_URL=postgres://notes_user:notes_pass@localhost:5432/notes_db`:

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

The time depends on your machine. 90 = 25 model tests (`test_models.py`: `unique_together` for tags, `SET_NULL` when a notebook or a group is deleted, `__str__`) + 65 service tests (`test_services.py`: `transaction.atomic()`, isolation between users, `F()` toggles).

What a selector gives you compared with a "naive" loop — on the demo data (`python manage.py seed_demo_data --force`, user `demo_alice`), the number of SQL queries measured with `CaptureQueriesContext`:

```text
naive notes: 10 queries: 11      # Note.objects.filter(user=alice) + n.notebook.title in the loop
selector notes: 16 queries: 2    # selectors.get_user_notes(alice) + notebook + tags
```

The selector returns more notes (16 also includes the notes of groups that `demo_alice` belongs to), but in 2 queries: `select_related('notebook', 'group')` and `prefetch_related('tags')`.

A migration is SQL that you can read before applying it:

```text
$ python manage.py sqlmigrate notes_app 0003
BEGIN;
--
-- Add field group to note
--
ALTER TABLE "notes_app_note" ADD COLUMN "group_id" integer NULL CONSTRAINT "notes_app_note_group_id_cd9a5f53_fk_auth_group_id" REFERENCES "auth_group"("id") DEFERRABLE INITIALLY DEFERRED; ...
```

## Checklist { #checklist }

**Design**

- [ ] The domain is described in words before the code; for "Notes" there is an ER diagram in Mermaid with the type of every relationship.
- [ ] For each FK, `on_delete` is chosen deliberately: `Note.notebook` and `Note.group` use `SET_NULL` (`null=True, blank=True`), `Reminder.note` and `TodoItem.todo_list` use `CASCADE`.
- [ ] The FK lives on the "Many" side: `Note.notebook`, not `Notebook.notes`.

**Models and migrations**

- [ ] `python manage.py showmigrations notes_app` — all migrations are `[X]`; `python manage.py makemigrations --check --dry-run` → `No changes detected`.
- [ ] The models have `related_name`, `__str__`, `Meta.ordering`; you know the difference between `null=True` (database) and `blank=True` (form).
- [ ] Before `migrate` you read the SQL: `python manage.py sqlmigrate notes_app 0003`; you do not edit an applied migration, and you name new ones with `--name`.

**Architecture and ORM**

- [ ] `selectors.py` only reads; `selectors.get_user_notes(user)` makes 2 SQL queries instead of N+1.
- [ ] `services.py` makes all the changes; `create_note`, `create_notebook`, `update_notebook`, `complete_todo_list` are wrapped in `transaction.atomic()`; `toggle_pin_note` does `update(is_pinned=~F('is_pinned'))`.
- [ ] `views.py` contains no `Model.objects.*` for business operations — it calls selectors and services.
- [ ] `python manage.py test notes_app.tests.test_models notes_app.tests.test_services` → `Ran 90 tests` and `OK`.

**CBV and PostgreSQL**

- [ ] In a CBV, `LoginRequiredMixin` comes first in the MRO, `success_url` uses `reverse_lazy`, and `form_valid()` calls a service, not `super().form_valid()`.
- [ ] PostgreSQL runs in Docker, `DATABASE_URL` is set as an environment variable; the data from SQLite is moved with `dumpdata`/`loaddata`.
- [ ] You can explain the READ flow (view → selector → QuerySet) and the WRITE flow (view → form → service → `transaction.atomic()`).
- [ ] You can explain why `Q(user=user) | Q(group__in=user_groups)` is an access rule and not just a filter.

## Common mistakes { #mistakes }

| Symptom | Cause | What to do |
|---|---|---|
| `hello_app.Page.notebook: (fields.E320) Field specifies on_delete=SET_NULL, but cannot be null.` | `SET_NULL` without `null=True` | `ForeignKey(..., on_delete=models.SET_NULL, null=True, blank=True)` |
| `ImproperlyConfigured: The included URLconf 'hello_project.urls' does not appear to have any patterns in it. ... probably caused by a circular import.` | `success_url = reverse(...)` in a class attribute runs while `views.py` is being imported, when `urls.py` is not loaded yet | `reverse_lazy('hello_app:note_list')` |
| Dozens of identical `SELECT`s on the list page in Debug Toolbar | N+1: `note.notebook` / `note.tags` in a loop without `select_related` / `prefetch_related` | read through a selector with `select_related('notebook')` and `prefetch_related('tags')` |
| The note is created, but the tags/notebook are not (or the other way round) | part of the operation failed in the middle of writing | a group of changes goes in `transaction.atomic()` in the service |
| The note is created without the service's checks (another user's tags got attached) | `form.save()` or `super().form_valid()` in the view bypasses `services` | in the view — `services.create_note(user=request.user, ...)` |
| `OperationalError: could not connect to server` / `Connection refused` on 5432 | PostgreSQL is not running or `DATABASE_URL` has a different port | start the container, check `DATABASE_URL` |
| `No changes detected` even though the model changed | the app is not in `INSTALLED_APPS` or the change is in the wrong `models.py` | check `INSTALLED_APPS` and the file path |

??? note "Fix examples in code"

    **Starting with `models.py`**

    ```
    ❌ WRONG: we start writing class Note(models.Model) right away
    ✅ RIGHT: first describe the domain in words, then the ER diagram, then the code
    ```

    **A QuerySet in the View**

    ```python
    # ❌ WRONG:
    def note_list(request):
        notes = Note.objects.filter(user=request.user).select_related(...)
        return render(...)

    # ✅ RIGHT:
    def note_list(request):
        notes = selectors.get_user_notes(request.user)
        return render(...)
    ```

    **Direct mutation in the View**

    ```python
    # ❌ WRONG:
    def note_create(request):
        if form.is_valid():
            note = form.save()  # bypasses services!

    # ✅ RIGHT:
    def note_create(request):
        if form.is_valid():
            note = services.create_note(user=request.user, ...)
    ```

    **N+1 when accessing an FK**

    ```python
    # ❌ WRONG:
    notes = Note.objects.all()
    for note in notes:
        print(note.notebook.title)  # N+1!

    # ✅ RIGHT:
    notes = Note.objects.select_related('notebook').all()
    for note in notes:
        print(note.notebook.title)  # 1 query!
    ```

    **SET_NULL without null=True**

    ```python
    # ❌ WRONG:
    notebook = models.ForeignKey(Notebook, on_delete=models.SET_NULL)
    # python manage.py check: fields.E320

    # ✅ RIGHT:
    notebook = models.ForeignKey(
        Notebook, on_delete=models.SET_NULL,
        null=True, blank=True,
    )
    ```

    **`super().form_valid()` in a CBV**

    ```python
    # ❌ WRONG in a CBV with services:
    def form_valid(self, form):
        super().form_valid(form)  # it calls form.save() — bypasses services!

    # ✅ RIGHT:
    def form_valid(self, form):
        note = services.create_note(user=self.request.user, ...)
        return redirect('hello_app:note_detail', pk=note.pk)
    ```

    **`reverse()` instead of `reverse_lazy()` in a class attribute**

    ```python
    # ❌ WRONG: runs when the class is loaded
    class NoteDeleteView(DeleteView):
        success_url = reverse('hello_app:note_list')  # ImproperlyConfigured (circular import)

    # ✅ RIGHT: runs lazily on request
    class NoteDeleteView(DeleteView):
        success_url = reverse_lazy('hello_app:note_list')
    ```

## Practice { #practice }

### Worked example: removing N+1 from the note list { #worked-example }

Task: the list page shows the notebook title and the tags for every note. How many SQL queries does this cost?

1. **Naive.** `for n in Note.objects.filter(user=alice): n.notebook.title` — one query for the list and one more for every notebook. For the 10 notes of `demo_alice` that is **11** queries (measured with `CaptureQueriesContext`).
2. **Selector.** What is already in `notes_app/selectors.py`:

    ```python title="notes_app/selectors.py (excerpt)"
    def get_user_notes(user, *, archived=False, notebook=None, tag=None, search=None):
        user_groups = user.groups.all()
        qs = Note.objects.filter(
            Q(user=user) | Q(group__in=user_groups), is_archived=archived
        ).select_related('notebook', 'group').prefetch_related('tags')
        ...
        return qs.order_by('-is_pinned', '-priority', '-updated_at')
    ```

    `select_related` is a JOIN in the same query (the "one" side of an FK); `prefetch_related` is one extra query for all the tags (the "many" side of an M2M).
3. **Result:** 16 notes (own + group notes), each with its notebook title and tags — **2** queries. The number of queries no longer depends on the number of notes.

### Modify the example { #modify }

1. Add a `pinned_only=False` parameter to `get_user_notes` with the condition `qs.filter(is_pinned=True)`. Check that there are still 2 queries.
2. Remove `prefetch_related('tags')` and count the queries again — how many are there now, and why?

### Try it yourself: "Library" { #library }

1. Draw an ER diagram for a "Library" system: `Book`, `Author`, `Genre`, `Review`. Give the relationship types and the `on_delete` for every FK.
2. Implement the models with `related_name`, `__str__`, `ordering`.
3. Write `get_books_by_genre(genre)` in `selectors.py` with `select_related('author').prefetch_related('genres')`.
4. Write `create_book(*, title, author_id, genre_ids)` in `services.py` with `transaction.atomic()`.
5. In the Django shell, change the `author` of a book and check that `updated_at` was updated.
6. Rewrite `BookListView` and `BookCreateView` from FBV to CBV with `LoginRequiredMixin` and `UserQuerySetMixin`.
7. Install `django-debug-toolbar` and check the number of queries on the book list page.

**Success criteria:** `python manage.py check` reports no errors; `makemigrations --check --dry-run` → `No changes detected`; the book list page makes a constant number of queries (2–3) regardless of the number of books; an error in the middle of `create_book` does not leave a "half-book" without genres.

## Summary { #summary }

| Concept | What to remember |
|---|---|
| Methodology | Domain → Entities → Attributes → Relationships → Normalization → `on_delete` → Django model |
| Relationships | `OneToOneField` (`UserProfile.user`), `ForeignKey` (`Note.notebook`), `ManyToManyField` (`Note.tags`) |
| `on_delete` | `CASCADE` — the child disappears together with the parent; `SET_NULL` — it stays without a parent, needs `null=True` |
| Migrations | `makemigrations` → `sqlmigrate` → `migrate`; never edit an applied one |
| Selectors | read only, `select_related` / `prefetch_related` against N+1 |
| Services | all changes; `transaction.atomic()` for a group of operations, `F()` for atomic updates |
| `Q` | OR conditions: `Q(user=user) \| Q(group__in=user_groups)` |
| `select_for_update()` | row locking against race conditions |
| `annotate()` | calculations in SQL instead of Python |
| CBV | `.as_view()`, `LoginRequiredMixin` first, `reverse_lazy`, `form_valid()` → service |
| PostgreSQL | Docker + `DATABASE_URL`; moving from SQLite — `dumpdata` / `loaddata` |

## Self-check { #self-check }

1. Why is `Note.notebook` `SET_NULL`, while `Reminder.note` is `CASCADE`?
2. How many SQL queries will a list of 100 notes make with `select_related('notebook')` and `prefetch_related('tags')`?
3. What happens if `create_note` fails after the note's `INSERT` but before the tags are attached — with `transaction.atomic()` and without it?
4. Why does `success_url = reverse(...)` break the project's startup, while `reverse_lazy` does not?
5. Why use `sqlmigrate` if `migrate` will do everything anyway?

??? success "Answers"

    1. A notebook is only a "folder": if you delete the folder, the notes must stay (they become "without a notebook"). A reminder without a note makes no sense — it disappears together with the note.
    2. Two, just as for 16: one query with a JOIN on the notebooks and one for all the tags. The number does not depend on N.
    3. With `atomic()` — ROLLBACK: there is no note, and the database is in its previous state. Without it — a note without tags, that is, a half-written object.
    4. `reverse()` runs when the class is defined, that is, while `views.py` is being imported, when the root `urls.py` is still importing this very module — a circular import. `reverse_lazy` postpones the evaluation until the first request.
    5. To see the SQL before it changes the database: which column, `NULL` or not, which constraint. This matters especially on a production database with data.

## What's next { #next }

[Step 4. Templates and Forms](../04_templates_and_forms/index.md) — 3-level Template Inheritance, Crispy Forms, a Context Processor for the Sidebar.
