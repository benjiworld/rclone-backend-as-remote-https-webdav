# Rclone backend as WebDAV with dockerized rclone and Caddy

Run an existing rclone remote bakend as an authenticated WebDAV endpoint, with Caddy handling HTTPS and automatically obtaining/renewing a publicly trusted TLS certificate.

> **Security:** This exposes access to your cloud files over the internet. Use a strong, unique WebDAV password, keep `rclone.conf` and `.env` private, expose only ports 80/443, and consider restricting access further (for example, with a VPN or IP allowlist) if the service does not need to be public.

## What this setup does

- Uses the existing `./config/rclone.conf` file; it does not recreate or replace the remote backend.
- Serves a selected path from that remote using rclone's WebDAV server.
- Uses a writable VFS cache directory at `./data/cache`.
- Places Caddy in a separate `caddy-compose.yaml` file beside `docker-compose.yaml`.
- Routes HTTPS requests through Caddy to rclone over the private Docker Compose network.
- Does not publish rclone's port 8080 on the host.

## Requirements

- A Linux host or another machine that can run Docker and Docker Compose.
- Docker Engine and Docker Compose. The examples use the `docker-compose` command, as in the setup tested here. If your installation uses Compose v2, replace `docker-compose` with `docker compose`.
- An existing rclone config file at `config/rclone.conf`, containing a working remote named `backend`.
- A domain or subdomain you control, such as `dav.example.com`.
- DNS for that hostname pointing to the server's public IP. If you publish an IPv6 `AAAA` record, IPv6 must also reach this server; otherwise remove the record.
- Inbound TCP ports 80 and 443 allowed through the host firewall and any router/NAT. UDP 443 is optional (HTTP/3).
- Enough disk space for the VFS cache and Caddy's certificate data.

## Directory layout

Create the following layout. Keep the existing `rclone.conf` in `config/`:

```text
rclone-webdav/
├── docker-compose.yaml
├── caddy-compose.yaml
├── Caddyfile
├── .env                 # local secrets; do not commit
├── .gitignore
├── config/
│   └── rclone.conf      # existing rclone configuration; do not commit
└── data/
    └── cache/           # writable VFS cache
```

Run the commands below from `rclone-webdav/`, the directory containing both Compose files.

## 1. Protect local credentials and configuration

Create `.gitignore` in the project directory:

```gitignore
.env
config/rclone.conf
```

If `rclone.conf` or `.env` has already been committed to a Git repository, adding it to `.gitignore` does not remove it from Git history. Remove it from tracking and rotate any exposed credentials/tokens.

The rclone configuration contains authentication tokens. Do not paste it into issues, commit it, or publish it. Mount the whole config directory (not just the file), since rclone may update tokens in its config.

## 2. Create the WebDAV credentials

Create `.env` beside the Compose files:

```dotenv
WEBDAV_USER=choose-a-username
WEBDAV_PASS=replace-with-a-long-unique-random-password
```

Choose a strong password and keep this file private. Compose substitutes these values into the service configuration. Do not include quotes around the values unless you intend the quote characters to be part of them.

## 3. Configure rclone's WebDAV server

Create or update `docker-compose.yaml`:

```yaml
services:
  rclone:
    image: rclone/rclone:latest
    container_name: rclone-webdav
    user: "${UID:-1000}:${GID:-1000}"
    volumes:
      - ./config:/config/rclone
      - ./data:/data
    working_dir: /data
    command:
      - --config
      - /config/rclone/rclone.conf
      - --cache-dir
      - /data/cache
      - serve
      - webdav
      - backend:some/folder
      - --addr
      - 0.0.0.0:8080
      - --user
      - "${WEBDAV_USER}"
      - --pass
      - "${WEBDAV_PASS}"
      - --vfs-cache-mode
      - writes
    restart: unless-stopped
```

### Set the remote path

Change `backend:some/folder` to the folder you want to serve. For example, use `backend:` to serve the remote root, or `backend:Documents/Shared` to serve a subfolder. The path is relative to the configured remote. The server log in the example setup reported `backend root 'folder'`; confirm the intended path in your account before exposing it.

The `--config` and `--cache-dir` global options are placed before `serve`. `--vfs-cache-mode writes` enables caching for write operations, as recommended by rclone for this remote. The cache is under `/data/cache`, which is backed by the host's `./data` directory.

There is deliberately no `ports:` entry for rclone. Caddy reaches it on the private Compose network; clients should not connect directly to port 8080.

### Prepare the cache directory

```sh
mkdir -p config data/cache
```

The container defaults to UID/GID `1000:1000` unless the environment supplies `UID` and `GID`. Make the cache directory writable by the container user. On Linux, if using UID/GID 1000:

