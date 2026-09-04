# osgar-datahub-ssh — Direct DB Access via `sqlcmd`

**Status:** Implemented 2026-09-04. Extends [`osgar-datahub-dev-setup.md`](osgar-datahub-dev-setup.md) — same session, same container. Not a new architecture, just a new tool + two new environment variables on an already-existing service.

**Why:** the Cursor agent inside `osgar-datahub-ssh` needed a way to inspect DB contents (schema, row data) without a Docker socket, without `sqlcmd` on the host, and without introducing a second app instance. `osgar-datahub-ssh` already has network access to `osgar-datahub-db:1433` (documented in [`dev-ssh-access.md`](dev-ssh-access.md)'s network isolation table) — this only adds the client tool and the credentials to actually use it.

**Related**

- Container/network layout: [`dev-ssh-access.md`](dev-ssh-access.md)
- Permissions, supervisord, SCSS watcher: [`osgar-datahub-dev-setup.md`](osgar-datahub-dev-setup.md)

---

## `Dockerfile.ssh` — `mssql-tools18` addition

**Base image is Ubuntu 24.04 (`noble`)**, not Debian, despite Microsoft's docs defaulting to Debian paths — using a `debian/12` package list against this image fails with `NO_PUBKEY`. The `apt-key add` approach also failed independently (`apt-key` is deprecated and doesn't reliably import the key on this image).

The working combination: Ubuntu 22.04's package list (works fine against 24.04 in practice), `gpg --dearmor` into a keyring file, and an explicit `signed-by=` — hand-written, not piped through `sed` against Microsoft's own `prod.list`. A naive `sed 's|deb |deb [signed-by=...] |'` against that file produces a malformed entry, since the line already contains an `[arch=...]` prefix that the substitution collides with.

```dockerfile
RUN curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg \
    && echo "deb [arch=amd64,arm64,armhf signed-by=/usr/share/keyrings/microsoft-prod.gpg] https://packages.microsoft.com/ubuntu/22.04/prod jammy main" > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y mssql-tools18 unixodbc-dev
```

`mssql-tools18` installs to `/opt/mssql-tools18/bin`, not on `$PATH` by default — appended to both users' `.bashrc`:

```dockerfile
RUN echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> /home/dominik/.bashrc \
    && echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> /home/pawel/.bashrc
```

Only takes effect in interactive/login shells that source `.bashrc` (e.g. Cursor's terminal). A `docker exec` without a shell that sources it needs the full path or an explicit `export PATH=...` prepended.

**Rebuild required** — `mssql-tools18` is baked into the image at build time, not bind-mounted. Any `osgar-datahub-ssh` rebuild from an older `Dockerfile.ssh` won't have it.

---

## `docker-compose.dev.yml` — credentials on `osgar-datahub-ssh`

`osgar-datahub-ssh` previously had no `environment:` block — only `osgar-datahub-dev` carried `ConnectionStrings__DefaultConnection`. Added the same connection string **plus** a standalone `MSSQL_SA_PASSWORD`, so `sqlcmd` doesn't need it parsed out of the composite string:

```yaml
  osgar-datahub-ssh:
    environment:
      - ConnectionStrings__DefaultConnection=Server=osgar-datahub-db;Database=OsgarDatahub;User Id=sa;Password=${MSSQL_SA_PASSWORD};TrustServerCertificate=True
      - MSSQL_SA_PASSWORD=${MSSQL_SA_PASSWORD}
```

**Considered and rejected:** extracting the password from `ConnectionStrings__DefaultConnection` via `grep -oP "Password=\K[^;]+"` at query time. Works, but adds a moving part to every query Cursor runs for no benefit — the standalone variable is simpler and just as safe (`sa` credentials for a dummy dev DB either way).

---

## Usage

```bash
sqlcmd -S osgar-datahub-db -U sa -P "$MSSQL_SA_PASSWORD" -C -d OsgarDatahub -Q "SELECT name FROM sys.tables"
```

`-C` skips TLS certificate validation — required, matches `TrustServerCertificate=True` in the app's own connection string (self-signed cert on the containerised SQL Server).

Confirmed working 2026-09-04 — returned all 18 application tables (`AppSettings`, `Suppliers`, `Catalogs`, `Articles`, etc.).

---

## Not done here

- **No read-only DB user.** `sa` has full admin rights — fine for a dummy dev database, not something to copy onto an app with production data in `osgar-datahub-db`. If this pattern is reused on such an app, create a dedicated login with `db_datareader` only rather than reusing `sa`.
- **No equivalent for `advoapp-ssh`.** That app doesn't use SQL Server and has no equivalent sibling DB container at the moment — N/A until it does.
