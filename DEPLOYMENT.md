# Render deployment notes

1. Create a **Web Service** from this repository/archive and use the Python runtime.
2. Build command: `pip install -r requirements.txt`
3. Start command: `gunicorn --worker-class gthread --workers 1 --threads 100 --timeout 120 --bind 0.0.0.0:$PORT app:sio`
4. Add environment variables: `SECRET_KEY` (long random secret), `NEXO_ADMIN` (owner username), `NEXO_SECURE_COOKIES=true`, `NEXO_DATA_DIR=/var/data`; add `NEXO_OPENROUTER_KEY` only if AI is enabled.
5. Attach a persistent disk mounted at `/var/data`. Without a persistent disk, SQLite database, uploads, and backups can be lost on redeploy/restart.
6. After deployment, open `/api/infinity/health` and confirm the version response.

## Database caveat

The supplied legacy app is SQLite-specific (`sqlite3`, `?` placeholders and SQLite SQL). This release preserves it and adds SQLite tables. It is **not yet PostgreSQL-compatible**. Do not point `DATABASE_URL` at PostgreSQL and assume it works. A safe PostgreSQL migration needs a dedicated data migration and a shared database adapter across all legacy routes.

## Bot API

- Create: `POST /api/bots` (authenticated NEXO session, CSRF header required), JSON `{ "name":"Helper Bot", "username":"helperbot" }`. The token is returned once.
- Token header: `X-NEXO-BOT-TOKEN: <token>` or use `/bot<TOKEN>/<method>`.
- Methods: `getMe`, `getUpdates`, `sendMessage`, `getChat`, `getUser`.
- Polling: `POST /bot<TOKEN>/getUpdates` with `{ "offset":0, "limit":20, "timeout":10 }`.
- Users must start the bot first: authenticated `POST /api/bots/<bot_id>/start`.
- Bot replies: `POST /bot<TOKEN>/sendMessage` with `{ "chat_id": <NEXO user id>, "text":"Hello" }`. Recipient reads messages from authenticated `GET /api/bot-inbox`.
- Webhook registration: authenticated `POST /api/bots/<bot_id>/webhook` with `{ "url":"https://example.com/hook" }`. This stores a validated HTTPS target and returns a one-time secret, but outbound delivery is deliberately not enabled until a separate retrying worker with SSRF protections is deployed.

## Market behavior

- Product prices are entered in Toman; checkout converts at 1 Star per 1,000 Toman, rounded up.
- Stars are charged atomically with stock reservation.
- Send an `Idempotency-Key` header (12–100 characters) to avoid duplicate checkout on retries.
- Seller marks paid order `shipped`; buyer marks shipped order `completed`.
- Seller internal wallet is credited only after buyer completion. Cancelled paid orders refund Stars and restore stock.
- Seller wallet is internal-only. There is no cash-out or bank-card transfer API.

## Internal Bot Studio runtime

Bot Studio Run/Stop runs through `nexo_internal_runner.py` and a lightweight worker registered by `infinity_ext.py`. It runs on the same NEXO web service and uses the existing SQLite database; no external runner URL or runner token is needed. Keep **one Gunicorn worker process** (the supplied Procfile does this), because the current queue worker is in-process.

Python callbacks may define `on_start(bot)` and `on_message(message, bot)`. `bot.send_message(chat_id, text)` queues a reply for users who opted in to that bot. Use **آزمایش /start**, then send a test message in Bot Studio and refresh execution logs to see the reply. The message API is `POST /api/bots/<bot_id>/message`.

Each callback runs in a short-lived child process with a 2.5-second timeout, OS resource limits (when `prlimit` is available), restricted built-ins/imports, and no exposed filesystem or network API. JavaScript source is save-only; Python is the currently executable language.

**Important security limitation:** this is a constrained starter runtime, not a hardened hostile-code sandbox. The runtime still shares the web service's host and resource pool. Do not open Bot Studio execution to untrusted public users; keep it limited to trusted accounts until a dedicated container/VM sandbox is available.
