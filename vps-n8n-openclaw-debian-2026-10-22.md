# VPS Update Guide (Debian 13): n8n + OpenClaw (incremental)

Add-on to `vps-vpn-web-debian-2026-10-21.md` (the **base guide**). It assumes the base guide is
finished and working: Xray on 443, native Nginx on 8443 behind Cloudflare (Origin Rule → 8443,
zone-level AOP), UFW, fail2ban, encrypted backups. Nothing from the base guide is reinstalled.

**Decisions locked in:**
- Login: same as the base guide, **`root` with its password** on port 2222. Every command runs as root.
- n8n and OpenClaw run in **Docker** (Compose plugin), each as its **own Compose project** with its
  own network, own data directory and no shared volumes. They never share a network with each other.
- Nginx stays **native** (base guide §4). Containers publish their port on **`127.0.0.1` only**,
  and Nginx proxies to `127.0.0.1:<port>`. Docker publishes ports by writing iptables rules that
  **bypass UFW**, so a port published on `0.0.0.0` would be open to the internet regardless of UFW.
  Binding to `127.0.0.1` avoids that. No new UFW rules are needed.
- Hostnames: `n8n.<yourdomain.com>` and `<OPENCLAW_SUBDOMAIN>.<yourdomain.com>`. Pick something
  non-obvious for OpenClaw (e.g. `ctrl`), not `openclaw`.
- Access layers: Cloudflare → AOP client cert → **Nginx basic auth** → the app's own login
  (n8n owner account / OpenClaw gateway token + device pairing).
- n8n webhook/form paths skip basic auth so external services can call them. n8n still checks them itself.
- OpenClaw image tag is **pinned** to an exact release (four critical RCEs in the past, patched within
  days each time) and rechecked weekly. Use the prebuilt image, never `latest`, and don't build
  locally (a local build needs ≥6 GB RAM).
