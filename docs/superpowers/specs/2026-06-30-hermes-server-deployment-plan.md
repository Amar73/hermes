# План развёртывания Hermes Agent / Paperclip на 193.228.139.46

Дата: 2026-06-30
Сервер: `193.228.139.46` (alias `hermes`, `ssh amar@hermes`)
Домен: `hermes.amar-home.ru`

## Стартовое состояние (уже сделано)

- Debian 12 установлен, `apt update && apt upgrade -y` выполнен.
- Пользователь `amar` с passwordless sudo (`/etc/sudoers.d/amar`, права 0440).
- Базовые пакеты: git vim curl gvfs-backends unzip exa tree ninja-build gettext make cmake build-essential gawk. vim — редактор по умолчанию.
- Вход: `ssh amar@hermes`.
- Файрвол не настроен.
- Старые данные сервера сохранять не нужно (чистый старт).
- **(Выполнено 2026-07-05) GitHub-доступ для пользователя `amar`:** `gh` CLI установлен (v2.96.0, через официальный apt-репозиторий `cli.github.com`), `gh auth login` пройден (device-flow, `Logged in as Amar73`, scopes `repo`/`gist`/`read:org`), credential helper подключен (`gh auth git-credential` для `github.com`/`gist.github.com`), git identity выставлен (`Andrey Maryanenko <a.maryanenko@gmail.com>`, как на локальной машине). В `~/Amar73/` склонированы и проверены на реальный push: `hermes`, `rclone`, `arch-niri`, `setup`. Это доступ **только под `amar`** — для пайплайна из раздела 10 (пуш в `Amar73/hermes-scripts` от лица `paperclip`) нужна отдельная настройка, см. шаг 10.4.
- **Сверка с реальным состоянием сервера (2026-07-07, через SSH):**
  - DNS: `hermes.amar-home.ru` уже резолвится в `193.228.139.46` — шаг 1 (DNS) фактически выполнен, можно пропустить.
  - SSH: `PasswordAuthentication no` и `PermitRootLogin no` в `/etc/ssh/sshd_config` уже стоят (вход только по ключу настроен вручную) — часть шага 2 уже сделана.
  - `ufw` и `fail2ban` **не установлены** (`command not found`) — весь остальной шаг 2 (правила ufw, IP-allowlist, fail2ban) ещё предстоит.
  - Комбо-стек **не установлен**: нет пользователя/дира `paperclip`, `caddy.service` не существует, `node`/`nvm` не найдены, `/opt/` пуст, из портов слушает только 22/tcp (ssh). Шаг 3 (`curl -fsSL https://virtua.sh/i/paperclip | bash`) ещё не запускался.
  - GitHub-доступ под `amar` подтверждён как есть (см. пункт выше) — `gh auth status` залогинен как `Amar73`, 4 репозитория на месте.

Везде ниже команды на сервере выполняются после `ssh amar@hermes`, если не указано иное.

---

## 1. DNS — A-запись для поддомена

В панели DNS вашего регистратора для `amar-home.ru` добавьте:

```
hermes.amar-home.ru.  A  193.228.139.46   (TTL 300, можно поднять позже)
```

Проверка с локальной машины:

```bash
dig +short hermes.amar-home.ru
# должно вернуть: 193.228.139.46
```

Дождитесь, пока запись разойдётся (обычно минуты, иногда дольше) — Caddy сможет выпустить TLS-сертификат только когда домен резолвится на сервер и порты 80/443 доступны снаружи.

---

## 2. Хардening сервера (ufw, fail2ban, SSH key-only, IP-allowlist)

Весь доступ, кроме Telegram, ограничен двумя IP:

- Дом: `46.34.141.146`
- Работа: `144.206.228.59`

Telegram-бот не требует входящего порта (long polling — соединение исходящее, к `api.telegram.org`), поэтому на него это ограничение не влияет.

Порт 80 оставляем открытым для всех — на нём Caddy отвечает только на ACME-проверку Let's Encrypt (выпуск/продление сертификата) и делает редирект на HTTPS; запросы Let's Encrypt идут с разных IP по всему миру, и никакого доступа к данным/дашборду через 80 нет.

**Важно:** меняя SSH-доступ, не закрывайте текущую сессию, пока не убедитесь, что новая сессия подключается. Если что-то пойдёт не так — вы рискуете потерять доступ.

