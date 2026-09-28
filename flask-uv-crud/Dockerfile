FROM python:3.14-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

ENV UV_PYTHON_DOWNLOADS=never

WORKDIR /app

# Dependency metadata first
COPY pyproject.toml uv.lock .python-version ./

# Install exactly what is locked
RUN uv sync --frozen --no-dev --no-install-project

# Application code afterwards
COPY . .


RUN uv sync --frozen --no-dev

RUN useradd --create-home appuser \
    && chown -R appuser:appuser /app

USER appuser

EXPOSE 8000

CMD ["uv", "run", "--no-dev", "gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "api:app"]