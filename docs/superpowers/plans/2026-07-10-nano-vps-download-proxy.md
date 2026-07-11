# Nano-VPS Download Proxy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route the media-downloader bot's yt-dlp downloads through a SOCKS5 proxy on the dual-stack nano-VPS (Helsinki), so YouTube videos served from IPv6-only CDN edges become downloadable from the IPv6-less `hermes` server — without migrating anything.

> **STATUS (2026-07-11): EXECUTED AND VERIFIED LIVE.** All 4 tasks done via subagent-driven-development, every task review clean. Bot code at `Amar73/telegram-bots` commit `a280e46`, deployed on `hermes` with `PROXY_URL` set. The decisive user-confirmed result: both previously-failing YouTube videos (IPv6-only CDN edges) now download and arrive in Telegram; Instagram and TikTok regression-tested working through the proxy. Notable deviation from plan: Task 1 found and neutralized a pre-existing `/etc/ssh/sshd_config.d/00password.conf` on the nano-VPS that was silently re-enabling password auth (include-order shadowing); renamed to `.disabled`.

**Architecture:** Two independent halves. Server half: harden the nano-VPS (ufw/fail2ban/ssh, mirroring `hermes`) and run `microsocks` with auth on port 12050, reachable only from `hermes`'s IP. Bot half: a new pure function `build_common_opts(proxy_url)` in `downloader.py` returns `{"proxy": ...}` when `PROXY_URL` is set and today's `{"source_address": "0.0.0.0"}` when it isn't; `_download_sync` uses it. Rollback = remove `PROXY_URL` from `.env` and restart.

**Tech Stack:** Debian 13 (nano-VPS), ufw, fail2ban, microsocks 1.0.5 (Debian package), systemd; Python + yt-dlp (bot, existing venv), pytest.

**Spec:** `docs/superpowers/specs/2026-07-10-nano-vps-download-proxy-design.md` (in the `hermes` repo). Bot code lives in a different repo: `Amar73/telegram-bots`, local checkout `/home/amar/Amar73/telegram-bots`.

## Global Constraints

- nano-VPS: `ssh nano-vps`, IPv4 `31.76.43.20`, user `amar` (passwordless sudo), Debian 13 trixie. Already verified dual-stack: yt-dlp downloaded a real YouTube file over forced IPv6 from this box on 2026-07-10.
- `hermes` server egress IP (the only IP allowed to reach the proxy): `193.228.139.46`.
- User's admin IPs (the only IPs allowed to SSH to the nano-VPS): home `46.34.141.146`, work `144.206.228.59`.
- SOCKS5: port **12050**, user **`mdbot`**, password generated once during Task 2 (`openssl rand -hex 16`) — lives ONLY in the nano-VPS systemd unit and in `hermes`'s `.env`. Never committed to git, never echoed into shell history files.
- Proxy URL scheme MUST be `socks5h://` (DNS resolved by the proxy) — `socks5://` would resolve AAAA records locally on `hermes`, which cannot connect to them, defeating the whole point.
- `PROXY_URL` unset/empty ⇒ bot behaves exactly as today (direct, force-IPv4). No other behavior change is in scope.
- yt-dlp has built-in SOCKS support (`yt_dlp/socks.py`) — `requirements.txt` does NOT change. If the smoke test in Task 3 shows otherwise, stop and investigate rather than adding dependencies silently.
- fail2ban on this Debian image needs `backend = systemd` (no rsyslog/auth.log — same issue already solved on `hermes`).
- Before `ufw enable`, ALWAYS confirm the current SSH session's source IP is in the allowlist (`echo $SSH_CONNECTION` on the box) — lesson from the `hermes` hardening.
- All commands run from the local machine via `ssh nano-vps "..."` or `ssh amar@hermes "..."`. None are long-running (no detached-launch pattern needed in this plan).

---

### Task 1: Harden the nano-VPS (ssh check, ufw, fail2ban)

**Files:** None in any repo — remote server state only. Creates on the nano-VPS: `/etc/fail2ban/jail.d/sshd-systemd-backend.local`.