- Telegram for OpenClaw uses **long polling** (OpenClaw's default), so it needs no inbound webhook.

**What changes on the existing server:**
- New: Docker Engine, `/opt/n8n`, `/opt/openclaw`, two Nginx sites, one Nginx `conf.d` snippet, `/etc/nginx/auth/`.
- Cloudflare: two DNS records, and the existing Origin Rule gets two more hostnames.
- `/opt/backups/backup.sh` is replaced by a version that also backs up n8n and OpenClaw.
- Unchanged: Xray, SSH, fail2ban, UFW rules, the website, the AOP certificate.

Replace `<ANGLE_BRACKETS>` with your own values. Version numbers are fetched live, not hardcoded.

**Reading the expected-output comments** (same convention as the base guide):
- `# expected: ...` describes what a command should print. `# expected: no output` means success is silent.
- Commands not marked print nothing you need to check (apt progress, `chmod`, `mkdir`, …).
- Every `tee ... <<'EOF'` command echoes the file content back to the terminal. That's normal.
- Version numbers, keys, IDs, IPs and dates in the expected output are placeholders.

---

## 0. Order of operations

1. Pre-flight checks (base stack healthy, RAM/disk, swap)
2. Docker Engine + Compose plugin
3. Cloudflare (DNS records, extend the Origin Rule)
4. n8n (Docker, `127.0.0.1:5678`)
5. OpenClaw (Docker, `127.0.0.1:18789`)
6. Nginx sites for n8n and OpenClaw (basic auth, WebSockets)
7. First login: n8n owner account, OpenClaw Control UI, Telegram
8. Extend the encrypted backups
9. Final verification checklist
10. Maintenance cadence
11. Troubleshooting

---

## 1. Pre-flight checks

### 1.1 Base stack is healthy
```
systemctl is-active xray nginx fail2ban ufw
# expected:
#   active
#   active
#   active
#   active
grep -r ssl_verify_client /etc/nginx/sites-enabled/
# expected: /etc/nginx/sites-enabled/website.conf:    ssl_verify_client on;
```
From your local machine, VPN off:
```
curl -sI https://<yourdomain.com> | grep -iE '^(HTTP|server)'
# expected:
#   HTTP/2 200
#   server: cloudflare
```
A 200 here while `ssl_verify_client on` is set proves zone-level AOP works (this also ticks the
open AOP item in the base guide's §8 checklist). If you get 400 "No required SSL certificate was
sent", fix AOP first (base guide §3.5). The new sites use the same client-cert check.

### 1.2 RAM, disk, swap
```
free -h
# expected: a "Mem:" line; note the "total" column (e.g. 3.8Gi) and the "Swap:" line
df -h /
# expected: the "Avail" column shows at least ~10G free (images + data + logs)
```
Rough footprint: n8n ~300–600 MB, OpenClaw gateway ~400 MB–1 GB, Xray + Nginx < 100 MB.
**4 GB RAM is comfortable, 2 GB is the minimum.** On ≤4 GB, add a 2 GB swap file if `Swap:` shows `0B`:
```
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
# expected: Setting up swapspace version 1, size = 2 GiB (2147479552 bytes) ...
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness=10' > /etc/sysctl.d/99-swap.conf
sysctl -p /etc/sysctl.d/99-swap.conf
# expected: vm.swappiness = 10
free -h | grep Swap
# expected: Swap:  2.0Gi  0B  2.0Gi
```
`vm.swappiness=10` keeps swap as an emergency buffer instead of slowing everything down.

### 1.3 Pick and record values
Put these in your password manager now; you'll need them several times:

| Placeholder | Example | How to get it |
|---|---|---|
| `<OPENCLAW_SUBDOMAIN>` | `ctrl` | your choice |
| `<Your/Timezone>` | `Europe/Berlin` | `timedatectl show -p Timezone --value` (or your local zone) |
| `<N8N_ENCRYPTION_KEY>` | 64 hex chars | `openssl rand -hex 32` (§4.2) |
| `<OPENCLAW_GATEWAY_TOKEN>` | 64 hex chars | `openssl rand -hex 32` (§5.2) |
| basic-auth users/passwords | — | password manager, two different passwords (§6.2) |

---

## 2. Docker Engine + Compose plugin

### 2.1 Remove distro packages that conflict
```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt-get remove -y $pkg 2>/dev/null; done
# expected: "Package 'docker.io' is not installed, so not removed" (or similar) for each one — fine on a fresh box
```

### 2.2 Add Docker's official Debian repository
```
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
# expected: the echoed file shows "Suites: trixie" (not an empty value and not a literal "$(...)")
apt update
# expected: a line "Get:... https://download.docker.com/linux/debian trixie InRelease", no errors
```
This `EOF` is intentionally **unquoted** so `$(...)` expands to `trixie`.

### 2.3 Install
```
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl is-active docker
# expected: active
docker --version
# expected: Docker version X.Y.Z, build ...
docker compose version
# expected: Docker Compose version vX.Y.Z
```

### 2.4 Log rotation for containers
By default Docker keeps container logs forever. Cap them:
```
tee /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "local",
  "log-opts": { "max-size": "20m", "max-file": "5" }
}
EOF
systemctl restart docker
docker info --format '{{.LoggingDriver}}'
# expected: local
```

### 2.5 Smoke test + check that UFW is untouched
```
docker run --rm hello-world
# expected (among other lines): Hello from Docker!
ufw status | head -3
# expected:
#   Status: active
#   (blank)
#   To                         Action      From
ufw status | grep -c Cloudflare
# expected: the same number as before (base guide §5) — Docker doesn't change UFW rules
```
Docker adds its own iptables chains (`DOCKER`, `DOCKER-USER`, …). That's normal and doesn't
affect UFW's rules for 2222/443/8443.

---

## 3. Cloudflare

### 3.1 DNS records
**DNS → Records → Add record**, same as base guide §3.1:

| Type | Name | IPv4 address | Proxy status | TTL |
|---|---|---|---|---|
| A | `n8n` | `<VPS_IP>` | **Proxied** (orange cloud) | Auto |
| A | `<OPENCLAW_SUBDOMAIN>` | `<VPS_IP>` | **Proxied** (orange cloud) | Auto |

If you added `AAAA` records for the site, add `AAAA` records for these two as well.

The origin certificate from base guide §3.3 already covers `*.<yourdomain.com>`, and Cloudflare's
free edge certificate covers first-level subdomains, so no new certificates are needed.

Check from any machine:
```
dig +short n8n.<yourdomain.com>
dig +short <OPENCLAW_SUBDOMAIN>.<yourdomain.com>
# expected: Cloudflare IPs (e.g. 104.21.x.x / 172.67.x.x), NOT <VPS_IP>
```

### 3.2 Extend the Origin Rule to the new hostnames (required)
Without this, Cloudflare connects to the new hostnames on 443 and hits Xray → error 521/525.

**Rules → Overview** → click the rule `website to 8443` → **Edit**:
1. Optionally rename it to `origin to 8443`.
2. **Edit expression** and replace it with:
   ```
   (http.host in {"<yourdomain.com>" "www.<yourdomain.com>" "n8n.<yourdomain.com>" "<OPENCLAW_SUBDOMAIN>.<yourdomain.com>"})
   ```
3. Keep **Destination Port → Rewrite to… → `8443`**.
4. **Save**.

Until §6 is done, the new hostnames return 404 or the website (Nginx's default server for 8443).
That's expected at this point.

### 3.3 Settings that break apps (check once)
- **Security → Settings → Bot Fight Mode**: keep **Off**. It challenges non-browser clients and
  breaks n8n webhooks called by other services.
- **Speed → Optimization → Rocket Loader**: keep **Off**. It rewrites JavaScript and can break the
  n8n editor and the OpenClaw Control UI.
- **Network → WebSockets**: **On** (the default). Both apps need it.

---

## 4. n8n (Docker, `127.0.0.1:5678`)

### 4.1 Look up the current version
```
curl -s https://api.github.com/repos/n8n-io/n8n/releases/latest | jq -r .tag_name
# expected: n8n@X.Y.Z
```
The image tag is the part after `n8n@`, e.g. `X.Y.Z`. Read the release notes at
`github.com/n8n-io/n8n/releases` before each upgrade (major versions have breaking changes).

### 4.2 Directory and secrets
```
mkdir -p /opt/n8n/data
chown -R 1000:1000 /opt/n8n/data
chmod 700 /opt/n8n
openssl rand -hex 32
# expected: 64 hex chars → this is <N8N_ENCRYPTION_KEY>; save it in the password manager
```
n8n runs as user `node` (UID 1000) inside the container, so the data directory must be owned by 1000.

The encryption key encrypts every credential stored in n8n. **If it's lost, restored backups
can't decrypt any credential.** Setting it explicitly (rather than letting n8n generate one inside
the data directory) means it lives in your password manager too.

```
tee /opt/n8n/.env <<'EOF'
N8N_VERSION=<X.Y.Z>
N8N_HOST=n8n.<yourdomain.com>
N8N_ENCRYPTION_KEY=<N8N_ENCRYPTION_KEY>
TZ=<Your/Timezone>
EOF
chmod 600 /opt/n8n/.env
```

### 4.3 Compose file
```
tee /opt/n8n/docker-compose.yml <<'EOF'
name: n8n

networks:
  n8n_net:
    name: n8n_net

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:${N8N_VERSION}
    container_name: n8n
    restart: unless-stopped
    networks: [n8n_net]
    # Loopback only: native Nginx proxies to it. Never "5678:5678" — Docker would bypass UFW.
    ports:
      - "127.0.0.1:5678:5678"
    security_opt:
      - no-new-privileges:true
    environment:
      N8N_HOST: ${N8N_HOST}
      N8N_PORT: "5678"
      N8N_PROTOCOL: https
      WEBHOOK_URL: https://${N8N_HOST}/
      N8N_EDITOR_BASE_URL: https://${N8N_HOST}/
      N8N_PROXY_HOPS: "1"
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS: "true"
      N8N_DIAGNOSTICS_ENABLED: "false"
      N8N_PERSONALIZATION_ENABLED: "false"
      GENERIC_TIMEZONE: ${TZ}
      TZ: ${TZ}
    volumes:
      - ./data:/home/node/.n8n
EOF
cd /opt/n8n
docker compose config --quiet
# expected: no output (any output = YAML or .env error)
```
- `N8N_PROTOCOL=https` + `WEBHOOK_URL`: n8n is reached over HTTP internally but shows and
  registers public `https://n8n.<yourdomain.com>/...` URLs.
- `N8N_PROXY_HOPS=1`: trust one proxy (Nginx) for `X-Forwarded-*` headers.
- Storage is SQLite inside `./data`. That's fine for a single user.

### 4.4 Start and verify
```
docker compose up -d
# expected: "Container n8n  Started" (first run also pulls the image: a progress bar, then "Pulled")
docker compose ps
# expected: n8n ... Up ... 127.0.0.1:5678->5678/tcp
docker compose logs --tail 20 n8n
# expected (near the end): "Editor is now accessible via:" followed by https://n8n.<yourdomain.com>
curl -s http://127.0.0.1:5678/healthz; echo
# expected: {"status":"ok"}
ss -tlnp | grep 5678
# expected: one LISTEN line on 127.0.0.1:5678 (docker-proxy). NOT 0.0.0.0:5678 or [::]:5678
```
If the logs show `EACCES: permission denied, open '/home/node/.n8n/config'`, re-run
`chown -R 1000:1000 /opt/n8n/data` and `docker compose up -d`.

Don't open the UI yet. The owner account is created in §7.1, after Nginx is in front of it.

---

## 5. OpenClaw (Docker, `127.0.0.1:18789`)

### 5.1 Look up the current version
```
curl -s https://api.github.com/repos/openclaw/openclaw/releases/latest | jq -r .tag_name
# expected: vYYYY.M.D (e.g. v2026.10.2), possibly without the leading "v"
```
The image tag is the version **without** the leading `v`, e.g. `2026.10.2`. Confirm it exists:
```
docker manifest inspect ghcr.io/openclaw/openclaw:<YYYY.M.D> >/dev/null && echo OK
# expected: OK
```
Check `github.com/openclaw/openclaw/releases` and `github.com/openclaw/openclaw/security/advisories`
before each upgrade.

### 5.2 Directories and secrets
```
mkdir -p /opt/openclaw/config /opt/openclaw/workspace
chown -R 1000:1000 /opt/openclaw/config /opt/openclaw/workspace
chmod 700 /opt/openclaw
openssl rand -hex 32
# expected: 64 hex chars → this is <OPENCLAW_GATEWAY_TOKEN>; save it in the password manager
tee /opt/openclaw/.env <<'EOF'
OPENCLAW_VERSION=<YYYY.M.D>
OPENCLAW_GATEWAY_TOKEN=<OPENCLAW_GATEWAY_TOKEN>
TZ=<Your/Timezone>
EOF
chmod 600 /opt/openclaw/.env
```
`config/` holds OpenClaw's state (`openclaw.json`, LLM API keys, channel tokens, paired devices);
`workspace/` is the only directory the agent works in. Neither is shared with n8n.

### 5.3 Compose file
```
tee /opt/openclaw/docker-compose.yml <<'EOF'
name: openclaw

networks:
  openclaw_net:
    name: openclaw_net
    ipam:
      config:
        # Fixed subnet so the gateway's view of the proxy IP (172.30.0.1) is predictable (§5.4).
        - subnet: 172.30.0.0/24
          gateway: 172.30.0.1

x-openclaw-common: &common
  image: ghcr.io/openclaw/openclaw:${OPENCLAW_VERSION}
  init: true
  cap_drop: [NET_RAW, NET_ADMIN]
  security_opt:
    - no-new-privileges:true
  environment:
    HOME: /home/node
    OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
    OPENCLAW_DISABLE_BONJOUR: "1"
    TZ: ${TZ}
  volumes:
    - ./config:/home/node/.openclaw
    - ./workspace:/home/node/.openclaw/workspace

services:
  openclaw-gateway:
    <<: *common
    container_name: openclaw-gateway
    restart: unless-stopped
    networks: [openclaw_net]
    command: ["node", "dist/index.js", "gateway", "--bind", "lan", "--port", "18789"]
    # Loopback only: native Nginx proxies to it. Never "18789:18789" — Docker would bypass UFW.
    ports:
      - "127.0.0.1:18789:18789"

  openclaw-cli:
    <<: *common
    # Shares the gateway's network namespace, so CLI commands reach it on its own loopback.
    network_mode: "service:openclaw-gateway"
    entrypoint: ["node", "dist/index.js"]
    depends_on: [openclaw-gateway]
    profiles: ["cli"]
EOF
cd /opt/openclaw
docker compose config --quiet
# expected: no output
```
- `--bind lan` makes the gateway listen on the container's network interface. That's required for
  the published port to reach it, and it's still only reachable from the host's `127.0.0.1`.
- No Docker socket is mounted, so the agent can't control Docker or other containers.
- `profiles: ["cli"]` keeps `docker compose up -d` from starting the CLI helper.
  `docker compose run openclaw-cli ...` still works.

### 5.4 Onboarding (before the gateway starts)
Pull the image, then run the interactive wizard in a one-off container:
```
docker compose pull
# expected: "openclaw-gateway Pulled" / "openclaw-cli Pulled" (or "Skipped - Image is already being pulled")
docker compose run --rm --no-deps --entrypoint node openclaw-gateway dist/index.js onboard --mode local --no-install-daemon
```
In the wizard:
- **Security/risk acknowledgement:** read it and accept. It reminds you the agent can run commands.
- **Model provider / auth:** pick your LLM provider and paste its API key (or sign in, if offered).
- **Gateway auth:** **token**. If it asks for a value, paste `<OPENCLAW_GATEWAY_TOKEN>`. If it offers
  to read it from the `OPENCLAW_GATEWAY_TOKEN` environment variable, choose that.
- **Gateway bind:** `lan`. **Port:** `18789`.
- **Channels:** skip for now (Telegram is added in §7.3).
- **Daemon / service install:** skip (Docker is the service manager).

```
# expected (end of the wizard): a summary and a line saying the config was written to
#   /home/node/.openclaw/openclaw.json  (on the host: /opt/openclaw/config/openclaw.json)
ls -l /opt/openclaw/config/
# expected: openclaw.json (owner 1000) plus a few state directories
```

Now set the reverse-proxy settings. Define a short helper for **offline** config commands (run
while the gateway is stopped):
```
oc-offline() { docker compose -f /opt/openclaw/docker-compose.yml run --rm -T --no-deps --entrypoint node openclaw-gateway dist/index.js "$@"; }

oc-offline config set gateway.bind lan
# expected: a confirmation line (e.g. "Updated gateway.bind"), no error
oc-offline config set gateway.controlUi.allowedOrigins '["https://<OPENCLAW_SUBDOMAIN>.<yourdomain.com>"]' --strict-json --dry-run
# expected: a preview of the change, no "invalid" / "unknown path" error
oc-offline config set gateway.controlUi.allowedOrigins '["https://<OPENCLAW_SUBDOMAIN>.<yourdomain.com>"]' --strict-json
# expected: a confirmation line, no error
oc-offline config set gateway.trustedProxies '["172.30.0.1"]' --strict-json
# expected: a confirmation line, no error
oc-offline config get gateway --json
# expected: JSON containing (secrets redacted):
#   "bind": "lan",
#   "trustedProxies": ["172.30.0.1"],
#   "auth": { "mode": "token", ... },
#   "controlUi": { "allowedOrigins": ["https://<OPENCLAW_SUBDOMAIN>.<yourdomain.com>"] ... }
oc-offline config validate
# expected: a "valid" / OK result
```
- `allowedOrigins` must match the browser's address bar **exactly** (scheme + host, no trailing
  slash). Non-loopback Control UI access is refused without it.
- `trustedProxies`: Nginx connects to `127.0.0.1:18789`, Docker forwards that into the container,
  and the gateway sees the connection coming from the network's gateway IP `172.30.0.1` (fixed in
  §5.3). Trusting only that IP lets OpenClaw read the `X-Forwarded-*` headers Nginx sets.
