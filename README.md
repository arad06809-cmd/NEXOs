# NEXO INFINITY — Foundation release

This release is an **additive upgrade** of the supplied `NEXO-Release-Features(1).py` source. Existing Flask pages, authentication, messaging, groups, Stars, AI, missions, badges and admin routes are preserved in `app.py`. The new `infinity_ext.py` registers additional backend modules without replacing the legacy application.

## Included in this delivery

- Existing NEXO application retained as the starting point.
- Telegram-style private BotFather conversation inside the chats list, with `/start`, `/help`, `/newbot`, `/mybots`, and `/cancel`.
- Bot management through the BotFather chat: create bots in two steps, view bot list and statistics, rename, rotate token, enable/disable, and delete.
- NEXO Bot API 1.0 core: `getMe`, `getUpdates` (long polling), `sendMessage`, `getChat`, `getUser`.
- User opt-in start flow and bot inbox endpoint.
- HTTPS webhook registration metadata. Actual outbound webhook delivery requires a separate worker; see `DEPLOYMENT.md`.
- NEXO Market backend foundation: seller stores, product CRUD, search/category listing, Stars-based checkout, idempotent order creation, seller shipping, buyer completion, cancellation/refund, internal seller wallet ledger, ratings.
- Seller wallet has no withdrawal or bank-card transfer route.
- Additive database schema and health endpoint.

## Not represented as complete

The requested Infinity scope is much larger than this foundation release. Voice/video calling, WebRTC signaling/TURN, Stories, livestreaming, disappearing/scheduled messages, full market storefront UI, outbound webhook delivery, PostgreSQL migration, and the complete redesigned frontend are not represented as finished. The Bot API's first methods are real endpoints; unsupported methods must not be treated as implemented.

## Local run

```bash
python -m venv .venv
# Windows: .venv\\Scripts\\activate
# Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
python app.py
```

Set `NEXO_ADMIN` before first registration to assign the owner role to the matching username. Set `NEXO_DATA_DIR` to a persistent directory. Existing databases are upgraded additively when the app starts.

## Test

```bash
python -m unittest discover -s . -p 'test*.py' -v
```

## Bot developer resources

- [Bot API 1.0 method reference](docs/BOT_API_1.0.md)
- [Implementation ledger and remaining work](docs/ROADMAP.md)
- Python SDK: `sdk/python/nexo_bot.py` (requires `requests`)
- JavaScript SDK: `sdk/javascript/nexo-bot.js` (Node 18+)


## NEXO Bot Studio

Each bot has its own Python/JavaScript code workspace inside its BotFather management chat. Each project is limited to **100 lines**, checked in both the editor and backend. The Run/Stop/logs actions use the **internal NEXO runtime**; no external runner URL or runner token is required. The app starts a lightweight worker in the same web process, stores project state and logs in the existing SQLite database, and processes queued bot updates.

The current internal runtime executes **Python only**. JavaScript source can be saved but is not executable yet. Python callbacks may define `on_start(bot)` and `on_message(message, bot)`. `bot.send_message(chat_id, text)` queues a reply for users who have opted in to that bot. Use the **آزمایش /start** button to enqueue a test start event, then send a test message in the Bot Studio panel and refresh execution logs to inspect the reply. Each callback runs in a short-lived child process with a 2.5-second timeout, memory/CPU/file limits, restricted built-ins and a small standard-library import allowlist. File access and network access are not provided to bot code.

**Security limitation:** this is a constrained starter runtime, not a hardened multi-tenant sandbox or a substitute for a dedicated container/VM isolation layer. Keep access to Bot Studio limited to trusted accounts until a stronger OS-level sandbox is deployed. The internal worker requires one Gunicorn worker process (the supplied Procfile uses one); do not increase the worker count without adding a cross-worker queue/lease mechanism.


## NEXO X patch: links, bot conversations, and search
- Web URLs in chat message text are rendered as safe clickable links (`http`, `https`, and `www`).
- Creating a BotFather bot also creates a persistent NEXO bot conversation in the creator's chat list.
- Existing bots are backfilled with a conversation during extension initialization when their username is available.
- Public bot conversations appear in global search; numeric user/chat IDs can be searched exactly.
- This patch creates the bot conversation record; automated bot replies still require the bot runtime to be running and connected.


## NEXO X reliability patch (2026-11)
- Fixed the message linkifier's invalid JavaScript regular expression, which could stop the entire frontend from booting and leave the app on a black screen.
- `@username` mentions in messages are now clickable and resolve to public channels/groups/bots or user profiles.
- Public-chat search now prioritizes exact username/title matches and returns a larger result set, including public channels and bots.
- Revalidated the frontend JavaScript syntax and ran the Python unit tests.


## NEXO Motion & Creator Studio (2026-11)
- Added one-shot entrance animation for single-emoji messages, with expressive tear accents for 😂, 🤣, 🥹 and 😭. The animation is played on new messages rather than looping continuously.
- Added animated reaction feedback and expanded the free reaction catalog; premium reactions remain gated.
- Added six original NEXO vector sticker designs (joy, love, fire, cry, star, party), sent as first-class sticker messages.
- Added Premium Emoji Studio at `/emoji-studio`: circular photo-to-emoji conversion, safe SVG authoring with optional SMIL animation, preview, owner-scoped library, and delete.
- Connected saved custom emojis to the in-chat picker (`ایموجی‌های من`), first-class message sending, and authenticated asset rendering for chat participants.
- Added the separate `/bot-studio` page with Monaco Editor, Python syntax highlighting, bot explorer, save/run/stop, `/start` test, and logs, wired to the existing NEXO bot project/runtime APIs.
- Bot Studio's in-browser Monaco assets are loaded from cdnjs and require browser internet access. The current isolated bot runtime executes Python only; the UI exposes Python only to avoid offering a non-runnable language.
- Photo conversion makes a circular, outlined emoji portrait and applies an entrance animation when sent. It does not infer facial expressions or synthesize a frame-by-frame character animation from a still photo.
- Existing app database is extended additively with `nx_custom_emoji`; no existing message tables are replaced.


## توسعه گروه‌ها و کانال‌ها (Telegram / All)

منطق سمت سرور گروه‌ها و کانال‌ها در `channels_ext.py` قرار دارد و از `app.py` ثبت می‌شود. راهنمای مسیرها و نمونه تنظیمات در `GROUPS_CHANNELS_API.md` است. برای این نسخه، پنل اختصاصی کنترل این قابلیت‌ها در رابط کاربری هنوز باید به APIها متصل شود؛ زمان‌بندی پست نیز به worker پایدار نیاز دارد و در این پچ فعال نشده است.
