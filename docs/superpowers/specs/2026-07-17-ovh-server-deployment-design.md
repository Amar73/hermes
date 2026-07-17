# Дизайн: развёртывание Claude Code / Antigravity CLI / Hermes Agent на `ovh` (51.75.66.60)

Дата: 2026-07-17
Сервер: `51.75.66.60` (alias `ovh` в `~/.ssh/config` через `/etc/hosts`, `ssh ovh` → пользователь `amar`)
Домен: `hermes.marko-lab.com` → `51.75.66.60` (A-запись уже стоит, проверено `dig`)
Заменяет: сервер `hermes` (`193.228.139.46`, домен `hermes.amar-home.ru`) — будет снесён отдельно, вне рамок этого документа

## Отличие от предыдущего развёртывания (`2026-06-30-hermes-server-deployment-plan.md`)

Прошлый план разворачивал комбо-стек Paperclip (мультиагентный оркестратор с дашбордом, отдельным системным пользователем `paperclip`, встроенным Postgres) поверх которого крутился Hermes Agent. В этот раз пользователь явно исключил Paperclip — нужны только сами инструменты, напрямую под пользователем `amar`, без оркестратора и без БД. Также пересмотрено решение по OAuth: в прошлый раз Pro-логины (Claude Code/Gemini CLI/Codex) были отложены из-за риска бана аккаунта за автоматизацию на headless-сервере; в этот раз пользователь осознанно решил попробовать OAuth-логин для Claude Code и Antigravity CLI (Codex по-прежнему вне скоупа).

Gemini CLI как отдельный продукт заменён на **Antigravity CLI** — Google переводит hosted-доступ Gemini CLI на Antigravity CLI (retirement объявлен на июнь 2026 для free/Pro/Ultra аккаунтов), поэтому ставим именно его.

## Стартовое состояние (проверено 2026-07-17 по SSH)

- Debian 13 (trixie), hostname `vps-caef402f`, 8 vCPU, 22 GiB RAM, 197 GB диска (использовано 1.3 GB).
- Пользователь `amar` (uid 1001), passwordless SSH по ключу уже работает.
- `PermitRootLogin no` и `PasswordAuthentication no` в `/etc/ssh/sshd_config` уже стоят — часть хардening готова из коробки.
- `sudo` требует пароль (не passwordless) — для шагов установки временно выдан `NOPASSWD` через `/etc/sudoers.d/99-amar-temp`, снять после завершения работ.
- Ничего из целевого стека не установлено: нет `node`, `npm`, `git`, `python3` есть (системный), `claude`/`gemini`/`agy`/`hermes`/`uv`/`rg`/`ffmpeg`/`fail2ban`/`ufw` — не найдены.
- `ufw` не настроен, файрвол не активен.
- Локальный репозиторий `~/Amar73/hermes` (эта документация) на машине пользователя синхронизирован с `origin` (`git@github.com:Amar73/hermes.git`), ветка `master`, чисто, без незакоммиченных изменений.

## Компоненты

### 1. Базовый hardening