**Interfaces:**
- Consumes: nothing.
- Produces: a nano-VPS where inbound traffic is default-deny, SSH (22) is reachable only from `46.34.141.146` and `144.206.228.59`, and fail2ban guards sshd. Task 2 adds one more ufw rule on top of this baseline.

- [x] **Step 1: Verify SSH is already key-only; fix if not**

```bash
ssh nano-vps "sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'"
```
Expected: `passwordauthentication no` and `permitrootlogin no` (or `prohibit-password`).

If `passwordauthentication yes`, apply a drop-in and restart — but FIRST confirm key auth works (it does — this session is key-based):
```bash
ssh nano-vps "printf 'PasswordAuthentication no\nPermitRootLogin no\n' | sudo tee /etc/ssh/sshd_config.d/90-hardening.conf && sudo systemctl restart ssh && sudo sshd -T | grep -i passwordauthentication"
```
Expected: `passwordauthentication no`.

- [x] **Step 2: Confirm the current session comes from an allowlisted IP**

```bash
ssh nano-vps 'echo "$SSH_CONNECTION"'
```
Expected: first field is `46.34.141.146` or `144.206.228.59`. **If it is anything else, STOP — enabling ufw with this ruleset would lock the session's network out. Ask the user before proceeding.**

- [x] **Step 3: Install ufw + fail2ban**

```bash
ssh nano-vps "sudo apt-get update -qq && sudo apt-get install -y -qq ufw fail2ban"
```
Expected: both install without errors (exit 0).

- [x] **Step 4: Configure fail2ban's sshd jail for journald BEFORE starting it**

```bash
ssh nano-vps "printf '[sshd]\nenabled = true\nbackend = systemd\n' | sudo tee /etc/fail2ban/jail.d/sshd-systemd-backend.local && sudo systemctl restart fail2ban && sleep 2 && sudo fail2ban-client status sshd"
```
Expected: `Status for the jail: sshd` with `|- Currently failed:` / `` `- Banned IP list:`` lines — the jail is up. If it errors with "Have not found any log file", the backend line didn't apply — re-check the file content.

- [x] **Step 5: Apply the ufw ruleset and enable**

```bash
ssh nano-vps "sudo ufw --force reset >/dev/null && sudo ufw default deny incoming && sudo ufw default allow outgoing && sudo ufw allow from 46.34.141.146 to any port 22 proto tcp && sudo ufw allow from 144.206.228.59 to any port 22 proto tcp && sudo ufw --force enable && sudo ufw status numbered"
```
Expected: `Status: active` and exactly two ALLOW rules (22/tcp from each admin IP).

- [x] **Step 6: Verify access survives with a fresh, second connection**

```bash
ssh -o ConnectTimeout=10 nano-vps "echo still-reachable && sudo ufw status | head -3"
```
Expected: `still-reachable`, `Status: active`. (A NEW connection proves the firewall rules, not just an already-open socket.)

---

### Task 2: microsocks SOCKS5 proxy on the nano-VPS

**Files:** None in any repo — remote server state only. Creates on the nano-VPS: `/etc/systemd/system/microsocks-mdbot.service`.

**Interfaces:**
- Consumes: Task 1's ufw baseline (default deny).
- Produces: a working SOCKS5 endpoint `socks5h://mdbot:<PASSWORD>@31.76.43.20:12050`, reachable only from `193.228.139.46`, with IPv6 egress. Task 3's smoke test and Task 4's `.env` use exactly this URL. The generated `<PASSWORD>` must be carried forward to Task 4 (it exists nowhere else but the unit file).

- [x] **Step 1: Install microsocks and generate the password**

```bash
ssh nano-vps "sudo apt-get install -y -qq microsocks && openssl rand -hex 16"
```
Expected: package installs; the last output line is a 32-char hex string. **Save it — it is `<PASSWORD>` in every later step of this plan.** The Debian package may auto-start a default `microsocks.service` (no auth, port 1080); the next step checks and disables it.

- [x] **Step 2: Disable any packaged default instance**

