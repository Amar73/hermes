# Media Downloader Telegram Bot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a working Telegram bot (`@amardownloader_bot`) that downloads YouTube/Instagram videos via `yt-dlp` and sends them back in-chat, running as a systemd service on the `hermes` server.

**Architecture:** Three small, independently-testable modules (`utils.py` for URL parsing/cleanup, `downloader.py` for `yt-dlp` format-selection and the actual download, `bot.py` for Telegram wiring) in a new repo `Amar73/telegram-bots`, subdirectory `media-downloader/`. Pure logic (URL extraction, format selection under the size limit) gets unit tests; the Telegram/yt-dlp I/O glue is verified by a real end-to-end run instead, since mocking Telegram's API and a live video extractor buys little confidence for a personal single-user bot.

**Tech Stack:** Python 3.10+, `python-telegram-bot` 21.6, `yt-dlp`, `pytest` (dev-only), systemd.

## Global Constraints

- File-size limit: cloud Bot API caps uploads at 50MB. Use `MAX_FILE_BYTES = 49 * 1024 * 1024` everywhere (1MB margin for Telegram's own overhead) — never attempt a Local Bot API Server workaround (out of scope, see spec).
- Only `.mp4` progressive formats (both video+audio in one file, no muxing) are eligible for selection — keeps the downloader simple and dependency-light (still needs `ffmpeg` for yt-dlp's own remuxing/thumbnail work, but not for combining separate video/audio streams).
- Repo: new `Amar73/telegram-bots`, local checkout `/home/amar/Amar73/telegram-bots`, this bot lives in `media-downloader/` inside it (repo holds multiple bots over time).
- Bot token: `@amardownloader_bot` — already created via BotFather, value handled in Task 5, never committed to git.
- Server hosting: `hermes` (193.228.139.46, `ssh amar@hermes`), runs as user `amar` (not `paperclip` — unrelated to the Hermes/Paperclip agent stack), own venv, own systemd unit.
- All Telegram-facing user messages are in Russian, matching the rest of this project's bots.

---

### Task 1: Project scaffolding + `utils.py`

**Files:**
- Create: `/home/amar/Amar73/telegram-bots/media-downloader/utils.py`
- Create: `/home/amar/Amar73/telegram-bots/media-downloader/tests/test_utils.py`
- Create: `/home/amar/Amar73/telegram-bots/.gitignore`

**Interfaces:**
- Produces: `extract_url(text: str) -> str | None`, `is_supported_url(url: str) -> bool`, `cleanup_file(path: str) -> None`. Tasks 2 and 3 do not depend on this task's internals, only these three function signatures.

- [ ] **Step 1: Initialize the repo and directory structure**

```bash
mkdir -p /home/amar/Amar73/telegram-bots/media-downloader/tests
cd /home/amar/Amar73/telegram-bots
git init
```

- [ ] **Step 2: Write `.gitignore`**

```
.venv/
__pycache__/
*.pyc
.env
```

- [ ] **Step 3: Create a venv and install dev/runtime deps**

```bash
cd /home/amar/Amar73/telegram-bots
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install pytest python-telegram-bot==21.6 yt-dlp
```

- [ ] **Step 4: Write the failing tests**

Tests import `utils` as a top-level module (not a package) and are run with
`media-downloader/` as the working directory — the dir on disk is named
with a hyphen (`media-downloader`, matching this repo's multi-bot
convention) so it can't be a Python package name itself; `utils.py` and
`downloader.py` inside it are plain top-level modules instead.

`media-downloader/tests/test_utils.py`:
```python
import os
import tempfile

from utils import cleanup_file, extract_url, is_supported_url


def test_extract_url_direct_link():
    assert extract_url("https://www.youtube.com/watch?v=abc123") == "https://www.youtube.com/watch?v=abc123"


def test_extract_url_from_surrounding_text():
    text = "смотри видео https://youtu.be/xyz это смешно"
    assert extract_url(text) == "https://youtu.be/xyz"


def test_extract_url_no_link():
    assert extract_url("привет, как дела?") is None


def test_extract_url_empty_string():
    assert extract_url("") is None


def test_is_supported_url_youtube():
    assert is_supported_url("https://www.youtube.com/watch?v=abc") is True


def test_is_supported_url_youtu_be():
    assert is_supported_url("https://youtu.be/abc") is True


def test_is_supported_url_instagram():
    assert is_supported_url("https://www.instagram.com/reel/abc/") is True


def test_is_supported_url_unsupported_domain():
    assert is_supported_url("https://vimeo.com/12345") is False


def test_cleanup_file_removes_existing_file():
    fd, path = tempfile.mkstemp()
    os.close(fd)
    assert os.path.exists(path)
    cleanup_file(path)
    assert not os.path.exists(path)


def test_cleanup_file_missing_file_does_not_raise():
    cleanup_file("/tmp/definitely-does-not-exist-12345.mp4")
```

- [ ] **Step 5: Run tests to verify they fail**

Run pytest from inside `media-downloader/` (not the repo root) so `utils.py`
is importable as a top-level module:
```bash
cd /home/amar/Amar73/telegram-bots/media-downloader
../.venv/bin/python -m pytest tests/test_utils.py -v
```
Expected: FAIL with `ModuleNotFoundError: No module named 'utils'`.

- [ ] **Step 6: Write `utils.py`**

```python
"""URL parsing and file-cleanup helpers for the media-downloader bot."""
import os
import re

URL_RE = re.compile(r"https?://\S+")

SUPPORTED_DOMAINS = ("youtube.com", "youtu.be", "instagram.com")


def extract_url(text: str) -> str | None:
    """Return the first http(s) URL found in `text`, or None if there isn't one."""
    if not text:
        return None
    match = URL_RE.search(text)
    return match.group(0) if match else None


def is_supported_url(url: str) -> bool:
    """True if `url`'s domain is one this bot knows how to handle."""
    return any(domain in url for domain in SUPPORTED_DOMAINS)


def cleanup_file(path: str) -> None:
    """Best-effort delete of a downloaded file. Never raises."""
    try:
        if path and os.path.exists(path):
            os.remove(path)
    except OSError:
        pass
```

- [ ] **Step 7: Run tests to verify they pass**

```bash
cd /home/amar/Amar73/telegram-bots/media-downloader
../.venv/bin/python -m pytest tests/test_utils.py -v
```
Expected: `9 passed`

- [ ] **Step 8: Commit**

```bash
cd /home/amar/Amar73/telegram-bots
git add .gitignore media-downloader/utils.py media-downloader/tests/test_utils.py
git commit -m "Add URL extraction and file-cleanup utilities"
```

---

### Task 2: `downloader.py` — format selection + download

**Files:**
- Create: `/home/amar/Amar73/telegram-bots/media-downloader/downloader.py`
- Create: `/home/amar/Amar73/telegram-bots/media-downloader/tests/test_downloader.py`

**Interfaces:**
- Consumes: nothing from Task 1.
- Produces: `select_format(formats: list[dict], max_bytes: int) -> str | None` (pure, unit-tested), `async def download_video(url: str, output_dir: str, max_bytes: int) -> str` (returns the downloaded file's path), exception classes `VideoTooLargeError` and `DownloadFailedError`. Task 3 imports all four names from this module.

- [ ] **Step 1: Write the failing tests for `select_format`**

`media-downloader/tests/test_downloader.py`:
```python
from downloader import select_format

MB = 1024 * 1024


def _fmt(format_id, ext, height, vcodec, acodec, filesize=None, filesize_approx=None):
    return {
        "format_id": format_id,
        "ext": ext,
        "height": height,
        "vcodec": vcodec,
        "acodec": acodec,
        "filesize": filesize,
        "filesize_approx": filesize_approx,
    }


def test_select_format_picks_highest_quality_under_limit():
    formats = [
        _fmt("18", "mp4", 360, "avc1", "mp4a", filesize=20 * MB),
        _fmt("22", "mp4", 720, "avc1", "mp4a", filesize=45 * MB),
        _fmt("37", "mp4", 1080, "avc1", "mp4a", filesize=90 * MB),
    ]
    assert select_format(formats, max_bytes=49 * MB) == "22"


def test_select_format_skips_video_only_formats():
    formats = [
        _fmt("137", "mp4", 1080, "avc1", "none", filesize=10 * MB),
        _fmt("18", "mp4", 360, "avc1", "mp4a", filesize=20 * MB),
    ]
    assert select_format(formats, max_bytes=49 * MB) == "18"


def test_select_format_skips_non_mp4():
    formats = [
        _fmt("248", "webm", 1080, "vp9", "opus", filesize=10 * MB),
        _fmt("18", "mp4", 360, "avc1", "mp4a", filesize=20 * MB),
    ]
    assert select_format(formats, max_bytes=49 * MB) == "18"


def test_select_format_uses_filesize_approx_when_filesize_missing():
    formats = [
        _fmt("22", "mp4", 720, "avc1", "mp4a", filesize=None, filesize_approx=30 * MB),
    ]
    assert select_format(formats, max_bytes=49 * MB) == "22"


def test_select_format_returns_none_when_nothing_fits():
    formats = [
        _fmt("37", "mp4", 1080, "avc1", "mp4a", filesize=90 * MB),
    ]
    assert select_format(formats, max_bytes=49 * MB) is None


def test_select_format_returns_none_when_size_unknown():
    formats = [
        _fmt("22", "mp4", 720, "avc1", "mp4a", filesize=None, filesize_approx=None),
    ]
    assert select_format(formats, max_bytes=49 * MB) is None
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /home/amar/Amar73/telegram-bots/media-downloader
../.venv/bin/python -m pytest tests/test_downloader.py -v
```
Expected: FAIL with `ModuleNotFoundError: No module named 'downloader'`.

- [ ] **Step 3: Write `downloader.py`**

```python
"""yt-dlp wrapper: format selection under the Telegram size limit, and download."""
import asyncio
import uuid
from pathlib import Path

import yt_dlp


class VideoTooLargeError(Exception):
    """Raised when no mp4 format fits under the size limit."""


class DownloadFailedError(Exception):
    """Raised when yt-dlp can't extract or download the video at all
    (private, deleted, unsupported, network error, etc.)."""


def select_format(formats: list[dict], max_bytes: int) -> str | None:
    """Pick the highest-quality progressive mp4 format that fits under max_bytes.

    Only considers progressive formats (single file has both video and audio
    tracks — vcodec and acodec both set) so no post-download muxing is needed.
    Falls back to filesize_approx when yt-dlp doesn't report an exact filesize
    (common for formats without a known content-length up front).
    """
    candidates = []
    for f in formats:
        if f.get("ext") != "mp4":
            continue
        if f.get("vcodec") in (None, "none") or f.get("acodec") in (None, "none"):
            continue
        size = f.get("filesize") or f.get("filesize_approx")
        if size is None or size > max_bytes:
            continue
        height = f.get("height") or 0
        candidates.append((height, size, f["format_id"]))

    if not candidates:
        return None

    candidates.sort(reverse=True)
    return candidates[0][2]


def _download_sync(url: str, output_dir: str, max_bytes: int) -> str:
    probe_opts = {"quiet": True, "no_warnings": True, "skip_download": True}
    try:
        with yt_dlp.YoutubeDL(probe_opts) as ydl:
            info = ydl.extract_info(url, download=False)
    except yt_dlp.utils.DownloadError as e:
        raise DownloadFailedError(str(e)) from e

    formats = info.get("formats") or [info]
    format_id = select_format(formats, max_bytes)
    if format_id is None:
        raise VideoTooLargeError(
            f"No mp4 format under {max_bytes // (1024 * 1024)}MB available for this video."
        )

    output_path = str(Path(output_dir) / f"{uuid.uuid4().hex}.mp4")
    download_opts = {
        "quiet": True,
        "no_warnings": True,
        "format": format_id,
        "outtmpl": output_path,
    }
    try:
        with yt_dlp.YoutubeDL(download_opts) as ydl:
            ydl.download([url])
    except yt_dlp.utils.DownloadError as e:
        raise DownloadFailedError(str(e)) from e

    if not Path(output_path).exists():
        raise DownloadFailedError("Download reported success but the output file is missing.")
    return output_path


async def download_video(url: str, output_dir: str, max_bytes: int) -> str:
    """Download `url` into `output_dir`, picking the best mp4 format under
    max_bytes. Runs yt-dlp (blocking) in a thread so the bot's event loop
    isn't blocked while it downloads."""
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, _download_sync, url, output_dir, max_bytes)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /home/amar/Amar73/telegram-bots/media-downloader
../.venv/bin/python -m pytest tests/test_downloader.py -v
```
Expected: `6 passed`

- [ ] **Step 5: Manual smoke test of the real download path (not unit-tested — needs network + yt-dlp + ffmpeg)**

```bash
cd /home/amar/Amar73/telegram-bots/media-downloader
../.venv/bin/python -c "
import asyncio
from downloader import download_video

async def main():
    path = await download_video('https://www.youtube.com/watch?v=jNQXAC9IVRw', '/tmp', 49 * 1024 * 1024)
    print('Downloaded to:', path)

asyncio.run(main())
"
```
Expected: prints `Downloaded to: /tmp/<uuid>.mp4`, and `ls -la /tmp/<uuid>.mp4` shows a real mp4 file under 49MB (this is "Me at the zoo", the first YouTube video ever uploaded — short, always public, stable test fixture). Delete it after confirming: `rm /tmp/*.mp4` (adjust to the actual printed filename).

- [ ] **Step 6: Commit**

```bash
cd /home/amar/Amar73/telegram-bots
git add media-downloader/downloader.py media-downloader/tests/test_downloader.py
git commit -m "Add yt-dlp format-selection and download wrapper"
```

---

### Task 3: `bot.py` — Telegram wiring

**Files:**
- Create: `/home/amar/Amar73/telegram-bots/media-downloader/bot.py`

**Interfaces:**
- Consumes: `extract_url`, `is_supported_url`, `cleanup_file` (Task 1); `download_video`, `VideoTooLargeError`, `DownloadFailedError` (Task 2).
- Produces: nothing consumed by later tasks — this is the entry point.

No unit tests for this file — it's Telegram-API glue code (handlers, message formatting) with no pure logic of its own; correctness is verified by Task 6's real end-to-end run, which is the only check that actually matters for a bot (mocking `python-telegram-bot`'s `Update`/`Context` objects would test the mock, not the bot).

- [ ] **Step 1: Write `bot.py`**

```python
"""Telegram bot: accepts YouTube/Instagram links, downloads via yt-dlp, sends the video back."""
import logging
import os

from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes, MessageHandler, filters

from downloader import DownloadFailedError, VideoTooLargeError, download_video
from utils import cleanup_file, extract_url, is_supported_url

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
logger = logging.getLogger(__name__)

MAX_FILE_BYTES = 49 * 1024 * 1024
DOWNLOAD_DIR = os.environ.get("DOWNLOAD_DIR", "/tmp/media-downloader")

START_TEXT = (
    "Привет! Пришлите ссылку (или перешлите сообщение со ссылкой) на видео "
    "YouTube или Instagram (Reels/посты/stories) — я скачаю его и пришлю сюда файлом.\n\n"
    "Команды: /start, /help"
)

HELP_TEXT = (
    "Как пользоваться:\n"
    "1. Отправьте ссылку, например:\n"
    "   https://www.youtube.com/watch?v=...\n"
    "   https://youtu.be/...\n"
    "   https://www.instagram.com/reel/...\n"
    "2. Либо перешлите сообщение, где есть такая ссылка.\n"
    "3. Дождитесь статуса «Готово» — видео придёт файлом.\n\n"
    "Ограничение: файлы больше 50 МБ Telegram не примет от бота — "
    "если видео слишком большое, я сообщу об этом честно, а не пришлю обрезанный кусок."
)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(START_TEXT)


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(HELP_TEXT)


async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    text = update.message.text or update.message.caption or ""
    url = extract_url(text)

    if not url:
        await update.message.reply_text(
            "Не нашёл ссылку в сообщении. Пришлите прямую ссылку на YouTube или Instagram."
        )
        return

    if not is_supported_url(url):
        await update.message.reply_text(
            "Эта ссылка не поддерживается. Работаю только с youtube.com, youtu.be и instagram.com."
        )
        return

    status_message = await update.message.reply_text("Скачиваю…")
    file_path = None
    try:
        file_path = await download_video(url, DOWNLOAD_DIR, MAX_FILE_BYTES)
        await status_message.edit_text("Загружаю…")
        with open(file_path, "rb") as video_file:
            await update.message.reply_video(
                video=video_file,
                caption=f"Скачано из: {url}",
                read_timeout=120,
                write_timeout=120,
            )
        await status_message.edit_text("Готово")
    except VideoTooLargeError:
        await status_message.edit_text(
            "Видео слишком большое для отправки ботом (лимит 50 МБ). "
            "Не смог подобрать формат под лимит."
        )
    except DownloadFailedError as e:
        logger.warning("Download failed for %s: %s", url, e)
        await status_message.edit_text(
            "Не удалось скачать видео — ссылка недоступна, видео приватное "
            "или произошла ошибка сети."
        )
    except Exception:
        logger.exception("Unexpected error handling %s", url)
        await status_message.edit_text("Произошла непредвиденная ошибка. Попробуйте ещё раз позже.")
    finally:
        if file_path:
            cleanup_file(file_path)


def main() -> None:
    token = os.environ.get("TELEGRAM_BOT_TOKEN")
    if not token:
        raise RuntimeError("TELEGRAM_BOT_TOKEN environment variable is not set.")

    os.makedirs(DOWNLOAD_DIR, exist_ok=True)

    application = Application.builder().token(token).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    logger.info("Starting media-downloader bot (polling)...")
    application.run_polling()


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Verify it imports and starts cleanly with a dummy token (catches syntax/import errors without needing a real Telegram connection)**

```bash
cd /home/amar/Amar73/telegram-bots/media-downloader
TELEGRAM_BOT_TOKEN=dummy ../.venv/bin/python -c "
import bot
print('bot module imported OK, main() is callable:', callable(bot.main))
"
```
Expected: `bot module imported OK, main() is callable: True` (importing must not raise — this catches typos/missing-import bugs before deploying to the server).

- [ ] **Step 3: Commit**

```bash
cd /home/amar/Amar73/telegram-bots
git add media-downloader/bot.py
git commit -m "Add Telegram bot handlers"
```

---

### Task 4: Packaging (requirements.txt, .env.example, README.md) + push to GitHub

**Files:**
- Create: `/home/amar/Amar73/telegram-bots/media-downloader/requirements.txt`
- Create: `/home/amar/Amar73/telegram-bots/media-downloader/.env.example`
- Create: `/home/amar/Amar73/telegram-bots/README.md`

- [ ] **Step 1: Write `requirements.txt`**

```
python-telegram-bot==21.6
yt-dlp
```

- [ ] **Step 2: Write `.env.example`**

```
# Telegram bot token from @BotFather (https://t.me/BotFather)
TELEGRAM_BOT_TOKEN=123456:ABC-DEF...

# Optional: directory for temporary downloads before they're sent and deleted
DOWNLOAD_DIR=/tmp/media-downloader
```

- [ ] **Step 3: Write `README.md`**

```markdown
# telegram-bots

Personal Telegram bots, one per subdirectory.

## media-downloader

Downloads YouTube (video/shorts) and Instagram (Reels/posts/stories) videos
and sends them back in-chat. Bot: `@amardownloader_bot`.

### Get a bot token

1. Message [@BotFather](https://t.me/BotFather) on Telegram.
2. `/newbot`, follow the prompts.
3. Copy the token it gives you.

### Run locally

```bash
cd media-downloader
python3 -m venv ../.venv
../.venv/bin/pip install -r requirements.txt
cp .env.example .env   # fill in TELEGRAM_BOT_TOKEN
export $(cat .env | xargs)
../.venv/bin/python bot.py
```

Requires `ffmpeg` on PATH (used by `yt-dlp` for remuxing/thumbnails).

### Run as a systemd service

See `media-downloader/media-downloader-bot.service` for a unit file template
(installed on the `hermes` server under user `amar` — not a general
deployment guide, specific to this project's setup).

### Limits

Telegram's cloud Bot API caps file uploads at 50MB. If a video doesn't have
an available format under that size, the bot says so rather than sending a
truncated/corrupted file.
```

- [ ] **Step 4: Create the GitHub repo and push**

```bash
gh repo create Amar73/telegram-bots --private --description "Personal Telegram bots" --confirm
cd /home/amar/Amar73/telegram-bots
git add media-downloader/requirements.txt media-downloader/.env.example README.md
git commit -m "Add packaging files and README"
git branch -M main
git remote add origin https://github.com/Amar73/telegram-bots.git
git push -u origin main
```

- [ ] **Step 5: Verify**

```bash
gh repo view Amar73/telegram-bots --json name,visibility,defaultBranchRef
```
Expected: `"name": "telegram-bots"`, `"visibility": "PRIVATE"`, `defaultBranchRef.name: "main"`.

---

### Task 5: Deploy on the `hermes` server

**Files:** None (server-side only; code already pushed in Task 4).

- [ ] **Step 1: Check/install system dependencies**

```bash
ssh amar@hermes "python3 --version; which ffmpeg || echo MISSING_FFMPEG"
```
If ffmpeg is missing:
```bash
ssh amar@hermes "sudo apt install -y ffmpeg"
```

- [ ] **Step 2: Clone the repo and set up the venv**

```bash
ssh amar@hermes "git clone https://github.com/Amar73/telegram-bots.git ~/Amar73/telegram-bots"
ssh amar@hermes "cd ~/Amar73/telegram-bots && python3 -m venv .venv && .venv/bin/pip install -r media-downloader/requirements.txt"
```

- [ ] **Step 3: Write the `.env` file on the server (token never goes in git)**

```bash
ssh amar@hermes 'cat > ~/Amar73/telegram-bots/media-downloader/.env <<EOF
TELEGRAM_BOT_TOKEN=<AMARDOWNLOADER_BOT_TOKEN>
EOF'
```

- [ ] **Step 4: Create the systemd unit**

```bash
ssh amar@hermes 'sudo tee /etc/systemd/system/media-downloader-bot.service >/dev/null <<EOF
[Unit]
Description=media-downloader Telegram bot
After=network-online.target

[Service]
Type=simple
User=amar
WorkingDirectory=/home/amar/Amar73/telegram-bots/media-downloader
EnvironmentFile=/home/amar/Amar73/telegram-bots/media-downloader/.env
ExecStart=/home/amar/Amar73/telegram-bots/.venv/bin/python bot.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF'
```

- [ ] **Step 5: Start and verify**

```bash
ssh amar@hermes "sudo systemctl daemon-reload && sudo systemctl enable --now media-downloader-bot"
ssh amar@hermes "systemctl is-active media-downloader-bot"
ssh amar@hermes "sudo journalctl -u media-downloader-bot --no-pager -n 20"
```
Expected: `active`, log line `Starting media-downloader bot (polling)...` with no traceback after it.

- [ ] **Step 6: Copy the systemd unit into the repo for documentation, commit**

```bash
ssh amar@hermes "cat /etc/systemd/system/media-downloader-bot.service" > /home/amar/Amar73/telegram-bots/media-downloader/media-downloader-bot.service
```
Edit the copied file to replace the real token line with a placeholder (`TELEGRAM_BOT_TOKEN=<set-in-.env-not-committed>`) if `EnvironmentFile` inlined it — it shouldn't, since the token lives in the separate `.env` referenced by path, not inline in the unit file. Confirm the copied file has no secret in it before committing.
```bash
cd /home/amar/Amar73/telegram-bots
git add media-downloader/media-downloader-bot.service
git commit -m "Document the systemd unit used to run the bot on hermes"
git push
```

---

### Task 6: End-to-end test

**Files:** None — live behavioral test.

- [ ] **Step 1: Send `/start` and `/help` to `@amardownloader_bot`**

Expected: both commands reply with the Russian text from `bot.py`.

- [ ] **Step 2: Send a real YouTube link**

Send `https://www.youtube.com/watch?v=jNQXAC9IVRw` (short, always-public test video).

Expected: "Скачиваю…" → "Загружаю…" → "Готово", then a video file arrives with caption `Скачано из: https://www.youtube.com/watch?v=jNQXAC9IVRw`.

- [ ] **Step 3: Send a real Instagram Reel link**

Send any public Instagram Reel URL.

Expected: same flow. If Instagram extraction fails (yt-dlp's Instagram support is fragile and sometimes needs cookies for content Instagram rate-limits) — this is a known, real risk (see spec) — check `sudo journalctl -u media-downloader-bot -f` for the actual `DownloadFailedError` detail before assuming it's a code bug; note the failure mode here rather than debugging blind if it happens.

- [ ] **Step 4: Send an unsupported link**

Send `https://vimeo.com/12345`.

Expected: bot replies with the "не поддерживается" message, no crash.

- [ ] **Step 5: Confirm temp files are cleaned up**

```bash
ssh amar@hermes "ls /tmp/media-downloader/"
```
Expected: empty (or only files from a run currently in flight) — confirms `cleanup_file` is actually being called after each send, not just present in the code.

**(Executed 2026-07-09/10) All 5 steps passed, but only after finding and fixing two real bugs live against real traffic:**

1. **YouTube: `[Errno 101] Network is unreachable`.** Root-caused via systematic-debugging: the `hermes` server has no global-scope IPv6 (`ip -6 addr show scope global` empty, only a link-local address, no default IPv6 route) — but some googlevideo CDN edge hosts resolve DNS-only-to-IPv6 (confirmed via `getent hosts` on a real failing edge hostname: single AAAA record, no A record at all). Fixed the "unreachable" case by forcing IPv4 (`source_address: "0.0.0.0"` in `downloader.py`, same as yt-dlp's own `--force-ipv4`) — this fixes videos assigned to dual-stack edges. **Known, still-open limitation**: videos assigned to an IPv6-only edge cannot be reached at all from this server regardless of any client-side option, since there's no IPv4 address for that host to connect to. This is consistently per-video (same edge host on repeated requests, not randomized), so retrying doesn't help for an affected video. User has asked their hosting provider whether global IPv6 is available for this VPS — if/when granted, this class of failure goes away entirely; until then, some fraction of YouTube videos will fail with "не удалось скачать видео" for reasons outside this bot's control.
2. **Instagram: every Reel rejected as "too large" regardless of actual size.** Root cause (inspected real format list via yt-dlp's `extract_info` for a real Reel): Instagram's DASH manifests have **no progressive format at all** (video-only WebM-in-mp4 streams + a separate audio-only m4a track, never combined) and **never report `filesize`/`filesize_approx`** on any format. `select_format()`'s progressive-with-known-size requirement (correct and by design for YouTube) always returned `None` for Instagram, and `_download_sync` treated "no format found" as unconditionally "too large" — even for a 14MB Reel, far under the 50MB limit. Fixed by adding a fallback path in `_download_sync`: when `select_format` finds nothing, use yt-dlp's own `bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]/best` selector with `merge_output_format: "mp4"` (ffmpeg muxes the separate streams), then check the actual file size **after** downloading (the only point it's knowable for this kind of source) and reject/delete only then if it's still too big. Verified against 3 real Reels that had all previously failed — one now downloads successfully at 14,962,986 bytes.

Both fixes are in `downloader.py`, deployed and confirmed live (not just unit-tested) against real Telegram traffic from the user. `select_format`'s existing unit tests (Task 2) were unaffected — the bug was entirely in `_download_sync`'s handling of `select_format`'s `None` return, not in `select_format` itself.