- `apt update && apt upgrade -y`, базовые пакеты: `git build-essential curl ca-certificates unzip`.
- `ufw`: default deny incoming / allow outgoing.
  - `80/tcp` — открыт всем (только ACME-проверка Let's Encrypt + редирект на HTTPS, доступа к данным нет).
  - `22/tcp` и `443/tcp` — только с 5 IP: `46.34.141.146` (дом), `144.206.228.59` (работа), `144.206.226.221`, `178.250.191.152`, `144.31.81.96`.
- `fail2ban`: jail `sshd` сразу с `backend = systemd` в `/etc/fail2ban/jail.d/sshd-systemd-backend.local` — на прошлом сервере (тоже минимальный Debian-образ без rsyslog) jail падал с дефолтным backend из-за отсутствия `/var/log/auth.log`; в этот раз ставим правильный backend сразу, не наступая на те же грабли.
- `unattended-upgrades` — автоматические security-обновления.
- **Риск** (как и в прошлый раз): если домашний/рабочий IP динамический и поменяется — доступ по SSH/HTTPS пропадёт, пока правила ufw не обновят из другого разрешённого места или через веб-консоль хостинг-провайдера OVH. Уточнить заранее, есть ли у OVH запасная консоль (KVM/VNC) на этот VPS.
- После завершения всех шагов — снять временный `/etc/sudoers.d/99-amar-temp` (`sudo rm /etc/sudoers.d/99-amar-temp`), проверить `sudo -n true` возвращает «нужен пароль».

### 2. Claude Code

- Нативный установщик: `curl -fsSL https://claude.ai/install.sh | bash` (кладёт бинарник в `~/.local/bin`, автообновляется в фоне, root не нужен).
- Проверка: `claude --version`.
- Авторизация — `claude auth login` (не `claude login`, этой подкоманды нет). Требует интерактивный терминал; на headless-сервере — через `tmux new-session -d -s claude-login 'claude auth login'` + `tmux capture-pane -p` для чтения URL/промптов, `tmux send-keys` для ввода кода. Пользователь входит по ссылке в браузере на своей машине под своим Pro-аккаунтом, код возвращает в терминал сервера. Делаем интерактивно вместе, не скриптуем вслепую.

### 3. Antigravity CLI

- Установка: `curl -fsSL https://antigravity.google/cli/install.sh | bash` — единый бинарник, без зависимостей Node/Python.
- Проверка: `agy --version` (уточнить точное имя команды после установки — подтверждено в доке как `agy`, но перепроверить `--help`).
- Авторизация: CLI в headless/SSH-сессии сама детектирует отсутствие браузера и печатает URL + одноразовый код; тот же паттерн, что и для Claude Code — открыть ссылку на своей машине, ввести код на сервере.

### 4. Hermes Agent

- Установка: `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash` — подтягивает `uv`, Python 3.11, Node.js, `ripgrep`, `ffmpeg`.
- `source ~/.bashrc` после установки, затем `hermes setup --portal`.
- Секреты в `~/.hermes/.env`: `OPENROUTER_API_KEY`, `PERPLEXITY_API_KEY`, `TELEGRAM_BOT_TOKEN` (токен бота у пользователя уже есть, как и в прошлый раз).
- Модель по умолчанию — через `hermes config set model ...` (конкретную модель уточнить с пользователем на шаге внедрения; в прошлый раз использовали `anthropic/claude-sonnet-5` через OpenRouter).
- **Web-search через Perplexity**: кастомный OpenAI-совместимый backend в `~/.hermes/config.yaml`:
  ```yaml
  custom_providers:
    - name: perplexity
      base_url: https://api.perplexity.ai
      key_env: PERPLEXITY_API_KEY
  ```
  Точный ключ, включающий этот backend именно для web-search toolset (а не как основной chat-провайдер), в документации описан нечётко (аналог `web.backend: xai` для xAI, но явного примера `web.backend: custom:perplexity` в доке не нашлось). **Не изобретать синтаксис заранее** — на шаге внедрения проверить через `hermes config --help` / `hermes status` на реально установленной версии, по аналогии с тем, как в прошлый раз выяснилось реальное поведение интеграции Perplexity (документация разошлась с фактическим кодом).
- Telegram-gateway — отдельный долгоживущий процесс, не часть основной установки:
  ```bash
  cd ~/.hermes/hermes-agent && uv run hermes --accept-hooks gateway install --system --run-as-user amar --start-now
  ```
  Создаёт `hermes-gateway.service` (systemd, переживает перезагрузку). Первое сообщение боту не отвечает, а выдаёт pairing-код — подтвердить через `hermes pairing approve telegram <КОД>`.
- Дашборд слушает `127.0.0.1:9119`, gateway OpenAI-совместимый API — `127.0.0.1:8642`. Оба только на loopback, наружу не торчат (наружу — только через Caddy, см. ниже).

### 5. Домен и HTTPS (Caddy)

```
hermes.marko-lab.com {
    reverse_proxy 127.0.0.1:9119
}
```
`sudo systemctl reload caddy`, проверка `curl -vI https://hermes.marko-lab.com` — валидный Let's Encrypt сертификат, редирект с HTTP.

### 6. Синхронизация репозитория `hermes` через GitHub

Цель: сервер `ovh` должен иметь тот же репозиторий `~/Amar73/hermes` (эта документация — спеки, планы, CLAUDE.md), что и локальная машина, и оставаться синхронизированным через GitHub в обе стороны — так же, как было настроено для старого сервера `hermes` (раздел «GitHub-доступ» в прошлом плане).

- Установить `gh` CLI (официальный apt-репозиторий `cli.github.com`).
- `gh auth login` (device-flow) под пользователем `amar` на сервере, credential helper (`gh auth git-credential`) для `github.com`.
- Git identity — та же, что на локальной машине (`Andrey Maryanenko <a.maryanenko@gmail.com>`, свериться с `git config --global user.email` локально).
- `git clone git@github.com:Amar73/hermes.git ~/Amar73/hermes` на сервере (или через `gh repo clone Amar73/hermes ~/Amar73/hermes`).
- По ходу и по завершении развёртывания — коммитить в этот репозиторий (локально или прямо с сервера) записи о том, что реально сделано/отличается от плана (как в прошлом плане — раздел «Сверка с реальным состоянием сервера»), и пушить в `origin`, чтобы GitHub оставался единым источником истины между локальной машиной и обоими серверами.

### 7. Вне скоупа

- Codex CLI — явно не нужен в этот раз.
- Paperclip / мультиагентный оргчарт-пайплайн — исключён по решению пользователя.
- Снос старого сервера `hermes` (`193.228.139.46`) — отдельная задача, не часть этого документа.

## Чек-лист готовности

- [x] Временный `NOPASSWD` sudo снят после установки — 2026-07-17, `sudo -n` снова требует пароль
- [x] `ufw status verbose` — 80 открыт всем, 22 и 443 — только с 5 согласованных IP — проверено 2026-07-17
- [x] `fail2ban-client status sshd` — jail активен (backend systemd), за первые часы уже забанил 2 IP — проверено 2026-07-17
- [x] `unattended-upgrades` включены — active
- [x] `claude --version` (2.1.212) работает, `claude auth login` пройден — loggedIn, subscriptionType pro, аккаунт a.marianenko.73@gmail.com
- [x] `agy --version` (1.1.3, имя команды подтвердилось) работает, OAuth пройден как a.maryanenko@gmail.com — **но инференс заблокирован**, см. «Реально сделано» п.2
- [x] Hermes Agent: `hermes status` — OpenRouter ✓, модель `anthropic/claude-sonnet-5`, `.env` ✓, живой запрос через OpenRouter проверен
- [x] Web-search подтверждён рабочим тестовым запросом — **но через ddgs, а не Perplexity**, см. «Реально сделано» п.3
- [x] `hermes-gateway.service` активен, Telegram-бот @AmarAiAgent_bot отвечает после pairing (amar/384955770), home channel установлен
- [x] `https://hermes.marko-lab.com` — валидный LE-сертификат, Caddy basic-auth (401 без пароля / 200 с), дашборд за ним
- [x] `~/Amar73/hermes` склонирован на сервере (9e2b4aa), `gh auth status` — Amar73, fetch проходит
- [x] Спек и план закоммичены и запушены в `Amar73/hermes`

## Реально сделано (сверка по итогам развёртывания, 2026-07-17)

Отклонения и находки относительно плана:

1. **Sudoers-ловушка**: на сервере уже существовал `/etc/sudoers.d/amar` (`amar ALL=(ALL:ALL) ALL`, к тому же с неверными правами 0644). Файлы в `sudoers.d` читаются по алфавиту, и последнее совпавшее правило побеждает — временный `99-amar-temp` (NOPASSWD) молча перебивался файлом `amar`. Решение: временный файл назван `zzz-amar-temp`, чтобы сортироваться последним; права `amar` исправлены на 0440. После завершения работ `zzz-amar-temp` удалён.
2. **Antigravity CLI — инференс заблокирован Google-ом**: установка и OAuth прошли (`agy` 1.1.3, аккаунт a.maryanenko@gmail.com с активной Google AI Pro), но каждый запрос отклоняется `Eligibility check failed: unknown reason`. Известная массовая проблема (форум Google AI Developers, июнь–июль 2026), на сервере не лечится — нужен appeal в Google / другой аккаунт / ждать. Сбор Interactions data при онбординге отключён по решению пользователя. Отдельная деталь: подкоманды `agy auth login` нет — логин запускается голым `agy` (TUI).
3. **Perplexity web-search невозможен в Hermes 0.18.2**: «кастомный OpenAI-совместимый backend» из документации оказался нереализованной фичей (issue #12832). Реально поддерживаются только встроенные плагины: firecrawl, parallel, tavily, exa, searxng, brave-free, ddgs, xai; `custom_providers` в config.yaml — только для LLM-инференса, не для поиска. По решению пользователя оставлен `ddgs` (DuckDuckGo, без ключа), закреплён явно: `web.backend: ddgs`. `PERPLEXITY_API_KEY` лежит в `~/.hermes/.env` про запас, ничем не используется. Если Perplexity понадобится — придётся писать свой `WebSearchProvider`-плагин по образцу `plugins/web/xai/provider.py`.
4. **Nous Portal**: `hermes setup --portal` запускает OAuth к порталу Nous; пользователь подтвердил подключение в браузере, сессия сохранилась, но кредитов на ней 0 — она неактивна и не мешает, рабочий провайдер OpenRouter.
5. **Дашборд — отдельный сервис, которого не было в плане**: `hermes dashboard` (127.0.0.1:9119) не запускается сам; создан юнит `/etc/systemd/system/hermes-dashboard.service` (User=amar, `--no-open --skip-build`, enabled). Первая сборка web UI выполнена отдельно перед установкой юнита.
6. **Caddy — два нюанса**: (а) добавлен basic-auth (`basic_auth`, Caddy 2.11.4; логин `amar`, пароль у пользователя в менеджере паролей) как второй слой поверх IP-allowlist; (б) потребовался `header_up Host 127.0.0.1` в `reverse_proxy` — иначе защита Hermes от DNS-rebinding отвечает 400 даже с верной авторизацией.
7. **Telegram**: тот же бот @AmarAiAgent_bot, что на старом сервере; старый `hermes-gateway` на 193.228.139.46 остановлен и отключён пользователем до запуска нового (иначе конфликт long-polling). Ловушка: в шаблоне `~/.hermes/.env` есть закомментированный `# TELEGRAM_BOT_TOKEN=` — проверка «ключ уже есть» по простому grep ловит комментарий; сверять по `^TELEGRAM_BOT_TOKEN=`. Gateway ставится с sudo: `hermes --accept-hooks gateway install --system --run-as-user amar --start-now`.
8. **PATH-нюансы**: нативный установщик Claude Code не дописал PATH в `.bashrc` (сделано вручную); `uv` у Hermes лежит в `~/.hermes/bin`, не в `~/.local/bin`; неинтерактивный `ssh ovh 'cmd'` не видит эти пути из-за early-return в Debian `.bashrc` — нужен префикс `export PATH="$HOME/.local/bin:$HOME/.hermes/bin:$PATH"`.
9. **Одношотовый запуск Hermes**: `echo ... | hermes` не работает (TUI закрывается), правильно `hermes -z "..."`.
10. **tmux не входил в базовые пакеты** — доставлен по ходу (нужен для всех headless-OAuth-флоу).
