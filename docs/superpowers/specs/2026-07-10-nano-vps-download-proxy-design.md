# Design: nano-VPS as a download egress proxy for the media-downloader bot

**Date:** 2026-07-10
**Status:** Approved by user (brainstorming session 2026-07-10)

## Problem

The `hermes` server (193.228.139.46) has no global IPv6, and the hosting provider has **refused** to allocate any. Some YouTube/googlevideo CDN edge hosts are IPv6-only in DNS (single AAAA record, no A record), so the videos assigned to those edges cannot be downloaded from `hermes` at all — the `source_address: "0.0.0.0"` (force-IPv4) workaround in `media-downloader/downloader.py` only helps for dual-stack edges. Retrying does not help: edge assignment is stable per video.

A full migration of the whole Paperclip/Hermes/MARKO Lab stack to a new server was considered and rejected: the user's only motivation is IPv6 (confirmed — no complaints about the provider, price, or geography), and migration is the most expensive way to obtain it, with no guarantee the new datacenter IP behaves better with YouTube.

## Decision

Keep `hermes` untouched. Use the already-provisioned second VPS as a **download-only egress proxy**:

- **nano-VPS**: `ssh nano-vps`, IPv4 `31.76.43.20`, IPv6 `2a01:ecc0:6000:a7d::2/64` (default route present), Helsinki FI, AS210546 (CHSL ONE — same operator family as `hermes`, different network), Debian 13 trixie, 1 CPU / 2GB RAM / 32GB disk, user `amar` with sudo.
- **Verified 2026-07-10 before this design was accepted**: outbound IPv6 works end-to-end (`curl -6` → HTTP/2 200 from Google), and — the decisive test — `yt-dlp -6` on this box both extracted metadata **and downloaded a real video file** from the googlevideo CDN over forced IPv6, with no bot-check/captcha wall.

## Architecture

```
Telegram user ──► @amardownloader_bot (hermes, unchanged service)
                          │ yt-dlp with proxy = socks5h://mdbot:***@31.76.43.20:12050
                          ▼
                  nano-VPS (Helsinki, dual-stack)
                     microsocks (SOCKS5, auth, ufw-allowlisted to hermes IP only)
                          │ native IPv4 + IPv6
                          ▼
                  YouTube / Instagram / TikTok CDNs
```

Two components change; everything else stays as-is.

### Component 1: nano-VPS setup (server-side only, no code)

1. **Hardening**, mirroring the light version of what `hermes` got:
   - Verify SSH is key-only (`PasswordAuthentication no`, `PermitRootLogin no`).
   - `ufw`: default deny incoming; port 22 allowed only from the user's two static IPs (home `46.34.141.146`, work `144.206.228.59`); the SOCKS port allowed **only from `hermes`'s IP `193.228.139.46`**. Nothing else open.
   - `fail2ban` for sshd with `backend = systemd` from the start (this minimal Debian image family has no rsyslog/auth.log — same issue already hit and solved on `hermes`).
2. **Proxy**: the Debian `microsocks` package (verified available: 1.0.5-1 in trixie), run as a systemd service with username/password auth (user `mdbot`, password generated at setup time and stored only in the two `.env`/service files), listening on the public interface on port **12050**. Defense in depth: ufw source-IP allowlist (primary) + SOCKS auth (secondary). No WireGuard — the proxied payload is HTTPS anyway; a tunnel can be added later if the whole server ever needs IPv6, but it is out of scope here (YAGNI).

### Component 2: bot change (`Amar73/telegram-bots`, `media-downloader/downloader.py`)

- New **optional** env var `PROXY_URL` (in `.env` on `hermes`, documented in `.env.example`), value `socks5h://mdbot:<generated password>@31.76.43.20:12050`.
  - `socks5h` (proxy-side DNS resolution) is **required**, not a style choice: IPv6-only hostnames must be resolved by the proxy, since `hermes` can resolve the AAAA record but cannot connect to it, and a locally-resolved address would be useless.
- Behavior in `downloader.py`:
  - `PROXY_URL` set → pass it as yt-dlp's `proxy` option for **all** downloads (YouTube, Instagram, TikTok — no per-site routing, one consistent egress) and **omit** `source_address: "0.0.0.0"` (force-IPv4 is pointless and misleading when the proxy does the connecting).
  - `PROXY_URL` unset/empty → behavior identical to today (force-IPv4 direct). This is the rollback path: remove one line from `.env`, restart the service.
- No other modules change. `bot.py`, `utils.py`, `select_format` logic, size limits — untouched.

## Verification

1. Unit: the `PROXY_URL`-handling logic in `downloader.py` (set → `proxy` present and `source_address` absent in ydl opts; unset → current behavior) — pure config assembly, unit-testable like `select_format`.
2. Live, via the real bot on Telegram, with the proxy enabled:
   - 1–2 YouTube links that previously failed with «не удалось скачать видео» due to IPv6-only edges (**to be supplied by the user from the bot chat history**) — must now succeed.
   - One Instagram Reel and one TikTok link — must still work through the proxy (regression check).
   - `sudo journalctl -u media-downloader-bot` on `hermes` shows no proxy auth/connect errors.

## Known costs and risks (accepted)

- Video traffic transits the nano-VPS twice (in + out). Check the tariff's traffic allowance; for a personal single-user bot this is expected to be far below any cap.
- SOCKS credentials travel unencrypted in the SOCKS handshake between `hermes` and the nano-VPS; accepted because the port is IP-allowlisted to `hermes` and the payload itself is HTTPS. (WireGuard would remove this; deliberately not done now.)
- If the Helsinki IP's reputation with YouTube degrades over time, the fix is swapping/replacing only the nano-VPS and updating one `.env` line — the main stack never depends on it.
- The nano-VPS is a second box to keep patched; its attack surface is one SOCKS port (IP-allowlisted) and SSH (IP-allowlisted).

## Out of scope

- Full migration of the Paperclip/Hermes stack (rejected — see Problem).
- WireGuard / whole-server IPv6 for `hermes`.
- Free tunnel options (HE tunnelbroker, Cloudflare WARP) — rejected: Google is known to captcha/block those ranges, which would worsen the exact problem being solved.
- Local Bot API server, per-site proxy routing, any change to the bot's feature set.