```bash
ssh nano-vps "systemctl list-unit-files 'microsocks*' --no-legend; sudo systemctl disable --now microsocks.service 2>/dev/null; ss -tlnp 2>/dev/null | grep 1080 || echo 'nothing on 1080'"
```
Expected: ends with `nothing on 1080` (an unauthenticated default instance must NOT be left running, even behind ufw).

- [x] **Step 3: Create our systemd unit (auth, port 12050, sandboxed)**

Replace `<PASSWORD>` with the Step 1 value:
```bash
ssh nano-vps "sudo tee /etc/systemd/system/microsocks-mdbot.service > /dev/null" <<'EOF'
[Unit]
Description=microsocks SOCKS5 proxy for the media-downloader bot (hermes egress)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/microsocks -i 0.0.0.0 -p 12050 -u mdbot -P <PASSWORD>
DynamicUser=yes
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
Then:
```bash
ssh nano-vps "sudo chmod 600 /etc/systemd/system/microsocks-mdbot.service && sudo systemctl daemon-reload && sudo systemctl enable --now microsocks-mdbot && systemctl is-active microsocks-mdbot && ss -tln | grep 12050"
```
Expected: `active` and a LISTEN line on `0.0.0.0:12050`. (chmod 600: the password sits in `ExecStart`; microsocks can't read it from a file, and on this single-admin box hiding it from non-root readers is the available mitigation — accepted in the spec.)

- [x] **Step 4: Open port 12050 to hermes only**

```bash
ssh nano-vps "sudo ufw allow from 193.228.139.46 to any port 12050 proto tcp && sudo ufw status numbered"
```
Expected: three ALLOW rules total now (2× port 22, 1× port 12050 from 193.228.139.46).

- [x] **Step 5: Positive test — the proxy works FROM HERMES and egresses over IPv6**

Replace `<PASSWORD>`:
```bash
ssh amar@hermes "curl -sS --max-time 15 -x 'socks5h://mdbot:<PASSWORD>@31.76.43.20:12050' https://api64.ipify.org && echo"
```
Expected: `2a01:ecc0:6000:a7d::2` — the request left through the proxy **and** the proxy reached an IPv6-only-preferring endpoint over IPv6. This single check proves auth, allowlist, and v6 egress at once.

- [x] **Step 6: Negative tests — wrong source IP and wrong password are rejected**

From the local machine (not hermes, so ufw must block):
```bash
curl -sS --max-time 8 -x 'socks5h://mdbot:wrong@31.76.43.20:12050' https://api64.ipify.org; echo "exit=$?"
```
Expected: timeout/connection failure, `exit=` non-zero (ufw drops the SYN — we never even reach auth).

From hermes with a wrong password (auth must reject):
```bash
ssh amar@hermes "curl -sS --max-time 10 -x 'socks5h://mdbot:wrongpass@31.76.43.20:12050' https://api64.ipify.org; echo exit=\$?"
```
Expected: curl error (SOCKS5 auth failed), non-zero exit.

---

### Task 3: Bot change — optional `PROXY_URL` (TDD)

**Files:**
- Modify: `/home/amar/Amar73/telegram-bots/media-downloader/downloader.py` (new function + `_download_sync` lines 45–51)
- Modify: `/home/amar/Amar73/telegram-bots/media-downloader/tests/test_downloader.py` (append tests)
- Modify: `/home/amar/Amar73/telegram-bots/media-downloader/.env.example` (document the new var)

**Interfaces:**
- Consumes: the proxy URL format from Task 2 (`socks5h://mdbot:<PASSWORD>@31.76.43.20:12050`) — but only in the smoke-test step; the code itself just reads `os.environ["PROXY_URL"]`.
- Produces: `build_common_opts(proxy_url: str | None) -> dict` in `downloader.py` — pure, unit-tested; `_download_sync` consumes it. Task 4 relies on the deployed bot honoring `PROXY_URL` from `.env`.

- [x] **Step 1: Write the failing tests**

Append to `/home/amar/Amar73/telegram-bots/media-downloader/tests/test_downloader.py` (and extend the import at line 1 to `from downloader import build_common_opts, select_format`):

