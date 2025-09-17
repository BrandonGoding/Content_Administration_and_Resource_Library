# The C.A.R.L.
**Content Administration & Resource Library** — a Django + Wagtail **multisite** with Tailwind (via `django-tailwind`) for theming each site.

## Tech Stack
- **Python / Django / Wagtail**
- **Tailwind CSS** (+ optional **DaisyUI**)
- **django-tailwind** for integrated Tailwind builds
- Optional: **pre-commit** with Black, isort, Flake8, mypy

---

## Project Layout (high-level)
```
.
├── manage.py
├── software_maine/                 # Django project (settings, urls, wsgi/asgi)
├── apps/...                        # Your Django/Wagtail apps
├── ui_theme/                       # django-tailwind app
│   ├── static_src/                 # Tailwind input(s)
│   ├── static/ui_theme/css/dist/   # Tailwind output(s)
│   └── tailwind.config.js
├── templates/
├── pyproject.toml
├── .flake8
└── .pre-commit-config.yaml
```

---

## Prerequisites
- Python 3.11+ (project targets 3.11; dev tooling may use 3.13)
- Node 18+ (for Tailwind build)
- PostgreSQL or SQLite (dev)

---

## Quick Start (Development)

### 1) Create & activate a virtualenv
```bash
python -m venv .venv
source .venv/bin/activate
pip install -U pip
```

### 2) Install Python deps
```bash
pip install -r requirements.txt
```

### 3) Configure environment
Create `.env` (or export env vars) with at least:
```
DJANGO_SETTINGS_MODULE=software_maine.settings
SECRET_KEY=change-me
DEBUG=1
DATABASE_URL=sqlite:///db.sqlite3           # or your Postgres URL
```

### 4) Database & admin user
```bash
python manage.py migrate
python manage.py createsuperuser
```

### 5) Tailwind deps
```bash
python manage.py tailwind install
```

---

## Running the Dev Server (Django + Tailwind)

### Option A — **Single theme build** (one CSS bundle; switch themes with DaisyUI)
Run these in **two terminals**:

**Terminal 1 — Tailwind watcher**
```bash
python manage.py tailwind start
```

**Terminal 2 — Django server**
```bash
python manage.py runserver
```

Your CSS will be served from `ui_theme/static/ui_theme/css/dist/styles.css` (the default path used by `django-tailwind` scaffolding).

### Option B — **Multiple bundles** (one CSS per site)
If you chose to output multiple CSS files (e.g., `fortkent.css`, `acsmaine.css`), use the NPM scripts you defined:

**Terminal 1 — Tailwind watcher**
```bash
# in ui_theme/
npm run dev   # runs both watchers via `concurrently`
```

**Terminal 2 — Django**
```bash
python manage.py runserver
```

---

## Multisite & Themes

### 1) Create sites in **Wagtail Admin**
1. Run the server and log in to `/admin/`.
2. Go to **Settings → Sites**.
3. Add a site per domain (or per host:port in dev):
   - **Hostname**: e.g., `localhost`
   - **Port**: e.g., `8000`
   - **Site name**: descriptive label
   - **Root page**: pick or create a home page for that site
   - **Is default site**: select one default for development
4. Repeat for each site (e.g., Fort Kent Cinema, ACS Maine, Software Maine).

### 2) Per-site theme selection
Add a Wagtail Site Setting that stores a theme key (one per site):

```python
# core/settings_models.py
from django.db import models
from wagtail.contrib.settings.models import BaseSiteSetting, register_setting

@register_setting
class ThemeSettings(BaseSiteSetting):
    THEME_CHOICES = [
        ("fortkent", "Fort Kent"),
        ("acsmaine", "ACS Maine"),
        ("softwaremaine", "Software Maine"),
    ]
    theme = models.CharField(max_length=50, choices=THEME_CHOICES, default="fortkent")
```

In your base template:

```django
{% load wagtailsettings_tags static %}
{% get_settings use_default_site=True as site_settings %}
<!doctype html>
<html lang="en" data-theme="{{ site_settings.core_themesettings.theme }}">
  <head>
    {# Option A: single bundle #}
    <link rel="stylesheet" href="{% static 'ui_theme/css/dist/styles.css' %}">
    {# Option B: multiple bundles #}
    {# <link rel="stylesheet" href="{% static 'ui_theme/css/dist/' %}{{ site_settings.core_themesettings.theme }}.css"> #}
  </head>
  <body class="min-h-screen bg-base-100 text-base-content">
    {% block content %}{% endblock %}
  </body>
</html>
```

### 3) Tailwind configuration

#### Option A (single bundle + DaisyUI multi-themes)
- Install DaisyUI in `ui_theme`:
  ```bash
  cd ui_theme
  npm i -D tailwindcss daisyui
  ```
- `ui_theme/tailwind.config.js`:
  ```js
  /** @type {import('tailwindcss').Config} */
  module.exports = {
    content: [
      '../../templates/**/*.html',
      '../../**/templates/**/*.html',
      '../../**/*.py',
    ],
    theme: { extend: {} },
    plugins: [require('daisyui')],
    daisyui: {
      themes: [
        {
          fortkent: {
            ...require("daisyui/src/theming/themes")["[data-theme=light]"],
            primary: "#0ea5e9",
          },
        },
        {
          acsmaine: {
            ...require("daisyui/src/theming/themes")["[data-theme=light]"],
            primary: "#4f46e5",
          },
        },
        {
          softwaremaine: {
            ...require("daisyui/src/theming/themes")["[data-theme=light]"],
            primary: "#16a34a",
          },
        },
      ],
    },
  };
  ```

Switching the `<html data-theme="...">` attribute flips the theme per site.

#### Option B (multiple bundles)
- Create multiple entry files under `ui_theme/static_src/styles/`:
  ```
  fortkent.css
  acsmaine.css
  softwaremaine.css
  ```
  Each:
  ```css
  @tailwind base;
  @tailwind components;
  @tailwind utilities;

  /* site-specific overrides here */
  ```
- Add NPM scripts in `ui_theme/package.json` to watch/build each CSS to `static/ui_theme/css/dist/`.

---

## Production Build
```bash
# Tailwind (Option A)
python manage.py tailwind build
# Tailwind (Option B)
cd ui_theme && npm run build

# Django static
python manage.py collectstatic
```

---

## Pre-commit (optional)
Install once:
```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files
```
The repo is set up to run **isort → Black → Flake8 → mypy** before each commit.

---

## Troubleshooting
- **CSS not updating?** Ensure the Tailwind watcher is running and that `tailwind.config.js` `content` paths include your templates and classnames.
- **Theme not changing?** Confirm the per-site **ThemeSettings** value in Wagtail Admin and check your base template.
- **404 for CSS in dev?** Verify outputs are written to `ui_theme/static/ui_theme/css/dist/` and static settings are correct.
- **Multisite routing issues?** Check **Settings → Sites** in Wagtail to match host/port.

---

## Common Commands
```bash
# Dev (Option A)
python manage.py tailwind start
python manage.py runserver

# Dev (Option B)
(cd ui_theme && npm run dev)   # watchers
python manage.py runserver

# Build
python manage.py tailwind build
python manage.py collectstatic

# Admin
python manage.py createsuperuser
```

---

## License
Choose a license appropriate for your goals. If you **don’t** want reuse, consider keeping the repo private or using a restrictive notice (note: most open-source licenses allow reuse).
