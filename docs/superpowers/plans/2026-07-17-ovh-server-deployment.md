# ovh (51.75.66.60) Server Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden the `ovh` server (51.75.66.60) and install Claude Code, Antigravity CLI, and Hermes Agent directly under user `amar` (no Paperclip), with Hermes's dashboard exposed via Caddy at `hermes.marko-lab.com`, and keep `~/Amar73/hermes` in sync with GitHub on both the local machine and the server.

**Architecture:** Everything runs under the existing unprivileged user `amar` on a single Debian 13 VPS — no dedicated service user, no orchestrator, no database. Each tool is installed via its own official install script. Hermes Agent's dashboard and gateway API stay bound to `127.0.0.1`; Caddy is the only process that binds a public port (80/443) and reverse-proxies to Hermes's dashboard. All work happens over `ssh ovh` (already configured, key-based, resolves via `/etc/hosts`).

**Tech Stack:** Debian 13 (trixie), ufw, fail2ban, unattended-upgrades, Caddy, Claude Code (native installer), Antigravity CLI (Google, single binary), Hermes Agent (NousResearch, `uv`/Python 3.11/Node.js), `gh` CLI, git.

## Global Constraints

- Server reached via `ssh ovh` (alias resolves `ovh` → `51.75.66.60` through `/etc/hosts`; user `amar`, key-based auth already working).
- Domain `hermes.marko-lab.com` → `51.75.66.60` — DNS A-record already live, do not touch DNS.
- SSH (22/tcp) and HTTPS (443/tcp) allowed **only** from: `46.34.141.146`, `144.206.228.59`, `144.206.226.221`, `178.250.191.152`, `144.31.81.96`. Port 80/tcp stays open to all (ACME challenge only).
- `fail2ban`'s `sshd` jail must use `backend = systemd` from the start — this Debian image has no rsyslog/`/var/log/auth.log`, and the default backend fails on it (hit on the previous server).
- Hermes dashboard binds `127.0.0.1:9119`; Hermes gateway API binds `127.0.0.1:8642`. Neither is ever exposed on a public interface — only Caddy reverse-proxies the dashboard.
- No Codex CLI, no Paperclip, no multi-agent org-chart pipeline in this round.
- `git@github.com:Amar73/hermes.git` is the repo of record; git identity on any machine touching it is `Andrey Maryanenko <a.maryanenko@gmail.com>`.
- Passwordless sudo for `amar` is a **permanent, intended** setting, at `/etc/sudoers.d/amar` (`amar ALL=(ALL:ALL) NOPASSWD:ALL`, `0440`). Every `sudo` command in this plan relies on it, and it stays after deployment — Claude Code sessions run on the server and do sudo work non-interactively, so a password prompt would hang them. (Decided 2026-07-21; earlier drafts of this plan wrongly framed it as temporary scaffolding to be removed — see Correction 2.)
- Do not run destructive ufw/sshd changes without verifying the current session's source IP is on the allowlist first (already confirmed: `178.250.191.152`).

---

### Task 1: Base packages and OS update

**Files:** none (server package state only)

**Interfaces:**
- Produces: `git`, `build-essential`, `curl`, `ca-certificates`, `unzip` available on `PATH` for all later tasks.

- [x] **Step 1: Update package index and upgrade existing packages**

Run:
```bash
ssh ovh 'sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y'
```
Expected: exits 0, ends with something like `0 upgraded, 0 newly installed` or a list of upgraded packages, no `E:` errors.

- [x] **Step 2: Install base packages**

Run:
```bash
ssh ovh 'sudo apt install -y git build-essential curl ca-certificates unzip'
```
Expected: exits 0, `git`, `gcc`, `make`, `curl`, `unzip` end up installed.

- [x] **Step 3: Verify**

Run:
```bash
ssh ovh 'git --version && curl --version | head -1 && unzip -v | head -1 && gcc --version | head -1'
```
Expected: four version lines print, no "command not found".

---

### Task 2: ufw firewall with SSH/HTTPS IP allowlist

