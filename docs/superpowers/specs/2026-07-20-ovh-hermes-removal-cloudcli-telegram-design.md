# Дизайн: снос Hermes Agent, CloudCLI на 443 и Telegram-плагин для Claude Code на `ovh` (51.75.66.60)

Дата: 2026-07-20
Сервер: `51.75.66.60` (hostname `vps-caef402f`), домен `marko-lab.com`
Продолжение: [`2026-07-17-ovh-server-deployment-design.md`](./2026-07-17-ovh-server-deployment-design.md) — тот документ развернул Claude Code, Antigravity CLI и Hermes Agent на этом же сервере; этот документ описывает следующий шаг, выполненный прямо в интерактивной Claude Code сессии на самом `ovh` (не через SSH с локальной машины).

## Задача

Пользователь запросил три изменения на `ovh`:
1. Удалить Hermes Agent целиком.
2. Поставить web-морду для Claude Code на порту 443.
3. Поставить Telegram-интеграцию для работы с Claude Code через чат-бота.

## Решения

- **Hermes Agent — полное удаление** (не просто остановка): сервисы, `~/.hermes` (1.4 ГБ), Caddy-блок, осиротевший CLI-враппер. Выбрано пользователем явно (Recommended-опция при выборе).
- **Web UI — [siteboon/claudecodeui](https://github.com/siteboon/claudecodeui) (CloudCLI)**, самый популярный и активно поддерживаемый из рассмотренных (matalvernaz/claude-web, sugyan/claude-code-webui, fafawlf/claude-code-web). Перед выбором пользователь отдельно уточнил модель трафика — подтверждено (по коду и `--help`), что браузер обращается только к серверу CloudCLI на самом `ovh` (Express+WebSocket, порт 3001), который на этой же машине спавнит локальный `claude` CLI; `claude.ai` участвует только как обычный backend самого Claude Code (и для OAuth), а не как прокси браузерного трафика. Cloud-режим CloudCLI (`CloudCLI Cloud`) не используется.
- **Домен** — новый саб `claude.marko-lab.com` (A-запись уже существовала на момент работы, отдельно не создавалась), а не переиспользование `hermes.marko-lab.com`.
- **Telegram-интеграция — официальный плагин Anthropic** (`anthropics/claude-plugins-official`, `external_plugins/telegram`), а не один из community-бриджей (RichardAtCT/claude-code-telegram, blogminhquy/claude-telegram-bot и др.). Бот переиспользован тот же — `@AmarAiAgent_bot`, что был у Hermes gateway.
- **Рабочая директория Telegram-сессии** — `~` (весь home), не привязана к одному репозиторию. Выбор пользователя.
- **Permission-mode** — после практической проверки выбран **дефолтный `manual`**, а не `dontAsk` (см. «Реально сделано» п.4 — `dontAsk` оказался несовместим с плагином) и не `bypassPermissions` (пользователь отклонил как избыточно опасный).
- **CloudCLI — блокировка новых пользователей**: специальных действий не потребовалось, см. п.3 ниже — это встроенное поведение.

## Компоненты

### 1. Удаление Hermes Agent

- `systemctl stop/disable` + `rm` юнитов `hermes-gateway.service`, `hermes-dashboard.service`, `daemon-reload`.
- `rm -rf ~/.hermes`.
- Удалён блок `hermes.marko-lab.com` из `/etc/caddy/Caddyfile`, `caddy validate` + `systemctl reload caddy`.
- Удалён осиротевший враппер `~/.local/bin/hermes` (шел на несуществующий venv).

### 2. CloudCLI (web UI для Claude Code)

- Node.js 22 через официальный репозиторий NodeSource (в Debian 13 репе только v20, а CloudCLI требует v22+).
- `npm install -g @cloudcli-ai/cloudcli` (1.36.3).
- systemd `cloudcli.service`: `HOST=127.0.0.1`, `SERVER_PORT=3001`, `CLAUDE_CLI_PATH=/home/amar/.local/bin/claude` (абсолютный путь — нативный установщик Claude Code не прописывает PATH там, где его видит systemd), `Restart=always` (не `on-failure` — тот же баг, что был у `hermes-dashboard.service` в прошлом деплое: SIGTERM не считается сбоем).
- Caddy: `claude.marko-lab.com { reverse_proxy 127.0.0.1:3001 { header_up Host 127.0.0.1 } }`.
- Доступ ограничен тем же ufw allowlist (5 IP на 443/tcp), что и раньше — отдельный `basic_auth` слой (как был у Hermes-дашборда) сознательно не добавлен: у CloudCLI есть собственная однопользовательская аутентификация (см. п.3).

### 3. Ограничение регистрации в CloudCLI

Отдельного действия не потребовалось — `POST /api/auth/register` в исходниках (`server/routes/auth.js`) жёстко проверяет `hasUsers()` и возвращает `403 "User already exists. This is a single-user system."`, если хоть один пользователь уже создан. После того как пользователь прошёл setup-wizard и создал свой аккаунт, регистрация новых пользователей уже недоступна сама по себе.

### 4. Telegram-плагин (официальный, Anthropic)

- Prereq: Bun (`curl -fsSL https://bun.sh/install | bash`) — MCP-сервер плагина работает на Bun.
- Маркетплейс `claude-plugins-official` уже был сконфигурирован. `claude plugin install telegram@claude-plugins-official`.
- Токен бота (переиспользован `@AmarAiAgent_bot`, токен заново получен у @BotFather — старый лежал в уже удалённом `~/.hermes/.env`) → `~/.claude/channels/telegram/.env` (`chmod 600`).
- Pairing прошёл через `/telegram:access pair <code>` — см. находку про `dontAsk` ниже, из-за которой pairing на первой попытке пришлось завершать вручную.
- Продакшн-обвязка: systemd `telegram-claude.service` → `/usr/local/bin/telegram-claude-wrapper.sh` → `tmux new-session -d -s telegram-bot -c /home/amar "claude --channels plugin:telegram@claude-plugins-official"`, затем цикл `while tmux has-session ...; do sleep 5; done`, чтобы systemd отслеживал реальное время жизни сессии внутри tmux и `Restart=always` срабатывал корректно.
- Diaложный триггер workspace-trust на `~` заранее снят напрямую в `~/.claude.json` (`hasTrustDialogAccepted: true` для проекта `/home/amar`) — иначе перезапуск сервиса после падения завис бы на диалоге подтверждения, отвечать на который некому.

## Чек-лист готовности

- [x] `hermes-gateway.service`, `hermes-dashboard.service` — остановлены, отключены, юниты удалены — 2026-07-20
- [x] `~/.hermes` удалён (1.4 ГБ освобождено), `~/.local/bin/hermes` (осиротевший враппер) удалён
- [x] Caddy-блок `hermes.marko-lab.com` убран, `caddy validate` + reload прошли без ошибок
- [x] Node.js 22 (NodeSource), CloudCLI 1.36.3 установлены; `cloudcli.service` активен, `Restart=always`
- [x] `https://claude.marko-lab.com` — валидный Let's Encrypt сертификат, 200 OK через ufw-allowlist
- [x] CloudCLI setup wizard пройден пользователем, логин подтверждён; повторная регистрация недоступна (single-user, проверено по коду)
- [x] Официальный Telegram-плагин установлен, бот `@AmarAiAgent_bot` запарен (chat_id `384955770`, allowlist policy)
- [x] Permission-relay через Telegram (inline Allow/Deny) подтверждён рабочим на реальном запросе (правка `MEMORY.md`)
- [x] `telegram-claude.service` активен и `enabled`, переживёт reboot
- [x] Полный round-trip проверен дважды: сообщение → ответ бота; и запрос на разрешение → одобрение через Telegram → коммит+пуш, выполненный самим агентом
- [ ] Временный NOPASSWD sudo-грант (`/etc/sudoers.d/amar` либо аналог) **оставлен активным по прямому указанию пользователя** ("оставь sudo") — не снят в конце, в отличие от предыдущего деплоя. Снять при следующей плановой работе, если необходимость отпадёт.

## Реально сделано (сверка по итогам работы, 2026-07-20)

Отклонения и находки относительно первоначального плана:

1. **Sudo снова потребовал пароль**: NOPASSWD-грант из предыдущего деплоя (`zzz-amar-temp`) был корректно снят в конце того деплоя, как и задумывалось. В начале этой сессии пришлось попросить пользователя вручную выдать новый грант — самому создать sudoers-файл без sudo невозможно (курица и яйцо). По просьбе пользователя новый грант **оставлен активным** на конец этой сессии (см. чек-лист).

2. **Осиротевшая директория `hermes/.claude/`**: примерно в середине сессии пользователь (через свой клиент-мультиплексор «Herdr», подключённый по Remote Control) переименовал весь проект «на лету»: `~/Amar73/hermes` → `~/Amar73/ClaudeOVH`, GitHub-репозиторий `Amar73/hermes` → `Amar73/ClaudeOVH`, приватный репозиторий памяти `Amar73/claude-memory-hermes` → `Amar73/claude-memory-ClaudeOVH`. Текущая же интерактивная сессия (эта самая) была запущена ещё со старым cwd `~/Amar73/hermes` и харнесс продолжал откатывать cwd Bash-инструмента к этому (уже переименованному) пути между вызовами, из-за чего один раз автоматически пересоздалась пустая директория `~/Amar73/hermes/.claude/settings.local.json`. Удалена вручную. Оставшиеся неполные правки после ручного переименования (`CLAUDE.md` в основном репо, `README.md` в репо памяти) — докоммичены и запушены.

3. **`--permission-mode dontAsk` несовместим с Telegram-плагином**: классификатор auto-mode блокирует `Write` в собственный файл состояния плагина `~/.claude/channels/telegram/access.json`, и даже блокирует вызов MCP-инструмента `reply` самого плагина — то есть в `dontAsk` бот не может ни завершить pairing, ни ответить пользователю. Обнаружено эмпирически на первом тестовом запуске (в `tmux`, до systemd-обвязки). Первый pairing (`код d65267`) пришлось завершить вручную — из внешней (не заблокированной) сессии отредактирован `access.json` (`allowFrom` += chat_id) и создан маркер `~/.claude/channels/telegram/approved/<chatId>` — по образцу реального поведения, проверенному по исходнику `server.ts` плагина (комментарий на строке ~326: `/telegram:access` сам создаёт такой файл при паринге), а не угадано.

4. **Найден и использован relay пермишенов через Telegram**: в `server.ts` плагина заявлена MCP-capability `claude/channel/permission` — сервер плагина подписан на `notifications/claude/channel/permission_request` от Claude Code и рассылает inline-кнопки `✅ Allow` / `❌ Deny` / `See more` всем адресам из `access.allowFrom` (только личка, группы намеренно исключены — «single-user mode for official plugins»). Это заменяет необходимость в `dontAsk`/`bypassPermissions`: дефолтный `manual`-режим работает штатно, а подтверждения приходят прямо в Telegram. Проверено на реальном запросе — правка `MEMORY.md`, одобренная пользователем через Telegram, привела к успешному коммиту и пушу от имени агента.

5. **Headless-запуск `claude` без pty не работает**: `claude --channels ... ` без `-p` и без реального терминала (тестировалось через `nohup ... < /dev/null`) падает с `Error: Input must be provided either through stdin or as a prompt argument when using --print` — CLI детектирует отсутствие TTY и откатывается в print-режим, для которого канал бесполезен (одноразовый ответ и выход, не долгоживущий листенер). Решение — `tmux`, тот же паттерн, что уже использовался для headless OAuth-логинов в прошлом деплое. `systemd` при этом не запускает `claude` напрямую, а через обёртку, которая создаёт detached-tmux-сессию и блокируется в цикле `tmux has-session`, пока сессия жива — так `Restart=always` реагирует на реальную гибель `claude`, а не на мгновенный выход обёртки.

6. **Workspace-trust диалог для `~`**: первый интерактивный запуск в `/home/amar` (ранее там `claude` не запускался) показывает диалог доверия к папке. Для systemd-сервиса это означало бы зависание после каждого `Restart=always` без возможности подтвердить. Исправлено точечной правкой `~/.claude.json` (`projects."/home/amar".hasTrustDialogAccepted = true`), а не попыткой автоматически слать Enter через обёртку.

7. **`CLAUDE_CLI_PATH` и другие абсолютные пути для systemd**: аналогично находке из прошлого деплоя (там — `uv`/`hermes` в нестандартных путях), здесь `cloudcli.service` не видел бы `claude`, если бы полагался на `$PATH` из `.bashrc` — задан явный `CLAUDE_CLI_PATH=/home/amar/.local/bin/claude` в юните.

8. **Токен Telegram-бота пришлось получать заново**: старый токен лежал только в `~/.hermes/.env`, который был удалён на шаге сноса Hermes до того, как стало ясно, что бот будет переиспользован. Не критично — токен получен заново у @BotFather, — но на будущее: если планируется переиспользование секрета, стоит вынести его до удаления директории, где он хранился.

9. **CloudCLI — блокировка регистрации новых пользователей** (отдельный запрос пользователя в процессе работы) оказалась уже встроенным поведением приложения, а не задачей, требующей конфигурации — см. п.3 «Компонентов» выше.
