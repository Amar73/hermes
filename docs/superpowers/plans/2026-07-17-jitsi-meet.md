# Jitsi Meet on ovh Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Jitsi Meet (docker-jitsi-meet) on the `ovh` server, served at `https://meet.marko-lab.com:50443` through the existing Caddy, with secure-domain auth (registered users create rooms, guests join by link with a mandatory display name).

**Architecture:** Four official Jitsi containers (web/prosody/jicofo/jvb) via docker compose in `/opt/jitsi-meet`; `web` bound to `127.0.0.1:8000`, Caddy terminates TLS on :50443 and reverse-proxies to it; JVB publishes `10000/udp` directly and advertises `51.75.66.60`. Existing Hermes stack (Caddy site `hermes.marko-lab.com`, hermes-gateway, hermes-dashboard) must remain untouched and working.

**Tech Stack:** Docker Engine + compose plugin (official `download.docker.com` apt repo), docker-jitsi-meet stable release, Caddy (already installed), ufw.

## Global Constraints

- Server: `ssh ovh` (51.75.66.60, user `amar`). `sudo` requires a password by default; Task 1 re-creates the temporary NOPASSWD grant as `/etc/sudoers.d/zzz-amar-temp` (must be `zzz-` — a pre-existing `/etc/sudoers.d/amar` overrides alphabetically-earlier files). Task 7 removes it.
- Public URL is exactly `https://meet.marko-lab.com:50443`. Port 443 stays allowlisted to 5 IPs — do not touch existing ufw rules; only ADD `50443/tcp` (all) and `10000/udp` (all).
- Auth: `ENABLE_AUTH=1`, `AUTH_TYPE=internal`, `ENABLE_GUESTS=1`. Internal XMPP domain default `meet.jitsi` (verify against the release's `env.example`).
- Guests must be forced to enter a display name (`requireDisplayName: true` in effective config.js — exact docker mechanism discovered empirically in Task 5).
- Do not edit `/etc/caddy/Caddyfile`'s existing `hermes.marko-lab.com` block.
- docker-jitsi-meet version: latest **stable** release tag from https://github.com/jitsi/docker-jitsi-meet/releases (not unstable).
- Docs of record: `Amar73/hermes` repo (clone on this machine at `~/Amar73/hermes` and on the server at `~/Amar73/hermes`).
- Known env quirks on ovh: non-interactive `ssh ovh 'cmd'` needs `export PATH="$HOME/.local/bin:$HOME/.hermes/bin:$PATH"` for hermes/claude tools (not needed for docker/caddy/ufw which live in system paths).

---

### Task 1: Temporary sudo + Docker Engine

**Files:** none (server state only)

**Interfaces:**
- Produces: passwordless sudo for the session; working `docker` + `docker compose` for Tasks 2-6.

- [ ] **Step 1: Human checkpoint — re-create the temp NOPASSWD grant**

Ask the human to run (they type their sudo password):
```bash
ssh -t ovh 'echo "amar ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/zzz-amar-temp && sudo chmod 440 /etc/sudoers.d/zzz-amar-temp && echo OK'
```
Then verify from here: `ssh ovh 'sudo -n whoami'` → `root`.

- [ ] **Step 2: Install Docker Engine from the official repo**

```bash
ssh ovh '
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
'
```
Expected: exits 0.

- [ ] **Step 3: Verify**

```bash
ssh ovh 'sudo docker --version && sudo docker compose version && sudo docker run --rm hello-world | head -3'
```
Expected: versions print; hello-world prints "Hello from Docker!".

---

### Task 2: docker-jitsi-meet project in /opt/jitsi-meet

**Files:** creates `/opt/jitsi-meet/` (release tree + `.env`) on server

**Interfaces:**
- Consumes: Docker from Task 1.
- Produces: 4 running containers; `web` on `127.0.0.1:8000`; `jvb` on `0.0.0.0:10000/udp`. Config dir `/opt/jitsi-meet/.jitsi-meet-cfg/` (CONFIG env) used by Task 5.

- [ ] **Step 1: Fetch the latest stable release**

```bash
ssh ovh '
TAG=$(curl -fsSL https://api.github.com/repos/jitsi/docker-jitsi-meet/releases/latest | grep -oP "\"tag_name\":\s*\"\K[^\"]+")
echo "release: $TAG"
sudo mkdir -p /opt/jitsi-meet && sudo chown amar:amar /opt/jitsi-meet
curl -fsSL "https://github.com/jitsi/docker-jitsi-meet/archive/refs/tags/${TAG}.tar.gz" | tar xz --strip-components=1 -C /opt/jitsi-meet
ls /opt/jitsi-meet | head'
```
Expected: a `stable-NNNN` tag; tree contains `docker-compose.yml`, `env.example`, `gen-passwords.sh`.

- [ ] **Step 2: Create .env with our settings**

```bash
ssh ovh '
cd /opt/jitsi-meet
cp env.example .env
./gen-passwords.sh
sed -i "s|^HTTP_PORT=.*|HTTP_PORT=127.0.0.1:8000|" .env
sed -i "s|^#\?PUBLIC_URL=.*|PUBLIC_URL=https://meet.marko-lab.com:50443|" .env
sed -i "s|^#\?ENABLE_AUTH=.*|ENABLE_AUTH=1|" .env
sed -i "s|^#\?ENABLE_GUESTS=.*|ENABLE_GUESTS=1|" .env
sed -i "s|^#\?AUTH_TYPE=.*|AUTH_TYPE=internal|" .env
sed -i "s|^#\?TZ=.*|TZ=Europe/Moscow|" .env
grep -nE "^(HTTP_PORT|PUBLIC_URL|ENABLE_AUTH|ENABLE_GUESTS|AUTH_TYPE|CONFIG)=" .env'
```
Then set the JVB advertise address — the variable name differs across releases; check which exists in `env.example` (`JVB_ADVERTISE_IPS` in current stable, `DOCKER_HOST_ADDRESS` in older) and set it to `51.75.66.60`, uncommenting if needed. Also confirm from `docker-compose.yml` that the web port mapping is `"${HTTP_PORT}:80"` so `127.0.0.1:8000` yields a loopback bind — if the mapping format differs, adapt (goal: web NOT reachable on public IP). If the release requires disabling its own TLS handling for reverse-proxy use (e.g. `ENABLE_LETSENCRYPT`), leave those flags at defaults that keep the web container plain-HTTP.
Expected: all values echo back correctly.

- [ ] **Step 3: Start and verify containers**

```bash
ssh ovh 'cd /opt/jitsi-meet && sudo docker compose up -d && sleep 20 && sudo docker compose ps'
ssh ovh 'ss -tlnp | grep 8000; ss -ulnp | grep 10000'
ssh ovh 'curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/'
```
Expected: 4 containers Up; `127.0.0.1:8000` (tcp, loopback only) and `*:10000` (udp) listening; curl returns 200.

---

### Task 3: ufw rules

**Files:** none

**Interfaces:**
- Consumes: nothing (independent).
- Produces: `50443/tcp` and `10000/udp` open to all; existing rules untouched.

- [ ] **Step 1: Add rules**

```bash
ssh ovh 'sudo ufw allow 50443/tcp && sudo ufw allow 10000/udp && sudo ufw status verbose'
```
Expected: both rules listed as `ALLOW IN Anywhere`; the pre-existing 22/443 five-IP rules and 80/tcp unchanged (compare against the deployment spec's list).

---

### Task 4: Caddy site on :50443

**Files:** Modify `/etc/caddy/Caddyfile` (append new block only)

**Interfaces:**
- Consumes: web container on 127.0.0.1:8000 (Task 2).
- Produces: `https://meet.marko-lab.com:50443` live with LE cert.

- [ ] **Step 1: Append the site block**

```bash
ssh ovh 'printf "\nhttps://meet.marko-lab.com:50443 {\n    reverse_proxy 127.0.0.1:8000\n}\n" | sudo tee -a /etc/caddy/Caddyfile && sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl reload caddy'
```
Expected: `Valid configuration`, reload OK. (If Hermes-style Host-header rejection appears later, `header_up Host` can be added — but Jitsi web has no such defense by default; don't add preemptively.)

- [ ] **Step 2: Verify TLS + UI from outside**

```bash
curl -sI https://meet.marko-lab.com:50443 | head -3
curl -s https://meet.marko-lab.com:50443 | grep -io "<title>[^<]*" | head -1
```
Expected: HTTP 200 with valid cert (no TLS errors), title mentions Jitsi Meet. Check `journalctl -u caddy` for ACME success if the cert takes a minute.

---

### Task 5: Secure-domain auth + mandatory display name

**Files:** `/opt/jitsi-meet/.jitsi-meet-cfg/web/custom-config.js` (or env var — discovered empirically)

**Interfaces:**
- Consumes: running stack (Task 2), public URL (Task 4).
- Produces: user `amar` registered; guests forced to enter a name.

- [ ] **Step 1: Register the first user**

Generate a password locally (`openssl rand -base64 15`), record it for the human, then:
```bash
ssh ovh 'cd /opt/jitsi-meet && sudo docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register amar meet.jitsi "<GENERATED_PASSWORD>"'
```
(Verify the internal domain is `meet.jitsi` via `grep ^XMPP_DOMAIN .env` first; use the actual value.) Expected: silent success or "User registered".

- [ ] **Step 2: Discover and set requireDisplayName**

Check whether the release's `env.example` has a dedicated variable (`grep -i display /opt/jitsi-meet/env.example`). If yes — set it in `.env` and `sudo docker compose up -d` to recreate. If no — append to the web config volume:
```bash
ssh ovh 'echo "config.requireDisplayName = true;" >> /opt/jitsi-meet/.jitsi-meet-cfg/web/custom-config.js && cd /opt/jitsi-meet && sudo docker compose restart web'
```
(Path assumption `CONFIG=~/.jitsi-meet-cfg` was overridden to the project dir in Task 2's .env — use the actual `CONFIG` value; custom-config.js is appended to config.js by the web container entrypoint.) Verify the effective config:
```bash
curl -s https://meet.marko-lab.com:50443/config.js | grep -i requireDisplayName
```
Expected: `requireDisplayName: true` (or the custom-config append visible).

- [ ] **Step 3: Human checkpoint — auth flow test in a real browser**

Ask the human to verify three things at `https://meet.marko-lab.com:50443`:
1. Creating a room prompts "Waiting for the host / I am the host" → logging in with `amar` + password works and starts the room;
2. A second participant (incognito window) joining the room link gets in WITHOUT a password;
3. The guest cannot join without entering a display name.
Do not proceed until the human confirms all three.

---

### Task 6: Real two-party video test

**Files:** none

- [ ] **Step 1: Human checkpoint — media path**

Ask the human: join the same room from two devices (e.g. laptop + phone on mobile network, not same LAN) and confirm both see and hear each other for >30 seconds. This validates UDP 10000 and the advertised IP. If media fails while chat works, check `sudo docker compose logs jvb | tail -30` for the advertised address and harvest errors — fix `JVB_ADVERTISE_IPS`/`DOCKER_HOST_ADDRESS` and recreate jvb.

---

### Task 7: Cleanup + docs

**Files:** Modify `docs/superpowers/specs/2026-07-17-jitsi-meet-design.md` (check off checklist, add «Реально сделано»)

- [ ] **Step 1: Remove temp sudo**

```bash
ssh ovh 'sudo rm /etc/sudoers.d/zzz-amar-temp'
ssh ovh 'sudo -n true 2>&1; echo "exit:$?"'
```
Expected: `exit:1` (password required again).

- [ ] **Step 2: Update the spec**

Check off the spec's checklist items with dates; add «Реально сделано» section: actual release tag, actual JVB advertise variable name, actual requireDisplayName mechanism, the user-management command verified working, any deviations.

- [ ] **Step 3: Commit, push, sync server**

```bash
cd ~/Amar73/hermes && git add -A && git commit -m "Record actual Jitsi Meet deployment results" && git push
ssh ovh 'cd ~/Amar73/hermes && git pull'
```
Expected: same commit hash on both.

---

## Self-Review Notes

- Spec coverage: Docker install (T1), compose+auth env (T2), ufw (T3), Caddy/TLS (T4), secure domain + display name + first user + documented user-add command (T5), real video check (T6), sudo cleanup + docs (T7). All spec checklist lines map to a task.
- Empirical-discovery points carried explicitly: JVB advertise variable name (T2), loopback bind mapping format (T2), requireDisplayName mechanism (T5), XMPP domain (T5).
- No placeholder passwords: generated at execution time and handed to the human, never committed.