**Files:** none (ufw rule state on server)

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: active `ufw` with default-deny incoming; ports 22 and 443 reachable only from the 5 allowlisted IPs; port 80 open to all.

- [x] **Step 1: Install ufw**

Run:
```bash
ssh ovh 'sudo apt install -y ufw'
```
Expected: exits 0.

- [x] **Step 2: Set default policies (before enabling — safe, doesn't cut current session)**

Run:
```bash
ssh ovh 'sudo ufw default deny incoming && sudo ufw default allow outgoing'
```
Expected: `Default incoming policy changed to 'deny'`, `Default outgoing policy changed to 'allow'`.

- [x] **Step 3: Allow port 80 from anywhere (ACME)**

Run:
```bash
ssh ovh 'sudo ufw allow 80/tcp'
```
Expected: `Rule added`.

- [x] **Step 4: Allow 22/tcp and 443/tcp from each of the 5 allowlisted IPs**

Run:
```bash
ssh ovh '
for ip in 46.34.141.146 144.206.228.59 144.206.226.221 178.250.191.152 144.31.81.96; do
  sudo ufw allow from "$ip" to any port 22 proto tcp
  sudo ufw allow from "$ip" to any port 443 proto tcp
done
'
```
Expected: 10 `Rule added` lines (2 per IP).

- [x] **Step 5: Enable ufw**

Run:
```bash
ssh ovh 'sudo ufw --force enable'
```
Expected: `Firewall is active and enabled on system startup`.

- [x] **Step 6: Verify from a fresh SSH connection (not reusing an existing multiplexed session)**

Run:
```bash
ssh -o ControlPath=none ovh 'echo still reachable; sudo ufw status verbose'
```
Expected: prints `still reachable`, then ufw status shows `Status: active`, port 80 `ALLOW Anywhere`, ports 22/443 `ALLOW` only from the 5 IPs listed above. If this command hangs or fails to connect, **do not close any existing open session** — investigate before doing anything else (recovery would require the OVH web console).

---

### Task 3: fail2ban with systemd backend

**Files:** create `/etc/fail2ban/jail.d/sshd-systemd-backend.local` (on server)

**Interfaces:**
- Consumes: nothing from earlier tasks (independent of Task 2).
- Produces: active `fail2ban` service with a working `sshd` jail.

- [x] **Step 1: Install fail2ban**

Run:
```bash
ssh ovh 'sudo apt install -y fail2ban'
```
Expected: exits 0.

- [x] **Step 2: Pre-empt the missing-rsyslog issue by setting the sshd jail's backend to systemd before first start**

Run:
```bash
ssh ovh "printf '[sshd]\nenabled = true\nbackend = systemd\n' | sudo tee /etc/fail2ban/jail.d/sshd-systemd-backend.local"
```
Expected: prints the 3 lines back (tee echoes to stdout).

- [x] **Step 3: Enable and start fail2ban**

Run:
```bash
ssh ovh 'sudo systemctl enable --now fail2ban'
```
Expected: exits 0, no error output.

- [x] **Step 4: Verify the sshd jail is active**

Run:
```bash
ssh ovh 'sudo fail2ban-client status sshd'
```
Expected: `Status for the jail: sshd` block with `Currently failed`, `Total failed`, `Currently banned`, `Total banned` lines — no "Sorry but the jail 'sshd' does not exist" error.

---

### Task 4: unattended-upgrades

**Files:** none (uses Debian's default `unattended-upgrades` config as shipped by the package)

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: automatic security updates applied going forward.

- [x] **Step 1: Install and enable**

Run:
```bash
ssh ovh 'sudo apt install -y unattended-upgrades && sudo systemctl enable --now unattended-upgrades'
```
Expected: exits 0.

- [x] **Step 2: Verify the timer/service is active**

Run:
```bash
ssh ovh 'systemctl is-enabled unattended-upgrades; systemctl is-active unattended-upgrades'
```
Expected: `enabled` then `active`.

---

### Task 5: Claude Code — install and authenticate

**Files:** none (installs to `~/.local/bin` on server, no root)

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: `claude` binary on `PATH`, authenticated session usable by later tasks (e.g. drafting deployment notes from the server itself, if ever needed).

- [ ] **Step 1: Run the native installer**

Run:
```bash
ssh ovh 'curl -fsSL https://claude.ai/install.sh | bash'
```
Expected: completes in well under a minute, ends with a success message pointing at `~/.local/bin/claude`.

- [ ] **Step 2: Verify the binary is on PATH in a fresh session**

Run:
```bash
ssh -o ControlPath=none ovh 'claude --version'
```
Expected: prints a version string, not "command not found". If `PATH` isn't picked up, check `~/.bashrc` for the line the installer appends and re-source it.

- [ ] **Step 3: Start an interactive OAuth login in tmux**

Run:
```bash
ssh ovh 'tmux new-session -d -s claude-login "claude auth login"'
sleep 2
ssh ovh 'tmux capture-pane -p -t claude-login'
```
Expected: pane output contains a `https://` URL and a short instruction to open it and enter a code.

- [ ] **Step 4: Hand the URL to the user, wait for them to complete browser login on their own machine with their Pro account**

This step has no command — it's a checkpoint. Report the captured URL to the user and wait for them to say they've completed the browser-side login and have a code (if the flow requires pasting one back) or confirm it auto-completed.

- [ ] **Step 5: If a code needs to be pasted back into the server-side prompt, send it via tmux**

Run (only if the captured pane is waiting for input):
```bash
ssh ovh "tmux send-keys -t claude-login '<CODE_FROM_USER>' Enter"
sleep 2
ssh ovh 'tmux capture-pane -p -t claude-login'
```
Expected: pane shows a success/logged-in message.

- [ ] **Step 6: Verify login and clean up the tmux session**

Run:
```bash
ssh ovh 'claude auth status'
ssh ovh 'tmux kill-session -t claude-login'
```
Expected: `claude auth status` reports a logged-in account, no error.

---

### Task 6: Antigravity CLI — install and authenticate

**Files:** none

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: `agy` binary on `PATH`, authenticated session.

- [ ] **Step 1: Run the installer**

Run:
```bash
ssh ovh 'curl -fsSL https://antigravity.google/cli/install.sh | bash'
```
Expected: completes, installs a single binary (no Node/Python dependency chain), prints where it was placed.

- [ ] **Step 2: Verify the binary and confirm the exact command name**

Run:
```bash
ssh -o ControlPath=none ovh 'agy --version || agy --help | head -5'
```
Expected: prints a version or help text. If `agy` isn't the right name, `--help` output or the installer's own success message will show the correct one — use that name in every following step instead.

- [ ] **Step 3: Start headless login**

Run:
```bash
ssh ovh 'tmux new-session -d -s agy-login "agy auth login"'
sleep 2
ssh ovh 'tmux capture-pane -p -t agy-login'
```
Expected: pane shows a URL plus a one-time code (headless/SSH sessions print both without needing `send-keys`, per Antigravity's documented headless flow — this is a detection, not an interactive prompt like Claude Code's).

- [ ] **Step 4: Hand the URL and code to the user, wait for them to complete it in a browser on their own machine**

Checkpoint, no command — report the URL/code, wait for confirmation.

- [ ] **Step 5: Verify login and clean up**

Run:
```bash
ssh ovh 'agy auth status || agy status'
ssh ovh 'tmux kill-session -t agy-login'
```
Expected: reports a logged-in account.

---

### Task 7: Hermes Agent — install and base configuration

**Files:** `~/.hermes/.env` and `~/.hermes/config.yaml` on server (created by the installer/CLI, not hand-authored from scratch)

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: working `hermes` CLI with a functioning default chat provider, ready for Task 8 (web-search) and Task 9 (Telegram gateway).

- [ ] **Step 1: Run the installer**

Run:
```bash
ssh ovh 'curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash'
```
Expected: pulls in `uv`, Python 3.11, Node.js, `ripgrep`, `ffmpeg`; ends with a success message.

- [ ] **Step 2: Reload shell and verify**

Run:
```bash
ssh -o ControlPath=none ovh 'source ~/.bashrc; hermes --help | head -5'
```
Expected: prints Hermes CLI help/usage, not "command not found".

- [ ] **Step 3: Run portal setup**

Run:
```bash
ssh ovh 'hermes setup --portal'
```
Expected: completes without error. If it's interactive, run it inside tmux the same way as Tasks 5/6 and report prompts to the user as they come up.

- [ ] **Step 4: Checkpoint — get API keys from the user**

No command. Ask the user for: `OPENROUTER_API_KEY`, `PERPLEXITY_API_KEY`, and (for Task 9) `TELEGRAM_BOT_TOKEN`. Do not proceed to Step 5 with placeholder values.

- [ ] **Step 5: Write the keys into `~/.hermes/.env`**

Run (substituting the real values received in Step 4 — never echo real secret values back into chat/logs beyond what's needed to confirm they were written):
```bash
ssh ovh "cat >> ~/.hermes/.env <<'EOF'
OPENROUTER_API_KEY=<value from user>
PERPLEXITY_API_KEY=<value from user>
EOF"
```
Expected: no output (silent success). Telegram token is added in Task 9 alongside the gateway install, to keep the two changes independently testable.

- [ ] **Step 6: Pick and set a default model**

Ask the user which model to default to (previous deployment used `anthropic/claude-sonnet-5` via OpenRouter — confirm whether that's still the choice, don't assume). Then run:
```bash
ssh ovh "cd ~/.hermes/hermes-agent && uv run hermes config set model <chosen-model>"
```
Expected: confirms the model was set.

- [ ] **Step 7: Verify Hermes can actually talk to the model**

Run:
```bash
ssh ovh 'hermes status'
```
Expected: shows `OpenRouter ✓` (or the relevant provider check), `.env file: ✓ exists`, and the model set in Step 6. If it shows an error or missing credit balance (HTTP 402 seen on the previous deployment when the OpenRouter account had $0 balance), tell the user — this blocks everything downstream that talks to a model.

---

### Task 8: Perplexity web-search backend

**Files:** Modify `~/.hermes/config.yaml` on server

**Interfaces:**
- Consumes: `PERPLEXITY_API_KEY` written to `~/.hermes/.env` in Task 7 Step 5.
- Produces: Hermes's `web_search` tool answering through Perplexity Sonar.

- [ ] **Step 1: Check the real config surface on the installed version before writing anything**

Run:
```bash
ssh ovh 'hermes config --help'
ssh ovh 'cd ~/.hermes/hermes-agent && uv run hermes config get web 2>&1'
```
Expected: `config --help` lists available subcommands/keys; the `get web` call either prints the current (likely empty/default) `web` config or a clear "not set" — either tells us the real key names to use next, since the docs for the exact `web.backend` value that selects a custom provider were unclear. Do not guess past this point — read what the command actually reports.

- [ ] **Step 2: Add the custom provider block**

Run:
```bash
ssh ovh "cat >> ~/.hermes/config.yaml <<'EOF'
custom_providers:
  - name: perplexity
    base_url: https://api.perplexity.ai
    key_env: PERPLEXITY_API_KEY
EOF"
```
Expected: silent success. (If Step 1 revealed a different required structure, e.g. an explicit `web.backend: custom:perplexity` key, add that here too, matching what Step 1 actually showed rather than the doc excerpt above.)

- [ ] **Step 3: Point the web-search tool at it**

Based on Step 1's findings, set the web backend — most likely:
```bash
ssh ovh "cd ~/.hermes/hermes-agent && uv run hermes config set web.backend custom:perplexity"
```
Expected: confirms the value was set. If this key name is rejected, fall back to whatever key `hermes config --help` (Step 1) actually listed for selecting a web-search backend.

- [ ] **Step 4: Verify with a real search query**

Run:
```bash
ssh ovh 'echo "search the web for today'"'"'s top Hacker News post title" | hermes'
```
Expected: the response includes a real, current result (not a hallucinated answer, not an error about missing web-search config). Report the actual output to the user so they can judge whether it's really hitting Perplexity.

---

### Task 9: Telegram gateway

**Files:** creates `/etc/systemd/system/hermes-gateway.service` on server (via `hermes gateway install`, not hand-authored)

**Interfaces:**
- Consumes: working `hermes` CLI from Task 7.
- Produces: `hermes-gateway.service` running and answering paired users on Telegram.

- [ ] **Step 1: Checkpoint — get the Telegram bot token from the user**

No command. The user already has a bot token from the previous deployment's bot, or a new one — confirm which, then get the value.

- [ ] **Step 2: Add the token to `.env`**

Run (substituting the real token):
```bash
ssh ovh "echo 'TELEGRAM_BOT_TOKEN=<value from user>' >> ~/.hermes/.env"
```
Expected: silent success.

- [ ] **Step 3: Verify the token is valid before installing the gateway**

Run:
```bash
ssh ovh "source ~/.hermes/.env && curl -s https://api.telegram.org/bot\$TELEGRAM_BOT_TOKEN/getMe"
```
Expected: `{"ok":true,"result":{...}}`. If `"ok":false`, stop and get a corrected token from the user before continuing.

- [ ] **Step 4: Install and start the gateway as a system service**

Run:
```bash
ssh ovh 'cd ~/.hermes/hermes-agent && uv run hermes --accept-hooks gateway install --system --run-as-user amar --start-now'
```
Expected: reports the systemd unit was created and started.

- [ ] **Step 5: Verify the service is running**

Run:
```bash
ssh ovh 'sudo systemctl status hermes-gateway --no-pager'
```
Expected: `Active: active (running)`.

- [ ] **Step 6: Checkpoint — user sends a first message to the bot in Telegram**

No command. Ask the user to message the bot. The first message won't get a real reply — it returns a one-time pairing code instead.

- [ ] **Step 7: List and approve the pairing**

Run:
```bash
ssh ovh 'cd ~/.hermes/hermes-agent && uv run hermes pairing list'
```
Ask the user for the code they saw *in the Telegram chat* (not the hash shown by `pairing list` — those are different, confirmed on the previous deployment). Then:
```bash
ssh ovh 'cd ~/.hermes/hermes-agent && uv run hermes pairing approve telegram <CODE_FROM_USER_TELEGRAM_CHAT>'
```
Expected: confirms the pairing was approved.

- [ ] **Step 8: Verify end-to-end**

Checkpoint — ask the user to send another message to the bot and confirm they got a real model response back (not another pairing code, not silence).

---

### Task 10: Caddy reverse proxy + TLS for hermes.marko-lab.com

**Files:** create/modify `/etc/caddy/Caddyfile` on server

**Interfaces:**
- Consumes: Hermes dashboard listening on `127.0.0.1:9119` (available since Task 7).
- Produces: `https://hermes.marko-lab.com` serving the Hermes dashboard with a valid Let's Encrypt certificate.

- [ ] **Step 1: Install Caddy from the official repo**

Run:
```bash
ssh ovh '
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf "https://dl.cloudsmith.io/public/caddy/stable/gpg.key" | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf "https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt" | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
'
```
Expected: exits 0, `caddy version` works afterward.

- [ ] **Step 2: Write the Caddyfile**

Run:
```bash
ssh ovh "printf 'hermes.marko-lab.com {\n    reverse_proxy 127.0.0.1:9119\n}\n' | sudo tee /etc/caddy/Caddyfile"
```
Expected: prints the 3-line Caddyfile back.

- [ ] **Step 3: Validate and reload**

Run:
```bash
ssh ovh 'sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl reload caddy'
```
Expected: `Valid configuration`, then no error from `systemctl reload`.

- [ ] **Step 4: Verify HTTPS from outside**

Run (from the local machine, which is on the allowlist):
```bash
curl -vI https://hermes.marko-lab.com 2>&1 | grep -E "SSL certificate|HTTP/"
```
Expected: shows a valid certificate chain (issuer Let's Encrypt) and an HTTP status line (likely 401/200 depending on whether Hermes's dashboard requires login — either is fine, a connection refused or TLS error is not).

---

### Task 11: Clone `~/Amar73/hermes` on the server and wire up GitHub access

**Files:** creates `~/Amar73/hermes/` directory tree on server (git clone of the existing repo — not new content)

**Interfaces:**
- Consumes: nothing from earlier tasks (independent — can run any time after Task 1's `git` install).
- Produces: a `git`-clean, GitHub-connected clone of the same repo the local machine uses, so deployment notes can be committed either from the server or the local machine and stay in sync.

- [ ] **Step 1: Install `gh` CLI from the official apt repo**

Run:
```bash
ssh ovh '
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install -y gh
'
```
Expected: exits 0, `gh --version` works afterward.

- [ ] **Step 2: Authenticate via device flow**

Run:
```bash
ssh ovh 'gh auth login --hostname github.com --git-protocol https --web'
```
Expected: prints a one-time code and a URL. Report both to the user, wait for them to confirm they authorized it in a browser on their own machine (same account, `Amar73`, used locally — confirmed via local `gh auth status` as `Amar73`).

- [ ] **Step 3: Verify auth and set up the git credential helper**

Run:
```bash
ssh ovh 'gh auth status'
ssh ovh 'gh auth setup-git'
```
Expected: `gh auth status` shows logged in as `Amar73`; `setup-git` completes silently or confirms the helper is configured.

- [ ] **Step 4: Set git identity to match the local machine**

Run:
```bash
ssh ovh "git config --global user.email 'a.maryanenko@gmail.com' && git config --global user.name 'Andrey Maryanenko'"
```
Expected: silent success.

- [ ] **Step 5: Clone the repo**

Run:
```bash
ssh ovh 'mkdir -p ~/Amar73 && gh repo clone Amar73/hermes ~/Amar73/hermes'
```
Expected: clone succeeds, ends with a commit hash or "already up to date" style message.

- [ ] **Step 6: Verify push/pull works from the server**

Run:
```bash
ssh ovh 'cd ~/Amar73/hermes && git log -1 --oneline && git fetch origin && git status --short'
```
Expected: shows the latest commit (the design spec added earlier in this project), fetch succeeds, working tree clean.

---

### Task 12: Deployment notes, cleanup, and final verification

**Files:** modify `docs/superpowers/specs/2026-07-17-ovh-server-deployment-design.md` (append a "what actually happened" section, following the pattern used in the previous server's spec)

**Interfaces:**
- Consumes: the full checklist from the design spec's "Чек-лист готовности" section.
- Produces: a closed-out, committed, pushed record of the real end state — matching what the previous deployment's spec did.

- [x] **Step 1: Passwordless sudo grant — keep (not removed)**

Superseded 2026-07-21. This step originally removed a "temporary" NOPASSWD grant, but passwordless sudo for `amar` was decided to be permanent (see the Global Constraints note and Correction 2) — the server's Claude Code sessions do sudo work non-interactively and a password prompt would hang them. The grant at `/etc/sudoers.d/amar` is left in place deliberately. Nothing to run.

- [ ] **Step 2: Run the full checklist from the design spec and record real results**

Run each of these and note the actual output:
```bash
ssh ovh 'sudo ufw status verbose'
ssh ovh 'sudo fail2ban-client status sshd'
ssh ovh 'systemctl is-active unattended-upgrades'
ssh ovh 'claude --version && claude auth status'
ssh ovh 'agy --version && (agy auth status || agy status)'
ssh ovh 'hermes status'
ssh ovh 'sudo systemctl status hermes-gateway --no-pager'
curl -vI https://hermes.marko-lab.com 2>&1 | grep -E "SSL certificate|HTTP/"
ssh ovh 'cd ~/Amar73/hermes && git status --short && git log -1 --oneline'
```
Expected: every command's actual output is captured, not assumed.

- [ ] **Step 3: Append a "Реально сделано" section to the design spec documenting any deviations**

Edit `/home/amar/Amar73/hermes/docs/superpowers/specs/2026-07-17-ovh-server-deployment-design.md`, adding a section after the checklist with what Step 2 actually showed — especially the real `web.backend` config key from Task 8 Step 1/3 if it differed from the doc's guess, and the confirmed `agy` command name from Task 6 Step 2 if it differed.

- [ ] **Step 4: Commit and push from the local machine**

Run:
```bash
cd ~/Amar73/hermes && git add docs/superpowers/specs/2026-07-17-ovh-server-deployment-design.md && git commit -m "Record actual ovh deployment results and close out checklist" && git push
```
Expected: commit succeeds, push updates `origin/master`.

- [ ] **Step 5: Pull on the server to confirm both sides are in sync**

Run:
```bash
ssh ovh 'cd ~/Amar73/hermes && git pull && git log -1 --oneline'
```
Expected: shows the same commit hash just pushed from the local machine.

---

## Self-Review Notes

- **Spec coverage:** Task 1↔base packages, Task 2↔ufw/allowlist, Task 3↔fail2ban, Task 4↔unattended-upgrades, Task 5↔Claude Code, Task 6↔Antigravity CLI, Task 7↔Hermes Agent base install, Task 8↔Perplexity web-search, Task 9↔Telegram gateway, Task 10↔Caddy/domain, Task 11↔GitHub repo sync, Task 12↔checklist closeout. All spec sections have a task.
- **Known open question carried forward, not hidden:** Task 8's exact `web.backend` config key is uncertain per the design spec's own caveat — Task 8 Step 1 makes checking the real CLI output a hard prerequisite before writing config, rather than guessing silently.
- **Command name uncertainty carried forward:** Task 6 Step 2 explicitly checks whether `agy` is the right binary name rather than assuming it.

---

## Execution Record — 2026-07-20 (Tasks 1–4)

Verified on the live server and marked complete. Two constraints in this document turned out to be wrong; corrections below.

### Correction 1: `ssh ovh` does not apply

The plan assumes every command runs as `ssh ovh '<cmd>'`. In practice the Claude Code session for this project runs **on `51.75.66.60` itself** (hostname `vps-caef402f`), and there is no `ovh` alias in `/etc/hosts` or `~/.ssh/config` there — `ssh ovh` fails with "Could not resolve hostname ovh". Run the commands directly, without the wrapper. Check `hostname -I` before assuming otherwise.

### Correction 2: the sudoers grant is `/etc/sudoers.d/amar`, and it is not temporary

Earlier drafts named a temporary grant at `/etc/sudoers.d/99-amar-temp` that Task 12 removes. **That file never existed.** What exists is `/etc/sudoers.d/amar` (`amar ALL=(ALL:ALL) NOPASSWD:ALL`, `0440`, created 2026-07-20 07:35).

**Resolved 2026-07-21:** passwordless sudo for `amar` is **kept as permanent, intended** configuration — `amar` has a usable password (so this is a convenience choice, not a lockout risk), and this server's Claude Code sessions run sudo non-interactively, where a password prompt would hang. The Global Constraints note and Task 12 Step 1 were rewritten to match; the file stays in place.

### Correction 3: leftover cloud-init `debian` account

`/etc/sudoers.d/90-cloud-init-users` grants `debian ALL=(ALL) NOPASSWD:ALL`. The `debian` account (uid 1000) has a **set, usable password** (`passwd -S` → `P`) and an **empty** `~/.ssh/authorized_keys`. It was genuinely used: `Accepted password for debian` from `46.34.141.146` and `178.250.191.152` on 2026-07-17, i.e. it was the initial provisioning login before user `amar` existed.

Not currently exploitable remotely — SSH password auth is now off (see hardening item 2 below) and there are no keys on the account — so this was a cleanup item, not an active hole.

**Resolved 2026-07-21:** `sudo passwd -l debian` — the account's password is now locked (`passwd -S` → `L`), removing the last login path (no password, no keys, SSH password auth off). Confirmed first that nothing depends on it: no running processes, no crontab, no systemd unit runs as `debian`, and it owns no files outside `/home/debian`. The `NOPASSWD:ALL` line in `/etc/sudoers.d/90-cloud-init-users` was left in place — harmless with no way to log in as `debian` — and the account itself was **not** deleted (only locking was requested; deleting the cloud-init account is a separate decision).

### Verified state

| Task | Status | Evidence |
|---|---|---|
| 1 — base packages | done | `git 1:2.47.3`, `build-essential 12.12`, `curl 8.14.1`, `ca-certificates 20250419`, `unzip 6.0-29` all `ii` |
| 2 — ufw allowlist | done | active, INPUT policy DROP on v4 **and** v6; 80/tcp all; 22+443/tcp from the 5 listed IPs; `ENABLED=yes` in `/etc/ufw/ufw.conf` |
| 3 — fail2ban | done | `sshd` jail up on `backend=systemd`, journal matches working, 2 bans to date |
| 4 — unattended-upgrades | done | package 2.12, `enabled` + `active` |

Note: `systemctl is-active ufw` reports `inactive (dead)` — this is normal for Debian's oneshot `ufw.service` and is **not** a fault. The real checks are `ufw status` (active), `iptables -S INPUT` (policy DROP), and `ENABLED=yes` in `/etc/ufw/ufw.conf`.

### Additional hardening applied beyond the plan

1. **Jitsi `web` container port 8443 was publicly exposed.** Docker writes DNAT rules into `nat/PREROUTING`, which bypasses the INPUT chain where ufw filters — so `0.0.0.0:8443->443/tcp` answered HTTP 200 from the internet while `ufw status` listed no such rule. Fixed by setting `HTTPS_PORT=127.0.0.1:8443` in `/opt/jitsi-meet/.env` (backup: `.env.bak-2026-07-20`) and recreating the container. External `:8443` now refuses; `https://meet.marko-lab.com:50443` still returns 200.

   **Audit rule going forward:** after adding any container with published ports, check `sudo iptables -t nat -S DOCKER | grep DNAT` — *not* `ufw status` — to see what is actually reachable. Only `10000/udp` (JVB) is intentionally exposed this way.

2. **SSH is now key-only.** `/etc/ssh/sshd_config.d/50-cloud-init.conf` ships `PasswordAuthentication yes` and was overriding the `no` in the main config. Added `/etc/ssh/sshd_config.d/10-hardening.conf` (`PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`) — the `10-` prefix wins because sshd keeps the first value it sees. Confirmed via `sshd -T`; recent `Accepted publickey` entries exist for all 5 allowlisted IPs, so no lockout risk.

3. **fail2ban tuned and a Caddy jail added** — `/etc/fail2ban/jail.d/hardening.local`: bantime 1h with `bantime.increment` (factor 2, cap 1w), the 5 trusted IPs in `ignoreip`, `sshd` maxretry 4. New jail `caddy-status` uses `/etc/fail2ban/filter.d/caddy-status.conf` to ban 15× 401/403/404 within 10m. Ban enforcement verified end-to-end (rule appears in `nft list table inet f2b-table`).

4. **Caddy access logging enabled** (prerequisite for the jail above — there was none). Both sites now `import accesslog` writing JSON to `/var/log/caddy/access.log`, rolling at 10 MiB × 5. Backup: `/etc/caddy/Caddyfile.bak-2026-07-20`.

   Two traps worth recording: the log file must be owned by `caddy:caddy` or `systemctl reload caddy` fails with HTTP 400 "permission denied"; and fail2ban strips the timestamp it matched via `datepattern` before applying `failregex`, so `^`-anchored patterns against JSON log lines never match.

### Drift from the plan's assumptions

The Caddyfile no longer serves `hermes.marko-lab.com` (the plan's Task 10 target). Current sites are `meet.marko-lab.com:50443` (Jitsi) and `claude.marko-lab.com` (proxying `127.0.0.1:3001`). Tasks 5–12 should be re-read against the live server before execution rather than assumed still accurate.
