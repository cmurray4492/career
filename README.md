# Texas Trauma Coordinators Forum — Website

A mobile-first Django 5 + Bootstrap 5 web application for TTCF.org, ready to run locally, in Docker, and to deploy on Railway with automatic deploys from GitHub.

---

## What's included

| Area | Details |
|---|---|
| **Security roles** | Superadmin → Admin → Staff → Member → Guest. Each role inherits everything below it. Superadmin can only be granted by another Superadmin. Every page's required role is editable in the admin panel under **Core → Page security levels**. |
| **Home page** | Guest access. Hero, membership sign-up block, Resource Center link, forum preview showing the 3 most recent discussions, upcoming meetings band. |
| **Resource Center** | Member access (changeable). Admins upload/update/delete documents in the admin panel with metadata (category, author, version, tags) and a summary shown on the page. Keyword search included. |
| **Meeting Information** | Guest access. Chronological upcoming meetings; each has date, time, title, location, notes, and an external sign-up link. Clicking a meeting opens a detail page. |
| **Committees & Workgroups** | Member access. Pre-seeded with Level 1–4, Registry, Injury Prevention, Special Populations committees and the four workgroups. Sections and groups can be added/removed in the admin panel. Each group page shows a summary (placeholder text seeded) and its members; members link to their profile pages. |
| **Discussion Forum** | Guests view; Members post questions/answers and vote (▲/▼ with tally); Admins mark final answers and close questions. Every post shows name + date. Feeds the home page preview. |
| **Membership** | Public application form → emailed to all Superadmins → approve in the admin panel (grants the Member role automatically when the applicant has an account). |
| **Member profiles** | Name/email required; phone, city, ZIP, facility, summary, bio, photo optional. Extra fields can be added anytime in the admin (**Accounts → Profile fields**). Members edit their own; Superadmins can edit any (including Staff). |
| **Contact form** | Emails all Superadmins. |
| **Sitemap** | Live at `/sitemap.xml`, plus a `generate_sitemap` management command for a nightly static file (Railway cron instructions below). |

---

## 1. Run it locally (first time)

You need **Python 3.11+** installed. In a terminal, from the project folder:

```bash
# 1. Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Create the database tables
python manage.py makemigrations accounts core resources meetings committees forum
python manage.py migrate

# 4. Seed the default committees, workgroups, and page security levels
python manage.py seed_site

# 5. Create your Superadmin account
python manage.py createsuperuser

# 6. Start the site
python manage.py runserver
```

Open **http://127.0.0.1:8000** — the site. Open **http://127.0.0.1:8000/admin/** — the admin panel.

> **Important:** commit the `migrations/` folders that step 3 creates to git. They tell the production database how to build its tables.

While developing locally, emails (contact form, membership applications) are printed to the terminal instead of being sent, so you can test without an email account.

### Or run locally with Docker

```bash
docker compose up --build
```

Then visit http://localhost:8000. Create your admin account with:

```bash
docker compose exec web python manage.py createsuperuser
```

---

## 2. Put the code on GitHub (`cmurray4492`)

```bash
git init
git add .
git commit -m "Initial TTCF website"
git branch -M main
git remote add origin https://github.com/cmurray4492/ttcf-website.git
git push -u origin main
```

(Create the empty `ttcf-website` repository on github.com first: **New repository**, no README, private is fine.)

`.gitignore` already excludes `.env`, the local database, and uploaded files — **never commit `.env`**; that's what keeps your secret keys out of GitHub.

---

## 3. Deploy on Railway (beginner walkthrough)

### 3.1 Create the project

