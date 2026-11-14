## 1. Reference Repos & Docs (Pin These)

* **Architecture & Conventions Playbook:** [zhanymkanov/fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices)
Read this once like a manual. It covers exact domain-module layouts (`router.py`, `schemas.py`, `models.py`, `service.py`, `dependencies.py`), when to use `async def` vs. `def` (so you don't block the event loop with sync DB/Redis calls), chaining `Depends()`, and Celery vs. `BackgroundTasks`.
* **Real Multi-Tenant Codebase to Grep:** [netflix/dispatch](https://github.com/netflix/dispatch)
Although archived, its Python backend is one of the best open-source examples of a modular, multi-tenant FastAPI + SQLAlchemy + Alembic + Pydantic application. Clone it locally and search (`rg`) inside `src/dispatch/` whenever an LLD asks for complex service patterns, mixins, or pagination.
* **SQLAlchemy 2.0 ORM Querying Guide:** [docs.sqlalchemy.org/en/latest/orm/queryguide/](http://docs.sqlalchemy.org/en/latest/orm/queryguide/)
Keep this open for exact 2.0 syntax: `select()`, `where()`, `join()`, `selectinload()` (to avoid N+1 queries on relationships), and ORM-level `update()` / `delete()`.
* **Pydantic V2 Validators & Config:** [pydantic.dev/docs/validation/2.0/usage/validators/](https://pydantic.dev/docs/validation/2.0/usage/validators/)
Bookmark this for `@field_validator`, `@model_validator`, `Annotated` types, and `model_config = ConfigDict(from_attributes=True)`.
* **Alembic Autogenerate Reference:** [alembic.sqlalchemy.org/en/latest/autogenerate.html](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)
Use this to check what `alembic revision --autogenerate` catches automatically versus what you must write manually inside `upgrade()` / `downgrade()` (such as custom constraints or data migrations).

---

## 2. How to Practice Translating an LLD into Code

Instead of following courses, pick one existing module in your Inventory Suite repo (like `Brand` or `Department`), imagine an LLD for a new adjacent module (e.g., `Category`, `Warehouse`, or `Supplier`), and build it end-to-end in four strict passes:

1. **Database (`models.py` + Alembic):** Translate the LLD's entity table into a SQLAlchemy 2.0 model using `Mapped[...]` and `mapped_column()`. Add the `tenant_id` foreign key and composite unique constraints (e.g., unique brand/category name *per tenant*, not globally). Generate the Alembic migration and read the generated file before running `upgrade head`.
2. **Contracts (`schemas.py`):** Translate the LLD's request/response payloads into `Base`, `Create`, `Update` (every field `Optional` defaulting to `None`), and `Response` (`from_attributes=True`).
3. **Repository & Service (`repository.py` / `service.py`):** Write the queries scoped strictly by `tenant_id`. Handle duplicate constraint violations by catching `sqlalchemy.exc.IntegrityError` and raising a clean domain exception.
4. **Router (`router.py`):** Keep endpoints thin—inject the DB session and current tenant via `Depends()`, call the service, and declare `response_model` and `status_code`.

---

## 3. How to Google Fast Without AI

Generic Google searches for FastAPI and SQLAlchemy are polluted with outdated 2021 tutorials using SQLAlchemy 1.4 (`db.query(...)`) and Pydantic V1 (`class Config:`). Use these search rules to hit exact answers on the first click:

* **Anchor searches to 2.0 / V2 keywords:**
* Instead of `sqlalchemy join filter`, search: `sqlalchemy 2.0 select join where mapped_column`
* Instead of `pydantic custom validation`, search: `pydantic v2 field_validator Annotated`
* Instead of `pydantic orm mode`, search: `pydantic v2 ConfigDict from_attributes`


* **Lock queries to official documentation with `site:`:**
* `site:docs.sqlalchemy.org/en/20/ selectinload`
* `site:fastapi.tiangolo.com Depends yield`
* `site:docs.pydantic.dev model_validator`


* **Search exact exception class names, not symptoms:**
When you hit a traceback, strip out your file paths and variable names and search the exact exception in quotes:
* `"sqlalchemy.exc.MissingGreenlet"` (happens when accessing a lazy-loaded relationship in async SQLAlchemy)
* `"pydantic_core._pydantic_core.ValidationError"`
* `"psycopg.errors.UniqueViolation"`


* **Grep your own repo first:**
Before Googling how to set up a Redis lock, enqueue a Celery task, or extract `tenant_id` from a JWT, search the existing `Brand`, `Department`, and `auth` directories in `Inventory-Suite`. Eighty percent of the wiring for any new LLD already exists in your repo's exact style.
* **Use `mypy`, `ruff`, and `/docs` as your live feedback loop:**
Run `ruff` and `mypy` in a terminal split while coding—`mypy` catches wrong SQLAlchemy return types and missing `None` checks before you even start the server. Keep `http://localhost:8000/docs` open to test edge-case JSON payloads directly against your Pydantic schemas.
