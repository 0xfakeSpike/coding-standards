# Django

## App structure

### Split apps by business capability [Recommended]

Each app maps to one business capability and contains its own models, views, migrations, and tests. Do not create catch-all `common` or `utils` apps. Put `AppConfig` in `apps.py`, and register signals in `ready()`.

**Source:** Django Docs

### Define a custom user model first in new projects [Recommended]

Inherit from `AbstractUser` and set `AUTH_USER_MODEL` at the start of the project. Foreign keys use `settings.AUTH_USER_MODEL`. Other code uses `get_user_model()`.

**Why:** Replacing the user model mid-project is expensive.

**Source:** Django Docs

## Models

### Use field types that express meaning [Recommended]

`CharField` and `TextField` use the empty string for no data, not `null=True`, except when a unique constraint is needed and blank values are allowed. Use `DecimalField` for money, with explicit `max_digits` and `decimal_places`, not `FloatField`. Use `models.TextChoices` or `IntegerChoices` for status fields, and refer to the enum members in code.

**Tooling:** `DJ001`.

**Source:** Django Docs

### Express invariants with database constraints [Recommended]

Write uniqueness and value ranges as `UniqueConstraint` and `CheckConstraint` in `Meta.constraints`. Write indexes in `Meta.indexes`.

**Why:** Validation in forms and `save()` does not stop concurrent writes or bulk operations that bypass it.

**Source:** Django Docs

## QuerySet and N+1

### Prefetch before accessing related objects in a loop [Recommended]

**Applies to:** production service

Use `select_related` for foreign keys and one-to-one relations, and `prefetch_related` for many-to-many relations and reverse foreign keys. On Django 6.1+, when it is hard to predict which fields will be accessed, you may use `fetch_mode(models.FETCH_PEERS)` to batch load on demand.

**Why:** Otherwise each loop iteration sends a query, and request time grows linearly with the data.

**Example:**

Avoid:

```python
for book in Book.objects.all():
    print(book.author.name)
```

Prefer:

```python
for book in Book.objects.select_related("author"):
    print(book.author.name)
```

**Tooling:** In tests for list endpoints, pin the query count with `assertNumQueries`.

**Source:** Django Docs

### Do not chain filters after prefetching [Recommended]

`book.reviews.filter(...)` ignores the prefetch cache and queries again. To filter, state the queryset in a `Prefetch` object.

**Source:** Django Docs

### Fetch only the data you need [Recommended]

When you need only field values, use `values()` or `values_list()`. For foreign keys, use `obj.author_id`, not `obj.author.id`. Use `iterator()` for large result sets. Do not sort when order does not matter. Use `exists()` when only checking existence and `count()` when only counting. When you will use the results afterwards, evaluate the queryset directly and reuse it.

**Source:** Django Docs

### Use bulk methods for bulk writes [Recommended]

Use `bulk_create`, `bulk_update`, and `QuerySet.update()`. These methods do not call `save()` and do not send `pre_save` and `post_save` signals, so do not use them when you depend on that logic.

**Source:** Django Docs

## Transactions

### Use atomic for multi-step writes [Required]

Put writes that must succeed together in `transaction.atomic()`. By default, Django commits each statement separately. Transactions are not yet available in async mode: write transaction logic as sync functions and call them with `sync_to_async`.

**Source:** Django Docs

### Do not catch database exceptions inside atomic [Required]

When catching exceptions such as `IntegrityError`, put the `try` outside `atomic`, or wrap the code in another `atomic`.

**Why:** After the catch, the transaction is already broken, and further queries raise `TransactionManagementError`.

**Example:**

```python
try:
    with transaction.atomic():
        create_membership(user, team)
except IntegrityError:
    raise AlreadyMemberError(user.id) from None
```

**Source:** Django Docs

### Use on_commit for side effects outside the transaction [Required]

Put sending email, enqueuing tasks, and refreshing caches in `transaction.on_commit()`. Use `functools.partial` to bind arguments.

**Why:** Side effects already sent cannot be recalled after a rollback, and a task may run before the commit and read old data.

**Source:** Django Docs

### Use F expressions or row locks for concurrent updates [Required]

Update counters with `F("count") + 1`, so the database does the work. For read-then-write, use `select_for_update()`, which must be inside `atomic` or it raises `TransactionManagementError`. `get_or_create` is atomic only when the fields have a unique constraint.

**Source:** Django Docs

## Migrations

### Commit migrations with their model changes [Recommended]

**Applies to:** team project

After running `makemigrations`, put the migration file and the model changes in the same commit, and give the migration a descriptive name with `-n`. Before applying, check the SQL with `sqlmigrate <app> <number>` and the plan with `migrate --plan`.

**Tooling:** CI runs `python manage.py makemigrations --check` and `migrate --check`.

**Source:** Django Docs

### Do not modify applied migrations [Required]

Do not edit or delete migrations already applied in a shared environment. Write changes as new migrations. When two branches conflict, merge them with `makemigrations --merge`. When there are too many migrations, compress them with `squashmigrations <app> <number>`. After confirming all environments have applied the result, delete the replaced files and remove the `replaces` attribute.

**Source:** Django Docs

### Use historical models in migrations [Required]

In `RunPython`, use `apps.get_model()`. Do not import models directly.

**Why:** A direct import gives you the model from the current code, which does not match the table schema at that migration.

**Source:** Django Docs

## Security settings

### Read secrets and switches from the environment [Required]

Read `SECRET_KEY` from an environment variable or key file, for example `os.environ["SECRET_KEY"]`. Do not share the production key with other environments. In production, set `DEBUG = False` and list the actual domains in `ALLOWED_HOSTS`.

**Tooling:** `S105`.

**Source:** Django Docs

### Pass the deployment check [Recommended]

**Applies to:** production service

CI runs `python manage.py check --deploy` with production settings. Do not deploy with `runserver`. Set `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SECURE_SSL_REDIRECT`, and `SECURE_HSTS_SECONDS`. When deploying behind a proxy, configure `SECURE_PROXY_SSL_HEADER` only after you understand the risks.

**Source:** Django Docs

### Bypass framework protections with care [Required]

Use `mark_safe`, `csrf_exempt`, `raw()`, `extra()`, and `RawSQL` only when truly necessary. User input must go through parameter binding. In `ModelForm`, list fields with `fields`, not `__all__` or `exclude`.

**Tooling:** `S308`, `S610`, `S611`, `DJ006`, `DJ007`.

**Source:** Django Docs, Ruff Docs

### Treat uploaded content as untrusted [Required]

Limit upload size at the web server layer. Serve uploaded files from a separate domain (a subdomain alone is not enough), and restrict the allowed extensions.

**Source:** Django Docs
