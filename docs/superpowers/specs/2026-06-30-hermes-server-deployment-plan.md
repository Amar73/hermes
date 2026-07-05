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

## 10. Мультиагентный workflow для кода (bash-бэкапы, python-боты)

Задача ставится через Telegram-бота («напиши bash-скрипт бэкапа X с шифрованием и ротацией») и дальше идёт по org chart внутри Paperclip — автономно, без ручных остановок между этапами:

| Роль | Инструмент | Задача |
|---|---|---|
| Researcher | Perplexity | ищет актуальные best practices/документацию перед написанием |
| Writer | Claude Code (Pro) | пишет код |
| Reviewer | Gemini CLI (Pro) | независимая проверка (другая модель — другой взгляд) |
| Auditor | Codex (Pro) | независимый аудит корректности/безопасности |
| Fallback | OpenRouter (API) | подстраховка, если основной провайдер недоступен, или доп. независимая проверка |

По завершении в Telegram приходит финальное сообщение: результат + сводка замечаний аудитора и ссылка на коммит.

Итоговый код коммитится в зависимости от типа задачи:
- **Bash-скрипты (бэкапы)** → существующий `Amar73/rclone` (там уже есть история наработок).
- **Python Telegram-боты** → новый `Amar73/hermes-scripts`.

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

### 10.4 Репозитории для результатов

- `Amar73/rclone` уже существует — репозиторий создавать не нужно, но с сервера потребуется доступ для пуша (SSH deploy-key или `gh auth login` под пользователем `paperclip`).
- Создать `Amar73/hermes-scripts` (с локальной машины, `gh` уже авторизован):
  ```bash
  gh repo create Amar73/hermes-scripts --public --confirm
  ```
  (или `--private`, если решите иначе) — и аналогично настроить доступ для пуша с сервера.

### 10.5 Проверка

Отправьте боту тестовую задачу (например, «напиши простой bash-скрипт, который бэкапит папку X в MinIO») и убедитесь, что: research → write → review → audit прошли по цепочке, код закоммичен в правильный репозиторий, а в Telegram пришёл итоговый отчёт с замечаниями аудитора.

---

## 11. Чек-лист готовности

- [ ] `dig +short hermes.amar-home.ru` → `193.228.139.46`
- [ ] `https://hermes.amar-home.ru` открывается, сертификат валиден
- [ ] `ufw status` → активен; 80 открыт всем, 22 и 443 — только с `46.34.141.146` и `144.206.228.59`
- [ ] Доступ по SSH/HTTPS с IP, не входящего в allowlist, действительно блокируется (проверить с третьего устройства/VPN)
- [ ] `fail2ban-client status sshd` → jail активен
- [ ] Вход по паролю отключён, ключ работает из нового окна терминала
- [ ] OpenRouter ключ в конфиге, Hermes может выполнить тестовый запрос через него
- [ ] `claude login` — авторизован под Pro-аккаунтом
- [ ] `gemini auth login` — авторизован под Google AI Pro
- [ ] Тестовое сообщение в Telegram-боте получает ответ от Hermes
- [ ] Hermes Desktop подключается к серверу и обрабатывает тестовую голосовую команду
- [ ] `codex login` — авторизован под Pro-аккаунтом
- [ ] Perplexity ключ в конфиге
- [ ] Org chart в Paperclip настроен (researcher/writer/reviewer/auditor привязаны к нужным инструментам)
- [ ] Репозиторий `Amar73/hermes-scripts` создан, доступ для пуша с сервера настроен (и для `Amar73/rclone` тоже)
- [ ] Тестовая задача через Telegram проходит весь конвейер (research → write → review → audit) и коммитится в правильный репозиторий

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