**Риск:** если домашний IP динамический и поменяется, доступ по SSH/HTTPS с дома пропадёт, пока вы не обновите правила ufw из другого разрешённого места (или через веб-консоль/VNC хостинг-провайдера). Уточните у провайдера сервера, есть ли такая запасная консоль — это единственный способ восстановить доступ, если оба IP вдруг окажутся недействительны.

```bash
sudo apt install -y ufw fail2ban

# ufw: запрещаем всё входящее по умолчанию
sudo ufw default deny incoming
sudo ufw default allow outgoing

# порт 80 — открыт всем (только ACME-проверка + редирект, без доступа к данным)
sudo ufw allow 80/tcp

# SSH и HTTPS-дашборд — только с дома и с работы
sudo ufw allow from 46.34.141.146 to any port 22 proto tcp
sudo ufw allow from 144.206.228.59 to any port 22 proto tcp
sudo ufw allow from 46.34.141.146 to any port 443 proto tcp
sudo ufw allow from 144.206.228.59 to any port 443 proto tcp

sudo ufw enable
sudo ufw status verbose

# fail2ban: дополнительный уровень защиты SSH (на случай попыток с других IP, которые не пройдут ufw, но полезен и сам по себе)
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

**Если `fail2ban` падает с ошибкой `Have not found any log file for sshd jail`:** на этом сервере (минимальный Debian-образ) нет `rsyslog`, поэтому нет `/var/log/auth.log`, который jail ищет по умолчанию. Вместо установки rsyslog переключите jail на чтение из systemd journal:

```bash
printf '[sshd]\nenabled = true\nbackend = systemd\n' | sudo tee /etc/fail2ban/jail.d/sshd-systemd-backend.local
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

Отключение входа по паролю (только ключ). Перед этим убедитесь, что ваш публичный ключ уже в `~/.ssh/authorized_keys` на сервере — раз вы заходите по `ssh amar@hermes` без пароля, скорее всего, это уже так.

```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload sshd
```

Проверка: откройте **новое** окно терминала и убедитесь, что `ssh amar@hermes` всё ещё работает, прежде чем закрывать текущую сессию.

---

## 3. Установка комбо-стека (Paperclip + Hermes + Caddy + PostgreSQL)

```bash
curl -fsSL https://virtua.sh/i/paperclip | bash
```

Установщик сам определит ветку Debian и поднимет Paperclip, Hermes, Caddy и встроенный PostgreSQL под системным пользователем `paperclip`.

**Важно про долгое выполнение:** установка занимает 15-20+ минут (сборка ~31 workspace-пакета Paperclip, клонирование и установка Hermes Agent, npm-установка 4 CLI-агентов). **Не запускайте установщик в SSH-сессии, обёрнутой в `timeout`, и не полагайтесь на то, что foreground SSH-сессия доживёт до конца** — при обрыве канала процесс на сервере может упасть по broken pipe (`set -euo pipefail` в скрипте). Вместо этого запускайте отвязанно от терминала и опрашивайте лог-файл отдельными короткими SSH-подключениями:

```bash
ssh amar@hermes 'sudo bash -c "setsid env PAPERCLIP_HOST=hermes.amar-home.ru nohup bash -c \"curl -fsSL https://virtua.sh/i/paperclip | bash\" > /root/paperclip-install.log 2>&1 < /dev/null & disown; echo LAUNCHED"'
# затем периодически:
ssh amar@hermes "sudo tail -40 /root/paperclip-install.log"
```

Если установка всё же прервалась на середине (проверить: `ps aux | grep -i paperclip`, `ls /opt/paperclip/server/dist`, `ls /home/paperclip/.hermes`) — **не запускайте установщик заново**, он упадёт на `check_clean_install`, увидев уже установленные node/pnpm/caddy. Нужно вручную доиграть оставшиеся шаги (`install_hermes`, npm-установка агентов, `configure_caddy`, `create_service`, `paperclip_credentials`, `start_services`) — код этих функций есть в самом установщике (`curl -fsSL https://virtua.sh/i/paperclip`) и в общем `https://virtua.sh/i/common/base.sh`.

**Если сборка прервалась именно во время `pnpm build` (parallel-сборка нескольких workspace-пакетов одновременно)** — некоторые пакеты могли не успеть собраться, даже если лог выглядит так, будто дошёл до конца (вывод от разных пакетов чередуется). Явный симптом: `ui/dist/` содержит статику (иконки, `assets/`), но **нет `index.html`** → сервер отвечает `Cannot GET /` (при этом `/api/health` работает нормально). Проверить и перестроить только пропущенное безопасно — `pnpm build` идемпотентен:

