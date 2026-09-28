Exactly. Since you already know **Flask, SQLAlchemy, CRUD and Docker**, don't spend time learning Flask here. The lab should use a familiar Flask API purely as a vehicle to understand **uv deeply**.

The key question is:

> **"What problem does uv solve in my Flask project's lifecycle, and how does that lifecycle change from local development → CI → Docker → production?"**

# Quick Start — Mastering `uv` with a Production Flask API

We'll build:

```text
Flask CRUD API
      │
      ▼
SQLAlchemy
      │
      ▼
   MySQL

       +
       
       uv
       │
       ├── Python version
       ├── Project creation
       ├── Dependencies
       ├── Virtual environment
       ├── Lock file
       ├── Dependency sync
       ├── Run commands
       └── Docker builds
```

The Flask code itself is intentionally boring.

---

# 1. What `uv` replaces

If you previously used:

```text
python
python -m venv
pip
pip-tools / requirements.txt
pyenv
```

`uv` brings most of that workflow together.

Think:

```text
                  uv
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Python     Packages    Project
     versions   + locking   workflow
        │          │
        └────┬─────┘
             ▼
          .venv
```

For your Flask project, the important files are:

```text
pyproject.toml
uv.lock
.venv/
```

Usually:

```text
pyproject.toml  → What the project depends on
uv.lock          → Exactly what versions were resolved
.venv/           → Local environment
```

**`uv.lock` is the big concept to master.**

---

# 2. Create the Flask project

```bash
uv init flask-uv-crud
cd flask-uv-crud
```

Inspect:

```bash
cat pyproject.toml
```

You will have a Python project definition.

Now tell uv which Python version you want:

```bash
uv python install 3.12
```

Then:

```bash
uv python pin 3.12
```

This creates:

```text
.python-version
```

So now the repository says:

```text
This project → Python 3.12
```

That's one of the first important `uv` concepts.

---

# 3. Add Flask dependencies

```bash
uv add flask
uv add flask-sqlalchemy
uv add pymysql
uv add gunicorn
```

Development dependencies:

```bash
uv add --dev pytest
```

Now inspect:

```bash
cat pyproject.toml
```

You'll have something conceptually like:

```toml
[project]
name = "flask-uv-crud"
version = "0.1.0"
requires-python = ">=3.12"

dependencies = [
    "flask",
    "flask-sqlalchemy",
    "gunicorn",
    "pymysql",
]

[dependency-groups]
dev = [
    "pytest",
]
```

And uv creates:

```text
uv.lock
```

---

# 4. Understand `uv.lock`

This is the most important part of your learning.

Suppose:

```text
pyproject.toml

Flask
SQLAlchemy
PyMySQL
Gunicorn
```

Those packages themselves have dependencies.

For example:

```text
Flask
 ├── Werkzeug
 ├── Jinja2
 ├── Click
 └── ItsDangerous

SQLAlchemy
 └── greenlet
```

`uv` resolves this dependency graph.

```text
                 pyproject.toml
                       │
                       ▼
                 Dependency graph
                       │
                       ▼
                    Resolver
                       │
                       ▼
                    uv.lock
```

So:

### `pyproject.toml`

```text
"I need Flask"
```

### `uv.lock`

```text
"Here is the exact resolved dependency graph
that this project should use."
```

This is why you commit:

```bash
git add pyproject.toml uv.lock .python-version
git commit
```

but normally **don't commit `.venv/`**.

---

# 5. `uv sync`

Now:

```bash
uv sync
```

This creates:

```text
.venv/
```

and installs the locked dependencies.

Your environment becomes:

```text
Project
 │
 ├── pyproject.toml
 ├── uv.lock
 ├── .python-version
 │
 └── .venv/
       ├── Flask
       ├── SQLAlchemy
       ├── PyMySQL
       └── Gunicorn
```

This is conceptually different from:

```bash
pip install flask
```

You aren't simply installing a package.

You're saying:

> Make my environment match the project's declared and locked dependency state.

---

# 6. Run Flask through uv

You don't even need to manually activate `.venv`.

Instead:

```bash
uv run flask --app app run
```

or:

```bash
uv run pytest
```

or:

```bash
uv run gunicorn ...
```

Think of:

```bash
uv run <command>
```

as:

> Run this command inside the project's managed environment.

This is a very useful habit to develop.

---

# 7. Your Flask API

Keep the application simple.

```text
flask-uv-crud/
│
├── app/
│   ├── __init__.py
│   ├── db.py
│   ├── models.py
│   └── routes.py
│
├── tests/
│
├── Dockerfile
├── compose.yaml
├── pyproject.toml
├── uv.lock
└── .python-version
```

For example:

```python
# app/__init__.py

import os

from flask import Flask
from .db import db
from .routes import users_bp


def create_app():
    app = Flask(__name__)

    app.config["SQLALCHEMY_DATABASE_URI"] = os.environ["DATABASE_URL"]
    app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

    db.init_app(app)

    app.register_blueprint(users_bp, url_prefix="/api")

    return app
```

