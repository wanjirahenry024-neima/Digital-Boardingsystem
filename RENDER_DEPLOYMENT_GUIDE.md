# Deploying the Digital Boarding Pass System on Render Free

This repository is a Laravel application. The included `render.yaml` and
Dockerfile configure it as a Render Docker web service, build the Vite assets,
connect Laravel to PostgreSQL, run migrations when the container starts, and
serve the application on port `10000`.

## Important free-plan limitations

- Render Free web services sleep after 15 minutes without traffic. The first
  request after sleep can take about a minute while the service starts.
- A free Render web service has an ephemeral filesystem. Files written locally,
  including SQLite databases and uploads, are lost on restart, spin-down, or
  redeploy. This deployment therefore stores application records, sessions,
  and cache in PostgreSQL, not in a local SQLite file.
- Render Free PostgreSQL databases expire 30 days after creation. They are
  suitable for a temporary demo, not durable free storage. Use a PostgreSQL
  provider with a free plan that suits your needs (for example, Neon Free as of
  this guide's writing), or accept the Render database's expiration and arrange
  an export/upgrade before it expires. Free-plan features and limits can
  change; check the provider's current terms.
- Render Free web services cannot send SMTP traffic on ports 25, 465, or 587.
  Configure an email service using a supported connection method/port if
  password-reset emails need to be delivered. The default Laravel `log` mailer
  does not deliver email.
- This free setup is for learning, demos, and hobby use, not production
  workloads that require uptime, backups, or support.

## What is already configured

- `render.yaml` declares a Free Docker web service and its production
  environment settings.
- The Docker image builds frontend assets and installs PHP's PostgreSQL PDO
  driver.
- The container runs `php artisan migrate --force` before starting Apache.
- Laravel uses PostgreSQL for application data, database sessions, and cache.
- Production-generated URLs and session cookies use HTTPS.
- Render's `/up` health endpoint is configured for health checks.

No separate Render build command is needed for a Docker service: Render builds
the repository's Dockerfile. Do not set the service's **Root Directory** to
`public`; it must remain the repository root.

## Step 1: Push the project to GitHub

Render deploys from a Git provider. Commit and push the project, including
`Dockerfile`, `.dockerignore`, and `render.yaml`. Do not commit `.env`, database
credentials, or other secrets.

## Step 2: Create a PostgreSQL database

### Recommended for a longer-running free demo

Create a PostgreSQL database with a provider that currently offers a suitable
free plan. For Neon:

1. Create a Neon account and a new PostgreSQL project.
2. Select a region close to the Render service's region (the Blueprint defaults
   to Oregon).
3. Use **Connect** in the database dashboard and copy a PostgreSQL connection
   URI. Prefer the pooled connection string if the provider recommends it for
   serverless clients.
4. Keep the connection URI private. You will enter it into Render as `DB_URL`.

The database URI should start with `postgresql://` or `postgres://`. For a
provider that requires TLS, its URI should include the provider's required SSL
option, commonly `sslmode=require`.

### Render's own free PostgreSQL option

You can instead create a Render Free PostgreSQL database and use its external
connection URI for `DB_URL`. Render documents this database as expiring 30 days
after creation; after a further 14-day grace period it deletes the database
and its data. It is not a permanent free database.

## Step 3: Create the Render Blueprint

1. Sign in to the [Render Dashboard](https://dashboard.render.com/).
2. Choose **New** > **Blueprint**.
3. Connect the GitHub repository and select the branch containing `render.yaml`.
4. Review the Blueprint. It creates a web service named
   `digital-boarding-pass-system` with the **Free** plan, in Oregon, using the
   repository's Dockerfile.
5. If Render asks for values for variables marked `sync: false`, enter:
   - `DB_URL`: the private PostgreSQL connection URI from Step 2.
   - `APP_URL`: the final service URL, such as
     `https://digital-boarding-pass-system.onrender.com`.
6. Apply the Blueprint and wait for the deployment.

The Blueprint generates `APP_KEY` and sets it for the service. Keep that key
stable across redeployments; do not generate or replace it for each deploy.

If Render creates the service before you can enter its secrets, open the
service's **Environment** settings, add `DB_URL` and `APP_URL`, save the
changes, and trigger a new deploy. A deployment cannot complete its startup
migration until `DB_URL` points to a reachable, initialized PostgreSQL
database.

If the generated Render domain differs from the example, set `APP_URL` to the
exact HTTPS URL shown on the service page, with no trailing slash, then save
and redeploy.

## Step 4: Verify the environment

In the service's **Environment** settings, confirm these values:

| Key | Value |
| --- | --- |
| `APP_ENV` | `production` |
| `APP_DEBUG` | `false` |
| `APP_KEY` | Generated by Render; keep it private and stable |
| `APP_URL` | The exact HTTPS Render service URL |
| `DB_CONNECTION` | `pgsql` |
| `DB_URL` | Your PostgreSQL connection URI |
| `SESSION_DRIVER` | `database` |
| `SESSION_SECURE_COOKIE` | `true` |
| `CACHE_STORE` | `database` |
| `QUEUE_CONNECTION` | `sync` |
| `LOG_CHANNEL` | `stderr` |
| `GATE_TERMINAL_MODE` | `false` |

`DB_URL` must be set as a secret environment variable in Render, not committed
to the repository. The repository's Laravel PostgreSQL connection reads this
variable directly.

## Step 5: Check the deployment

1. Open the Render service's **Logs** and confirm the image builds successfully.
2. Confirm the startup log shows migrations completing and Apache starting.
3. Open the service's HTTPS URL. The `/up` health check should report the
   service healthy.
4. Test registration and login, then verify you remain logged in when moving
   between pages.
5. Try flight browsing and other key workflows.
6. In the database provider, confirm the Laravel tables were created.

Do not run `php artisan migrate --seed` against this database. The project
seeder includes sample data, and test/demo accounts are not appropriate for a
public service. Create any required production data and administrator account
through a controlled process.

## Step 6: Subsequent updates

With `autoDeployTrigger: commit`, pushing to the connected branch triggers a
new Docker build and deploy. The startup command applies any pending
migrations before Apache starts. Check Render logs after each deploy.

Use backward-compatible database migrations when possible. The free service
does not provide a separate pre-deploy migration command, so migrations run as
the container starts.

## If deployment fails

- **Database connection or migration error:** verify `DB_URL`, that the
  PostgreSQL database is active, and that its URI includes any required TLS
  settings. Review Render's service logs.
- **Missing PostgreSQL driver:** confirm your service built the latest commit
  containing `pdo_pgsql` in the Dockerfile; trigger a new deploy.
- **Service fails its health check:** confirm the startup log has no migration
  error and visit `/up` on the service's HTTPS URL.
- **Slow first page:** this is expected when a Free web service has spun down.
- **Data disappears:** verify the app uses the external PostgreSQL database;
  never use the Render container's local SQLite file for persisted data.
- **Password-reset messages do not arrive:** configure a real mail provider
  and verify it supports a connection method/port available from Render Free.

## Sources

- [Render: Docker deployments](https://render.com/docs/docker)
- [Render: Free instance limitations](https://render.com/docs/free)
- [Render: Deploy Laravel with Docker](https://render.com/docs/deploy-php-laravel-docker)
- [Render: Blueprint specification](https://render.com/docs/blueprint-spec)
- [Neon: Connect from Laravel](https://neon.com/docs/guides/laravel)