- The value is a single IP, not the whole subnet, so nothing else on `openclaw_net` is trusted.

### 5.5 Start and verify
```
docker compose up -d
# expected: "Network openclaw_net  Created", "Container openclaw-gateway  Started"
docker compose ps
# expected: openclaw-gateway ... Up ... (healthy) ... 127.0.0.1:18789->18789/tcp
#   ("health: starting" for the first ~30 s is fine)
docker compose logs --tail 30 openclaw-gateway
# expected: a "gateway listening" / "ready" line on port 18789; no "allowedOrigins" or config errors
curl -fsS http://127.0.0.1:18789/healthz; echo
# expected: a short OK/JSON body and exit code 0 (no "curl: (7)" or "(22)" error)
ss -tlnp | grep 18789
# expected: one LISTEN line on 127.0.0.1:18789 (docker-proxy). NOT 0.0.0.0 or [::]
docker compose exec openclaw-gateway sh -lc 'node dist/index.js gateway health --token "$OPENCLAW_GATEWAY_TOKEN"'
# expected: a healthy/OK status
```
Helper for **runtime** CLI commands (gateway running). Add both helpers to root's shell so they
survive logout:
```
tee -a /root/.bashrc <<'EOF'
oc-offline() { docker compose -f /opt/openclaw/docker-compose.yml run --rm -T --no-deps --entrypoint node openclaw-gateway dist/index.js "$@"; }
oc() { docker compose -f /opt/openclaw/docker-compose.yml run --rm openclaw-cli "$@"; }
EOF
source /root/.bashrc
oc --version
# expected: the pinned version, e.g. 2026.10.2
```

