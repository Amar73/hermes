# MARKO Lab Paperclip Company — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the auto-onboarded, currently-broken Paperclip company into a working two-agent org ("MARKO Lab": chief-of-staff + Engineer), both running on OpenRouter/`claude-sonnet-5` via the already-configured Hermes Agent, with a real Telegram-to-commit path into `Amar73/hermes-scripts`.

**Architecture:** Paperclip's `hermes_local` adapter shells out to the local `hermes` CLI for both agents, reusing the exact OpenRouter/`claude-sonnet-5` config already verified working (plan doc steps 5+8). A new Hermes skill (`paperclip-task-bridge`, shipped with Paperclip but not yet installed into Hermes) lets the already-running Telegram-facing Hermes gateway create Paperclip issues, which is how a Telegram message becomes a task chief-of-staff can delegate to Engineer.

**Tech Stack:** Paperclip REST API (curl), Hermes Agent CLI/config (`~/.hermes`), `gh` CLI, systemd (`hermes-gateway.service`).

## Global Constraints

- No Claude Code / Gemini CLI / Codex OAuth logins on the server — user vetoed this 2026-07-09 (account-ban risk). Every agent must use `hermes_local` (OpenRouter), never `claude_local`/`codex_local`/`gemini_local`.
- Model for all agents in this plan: `anthropic/claude-sonnet-5` via OpenRouter — same as Hermes's own default (plan doc step 5), no new provider setup.
- Paperclip dashboard/API auth: `https://hermes.amar-home.ru` gated by Caddy basicauth (`paperclip` / see `/root/.paperclip-credentials` on the server). The Paperclip app itself runs in `deploymentMode: local_trusted`, which accepted plain basicauth-tunneled requests with no additional bearer token in earlier read-only testing — if any write call in this plan returns 401/403, create a Board API key first (`POST /api/board-api-keys`, body `{"name": "plan-setup"}`) and add `-H "Authorization: Bearer <token>"` to the failing call.
- Known IDs (verified live via `/api/openapi.json` and `GET` calls on 2026-07-09, re-verify with the `GET` shown in each task if time has passed):
  - `companyId` = `72c6d782-a89c-404a-a386-cc299a77cff4`
  - chief-of-staff `agentId` = `67e44763-4e48-49ed-8cd8-1c5131b660d2`
  - "Local" environment `id` = `96dfd510-da1f-41a0-a831-5a398bc4bbed`
- Never print secrets (API keys, bridge tokens) into git history, chat, or logs — write them straight into `.env`/`config.json` on the server and reference them by variable name afterward.
- All server-side commands go through `ssh amar@hermes`; Hermes Agent files/commands need `sudo -u paperclip`.
- **`hermes_local` `adapterConfig` must always set `"provider": "openrouter"` explicitly whenever `model` is an `anthropic/...`-style string.** Discovered 2026-07-09 during Task 7: leaving `provider` unset ("auto") makes Hermes mis-detect the model's `anthropic/` prefix as a request for the *native* Anthropic provider (requires `claude`/OAuth credentials we don't have) instead of routing it through OpenRouter, even though `anthropic/claude-sonnet-5` is OpenRouter's own model-id convention. Symptom: every run fails instantly with `No Anthropic credentials found...`. Tasks 1 and 3 in this plan were retroactively patched with this field after the bug surfaced — any future agent created with this same model string needs it from the start.

---

### Task 1: Fix chief-of-staff — switch its adapter off `claude_local`

**Problem:** `GET /api/companies/72c6d782-a89c-404a-a386-cc299a77cff4/agents` currently shows chief-of-staff `"status": "error"`, `"adapterType": "claude_local"`, `"errorReason": "Claude run failed: subtype=success: Not logged in · Please run /login"`. It has been broken since onboarding because it needs a Claude Code OAuth login that we are not doing (see Global Constraints). This task switches it to `hermes_local`, reusing Hermes's already-working OpenRouter setup.

**Files:** None (remote API + config only, no local repo files).

- [ ] **Step 1: Confirm current broken state**

```bash
curl -s -u paperclip:$(ssh amar@hermes "sudo grep -oP '(?<=Password: ).*' /root/.paperclip-credentials") \
  "https://hermes.amar-home.ru/api/companies/72c6d782-a89c-404a-a386-cc299a77cff4/agents" | python3 -m json.tool
```