Your normal SQLAlchemy CRUD implementation can remain exactly as you're accustomed to.

---

# 8. Now the interesting part — Docker + uv

This is where you should pay attention.

A production-oriented Dockerfile:

```dockerfile
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Dependency metadata first
COPY pyproject.toml uv.lock .python-version ./

# Install exactly what is locked
RUN uv sync --frozen --no-dev

# Application code afterwards
COPY app ./app

RUN useradd --create-home appuser \
    && chown -R appuser:appuser /app

USER appuser

EXPOSE 8000

CMD [
    "uv",
    "run",
    "--no-dev",
    "gunicorn",
    "--bind", "0.0.0.0:8000",
    "--workers", "2",
    "app:create_app()"
]
```

Now understand this line:

```dockerfile
RUN uv sync --frozen --no-dev
```

### Why `--frozen`?

Production shouldn't modify your lock file.

You already committed:

```text
uv.lock
```

So:

```text
CI / Production
      │
      ▼
pyproject.toml + uv.lock
      │
      ▼
uv sync --frozen
      │
      ▼
exact locked environment
```

If your lock file is inconsistent with `pyproject.toml`, the build fails rather than silently resolving something new.

That's exactly what you want in reproducible builds.

---

# 9. Why copy dependency files first?

Notice:

```dockerfile
COPY pyproject.toml uv.lock .python-version ./

RUN uv sync --frozen --no-dev

COPY app ./app
```

Not:

```dockerfile
COPY . .
RUN uv sync
```

Because Docker caching.

Imagine you modify:

```text
app/routes.py
```

The dependency files haven't changed.

Therefore:

```text
Docker build

pyproject.toml ─┐
uv.lock         ├──> dependency layer ── cached
.python-version ┘

app/routes.py ───────> application layer ── rebuilt
```

This is one of the nice places where **uv + Docker work together naturally**.

---

# 10. Docker Compose

```yaml
services:

  api:
    build: .
    ports:
      - "8000:8000"

    environment:
      DATABASE_URL: mysql+pymysql://appuser:apppassword@mysql:3306/appdb

    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.4

    environment:
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
      MYSQL_ROOT_PASSWORD: rootpassword

    volumes:
      - mysql_data:/var/lib/mysql

    healthcheck:
      test:
        [
          "CMD",
          "mysqladmin",
          "ping",
          "-h",
          "localhost",
          "-u",
          "root",
          "-prootpassword"
        ]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  mysql_data:
```

Start:

```bash
docker compose up --build
```

---

# 11. The complete `uv` workflow you should master

This is the cheat sheet I'd actually recommend memorizing:

### Create project

```bash
uv init
```

### Select Python

```bash
uv python install 3.12
uv python pin 3.12
```

### Add runtime dependency

```bash
uv add flask
```

### Add development dependency

```bash
uv add --dev pytest
```

### Remove dependency

```bash
uv remove flask
```

### Create/update environment

```bash
uv sync
```

### Run command

```bash
uv run pytest
uv run flask --app app run
```

### Inspect dependency tree

```bash
uv tree
```

This is particularly useful for understanding transitive dependencies.

### Update dependencies

```bash
uv lock --upgrade
```

### Check project

```bash
uv lock --check
```

### Production installation

```bash
uv sync --frozen --no-dev
```

---

# 12. The mental model I want you to have

Don't think:

> **uv = faster pip**

That's true, but too shallow for your goal.

Think:

```text
                    UV
                     │
       ┌─────────────┼──────────────┐
       │             │              │
       ▼             ▼              ▼
    Python        Project        Packages
   management     workflow       management
       │             │              │
       │             ▼              │
       │        pyproject.toml       │
       │             │              │
       │             ▼              │
       │          Resolver           │
       │             │              │
       │             ▼              │
       │          uv.lock            │
       │             │              │
       └─────────────┼──────────────┘
                     ▼
                   .venv
                     │
                     ▼
                 uv run
                     │
                     ▼
              Flask / Gunicorn
```

And then Docker takes that reproducible project definition and turns it into a production image:

```text
Developer machine
       │
       │ uv add
       ▼
pyproject.toml
       │
       │ uv lock
       ▼
uv.lock
       │
       │ git
       ▼
CI
       │
       │ uv sync --frozen
       ▼
Docker image
       │
       ▼
Production
```

## Your actual `uv` mastery checklist

For this lab, don't spend time studying Flask. Experiment with these scenarios:

```text
1. uv init
2. uv python pin
3. uv add
4. uv remove
5. uv sync
6. uv run
7. uv tree
8. uv.lock
9. dependency groups
10. uv lock --upgrade
11. uv sync --frozen
12. uv + Docker caching
13. uv in CI
14. production dependency-only installation
15. multi-stage Docker builds with uv
```

Once you understand **why each of those exists and when you'd use it**, you'll have the useful mental model of `uv` rather than merely knowing its commands.