```sh
sudo chown -R 1000:1000 data/cache
```

If you use different IDs, substitute them. On systems where `UID`/`GID` are not set, the Compose defaults are 1000. The existing config file must also be readable by the container user.

## 4. Configure Caddy and HTTPS

Create `caddy-compose.yaml` beside `docker-compose.yaml`:

```yaml
services:
  caddy:
    image: caddy:2
    container_name: caddy-webdav
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp" # optional; enables HTTP/3 where supported
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

Create `Caddyfile` beside the Compose files. Replace the example hostname with the exact DNS name you control:

```caddyfile
dav.example.com {
    reverse_proxy rclone:8080
}
```

When both Compose files are started together as below, the services join the same default Compose network. `rclone` in the `reverse_proxy` directive is the **service key** in `docker-compose.yaml`, not the container name. Caddy's named volumes preserve certificate and configuration data across container recreation.

### Certificate details

Caddy automatically obtains and renews a publicly trusted domain-validated certificate when the hostname's DNS and network access are correct. The certificate identifies the hostname; a missing organization field does not make a browser reject it. If a certificate with a verified organization name is specifically required, obtain an OV/EV certificate from a certificate authority and configure Caddy to use that certificate and key instead.

## 5. Start both Compose files

First check that DNS resolves to the server and ports 80/443 can reach it. Then, from the project directory, start the two files together:

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml up -d
```

Use **both** `-f` options for later Compose operations too, because the Caddy definition is in a separate file:

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml ps
docker-compose -f docker-compose.yaml -f caddy-compose.yaml logs -f
```

To view only Caddy logs:

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml logs -f caddy
```

The first certificate request may take a short time. If issuance fails, inspect the Caddy logs and check DNS, firewall/NAT rules, and whether another service is already using ports 80 or 443.

## 6. Connect a WebDAV client

Use this base URL in your WebDAV client:

```text
https://dav.example.com/
```

Provide the `WEBDAV_USER` and `WEBDAV_PASS` values from `.env`. Use HTTPS, not plain HTTP. Since Caddy's certificate is issued by a public CA and matches the hostname, a normally updated browser/client should trust it without manually installing a certificate.

## Useful operations

### Check rclone's configured remotes

For a one-off check using the existing config:

```sh
docker-compose -f docker-compose.yaml run --rm rclone listremotes
```

### Follow service logs

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml logs -f rclone caddy
```

### Restart after editing a Compose file

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml up -d
```

### Stop services

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml down
```

This stops and removes the containers and project network, but named volumes and host files are retained. Do not use `down -v` unless you intend to delete Caddy's stored certificate data.

## Troubleshooting

### `unknown flag: --rm` with `docker compose`

This can indicate that the installed command is the legacy Compose executable. Try `docker-compose run --rm rclone listremotes`, or install/enable the Compose plugin to use `docker compose`.

### `mkdir /.cache: permission denied`

Confirm `--cache-dir /data/cache` is before `serve` in the command, `./data:/data` is mounted, and `data/cache` exists and is writable by the configured container UID/GID. Recreate the service after changing the YAML:

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml up -d --force-recreate
```

### Caddy cannot obtain a certificate

Check that:

- the hostname in the Caddyfile is spelled correctly and has a public DNS record pointing to this server;
- inbound TCP 80 and 443 reach this host and are not blocked or already occupied;
- any public IPv6 DNS record points to a working IPv6 route;
- the Caddy container can reach the internet;
- the Caddyfile is mounted from the expected directory and parses successfully.

Inspect logs with:

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml logs caddy
```

### Caddy reports upstream connection errors

Check that the rclone service is running and listening on `0.0.0.0:8080` inside its container. Confirm the service is named `rclone` in `docker-compose.yaml` and that both files are started in the same Compose invocation. Caddy's upstream is `rclone:8080`; do not use `localhost:8080` from inside the Caddy container.

### WebDAV reads work but writes fail or are unreliable

Confirm `--vfs-cache-mode writes` is present and `/data/cache` is writable. Check rclone logs and available host disk space. Avoid deleting cache files while rclone is running.

### The hostname serves a certificate warning

Use the exact hostname listed in the Caddyfile and WebDAV URL, not the server IP. Check Caddy logs to confirm certificate issuance succeeded, and verify DNS points to this server. Do not bypass a certificate warning for a public service.

## Updating images

From the project directory:

```sh
docker-compose -f docker-compose.yaml -f caddy-compose.yaml pull
docker-compose -f docker-compose.yaml -f caddy-compose.yaml up -d
```

For production, consider pinning images to tested version tags rather than always using `latest` or `2`, and periodically review the rclone and Caddy release notes.
