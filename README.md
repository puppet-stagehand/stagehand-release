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

### Connecting to your Puppet Server / PuppetDB

The console talks to Puppet Server and PuppetDB over mutual-TLS -- it needs its own
client certificate, signed by your Puppet CA, the same way a real Puppet agent does.
When installed via the Puppet Installer this cert is generated automatically; running
standalone, you generate it yourself, once:

1. **On your Puppet primary server**, generate and auto-sign a client cert for the console:
   ```bash
   sudo /opt/puppetlabs/bin/puppetserver ca generate --certname console.example.com --ca-client
   ```
   (Use any certname you like -- it just identifies the console to your Puppet CA.)

2. **Copy the three resulting files** to a `certs/` folder next to this repo's `docker-compose.yaml`:
   ```bash
   mkdir -p certs
   scp your-primary:/etc/puppetlabs/puppet/ssl/certs/console.example.com.pem       certs/console.pem
   scp your-primary:/etc/puppetlabs/puppet/ssl/private_keys/console.example.com.pem certs/console.key
   scp your-primary:/etc/puppetlabs/puppet/ssl/certs/ca.pem                        certs/ca.pem
   ```
   `docker-compose.yaml` already mounts `./certs` into the container at `/certs`, read-only.

3. **In the console**, go to **Settings -> Connections** and fill in each service (Puppet Server, PuppetDB):
   - **Host**: your Puppet primary's address, reachable from inside the container.
     If Puppet runs on the *same* machine as Docker Desktop (macOS/Windows), use
     `host.docker.internal`. Otherwise use its real hostname or IP.
   - **Cert path**: `/certs/console.pem`
   - **Key path**: `/certs/console.key`
   - **CA cert path**: `/certs/ca.pem`
   - Click **Save & test** -- it won't let you save until the connection actually works.

   Puppet Server defaults to port 8140, PuppetDB to 8081.

If you're managing more than one Puppet installation, each registered **instance** in
Estate Viewer gets its own set of these connections -- switch the active instance in
the sidebar first to edit a different one.

### Task execution (Bolt)

Patching, r10k control-repo deploys, and other task/plan runs execute through
[Puppet Bolt](https://www.puppet.com/docs/bolt/latest/bolt.html), which the
console image already bundles -- there's nothing extra to install. It does,
though, need a writable **Bolt project directory** to keep its task module,
project manifest, and a persistent SSH keypair in; `docker-compose.yaml`
already points it at one (`PSH_BOLT_PROJECT=/bolt-project`, backed by the
`bolt-project` named volume) and the console stages everything into it
automatically on boot -- no separate module install or `bolt project init`
step. Settings -> Execution shows the exact path this console resolved and
where each check landed; **Service health** on the dashboard rolls that up
into a single "Bolt execution" status.

If you're running the containers manually (below) rather than through
Compose, remember to pass the same `-e PSH_BOLT_PROJECT=/bolt-project -v
bolt-project:/bolt-project` flags -- without them, task execution stays
unavailable and Settings -> Execution reports no Bolt project configured.

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

docker volume create psh-bolt-project

docker run -d --name psh-console --network psh-net \
  -p 8767:8767 \
  -e PSH_SECRET_KEY="$PSH_SECRET_KEY" \
  -e PSH_DATABASE_URL="postgres://psh:psh@psh-postgres:5432/psh?sslmode=disable" \
  -e PSH_BOLT_PROJECT=/bolt-project \
  -v "$(pwd)/certs:/certs:ro" \
  -v psh-bolt-project:/bolt-project \
  ghcr.io/puppet-stagehand/stagehand-release/console:latest
```

`psh-bolt-project` is a **named** volume, not a bind-mounted host directory --
the console container runs as a non-root user, and Docker creates a fresh
bind-mount host directory owned by root, which that user can't write into. A
named volume avoids that (see "Task execution (Bolt)" above).

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

**Filing an issue?** Since `latest`/`test-pilots` are floating tags, include the exact
build you're running:

```bash
curl http://localhost:8767/api/v1/meta/version
```

```json
{"version":"1.0.2","commit":"199d4fe9f0e2f51a92a4e5d33bf19edd68b52665","date":"2026-09-04T03:27:14Z","channel":"test-pilots","go_version":"go1.26.8","product_name":"Puppet Stagehand"}
```

No login required. The same version string is also shown under the logo in the console's own sidebar.

## Required environment variables

| Variable | Required | Notes |
|---|---|---|
| `PSH_SECRET_KEY` | Yes | Base64-encoded 32-byte key (`openssl rand -base64 32`). Used to encrypt stored secrets (connection credentials, etc). Losing it makes existing encrypted data unreadable. |
| `PSH_DATABASE_URL` | No (has a local-dev default) | Postgres connection string. Default is `postgres://psh:psh@localhost:5432/psh?sslmode=disable` -- override this for anything beyond local testing. |
| `PSH_ADDR` | No | Listen address. Defaults to `:8767`. |
| `PSH_BOLT_PROJECT` | No, but required for task execution | Writable directory the console stages its Bolt project into (module, manifest, SSH keypair) -- see "Task execution (Bolt)" above. Without it, patching/r10k/task runs stay unavailable; everything else works. |

The console has no environment variable for connecting to Puppet/PuppetDB -- see "Quick start" above.

## What this is not

This is not the supported production install path. For a production deployment -- OS package management, TLS, backups, upgrades -- use the [Puppet Installer](https://github.com/puppet-stagehand/stagehand-installer). This repo exists for evaluating the console quickly, or for environments where the installer's assumptions don't fit.

## Source

The console's source, issue tracker, and full documentation live in the (private) [stagehand-console](https://github.com/puppet-stagehand/stagehand-console) repo. This repo only mirrors its release images and hosts this quick-start.
