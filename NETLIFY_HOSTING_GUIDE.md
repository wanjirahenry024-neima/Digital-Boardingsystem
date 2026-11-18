# Hosting the Digital Boarding Pass System

## Important: Netlify cannot run this application as it is

This repository is a server-rendered Laravel 13 application. Its Blade pages,
login and session handling, database-backed booking workflows, and other
routes require a PHP application server and a persistent relational database.
Netlify can build and publish static files and run its supported serverless
functions, but it does not run this repository as a conventional Laravel/PHP
web server.

In particular:

- `npm run build` only compiles the Vite assets in `public/build`; it does not
  turn the Laravel application into a static website.
- Publishing `public/` or `public/build/` on Netlify will not make Laravel
  routes, Blade templates, login, bookings, or migrations work.
- The application uses Laravel session authentication and CSRF-protected web
  routes. These cannot simply be pointed at a static Netlify deployment.

**Recommended:** deploy the complete application to a host that runs PHP and
MySQL, such as [Laravel Cloud](https://cloud.laravel.com/). Netlify is suitable
for an independent static marketing site, but that would be a separate site;
the Laravel application still needs its own backend host.

## What you need

- The project repository on GitHub, GitLab, or Bitbucket.
- A Laravel/PHP host that supports PHP 8.3 or newer and Laravel 13.
- A production MySQL database. The application can use SQLite locally, but its
  production filesystem/database must be durable; this guide uses MySQL.
- An application encryption key (`APP_KEY`).
- A mail provider and SMTP credentials if password-reset email is required.
- A domain name (optional to begin with; a host-provided domain can be used
  first).
- A persistent session/cache store. This guide uses the application database.

The repository includes Composer and npm lockfiles, a Vite production build
script, and a Dockerfile for Docker-capable hosts. Do not commit `.env` or
production credentials.

## Recommended deployment: Laravel Cloud

Laravel Cloud provides a PHP runtime and managed database resources. The
dashboard labels can change, so use the current environment settings shown in
your account.

### 1. Prepare the repository

1. Make sure your latest working code is committed and pushed to your Git
   provider.
2. Verify `composer.lock` and `package-lock.json` are in the repository.
3. Keep production secrets out of Git. `.env` should remain local and ignored.
4. Do not run the development server (`php artisan serve`) as your production
   server.

### 2. Create the production session-table migration

The project's checked-in migrations create the database cache tables but do
not currently create Laravel's database-backed session table. Since hosted
application filesystems can be replaced during deployment, database-backed
sessions are a safer starting point than file-backed sessions.

From the project directory, create the migration:

```sh
php artisan make:session-table
```

Commit the generated migration and push it before deploying. It will be applied
with the other migrations in step 5.

### 3. Create the Laravel Cloud application

1. Sign in to Laravel Cloud and choose **New application**.
2. Connect your Git provider and select this repository.
3. Select the production branch and a region close to your users.
4. Confirm the application uses the **PHP** runtime and select PHP 8.3 or a
   newer version supported by the host.
5. Add a managed **MySQL** database in the same region as the application, and
   attach it to the environment.

The host provides database connection values after you attach the database.
Do not copy the example local database password into production.

### 4. Set build and deployment commands

Use the host's build-command setting for dependency installation, Vite asset
compilation, and Laravel optimization:

```sh
composer install --no-dev
npm ci --audit false
npm run build
php artisan optimize
```

Use the deploy-command setting to apply database migrations:

```sh
php artisan migrate --force
```

Migrations update the production schema. Back up the database before deploying
schema changes to an existing production system.

### 5. Configure production environment variables

Set these in the hosting provider's environment-variable settings, not in Git:

| Variable | Production value |
| --- | --- |
| `APP_NAME` | `Digital Boarding Pass System` |
| `APP_ENV` | `production` |
| `APP_DEBUG` | `false` |
| `APP_URL` | Your final `https://` application URL |
| `APP_KEY` | Generate as shown below; keep the same key across deployments |
| `DB_CONNECTION` | `mysql` |
| `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` | Values supplied for the attached production database |
| `SESSION_DRIVER` | `database` |
| `CACHE_STORE` | `database` |
| `QUEUE_CONNECTION` | `sync` to start with |
| `MAIL_MAILER` | A real mail transport such as `smtp`, not `log` |
| `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM_ADDRESS` | Values from your mail provider |
| `GATE_TERMINAL_MODE` | `false` for the central hosted application |

Generate a new key once, from your local project directory:

```sh
php artisan key:generate --show
```

Copy the printed value into the provider's `APP_KEY` setting. Do not share it
or commit it. **Do not generate a new key on every deployment**: changing it
can invalidate sessions and data encrypted with the old key.

If you do not yet have SMTP details, the app may start with the log mailer, but
password-reset emails will not be delivered to users. Configure a mail provider
before relying on password reset in production.

The project has no database session migration yet; complete step 2 before
setting `SESSION_DRIVER=database`. The database cache tables are included in
the current migrations, so `CACHE_STORE=database` can use them.

### 6. Deploy and verify

1. Deploy the production branch.
2. Check the deployment/build logs for failed Composer/npm commands or
   migrations.
3. Open the production URL and verify the welcome page, registration, login,
   flight search, and booking flow.
4. Verify a password-reset email using the configured mail provider.
5. Confirm the database contains the expected schema and that sessions persist
   between requests.
6. Add your custom domain in the host dashboard, update `APP_URL` to the final
   HTTPS URL, and redeploy if needed. Enable/confirm HTTPS.

### 7. Create production accounts and data safely

Do **not** run `php artisan migrate --seed` against production by default. This
project's seeder adds sample flights, and its passenger/staff/admin test users
with the password `password` are intended for local/testing environments.
Public registration creates passenger accounts; it does not create an
administrator.

Provision the first administrator through a deliberate, secured process, then
verify that development/test credentials are not present in the production
database. Do not expose an unauthenticated account-promotion route.

## If you must use Netlify

### Option A: Netlify for a separate static site

You can create a separate static marketing site on Netlify (for example, a
landing page linking to the hosted Laravel application). Deploy that separate
site from a static-site repository. The Laravel application remains deployed
to a PHP host, and users use that host's URL for registration, login, bookings,
and administration.

Do not configure this Laravel repository's publish directory as `public/` or
`public/build/` and expect the full app to work. Vite's output is only the
compiled CSS and JavaScript assets.

### Option B: Rebuild the frontend as a separate Netlify app

This requires a development project, not just Netlify build settings:

1. Keep Laravel on a PHP host and expose a properly designed API.
2. Build a separate frontend (for example, a JavaScript application) that
   Netlify can build and publish.
3. Replace the current server-rendered Blade screens with that frontend.
4. Deliberately redesign and test authentication, cookies, CSRF, CORS, file
   downloads, and API authorization across the separate frontend/backend
   origins.
5. Configure Netlify's build settings for the new frontend and point it to the
   API URL.

This is a larger architecture change. It is not achieved by adding a
`netlify.toml` file to the current Laravel repository.

## Docker-capable hosting alternative

The repository includes a `Dockerfile` that builds the Vite assets and runs
Laravel under Apache. A provider that supports deploying a Dockerfile can be
used instead of Laravel Cloud:

1. Create a web service from this repository and select its Dockerfile.
2. Configure the service to listen on port `10000`, as specified by the
   repository's Dockerfile.
3. Create or attach a production MySQL database and set the `DB_*` values in
   the service's secret environment settings.
4. Set the same production application, key, session, cache, and mail
   environment variables described above.
5. Configure the service's pre-deploy/release command to run
   `php artisan migrate --force`, if the provider supports one.
6. Ensure application state that must survive deployments (database sessions,
   cache, and any user-uploaded files) is stored in durable database/object
   storage, not only in the container filesystem.

The repository's Docker build does not copy `.env`; configure secrets in the
provider dashboard. Do not bake credentials into the image.

## Production checklist

- [ ] Production PHP runtime is 8.3 or newer and supports Laravel 13.
- [ ] Production uses MySQL, not the local SQLite database file.
- [ ] `APP_ENV=production`, `APP_DEBUG=false`, and `APP_URL` uses HTTPS.
- [ ] A stable, private `APP_KEY` is configured.
- [ ] Database session migration is committed and applied.
- [ ] `SESSION_DRIVER=database` and `CACHE_STORE=database` are configured.
- [ ] SMTP is configured if password resets or other application mail is needed.
- [ ] Migrations run with `--force`; production seeding is not run blindly.
- [ ] No local test accounts or default passwords are used in production.
- [ ] Database backups and a recovery plan are in place.
- [ ] The provider's logs are monitored after deployment.
- [ ] A custom domain and HTTPS are configured when ready.

## Further reading

- [Netlify: deploy a project from a Git repository](https://docs.netlify.com/start/quickstarts/deploy-from-repository/)
- [Netlify: configure builds](https://docs.netlify.com/build/configure-builds/overview/)
- [Netlify: Functions overview](https://docs.netlify.com/build/functions/overview/)
- [Laravel Cloud: quickstart](https://cloud.laravel.com/docs/quickstart.md)
- [Laravel Cloud: Laravel deployment guide](https://cloud.laravel.com/docs/deploy-guides/laravel.md)
- [Laravel Cloud: deployments](https://cloud.laravel.com/docs/deployments.md)