---

## 6. Nginx sites for n8n and OpenClaw

### 6.1 WebSocket helper (shared)
Both apps use WebSockets (n8n's editor push, OpenClaw's Control UI). This map sends
`Connection: upgrade` only when the client asked for an upgrade, and keeps plain requests normal:
```
tee /etc/nginx/conf.d/websocket-map.conf <<'EOF'
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
EOF
```
Debian's `/etc/nginx/nginx.conf` already includes `conf.d/*.conf` inside the `http {}` block.

### 6.2 Basic-auth credentials
Use **two different** passwords from your password manager. They are not the n8n login or the
OpenClaw token.
```
apt install -y apache2-utils
mkdir -p /etc/nginx/auth
htpasswd -B -c /etc/nginx/auth/n8n.htpasswd <n8n_basic_user>
# expected: "New password:" / "Re-type new password:" (hidden), then "Adding password for user <n8n_basic_user>"
htpasswd -B -c /etc/nginx/auth/openclaw.htpasswd <openclaw_basic_user>
# expected: same prompts, then "Adding password for user <openclaw_basic_user>"
chown root:www-data /etc/nginx/auth/*.htpasswd
chmod 640 /etc/nginx/auth/*.htpasswd
head -c 30 /etc/nginx/auth/n8n.htpasswd; echo
# expected: <n8n_basic_user>:$2y$05$...   ($2y$ = bcrypt)
```
`-c` creates (overwrites) the file. To add a second user later, run the command **without** `-c`.