```bash
ssh amar@hermes 'sudo -u paperclip bash -c "setsid env NODE_OPTIONS=--max-old-space-size=3072 nohup bash -c \"cd /opt/paperclip && pnpm build\" > /home/paperclip/paperclip-rebuild.log 2>&1 < /dev/null & disown"'
# опрашивать: ssh amar@hermes "sudo -u paperclip tail -30 /home/paperclip/paperclip-rebuild.log"
# после завершения: ls /opt/paperclip/ui/dist/index.html && sudo systemctl restart paperclip
```

**Ещё одна вещь, которая всплывает при первом заходе в дашборд:** если открыть `https://<домен>` в браузере, Paperclip может ответить `Hostname '<домен>' is not allowed for this Paperclip instance.` — это отдельная от Caddy проверка на уровне самого приложения (`allowedHostnames` в конфиге). Если конфига ещё нет (`No config found ... Run paperclip onboard first`), сначала выполните ониординг **без интерактивных промптов** (`-y` = принять quickstart-настройки: `local_trusted`/`private`/loopback, что и нужно — приложение слушает только 127.0.0.1, наружу его пускает Caddy):

```bash
sudo -u paperclip bash -c 'cd /opt/paperclip && pnpm paperclipai onboard -y'
```

**Осторожно:** `onboard -y` в некоторых версиях CLI не просто сохраняет конфиг, а следом сам стартует `paperclipai run` в текущей сессии (доп. dev-инстанс сервера, на свободном порту, если 3100 уже занят настоящим systemd-сервисом). Если это произошло — оборвите SSH-команду и **убейте весь процесс-дерево** этого лишнего инстанса (`ps aux | grep -E 'onboard|node.*cli/src/index'`, затем `sudo kill -TERM <pids>`), не трогая настоящий `paperclip.service`. Затем добавьте домен в allowlist и перезапустите настоящий сервис:

```bash
sudo -u paperclip bash -c 'cd /opt/paperclip && pnpm paperclipai allowed-hostname hermes.amar-home.ru'
sudo systemctl restart paperclip
```

**По ходу установки:**
- Если установщик спросит домен — укажите `hermes.amar-home.ru`.
- В конце установщик обычно выводит сгенерированные креды (БД, админ-токен и т.п.) и пути к конфигам — **сохраните их сразу** в менеджер паролей, повторно они могут не показаться.
- Запишите путь к `.env`/конфигу Hermes (понадобится в шагах 5, 6, 8) и путь к Caddyfile (шаг 4).

---

## 4. Домен и HTTPS в Caddy

Если установщик не настроил домен на шаге 3, отредактируйте Caddyfile (обычно `/etc/caddy/Caddyfile`, уточните по выводу установщика):

```
hermes.amar-home.ru {
    reverse_proxy localhost:<порт_приложения>
}
```

Применить и проверить конфиг, затем перезагрузить Caddy:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Проверка HTTPS:

```bash
curl -vI https://hermes.amar-home.ru
```