Expected: one agent, `"adapterType": "claude_local"`, `"status": "error"`.

- [ ] **Step 2: Patch the adapter to `hermes_local`**

```bash
curl -s -u paperclip:<PASSWORD> -X PATCH \
  "https://hermes.amar-home.ru/api/agents/67e44763-4e48-49ed-8cd8-1c5131b660d2" \
  -H "Content-Type: application/json" \
  -d '{
    "adapterType": "hermes_local",
    "adapterConfig": {
      "model": "anthropic/claude-sonnet-5",
      "provider": "openrouter",
      "toolsets": "terminal,file,web",
      "persistSession": true,
      "timeoutSec": 300
    }
  }' | python3 -m json.tool
```

Expected: HTTP 200, response body shows `"adapterType": "hermes_local"`, `"adapterConfig.model": "anthropic/claude-sonnet-5"`.

- [ ] **Step 3: Verify the error clears**

```bash
curl -s -u paperclip:<PASSWORD> "https://hermes.amar-home.ru/api/agents/67e44763-4e48-49ed-8cd8-1c5131b660d2" | python3 -m json.tool
```

Expected: `"status"` is no longer `"error"` (likely `"idle"` or similar — it clears once the next run attempt succeeds; if still `"error"` with the old message, call `POST /api/agents/67e44763-4e48-49ed-8cd8-1c5131b660d2/wakeup` with an empty body to force a fresh run and re-check).

- [ ] **Step 4: Commit the plan-doc/checklist update**

No code to commit for this task (pure remote config). Proceed to Task 2 before committing anything — Task 6 covers updating the deployment-plan doc's checklist for all of this work at once.

---

### Task 2: Rename the company to "MARKO Lab"

**Files:** None (remote API only).

- [ ] **Step 1: Rename via PATCH**

```bash
curl -s -u paperclip:<PASSWORD> -X PATCH \
  "https://hermes.amar-home.ru/api/companies/72c6d782-a89c-404a-a386-cc299a77cff4" \
  -H "Content-Type: application/json" \
  -d '{"name": "MARKO Lab"}' | python3 -m json.tool
```

Expected: HTTP 200, response body `"name": "MARKO Lab"`.

- [ ] **Step 2: Verify**

```bash
curl -s -u paperclip:<PASSWORD> "https://hermes.amar-home.ru/api/companies/72c6d782-a89c-404a-a386-cc299a77cff4" | python3 -c "import json,sys; print(json.load(sys.stdin)['name'])"
```

Expected output: `MARKO Lab`

---

### Task 3: Create the Engineer agent

**Files:** None (remote API only).