### 6.3 n8n site
```
tee /etc/nginx/sites-available/n8n.conf <<'EOF'
server {
    listen 8443 ssl;
    listen [::]:8443 ssl;
    http2 on;
    server_name n8n.<yourdomain.com>;

    ssl_certificate     /etc/nginx/ssl/origin.pem;
    ssl_certificate_key /etc/nginx/ssl/origin.key;

    ssl_client_certificate /etc/nginx/ssl/aop-ca.pem;
    ssl_verify_client on;

    client_max_body_size 50m;

    proxy_http_version 1.1;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        $connection_upgrade;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $remote_addr;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Host  $host;
    proxy_buffering off;
    proxy_read_timeout 300s;

    # Webhooks and forms: called by external services or the public, so no basic auth.
    # n8n validates these itself (unknown paths return 404; add header/basic auth per webhook node).
    location ~ ^/(webhook|webhook-test|webhook-waiting|form|form-test|form-waiting)/ {
        proxy_pass http://127.0.0.1:5678;
    }

    # Editor UI and REST API: basic auth in front of n8n's own login.
    location / {
        auth_basic           "Restricted";
        auth_basic_user_file /etc/nginx/auth/n8n.htpasswd;
        proxy_pass http://127.0.0.1:5678;
    }
}
EOF
```
- `proxy_set_header` lines at `server` level are inherited by both `location` blocks.
- `proxy_buffering off` lets n8n stream responses (executions, server-sent events) without delay.
- Webhooks that run longer than ~100 s hit Cloudflare's timeout (error 524) no matter what Nginx
  allows; use "Respond immediately" in the Webhook node for long workflows.
- If you never use webhooks or forms, delete the first `location` block so everything requires basic auth.