Должен вернуться валидный сертификат (Let's Encrypt, автоматически выпущенный Caddy) и редирект с HTTP на HTTPS. Порт приложения наружу торчать не должен — снаружи доступны только 80/443 (см. ufw в шаге 2).

---

## 5. OpenRouter (API)

1. Зарегистрируйтесь / войдите на openrouter.ai.
2. В Settings → Keys создайте новый API-ключ.
3. Добавьте его в конфиг Hermes Agent на сервере (путь определён в шаге 3):
   ```bash
   echo 'OPENROUTER_API_KEY=sk-or-...' | sudo tee -a /путь/к/.env
   ```
4. Перезапустите сервис Hermes, чтобы он подхватил переменную (имя сервиса уточните по выводу установщика, обычно `systemctl status hermes` покажет точное имя):
   ```bash
   sudo systemctl restart hermes
   ```

---

## 6. Авторизация Claude Code (Pro, OAuth)

На сервере, под пользователем, от которого запускается Hermes (обычно `paperclip`):

```bash
sudo -u paperclip claude login
```

Откроется ссылка для OAuth — переходите по ней в браузере на своём компьютере, входите аккаунтом с подпиской Pro, вставляете код обратно в терминал сервера. Вы уже знакомы с этим флоу. Если команда `login` не сработает как ожидается — проверьте `claude --help`, синтаксис может отличаться между версиями.

---

## 7. Авторизация Gemini CLI (Google AI Pro, OAuth)

Аналогично, тем же device-code flow:

```bash
sudo -u paperclip gemini auth login
```

Вход через аккаунт с подпиской Google AI Pro. Точное имя команды авторизации может отличаться в зависимости от версии Gemini CLI — проверьте `gemini --help`, если `auth login` не подойдёт.

---

## 8. Подключение Telegram-бота

Токен уже есть. Добавьте его в конфиг Hermes:

```bash
echo 'TELEGRAM_BOT_TOKEN=123456:ABC-...' | sudo tee -a /путь/к/.env
sudo systemctl restart hermes
```

Проверка токена (с локальной машины, разрешено в settings.local.json):

```bash
curl https://api.telegram.org/bot<TOKEN>/getMe
# {"ok":true,"result":{...}}
```

Затем напишите боту в Telegram тестовое сообщение и убедитесь, что Hermes отвечает.

---

## 9. Hermes Desktop (локально, на вашей Arch-машине)

```bash
yay -S hermes-desktop-bin
```

После установки запустите приложение и в настройках укажите адрес сервера `https://hermes.amar-home.ru` и токен доступа (формат зависит от того, как Hermes Agent выдаёт токены клиентам — уточните в документации/настройках самого приложения после первого запуска). Проверьте голосовую команду end-to-end.

---

## 10. Мультиагентный workflow для кода (Telegram-боты на Python)

Из проекта исключено написание bash-скриптов бэкапа — `Amar73/rclone` этим планом больше не затрагивается. Сейчас в скоупе только написание Telegram-ботов на Python; список задач может расшириться позже — org chart ниже рассчитан так, чтобы добавление новых типов задач не требовало смены архитектуры, только новых правил маршрутизации в репозитории.

Задача ставится через Telegram-бота («напиши Telegram-бота на Python, который делает X») и дальше идёт по org chart внутри Paperclip — автономно, без ручных остановок между этапами:

| Роль | Инструмент | Задача |
|---|---|---|
| Researcher | Perplexity | ищет актуальные best practices/документацию перед написанием |
| Writer | Claude Code (Pro) | пишет код |
| Reviewer | Gemini CLI (Pro) | независимая проверка (другая модель — другой взгляд) |
| Auditor | Codex (Pro) | независимый аудит корректности/безопасности |
| Fallback | OpenRouter (API) | подстраховка, если основной провайдер недоступен, или доп. независимая проверка |

По завершении в Telegram приходит финальное сообщение: результат + сводка замечаний аудитора и ссылка на коммит.

Итоговый код коммитится в `Amar73/hermes-scripts` (новый репозиторий).

### 10.1 Авторизация Codex CLI (Pro, OAuth)

```bash
sudo -u paperclip codex login
```

Тот же device-code flow, что и для Claude Code/Gemini (шаги 6–7): ссылка → браузер на своём компьютере → код обратно в терминал. Точный синтаксис проверьте через `codex --help`, если `login` не подойдёт.

### 10.2 Ключ Perplexity

1. Создайте API-ключ в аккаунте Perplexity.
2. Добавьте в конфиг Hermes: `echo 'PERPLEXITY_API_KEY=pplx-...' | sudo tee -a /путь/к/.env`
3. Перезапустите сервис (имя уточните по выводу установщика): `sudo systemctl restart hermes`

### 10.3 Org chart в Paperclip

Настройте роли writer / reviewer / auditor / researcher через дашборд или конфиг Paperclip (`https://hermes.amar-home.ru`), связав каждую роль с соответствующим инструментом из таблицы выше. **Точный формат настройки (yaml-конфиг, CLI или только дашборд) не известен заранее** — уточните по документации/UI Paperclip, которые появятся после установки (шаг 3); не изобретайте синтаксис заранее.

### 10.4 Репозиторий для результатов

Создать `Amar73/hermes-scripts` (с локальной машины, `gh` уже авторизован):
```bash
gh repo create Amar73/hermes-scripts --public --confirm
```
(или `--private`, если решите иначе) — и настроить доступ для пуша с сервера (SSH deploy-key или `gh auth login` под пользователем `paperclip`).

### 10.5 Проверка

Отправьте боту тестовую задачу (например, «напиши простого Telegram-бота на Python, который отвечает /start и /help») и убедитесь, что: research → write → review → audit прошли по цепочке, код закоммичен в `Amar73/hermes-scripts`, а в Telegram пришёл итоговый отчёт с замечаниями аудитора.

---

## 11. Чек-лист готовности

- [x] GitHub-доступ под `amar` настроен (`gh auth login`, credential helper, `~/Amar73/{hermes,rclone,arch-niri,setup}` склонированы, push проверен) — выполнено 2026-07-05
- [x] `dig +short hermes.amar-home.ru` → `193.228.139.46` — подтверждено 2026-07-07
- [x] Комбо-стек (Paperclip + Hermes Agent + Caddy + Claude Code/Codex/Gemini CLI/OpenCode) установлен — выполнено 2026-07-07, см. заметку в шаге 3 про прерывание/докат установки
- [x] `https://hermes.amar-home.ru` открывается, сертификат валиден — подтверждено 2026-07-07 (HTTP/2, basicauth возвращает 401 как и ожидалось)
- [x] `ufw status` → активен; 80 открыт всем, 22 и 443 — только с `46.34.141.146` и `144.206.228.59` — выполнено 2026-07-07
- [ ] Доступ по SSH/HTTPS с IP, не входящего в allowlist, действительно блокируется (проверить с третьего устройства/VPN)
- [x] `fail2ban-client status sshd` → jail активен — выполнено 2026-07-07 (потребовалась правка: backend=systemd, т.к. на сервере нет rsyslog/`/var/log/auth.log`, см. заметку в шаге 2)
- [x] Вход по паролю отключён, ключ работает из нового окна терминала — подтверждено 2026-07-07 (новая SSH-сессия после `ufw enable` прошла успешно)
- [ ] OpenRouter ключ в конфиге, Hermes может выполнить тестовый запрос через него
- [ ] `claude login` — авторизован под Pro-аккаунтом
- [ ] `gemini auth login` — авторизован под Google AI Pro
- [ ] Тестовое сообщение в Telegram-боте получает ответ от Hermes
- [ ] Hermes Desktop подключается к серверу и обрабатывает тестовую голосовую команду
- [ ] `codex login` — авторизован под Pro-аккаунтом
- [ ] Perplexity ключ в конфиге
- [ ] Org chart в Paperclip настроен (researcher/writer/reviewer/auditor привязаны к нужным инструментам)
- [ ] Репозиторий `Amar73/hermes-scripts` создан, доступ для пуша с сервера настроен
- [ ] Тестовая задача через Telegram проходит весь конвейер (research → write → review → audit) и коммитится в `Amar73/hermes-scripts`

---

## 12. Skills и MCP в Claude Code для этого проекта

Настроено заранее (2026-06-30), на локальной машине, в `.claude/settings.local.json` репозитория `hermes`:

- **Allowlist для безопасных команд**: точечные read-only правила вместо широкого `Bash(ssh *)` — `systemctl status`, `journalctl`, `ufw status`, `fail2ban-client status`, `caddy validate`, `dig`/`nslookup`, `curl` к `api.telegram.org`. Деструктивные команды (restart/reboot/rm/изменение правил ufw) по-прежнему требуют подтверждения.
- **Telegram**: без MCP — тестирование через `curl` к Bot API напрямую, чтобы боевой токен бота не передавался стороннему неаудированному коду.
- **GitHub**: без MCP — `gh` CLI уже авторизован и полностью покрывает работу с репозиторием `Amar73/hermes`.

Отложено до завершения установки (шаг 3), когда появятся реальные креды БД:

- **Postgres MCP** (`@modelcontextprotocol/server-postgres`, официальный, read-only) — для инспекции схемы и данных Hermes Agent без ручного `psql` по SSH. БД слушает только `localhost` на сервере, поэтому потребуется SSH-туннель (например, `ssh -L 5432:localhost:5432 amar@hermes`) перед подключением MCP-сервера с локальной машины.

Полезные встроенные skills Claude Code на время развёртывания и дальше:
- `verify` / `run` — чтобы реально проверить, что дашборд и бот работают, а не полагаться только на статус команд.
- `security-review` — стоит прогнать перед тем, как считать хардening (шаг 2) и обработку токенов (шаги 5, 8) завершёнными.
