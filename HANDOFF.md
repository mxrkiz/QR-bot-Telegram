# QR Bot — Handoff

## Context
Telegram bot for generating custom QR codes. Multi-step ConversationHandler: color → logo (optional) → text → generate + send.

Repo: https://github.com/mxrkiz/QR-generation-bot-for-Telegram-messenger

## Fixes needed (priority order)

### 1. `handle_new_start` — antipattern (HIGH)
**File:** `handlers.py`, function `handle_new_start`

Currently sends the literal text `"/start"` as a message to the chat — this is a hack.
Replace with a direct call to `start_custom_qr`:

```python
async def handle_new_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await query.edit_message_text("Resetting flow...", reply_markup=None)
    return await start_custom_qr(update, context)
```

Also remove the now-unused persistent handler in `main.py`:
```python
# DELETE this block:
application.add_handler(
    CallbackQueryHandler(handle_new_start, pattern=f"^{re.escape(START_QR_CALLBACK_DATA)}$")
)
```
Wire `START_QR_CALLBACK_DATA` into ConversationHandler entry_points instead.

---

### 2. `box_size=100` generates oversized images (HIGH)
**File:** `handlers.py`, function `create_custom_qr`

`box_size=100` produces ~3000×3000 px images. Change to `box_size=10`:

```python
qr = qrcode.QRCode(
    error_correction=qrcode.constants.ERROR_CORRECT_H,
    box_size=10,  # was 100
    border=3,
)
```

---

### 3. `go_back` duplicates the color keyboard (MEDIUM)
**File:** `handlers.py`, function `go_back`, branch `if current_state == GET_LOGO`

Keyboard is rebuilt manually instead of using `COLOR_REPLY_MARKUP` from `config.py`.
Replace local construction with:

```python
await query.message.reply_text("Color menu:", reply_markup=COLOR_REPLY_MARKUP)
```

---

### 4. Pin versions in `requirements.txt` (MEDIUM)
No versions pinned — breaks on fresh installs (PTB v13 vs v20 are incompatible).
Run `pip freeze` in the venv, lock exact versions. Also remove `numpy` — unused in the codebase.

---

### 5. Add `.gitignore` (LOW)
```
__pycache__/
*.pyc
.env
```

---

### 6. Add run instructions to `README.md` (LOW)
Add a Setup section:
```
git clone <repo>
cd QR-generation-bot-for-Telegram-messenger
pip install -r requirements.txt
# Set TELEGRAM_TOKEN in config.py or via .env
python main.py
```

---

## Do not change
- Module structure — clean separation is correct
- `ImageDocumentFilter` — correct implementation
- `ERROR_CORRECT_H` — required for logo embedding, do not downgrade
- `TELEGRAM_TOKEN = "BOT_TOKEN"` placeholder — token is not leaked
- README screenshots