### 6.4 OpenClaw site
```
tee /etc/nginx/sites-available/openclaw.conf <<'EOF'
server {
    listen 8443 ssl;
    listen [::]:8443 ssl;
    http2 on;
    server_name <OPENCLAW_SUBDOMAIN>.<yourdomain.com>;

    ssl_certificate     /etc/nginx/ssl/origin.pem;
    ssl_certificate_key /etc/nginx/ssl/origin.key;

    ssl_client_certificate /etc/nginx/ssl/aop-ca.pem;
    ssl_verify_client on;

    auth_basic           "Restricted";
    auth_basic_user_file /etc/nginx/auth/openclaw.htpasswd;

    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
EOF
```
- `X-Forwarded-For $remote_addr` **overwrites** any client-supplied value instead of appending to it,
  so nobody can spoof an address to the gateway through Cloudflare.
- The long timeouts keep the Control UI's WebSocket open while idle.
- Auth layers: Nginx basic auth, then the gateway token, then per-browser device pairing (§7.2).

### 6.5 Enable and reload
```
ln -sf /etc/nginx/sites-available/n8n.conf      /etc/nginx/sites-enabled/n8n.conf
ln -sf /etc/nginx/sites-available/openclaw.conf /etc/nginx/sites-enabled/openclaw.conf
ls -l /etc/nginx/sites-enabled/
# expected: three symlinks — website.conf, n8n.conf, openclaw.conf
nginx -t
# expected:
#   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
#   nginx: configuration file /etc/nginx/nginx.conf test is successful
systemctl reload nginx
systemctl is-active nginx
# expected: active
```
Check from your local machine (VPN off):
```
curl -sI https://n8n.<yourdomain.com> | grep -iE '^(HTTP|www-authenticate)'
# expected:
#   HTTP/2 401
#   www-authenticate: Basic realm="Restricted"
curl -sI -u '<n8n_basic_user>:<password>' https://n8n.<yourdomain.com> | head -1
# expected: HTTP/2 200
curl -s -o /dev/null -w '%{http_code}\n' https://n8n.<yourdomain.com>/webhook/does-not-exist
# expected: 404   (reaches n8n without basic auth; n8n has no such webhook)
curl -sI https://<OPENCLAW_SUBDOMAIN>.<yourdomain.com> | grep -iE '^(HTTP|www-authenticate)'
# expected:
#   HTTP/2 401
#   www-authenticate: Basic realm="Restricted"
curl -sI -u '<openclaw_basic_user>:<password>' https://<OPENCLAW_SUBDOMAIN>.<yourdomain.com> | head -1
# expected: HTTP/2 200
curl -sI https://<yourdomain.com> | head -1
# expected: HTTP/2 200   (the website still works)
```
Your shell history now contains the basic-auth passwords. Clear those lines: `history -c` on
bash, or delete them from `~/.zsh_history` on macOS.

---

## 7. First login

### 7.1 n8n owner account
1. Open `https://n8n.<yourdomain.com>` → browser basic-auth prompt → n8n basic user/password.
2. n8n shows **Set up owner account**. Enter your email, name and a **third, separate** password
   from your password manager → **Next**. Skip the optional questionnaire/license prompts as you like.
3. Expected: the empty workflow editor.

Do this right away. Until the owner exists, anyone past basic auth could claim the instance.

Check WebSockets work: create a workflow with a **Manual Trigger** node → **Execute workflow**.
Expected: the node turns green within a second. If the editor shows "Connection lost" in the
corner, see §11.

Optional hardening: **Settings → Personal → Enable 2FA**.

### 7.2 OpenClaw Control UI and device pairing
1. Open `https://<OPENCLAW_SUBDOMAIN>.<yourdomain.com>` → basic-auth prompt → OpenClaw basic user/password.
2. The Control UI asks for the **Gateway secret**. Paste `<OPENCLAW_GATEWAY_TOKEN>` → connect.
3. The first connection from a new browser is held for **device pairing**: the UI says pairing is
   required / waiting for approval. Approve it on the VPS:
   ```
   oc devices list
   # expected: a table with one pending request (request ID, browser/device name, created time)
   oc devices approve <requestId>
   # expected: a confirmation that the device was approved
   ```
4. Reload the page. Expected: the Control UI connects and shows the gateway status and chat.

Each new browser or device has to be approved once. To remove one later: `oc devices list`, then
the revoke/remove subcommand that `oc devices --help` shows.

If the URL ever needs to be printed again: `oc dashboard --no-open` (replace the host part with
your public hostname).

### 7.3 Telegram channel (long polling)
1. In Telegram, message `@BotFather` → `/newbot` → pick a name and a username ending in `bot` →
   copy the bot token. Store it in your password manager.
2. Register it:
   ```
   oc channels add --channel telegram --token "<BOT_TOKEN>"
   # expected: confirmation that the telegram channel was added
   docker compose -f /opt/openclaw/docker-compose.yml restart openclaw-gateway
   # expected: Container openclaw-gateway  Started
   docker compose -f /opt/openclaw/docker-compose.yml logs --tail 20 openclaw-gateway | grep -i telegram
   # expected: a line showing the Telegram channel started / polling
   ```
3. From **your own** Telegram account, send the bot any message. With the default DM policy
   (`pairing`), the bot replies with a **pairing code** instead of answering.