```python
def test_build_common_opts_with_proxy_uses_proxy_and_drops_ipv4_forcing():
    opts = build_common_opts("socks5h://mdbot:secret@31.76.43.20:12050")
    assert opts == {"proxy": "socks5h://mdbot:secret@31.76.43.20:12050"}


def test_build_common_opts_without_proxy_forces_ipv4():
    assert build_common_opts(None) == {"source_address": "0.0.0.0"}


def test_build_common_opts_empty_string_means_unset():
    assert build_common_opts("") == {"source_address": "0.0.0.0"}
```

- [x] **Step 2: Run the new tests to verify they fail**

```bash
cd /home/amar/Amar73/telegram-bots/media-downloader && ../.venv/bin/python -m pytest tests/test_downloader.py -v -k build_common_opts
```
Expected: collection error / ImportError — `build_common_opts` doesn't exist yet.

- [x] **Step 3: Implement `build_common_opts` and wire it into `_download_sync`**

In `/home/amar/Amar73/telegram-bots/media-downloader/downloader.py`, add `import os` to the imports (after `import asyncio`), then replace lines 45–51 (`def _download_sync(...)`, the force-IPv4 comment block, and the `common_opts = {...}` line) with exactly this — the new function goes right above `_download_sync`:

```python
def build_common_opts(proxy_url: str | None) -> dict:
    """yt-dlp options shared by the probe and download calls.

    With a proxy (PROXY_URL set): route everything through it. The proxy host
    (nano-vps, Helsinki) is dual-stack, which fixes YouTube videos served from
    IPv6-only CDN edges — this server has no IPv6 at all and its provider
    refused to add it. The URL must be socks5h:// so DNS is resolved by the
    proxy; a locally-resolved IPv6 address would be unreachable from here.

    Without a proxy: force IPv4 (same as yt-dlp's --force-ipv4). Some
    googlevideo hosts resolve to IPv6 anyway and fail with "Network is
    unreachable" (Errno 101) instead of falling back — found via a real
    failed download during the original end-to-end test.
    """
    if proxy_url:
        return {"proxy": proxy_url}
    return {"source_address": "0.0.0.0"}


def _download_sync(url: str, output_dir: str, max_bytes: int) -> str:
    common_opts = build_common_opts(os.environ.get("PROXY_URL"))
```

(The rest of `_download_sync` from `probe_opts = ...` onward is unchanged.)

- [x] **Step 4: Run the full test suite**

```bash
cd /home/amar/Amar73/telegram-bots/media-downloader && ../.venv/bin/python -m pytest tests/ -v
```
Expected: all tests pass (the 3 new ones plus every pre-existing test — `select_format` behavior is untouched).

- [x] **Step 5: Document the variable in `.env.example`**

Append to `/home/amar/Amar73/telegram-bots/media-downloader/.env.example`:

```
# Optional: SOCKS5 proxy for ALL downloads (dual-stack egress via the nano-vps
# in Helsinki — fixes YouTube videos on IPv6-only CDN edges, since this server
# has no IPv6). Scheme must be socks5h:// so DNS resolves on the proxy side.
# Unset/empty = download directly, forcing IPv4 (previous behavior).
PROXY_URL=socks5h://mdbot:password@31.76.43.20:12050
```

- [x] **Step 6: Smoke-test yt-dlp's SOCKS support through the real proxy (no bot, no Telegram)**