**Interfaces:**
- Consumes: `companyId` and chief-of-staff `agentId` from Global Constraints; "Local" environment `id` from Global Constraints.
- Produces: a new `agentId` (capture it from the response — needed by Task 5's end-to-end test, not by any other task in this plan).

- [ ] **Step 1: Create the agent**

```bash
curl -s -u paperclip:<PASSWORD> -X POST \
  "https://hermes.amar-home.ru/api/companies/72c6d782-a89c-404a-a386-cc299a77cff4/agents" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Engineer",
    "role": "engineer",
    "reportsTo": "67e44763-4e48-49ed-8cd8-1c5131b660d2",
    "defaultEnvironmentId": "96dfd510-da1f-41a0-a831-5a398bc4bbed",
    "adapterType": "hermes_local",
    "adapterConfig": {
      "model": "anthropic/claude-sonnet-5",
      "provider": "openrouter",
      "toolsets": "terminal,file,web",
      "persistSession": true,
      "timeoutSec": 300
    }
  }' | python3 -m json.tool
```

Expected: HTTP 201, response body has `"role": "engineer"`, `"adapterType": "hermes_local"`, `"reportsTo": "67e44763-4e48-49ed-8cd8-1c5131b660d2"`. **Copy the returned `"id"` — write it down, it's referenced as `<ENGINEER_AGENT_ID>` in Task 5.**

- [ ] **Step 2: Verify org chain**

```bash
curl -s -u paperclip:<PASSWORD> "https://hermes.amar-home.ru/api/companies/72c6d782-a89c-404a-a386-cc299a77cff4/org" | python3 -m json.tool
```

Expected: tree showing chief-of-staff as root with Engineer as a direct report.

---

### Task 4: Wire Telegram → Paperclip via the `paperclip-task-bridge` skill

**Problem:** Hermes's Telegram gateway (already running, `hermes-gateway.service`) only handles Hermes's own personal-assistant conversation — it has no path into Paperclip's issue system yet. Paperclip ships a bundled Hermes skill (`paperclip-task-bridge`, at `/opt/paperclip/packages/adapters/hermes/skills/paperclip-task-bridge/`) that gives Hermes a `create-task`/`comment`/`update-status`/`list-assigned` CLI it can call — that's the missing link. This task creates a scoped API key, installs the skill, and configures Hermes's `.env`.

**Files:**
- Create (on server): `/home/paperclip/.hermes/skills/paperclip-task-bridge/` (copy of two files)
- Modify (on server): `/home/paperclip/.hermes/hermes-agent/.env` (append 3 vars)

**Interfaces:**
- Consumes: chief-of-staff `agentId` (`67e44763-4e48-49ed-8cd8-1c5131b660d2`), `companyId` (`72c6d782-a89c-404a-a386-cc299a77cff4`).
- Produces: a parent issue id (`<PARENT_ISSUE_ID>`) and a bridge API key value — both referenced later in this task only.

- [ ] **Step 1: Create a parent issue to scope the bridge key**

All Telegram-originated tasks will be created as children of this one issue — this is the security boundary for the bridge key (it can only create issues under this parent, not read/list anything else company-wide).

```bash
curl -s -u paperclip:<PASSWORD> -X POST \
  "https://hermes.amar-home.ru/api/companies/72c6d782-a89c-404a-a386-cc299a77cff4/issues" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Входящие задачи из Telegram (MARKO Lab)",
    "assigneeAgentId": "67e44763-4e48-49ed-8cd8-1c5131b660d2"
  }' | python3 -m json.tool
```

Expected: HTTP 201, response has `"title": "Входящие задачи из Telegram (MARKO Lab)"`. **Copy the returned `"id"` as `<PARENT_ISSUE_ID>`.**

- [ ] **Step 2: Create the task-bridge-scoped API key**

```bash
curl -s -u paperclip:<PASSWORD> -X POST \
  "https://hermes.amar-home.ru/api/agents/67e44763-4e48-49ed-8cd8-1c5131b660d2/keys" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Hermes Telegram task bridge",
    "scope": {
      "kind": "task_bridge",
      "parentIssueId": "<PARENT_ISSUE_ID>"
    }
  }' | python3 -m json.tool
```

Expected: HTTP 201. The response schema isn't documented (`additionalProperties` in the OpenAPI spec), so **read the actual JSON returned** — look for a field holding the raw token (likely `token`, `key`, or `apiKey`). **Copy that value once — it will not be shown again** (matches the SKILL.md's own warning: "store the returned token once").

- [ ] **Step 3: Copy the skill into Hermes's skills directory**

```bash
ssh amar@hermes "sudo -u paperclip mkdir -p /home/paperclip/.hermes/skills && sudo cp -r /opt/paperclip/packages/adapters/hermes/skills/paperclip-task-bridge /home/paperclip/.hermes/skills/paperclip-task-bridge && sudo chown -R paperclip:paperclip /home/paperclip/.hermes/skills/paperclip-task-bridge"
```

Expected: no error. Verify with:

```bash
ssh amar@hermes "sudo -u paperclip ls /home/paperclip/.hermes/skills/paperclip-task-bridge"
```

Expected output: `SKILL.md` and `paperclip-task.mjs`.

- [ ] **Step 4: Add bridge credentials to Hermes's `.env`**

Use the loopback URL (not the public HTTPS/basicauth one) since Hermes runs on the same host as Paperclip:

```bash
ssh amar@hermes 'sudo -u paperclip tee -a /home/paperclip/.hermes/hermes-agent/.env >/dev/null <<EOF
PAPERCLIP_API_URL=http://127.0.0.1:3100
PAPERCLIP_BRIDGE_API_KEY=<TOKEN_FROM_STEP_2>
PAPERCLIP_COMPANY_ID=72c6d782-a89c-404a-a386-cc299a77cff4
EOF'
```

- [ ] **Step 5: Restart the gateway and verify the skill loaded**