4. Approve it on the VPS:
   ```
   oc pairing list telegram
   # expected: one pending request with your Telegram name/ID and a code
   oc pairing approve telegram <CODE>
   # expected: a confirmation that the sender was approved
   ```
5. Message the bot again. Expected: an answer from the agent.

Only approved senders can talk to the agent; anyone else just gets a pairing code that you never
approve. Don't add the bot to groups unless you've configured a group allowlist.

Clear the bot token from shell history (`history -c`).

---

## 8. Extend the encrypted backups

This replaces `/opt/backups/backup.sh` from base guide §7. It adds n8n and OpenClaw data,
their `.env` files (which hold the n8n encryption key and gateway token), and the Nginx auth files.
Both apps keep SQLite files, which can be corrupted if copied mid-write, so the script stops the
containers for a few seconds and always starts them again.

```
cp /opt/backups/backup.sh /opt/backups/backup.sh.bak-$(date +%F)
tee /opt/backups/backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
umask 077
STAMP=$(date +%F)
WORK=$(mktemp -d)

restart_apps() {
  docker compose -f /opt/n8n/docker-compose.yml start >/dev/null 2>&1 || true
  docker compose -f /opt/openclaw/docker-compose.yml start openclaw-gateway >/dev/null 2>&1 || true
}
trap 'restart_apps; rm -rf "$WORK"' EXIT

# Base stack (unchanged)
cp -r /usr/local/etc/xray        "$WORK/xray-config"
cp -r /etc/nginx/sites-available "$WORK/nginx-sites"
cp -r /etc/nginx/conf.d          "$WORK/nginx-conf.d"
cp -r /etc/nginx/ssl             "$WORK/nginx-ssl"
cp -r /etc/nginx/auth            "$WORK/nginx-auth"
cp -r /etc/ssh/sshd_config.d     "$WORK/sshd_config.d"
cp -r /var/www/site              "$WORK/site"

# Apps: stop briefly for a consistent copy of their SQLite files
docker compose -f /opt/n8n/docker-compose.yml stop >/dev/null 2>&1
docker compose -f /opt/openclaw/docker-compose.yml stop openclaw-gateway >/dev/null 2>&1
tar czf "$WORK/n8n.tar.gz"      -C /opt n8n
tar czf "$WORK/openclaw.tar.gz" -C /opt openclaw
restart_apps

tar czf - -C "$WORK" . | age -R /opt/backups/age-recipient.txt \
  -o "/opt/backups/vps-backup-$STAMP.tar.gz.age"

# Keep 14 days locally (off-box copies are kept by rclone)
find /opt/backups -name 'vps-backup-*.tar.gz.age' -mtime +14 -delete
EOF
chmod 700 /opt/backups/backup.sh
/opt/backups/backup.sh
# expected: no output (takes ~5–20 s; n8n and OpenClaw are down for part of that)
ls -lh /opt/backups/vps-backup-$(date +%F).tar.gz.age
# expected: one file, now MBs instead of KBs (it includes the app data)
docker compose -f /opt/n8n/docker-compose.yml ps --format '{{.Name}} {{.Status}}'
docker compose -f /opt/openclaw/docker-compose.yml ps --format '{{.Name}} {{.Status}}'
# expected:
#   n8n Up X seconds
#   openclaw-gateway Up X seconds (healthy)   ("health: starting" right after the backup is fine)
```
The cron jobs from base guide §7 (backup at 03:00, rclone at 03:30) stay as they are:
```
crontab -l
# expected: still three lines – cf-ufw-refresh.sh, backup.sh, rclone copy
```
The OpenClaw workspace can grow with whatever the agent writes. If backups get large, check
`du -sh /opt/openclaw/workspace`.

Restore test (on your local machine):
```
age -d -i vps-backup.agekey vps-backup-<DATE>.tar.gz.age | tar xzf -
# expected: no output; xray-config/, nginx-sites/, nginx-conf.d/, nginx-ssl/, nginx-auth/,
#           sshd_config.d/, site/, n8n.tar.gz, openclaw.tar.gz appear
tar tzf n8n.tar.gz | head
# expected: n8n/, n8n/.env, n8n/docker-compose.yml, n8n/data/..., including n8n/data/database.sqlite
```
To restore an app on a server: stop it, extract its tarball into `/opt`, `chown -R 1000:1000` its
data directories, `docker compose up -d`.

---

## 9. Final verification checklist