Replace `<PASSWORD>` with the Task 2 value; runs on hermes with the bot's own venv and a real yt-dlp call:
```bash
ssh amar@hermes "cd ~/Amar73/telegram-bots/media-downloader && PROXY_URL='socks5h://mdbot:<PASSWORD>@31.76.43.20:12050' ../.venv/bin/python -c \"
import asyncio, os, tempfile
from downloader import download_video
with tempfile.TemporaryDirectory() as d:
    p = asyncio.run(download_video('https://www.youtube.com/watch?v=jNQXAC9IVRw', d, 49*1024*1024))
    print('OK', os.path.getsize(p), 'bytes')
\""
```
Expected: `OK 6xxxxx bytes` — proves yt-dlp's built-in SOCKS client works with our exact URL from the exact machine that will use it. NOTE: this needs Task 4's Step 1 `git pull` to have happened OR run it after pushing; if running Task 3 standalone, do this step from the local checkout instead (same command without `ssh amar@hermes`, using the local venv — it proves yt-dlp+socks5h; the from-hermes variant is re-proven by Task 4's e2e anyway).

- [x] **Step 7: Commit and push**

```bash
cd /home/amar/Amar73/telegram-bots && git add media-downloader/downloader.py media-downloader/tests/test_downloader.py media-downloader/.env.example && git commit -m "Route downloads through an optional SOCKS5 proxy (PROXY_URL)

The hermes server has no IPv6 and its provider refused to add it, so
YouTube videos on IPv6-only CDN edges were undownloadable. With PROXY_URL
set (socks5h via the dual-stack nano-vps in Helsinki) all downloads get
IPv6-capable egress; unset, behavior is unchanged (direct, force-IPv4)." && git push
```
Expected: clean push to `origin/main`.

---

### Task 4: Deploy to hermes + live end-to-end test

**Files:** None in any repo — remote server state (`/home/amar/Amar73/telegram-bots` checkout and `media-downloader/.env` on hermes) + live Telegram verification.

**Interfaces:**
- Consumes: Task 2's proxy URL (with the real `<PASSWORD>`), Task 3's pushed commit.
- Produces: the running production bot downloading through Helsinki.

- [x] **Step 1: Pull the new code on hermes**

```bash
ssh amar@hermes "cd ~/Amar73/telegram-bots && git pull --ff-only && git log --oneline -1"
```
Expected: fast-forward to the Task 3 commit.

- [x] **Step 2: Add PROXY_URL to the production `.env`**

Replace `<PASSWORD>`:
```bash
ssh amar@hermes "printf 'PROXY_URL=socks5h://mdbot:<PASSWORD>@31.76.43.20:12050\n' >> ~/Amar73/telegram-bots/media-downloader/.env && grep -c PROXY_URL ~/Amar73/telegram-bots/media-downloader/.env"
```
Expected: `1` (exactly one PROXY_URL line — if >1, an earlier attempt already added it; dedupe by editing the file instead of appending again).

- [x] **Step 3: Restart and check the service**

```bash
ssh amar@hermes "sudo systemctl restart media-downloader-bot && sleep 3 && systemctl is-active media-downloader-bot && sudo journalctl -u media-downloader-bot -n 5 --no-pager"
```
Expected: `active`, log shows "Application started", no tracebacks.

- [x] **Step 4: Live test — previously-failing YouTube links**

Ask the user for 1–2 YouTube links that previously failed with «не удалось скачать видео» (they're in their chat history with `@amardownloader_bot`), and ask them to resend those links to the bot now. In the same message, ask them to check the nano-VPS tariff's monthly traffic allowance in the provider panel — video traffic now transits that box twice (spec's accepted cost, but the number should be known, not assumed).

Expected: the videos ARRIVE (this is the whole point of the project). If one still fails, read `sudo journalctl -u media-downloader-bot -n 50` — distinguish proxy errors (auth/connect — a Task 2/config problem) from a genuine yt-dlp failure (different root cause; investigate with systematic-debugging, don't paper over).

- [x] **Step 5: Regression — Instagram Reel and TikTok through the proxy**

Ask the user to send one Instagram Reel link and one TikTok link (any that worked before).

Expected: both still download fine (all traffic now egresses via Helsinki — this checks those extractors tolerate the new exit IP).

- [x] **Step 6: Confirm cleanup + quiet logs, then document**

```bash
ssh amar@hermes "ls /tmp/media-downloader/ | wc -l && sudo journalctl -u media-downloader-bot --since '-15 min' --no-pager | grep -ciE 'error|traceback' || true"
```
Expected: `0` files (or only in-flight ones), `0` error lines.

Then mark this plan's checkboxes done and commit the plan-doc update in the `hermes` repo:
```bash
cd /home/amar/Amar73/hermes && git add docs/superpowers/plans/2026-07-10-nano-vps-download-proxy.md && git commit -m "Mark nano-VPS download proxy plan executed and verified live"
```