```bash
ssh amar@hermes "sudo systemctl restart hermes-gateway && sleep 3 && systemctl is-active hermes-gateway"
ssh amar@hermes "sudo -u paperclip bash -lc 'cd /home/paperclip/.hermes/hermes-agent && uv run hermes skills list 2>&1 | grep -i paperclip'"
```

Expected: `active`, and the skills list includes `paperclip-task-bridge`.

- [ ] **Step 6: Smoke-test the bridge helper directly (bypassing chat) before trusting the LLM to call it**

```bash
ssh amar@hermes "sudo -u paperclip bash -lc 'cd /home/paperclip/.hermes/skills/paperclip-task-bridge && PAPERCLIP_API_URL=http://127.0.0.1:3100 PAPERCLIP_BRIDGE_API_KEY=<TOKEN> node paperclip-task.mjs list-assigned'"
```

Expected: JSON output (empty list or the parent issue, no auth error). If this fails with an auth error, the token or `parentIssueId` scope from Steps 1–2 is wrong — fix before moving to Task 5.

---

### Task 5: Create `Amar73/hermes-scripts` and grant `paperclip` push access

**Files:** None (GitHub + server SSH config only).

- [ ] **Step 1: Create the repo from the local machine (`gh` already authorized for `Amar73`)**

```bash
gh repo create Amar73/hermes-scripts --private --description "Python Telegram bots written by the MARKO Lab Paperclip pipeline" --confirm
```

Expected: repo created, URL printed.

- [ ] **Step 2: Give `paperclip` its own GitHub identity for pushing**

Reuse the pattern from the `amar` GitHub setup (see deployment-plan doc, "Стартовое состояние"): install `gh` for the `paperclip` user context and authenticate.

```bash
ssh amar@hermes "sudo -u paperclip gh auth login"
```

This is an interactive device-flow login (like the git/gh setup already done for `amar`) — **not** the same account-ban risk as Claude/Gemini/Codex (GitHub has no such restriction on this pattern; the user already did this exact flow for `amar`). Follow the printed URL + code prompts.

- [ ] **Step 3: Verify push access**

```bash
ssh amar@hermes "sudo -u paperclip bash -c 'cd /tmp && rm -rf hermes-scripts-test && git clone https://github.com/Amar73/hermes-scripts.git hermes-scripts-test && cd hermes-scripts-test && git commit --allow-empty -m \"paperclip push test\" && git push && cd .. && rm -rf hermes-scripts-test'"
```

Expected: push succeeds with no auth error. Check on GitHub (or `gh repo view Amar73/hermes-scripts --json pushedAt`) that the empty commit landed, then this test commit can stay (harmless) or be reverted later — not worth the extra complexity of un-pushing it.

---

### Task 6: Update deployment-plan doc and memory

**Files:**
- Modify: `docs/superpowers/specs/2026-06-30-hermes-server-deployment-plan.md` (checklist section, step 10.3/10.4 notes)

- [ ] **Step 1: Update the checklist**

Mark these as done with today's date and a one-line note pointing at this plan doc:
- "Org chart в Paperclip настроен" → done, MARKO Lab company with chief-of-staff (CEO) + Engineer, both on `hermes_local`/OpenRouter/`claude-sonnet-5`.
- "Репозиторий `Amar73/hermes-scripts` создан, доступ для пуша с сервера настроен" → done.

- [ ] **Step 2: Commit**

```bash
git add docs/superpowers/specs/2026-06-30-hermes-server-deployment-plan.md
git commit -m "Mark org chart + hermes-scripts repo complete (MARKO Lab)"
```

---

### Task 7: End-to-end test

**Files:** None — this is a live behavioral test, not a code change.

- [ ] **Step 1: Send a real task via Telegram**

Message the bot (`@AmarAiAgent_bot`): *"Создай в Paperclip задачу для chief-of-staff: напиши простого Telegram-бота на Python, который отвечает на /start и /help, и закоммить его в hermes-scripts."*

- [ ] **Step 2: Watch it land as an issue**

```bash
curl -s -u paperclip:<PASSWORD> "https://hermes.amar-home.ru/api/companies/72c6d782-a89c-404a-a386-cc299a77cff4/issues?parentId=<PARENT_ISSUE_ID>" | python3 -m json.tool
```

Expected: a new child issue under the Task 4 parent issue, with a title resembling the request.