- [ ] `docker compose ps` in `/opt/n8n` and `/opt/openclaw` shows `n8n` and `openclaw-gateway` **Up** (gateway `healthy`).
- [ ] `ss -tlnp | grep -E ':(5678|18789)'` shows only `127.0.0.1` listeners.
- [ ] From outside, `curl --max-time 10 http://<VPS_IP>:5678` and `:18789` time out.
- [ ] `ufw status` is unchanged: only 2222/tcp, 443/tcp and 8443/tcp from Cloudflare CIDRs.
- [ ] `docker network inspect n8n_net openclaw_net --format '{{.Name}}: {{range .Containers}}{{.Name}} {{end}}'` shows `n8n_net: n8n` and `openclaw_net: openclaw-gateway` (no overlap).
- [ ] `https://<yourdomain.com>` still loads the website, and the VPN still connects.
- [ ] `https://n8n.<yourdomain.com>` asks for basic auth, then n8n's own login; a manual test workflow runs.
- [ ] `https://n8n.<yourdomain.com>/webhook/does-not-exist` returns 404 without basic auth.
- [ ] `https://<OPENCLAW_SUBDOMAIN>.<yourdomain.com>` asks for basic auth, then the gateway token, then device approval.
- [ ] The Telegram bot answers you, and gives an unapproved account only a pairing code.
- [ ] `/opt/backups/backup.sh` runs clean, both apps are Up afterwards, and a test restore decrypts and lists `n8n/data/database.sqlite`.
- [ ] n8n encryption key, gateway token, both basic-auth passwords, n8n owner password and bot token are in the password manager.

## 10. Maintenance cadence

- **Weekly:** check OpenClaw releases/advisories against `OPENCLAW_VERSION` in `/opt/openclaw/.env`.
  Update immediately on a critical advisory:
  ```
  cd /opt/openclaw
  sed -i 's/^OPENCLAW_VERSION=.*/OPENCLAW_VERSION=<NEW_YYYY.M.D>/' .env
  docker compose pull && docker compose up -d
  # expected: "Pulled", then "Container openclaw-gateway  Started"
  oc doctor
  # expected: checks pass; if it suggests fixes and the release notes call for it: oc doctor --fix
  oc --version
  # expected: <NEW_YYYY.M.D>
  ```
- **Monthly:** everything from base guide §9, plus:
  - n8n: compare `N8N_VERSION` with §4.1, read the release notes, then
    ```
    cd /opt/n8n
    /opt/backups/backup.sh          # fresh backup before every n8n upgrade (DB migrations aren't reversible)
    sed -i 's/^N8N_VERSION=.*/N8N_VERSION=<NEW_X.Y.Z>/' .env
    docker compose pull && docker compose up -d
    docker compose logs --tail 30 n8n
    # expected: migration lines (if any), then "Editor is now accessible via:"
    ```
  - Docker Engine updates come through `apt full-upgrade` (Docker's repo). Containers restart
    automatically (`restart: unless-stopped`).
  - Clean up old images: `docker image prune -a -f` → expected: `Total reclaimed space: ...`.
  - `docker system df` → expected: images/containers/volumes sizes you can live with.
- **Quarterly:** rotate the basic-auth passwords (`htpasswd -B /etc/nginx/auth/<file> <user>`, no
  `-c`, then `systemctl reload nginx`), review `oc devices list` and remove devices you no longer
  use, and test a restore of `n8n.tar.gz`.
- **Never rotate** `N8N_ENCRYPTION_KEY` casually: changing it makes every stored n8n credential unreadable.

## 11. Troubleshooting

| Symptom | Likely cause | Check / fix |
|---|---|---|
| 521 on `n8n.` / `<OPENCLAW_SUBDOMAIN>.` | Hostname missing from the Origin Rule → Cloudflare hit Xray on 443 | §3.2 expression |
| 502 Bad Gateway (nginx page) | Container down or not on 127.0.0.1 | `docker compose ps`; `curl http://127.0.0.1:5678/healthz` / `:18789/healthz` |
| 400 "No required SSL certificate was sent" | AOP not active, or the server block lacks `ssl_client_certificate` | Base guide §3.5; compare with `website.conf` |
| 500 right after the basic-auth prompt | Nginx can't read the htpasswd file, or doesn't support its hash | `tail /var/log/nginx/error.log`; `chmod 640` + `chown root:www-data`; if the log says `crypt_r() failed`, recreate the file without `-B` |
| n8n "Connection lost" banner | WebSocket upgrade not passed | `/etc/nginx/conf.d/websocket-map.conf` exists, `nginx -t`, Cloudflare WebSockets On |
| n8n shows `http://` or `localhost` webhook URLs | `WEBHOOK_URL` / `N8N_HOST` wrong | `/opt/n8n/.env`, then `docker compose up -d` |
| OpenClaw UI: origin not allowed | `allowedOrigins` doesn't match the address bar exactly | §5.4; `oc config get gateway.controlUi.allowedOrigins` |
| OpenClaw UI: unauthorized / 1008 | Wrong gateway token | Paste the value from `/opt/openclaw/.env` |
| OpenClaw UI: pairing required, stuck | Device not approved yet | `oc devices list` → `oc devices approve <requestId>` |
| OpenClaw logs warn about untrusted proxy / wrong client IP | Proxy IP differs from `172.30.0.1` | `docker network inspect openclaw_net --format '{{(index .IPAM.Config 0).Gateway}}'` must print `172.30.0.1` |
| `oc ...` fails: container not running | Gateway stopped; `openclaw-cli` shares its network | `docker compose -f /opt/openclaw/docker-compose.yml up -d`, or use `oc-offline` |
| Containers killed, `dmesg` shows "Out of memory" | Not enough RAM | Add swap (§1.2) or a bigger plan |
| 524 on a webhook | Workflow ran > ~100 s before responding | Webhook node → Respond: Immediately |