1. Go to [railway.app](https://railway.app) and sign in **with your GitHub account**.
2. Click **New Project → Deploy from GitHub repo** and pick `cmurray4492/ttcf-website`.
3. Railway detects the `Dockerfile` and starts building. The first deploy will crash — that's expected, we haven't added the database or secrets yet.

From now on, **every `git push` to `main` automatically redeploys the site.** That's the GitHub → Railway pipeline you asked for; nothing else to configure.

### 3.2 Add a PostgreSQL database

1. In your Railway project, click **+ New → Database → Add PostgreSQL**.
2. Click your **web service → Variables → + New Variable → Add Reference**, and pick **`DATABASE_URL`** from the Postgres service.

That's it — the app reads `DATABASE_URL` automatically.

### 3.3 Add your secret keys (environment variables)

In your **web service → Variables**, add:

| Variable | Value |
|---|---|
| `SECRET_KEY` | A long random string. Generate one: `python -c "import secrets; print(secrets.token_urlsafe(50))"` |
| `DEBUG` | `False` |
| `SITE_BASE_URL` | `https://<your-app>.up.railway.app` (you'll see the domain after step 3.4) |

Secrets in Railway Variables are encrypted and injected at runtime — this is the correct way to "upload" secret keys. They never touch GitHub.

You do **not** need to set `ALLOWED_HOSTS` for the Railway domain — the app picks up `RAILWAY_PUBLIC_DOMAIN` automatically. If you later attach a custom domain (e.g. `www.ttcf.org`), add:

| Variable | Value |
|---|---|
| `ALLOWED_HOSTS` | `www.ttcf.org` |
| `CSRF_TRUSTED_ORIGINS` | `https://www.ttcf.org` |

### 3.4 Get your public URL

Web service → **Settings → Networking → Generate Domain**. You'll get something like `ttcf-website-production.up.railway.app`.

### 3.5 Set up email (contact form + membership notifications)

The site emails all Superadmins when someone submits the contact or membership form. Add SMTP variables from any provider (a Gmail **App Password**, SendGrid, Mailgun, etc.):

| Variable | Example |
|---|---|
| `EMAIL_HOST` | `smtp.gmail.com` |
| `EMAIL_PORT` | `587` |
| `EMAIL_HOST_USER` | `you@gmail.com` |
| `EMAIL_HOST_PASSWORD` | your app password |
| `EMAIL_USE_TLS` | `True` |
| `DEFAULT_FROM_EMAIL` | `TTCF Website <you@gmail.com>` |

If these are not set, the app still works — emails are just skipped/logged.

### 3.6 Create your Superadmin in production

Install the Railway CLI (`npm i -g @railway/cli`), then:

```bash
railway login
railway link        # pick your project + web service
railway run python manage.py createsuperuser
```

(Alternatively use the **shell** tab on the service in the Railway dashboard.)

### 3.7 Keep uploaded files across deploys (important)

Railway's filesystem is wiped on every deploy. So resource documents and profile photos survive:

1. Web service → **Settings → Volumes → Add Volume**, mount path: `/data`.
2. Add variable `MEDIA_ROOT` = `/data/media`.
3. Redeploy.

### 3.8 Nightly sitemap

`/sitemap.xml` is always available live. To also generate the static nightly file per the spec, create a **cron job** in Railway:

1. **+ New → Empty Service** in the same project, connect the same GitHub repo.
2. Service **Settings → Cron Schedule**: `0 8 * * *` (08:00 UTC ≈ 2–3 AM Texas time).
3. Service **Settings → Deploy → Custom Start Command**:
   `python manage.py generate_sitemap --base-url $SITE_BASE_URL`
4. Give this service the same `DATABASE_URL` reference and `SECRET_KEY`/`SITE_BASE_URL` variables.

---

## 4. Day-to-day administration

Everything below happens in **`/admin/`**:

* **Approve members:** Accounts → Membership applications → select → action *"Approve selected applications"*.
* **Change a page's security level:** Core → Page security levels → pick the role per page.
* **Upload resources:** Resources → Documents → Add.
* **Manage meetings:** Meetings → Meetings → Add (date, time, title, location, notes, sign-up link).
* **Add/remove committees & workgroups:** Committees → Group sections / Committee groups; assign members and a chair there.
* **Add profile fields:** Accounts → Profile fields (they appear on every member's edit form automatically).
* **Elevate roles:** Accounts → Users → set *TTCF role*. Only a Superadmin can grant Superadmin.
* **Moderate the forum:** mark final answers and close questions directly on the question page (as an Admin) or in Forum → Questions.

## 5. Making changes later

```bash
# edit code …
git add .
git commit -m "Describe the change"
git push          # Railway redeploys automatically
```

If you ever change the models (database fields), also run
`python manage.py makemigrations` locally and commit the new migration files
before pushing.
