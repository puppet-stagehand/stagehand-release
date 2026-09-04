# stagehand-release

Public container mirror and quick-start for the [Puppet Stagehand](https://github.com/puppet-stagehand/stagehand-console) console -- no installer required.

The console's source repo is private. This repo mirrors its published container images here, publicly, so anyone can pull and run the console directly with Docker.

## Quick start (Docker Compose)

Requires Docker and Docker Compose.

```bash
git clone https://github.com/puppet-stagehand/stagehand-release.git
cd stagehand-release

# Generate a one-time encryption key. Save it -- if you lose it, encrypted
# data (connection credentials, secrets) already stored by the console
# becomes unreadable.
echo "PSH_SECRET_KEY=$(openssl rand -base64 32)" > .env

docker compose up -d
```

Then open **http://localhost:8767** and complete the first-run setup wizard (creates the console's one Global Administrator account).

This starts two containers: the console itself, and a Postgres 17 database for its own data. It does **not** include Puppet Server or PuppetDB -- the console doesn't need either to boot. Connect it to your existing Puppet installation (PE, Core, or OpenVox) from inside the console after setup: **Estate Viewer -> Add instance**, or **Settings -> Connections** for the default instance. There's no environment variable for this -- it's entirely UI-driven, and the wizard won't let you confirm an instance until every connection you fill in has actually passed its own test.

To stop it:

```bash
docker compose down       # keeps your data (the psh-pgdata volume)
docker compose down -v    # also deletes your data
```

## Manual Docker (no Compose)

If you'd rather run the two containers yourself:

```bash
docker network create psh-net

docker run -d --name psh-postgres --network psh-net \
  -e POSTGRES_USER=psh -e POSTGRES_PASSWORD=psh -e POSTGRES_DB=psh \
  postgres:17-alpine

export PSH_SECRET_KEY=$(openssl rand -base64 32)

docker run -d --name psh-console --network psh-net \
  -p 8767:8767 \
  -e PSH_SECRET_KEY="$PSH_SECRET_KEY" \
  -e PSH_DATABASE_URL="postgres://psh:psh@psh-postgres:5432/psh?sslmode=disable" \
  ghcr.io/puppet-stagehand/stagehand-release/console:latest
```

## Image tags

| Tag | What it tracks |
|---|---|
| `latest` | The most recently mirrored `test-pilots` build |
| `test-pilots` | Mirrors the console repo's `main` branch -- frequent, less tested |
| `<version>` (e.g. `1.0.2`) | A specific tagged release |

```
ghcr.io/puppet-stagehand/stagehand-release/console:latest
ghcr.io/puppet-stagehand/stagehand-release/console:1.0.2
```

Pull a specific version instead of `latest` for anything other than casual testing.

## Required environment variables

| Variable | Required | Notes |
|---|---|---|
| `PSH_SECRET_KEY` | Yes | Base64-encoded 32-byte key (`openssl rand -base64 32`). Used to encrypt stored secrets (connection credentials, etc). Losing it makes existing encrypted data unreadable. |
| `PSH_DATABASE_URL` | No (has a local-dev default) | Postgres connection string. Default is `postgres://psh:psh@localhost:5432/psh?sslmode=disable` -- override this for anything beyond local testing. |
| `PSH_ADDR` | No | Listen address. Defaults to `:8767`. |

The console has no environment variable for connecting to Puppet/PuppetDB -- see "Quick start" above.

## What this is not

This is not the supported production install path. For a production deployment -- OS package management, TLS, backups, upgrades -- use the [Puppet Installer](https://github.com/puppet-stagehand/stagehand-installer). This repo exists for evaluating the console quickly, or for environments where the installer's assumptions don't fit.

## Source

The console's source, issue tracker, and full documentation live in the (private) [stagehand-console](https://github.com/puppet-stagehand/stagehand-console) repo. This repo only mirrors its release images and hosts this quick-start.