- [ ] **Step 3: Watch chief-of-staff delegate and Engineer run**

```bash
ssh amar@hermes "sudo journalctl -u paperclip --no-pager -n 100 | grep -iE 'engineer|chief-of-staff|heartbeat'"
```

Expected: run activity for both agents (chief-of-staff picking up the issue, then Engineer being invoked).

- [ ] **Step 4: Confirm the commit landed**

```bash
gh api repos/Amar73/hermes-scripts/commits --jq '.[0].commit.message, .[0].html_url'
```

Expected: a recent commit with a message related to the Telegram bot, matching what Engineer was asked to build.

- [ ] **Step 5: Confirm the Telegram report**

Check the Telegram chat for a final message from the bot summarizing what was built and linking the commit — this is the acceptance criterion for the whole plan (matches the spec's "Поток задачи" section).

If any step in this task fails, debug with `sudo journalctl -u paperclip -f` and `sudo journalctl -u hermes-gateway -f` live while resending the Telegram message, rather than guessing — this is new, unverified wiring (Task 4 especially) and the first real run is the actual test of it.

**(Executed 2026-07-09) Two real bugs found and fixed during this run:**

1. **Provider auto-detection bug** (root-caused via systematic-debugging): every chief-of-staff run failed instantly with "No Anthropic credentials found..." because `hermes_local` mis-detected `provider=anthropic` from the `anthropic/claude-sonnet-5` model string instead of `openrouter`. Fixed by adding `"provider": "openrouter"` explicitly to both agents' `adapterConfig` (now folded back into Tasks 1 and 3 above and the Global Constraints). After the fix, chief-of-staff picked up the real Telegram-originated issue (MAR-3) and completed it for real: wrote a working `python-telegram-bot` starter (`/start`, `/help`, README, requirements.txt), committed it to `Amar73/hermes-scripts` (commit `745c2db`), and marked the issue done — confirmed via the Paperclip issues API and `gh api .../commits`.
2. **Stale company name in chief-of-staff's instructions**: `AGENTS.md` still said "AmarIko Lab" (a leftover from onboarding) even after Task 2's rename — renaming the company via `PATCH /api/companies/{id}` does **not** touch the baked-in instructions bundle text. Fixed via `PUT /api/agents/{id}/instructions-bundle/file?path=AGENTS.md` with the string replaced. Engineer's instructions never mentioned the company name (generic template), so no fix needed there.

**Known gap found and fixed (Task 8, added after Task 7 revealed it):** the design spec promised an autonomous final report delivered to Telegram, but nothing in the Task 4 wiring pushes a notification when a Paperclip issue completes — `paperclip-task-bridge` is pull-only (Hermes must be prompted to check). Chief-of-staff's actual completion report (with the commit link) was sitting in the issue's Paperclip comments the whole time, just never forwarded. See Task 8 below.

---

### Task 8: Push completion reports to Telegram (closes the Task 7 gap)

**Problem:** confirmed via a live test — after MAR-3 completed, the user only ever received Hermes's initial "task created" acknowledgment in Telegram, never a completion report, even though chief-of-staff had written one into the issue's Paperclip comments. Closing this needs something to proactively check for newly-completed issues and push the existing report to Telegram — `hermes cron` (already available, unused until now) is the natural mechanism.

**Files:**
- Create (on server): `/home/paperclip/.hermes/scripts/notify_paperclip_done.py`

**Interfaces:**
- Consumes: `PARENT_ISSUE_ID` = `e78be65c-87a2-400c-aba0-81161c78af41` (Task 4's parent issue), `COMPANY_ID` = `72c6d782-a89c-404a-a386-cc299a77cff4`. Reads Paperclip's REST API over loopback (`http://127.0.0.1:3100`) with **no auth header** — confirmed the `local_trusted` deployment mode accepts unauthenticated reads from the loopback interface (Caddy's basicauth only gates the public hostname, not `127.0.0.1:3100` directly).
- Produces: state file `~/.hermes/state/paperclip-notified.json` (list of already-notified issue ids) — local to this script, nothing else reads it.

- [x] **Step 1: Write the script**

```python
#!/usr/bin/env python3
"""Poll Paperclip for newly-completed MARKO Lab issues and print a Telegram-ready
summary for each one (using the agent's own last comment as the report body).
Intended to run via `hermes cron ... --no-agent` so stdout is delivered verbatim.
Silent (no output) when nothing new has completed.
"""
import json
import os
import urllib.request
from pathlib import Path

BASE_URL = "http://127.0.0.1:3100"
COMPANY_ID = "72c6d782-a89c-404a-a386-cc299a77cff4"
PARENT_ISSUE_ID = "e78be65c-87a2-400c-aba0-81161c78af41"  # "Входящие задачи из Telegram (MARKO Lab)"
STATE_PATH = Path.home() / ".hermes" / "state" / "paperclip-notified.json"
TERMINAL_STATUSES = {"done", "cancelled"}


def get_json(url):
    with urllib.request.urlopen(url, timeout=15) as resp:
        return json.load(resp)


def load_state():
    if STATE_PATH.exists():
        try:
            return set(json.loads(STATE_PATH.read_text()))
        except (json.JSONDecodeError, OSError):
            return set()
    return set()


def save_state(notified_ids):
    STATE_PATH.parent.mkdir(parents=True, exist_ok=True)
    STATE_PATH.write_text(json.dumps(sorted(notified_ids)))


def latest_agent_comment(issue_id):
    comments = get_json(f"{BASE_URL}/api/issues/{issue_id}/comments")
    agent_comments = [c for c in comments if c.get("authorType") == "agent"]
    if not agent_comments:
        return None
    agent_comments.sort(key=lambda c: c.get("createdAt", ""), reverse=True)
    return agent_comments[0].get("body", "").strip()


def main():
    notified = load_state()
    issues = get_json(
        f"{BASE_URL}/api/companies/{COMPANY_ID}/issues?parentId={PARENT_ISSUE_ID}"
    )

    messages = []
    newly_notified = set()
    for issue in issues:
        if issue["status"] not in TERMINAL_STATUSES:
            continue
        if issue["id"] in notified:
            continue

        report = latest_agent_comment(issue["id"]) or "(агент не оставил текстовый отчёт)"
        messages.append(
            f"✅ {issue['identifier']}: {issue['title']}\n"
            f"Статус: {issue['status']}\n\n"
            f"{report}"
        )
        newly_notified.add(issue["id"])

    if messages:
        save_state(notified | newly_notified)
        print("\n\n---\n\n".join(messages))


if __name__ == "__main__":
    main()
```

Deploy: `scp` to the server, place at `/home/paperclip/.hermes/scripts/notify_paperclip_done.py`, `chown paperclip:paperclip`, `chmod +x`.

- [x] **Step 2: Verify manually before scheduling**

```bash
ssh amar@hermes "sudo -u paperclip python3 /home/paperclip/.hermes/scripts/notify_paperclip_done.py"
```

Expected (first run, with MAR-3 already done): prints the full report for MAR-3 including the commit link. Run it a second time immediately after — expected: no output (idempotent, already recorded in the state file).

- [x] **Step 3: Register the cron job**

```bash
ssh amar@hermes "sudo -u paperclip bash -lc 'cd /home/paperclip/.hermes/hermes-agent && uv run hermes cron create \"every 3m\" --name paperclip-completions --script notify_paperclip_done.py --no-agent --deliver telegram:384955770'"
```

Note: `--script` must be a bare filename (just `notify_paperclip_done.py`), not an absolute path — `hermes cron create` rejects absolute/home-relative paths and resolves everything against `~/.hermes/scripts/` itself.

Verify: `hermes cron status` shows the gateway running with the job counted as active; `hermes cron list` shows it with `Deliver: telegram:384955770`.

- [x] **Step 4: Force a real delivery test**

```bash
ssh amar@hermes "sudo -u paperclip rm -f /home/paperclip/.hermes/state/paperclip-notified.json"
ssh amar@hermes "sudo -u paperclip bash -lc 'cd /home/paperclip/.hermes/hermes-agent && uv run hermes cron run <job_id>'"
```

Expected: ask the human to confirm the Telegram message actually arrived (this is the only way to check — Hermes delivers it directly, there's no server-side log of successful Telegram delivery to inspect). **Confirmed 2026-07-09**: user received the full MAR-3 report (title, status, complete agent write-up, commit link) as a Telegram message from the bot, formatted as a "Cronjob Response: paperclip-completions" message.
