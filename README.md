# QR generation bot for Telegram messenger

Telegram bot built on Python that allows users to generate QR codes with multiple design options.
This bot was developed for a semestral project at BUT (Brno University of Technology) as part of my programming course.

## Setup

```bash
git clone https://github.com/mxrkiz/QR-generation-bot-for-Telegram-messenger
cd QR-generation-bot-for-Telegram-messenger
pip install -r requirements.txt
```

Create a `.env` file in the project root with your bot token:

```
TELEGRAM_TOKEN=your_token_here
```

```bash
python main.py
```

## Required libraries

```
python-telegram-bot  # Foundational framework
qrcode               # QR code generating engine
Pillow               # Image processing and logo embedding
python-dotenv        # Loading token from .env
```

## Features

**Custom Color Selection:** Users can select from a predefined color palette.

TO-DO: image

```python
### handlers.py

async def get_color(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Validates color input, stores it, and asks for the logo."""
    color = update.message.text.lower()

    if color not in VALID_COLORS:
        await update.message.reply_text(
            f"Invalid color choice: '{color.title()}'. Please choose one of the available options."
        )
        return GET_COLOR

    context.user_data['color'] = color
    logger.info(f"User {update.effective_user.id} set color to: {color}")

    return await prompt_for_logo(update, context)
```

**Logo Embedding:** Supports uploading a custom image to be placed in the center of the QR code.

TO-DO: image

```python
### handlers.py

def create_custom_qr(data: str, fill_color: str = 'black', logo_bytes: bytes | None = None):
    qr = qrcode.QRCode(
        error_correction=qrcode.constants.ERROR_CORRECT_H,
        box_size=10,
        border=3,
    )
    qr.add_data(data)
    qr.make(fit=True)

    img = qr.make_image(fill_color=hex_code, back_color="white").convert("RGB")

    if logo_bytes:
        try:
            logo_pil = Image.open(io.BytesIO(logo_bytes)).convert("RGBA")
            img_size = img.size[0]
            logo_max_size = img_size // 4
            logo_pil.thumbnail((logo_max_size, logo_max_size))
            logo_pos = ((img_size - logo_pil.width) // 2, (img_size - logo_pil.height) // 2)
            img.paste(logo_pil, logo_pos, mask=logo_pil)
        except Exception as e:
            logger.error(f"Failed to add logo to QR code: {e}")
```

**Skip Button on Logo Step:** Users can skip the logo upload using either the inline "Skip →" button or the `/skip` command.

```python
### config.py

SKIP_LOGO_DATA = "SKIP_LOGO"
LOGO_STEP_KEYBOARD = InlineKeyboardMarkup([[
    InlineKeyboardButton("⬅️ Go Back", callback_data=INLINE_BACK_DATA),
    InlineKeyboardButton("Skip →", callback_data=SKIP_LOGO_DATA),
]])
```

**Robust Navigation:** Uses inline buttons for "Go Back" and "Skip →" functionality and clean flow management.

TO-DO: image

```python
### handlers.py

async def go_back(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Handles the 'Go Back' callback query."""

    query = update.callback_query
    await query.answer()

    current_state = context.user_data.get('current_state')

    if current_state == GET_LOGO:
        context.user_data.pop('color', None)
        context.user_data['current_state'] = GET_COLOR

        await query.edit_message_text("You went back. Choose a color from the panel below.", reply_markup=None)
        await query.message.reply_text("Color menu:", reply_markup=COLOR_REPLY_MARKUP)
        return GET_COLOR

    elif current_state == GET_TEXT:
        context.user_data.pop('logo', None)
        context.user_data['current_state'] = GET_LOGO

        await query.edit_message_text(
            f"You went back. Color is still: {context.user_data.get('color', 'black').title()}\n\n"
            "Now, send me an image for the middle logo (optional), or send /skip.",
            reply_markup=LOGO_STEP_KEYBOARD
        )
        return GET_LOGO
```

**"Start New QR" from text:** Typing "Start New QR" in chat restarts the flow, in addition to pressing the inline button.

**Error Handling:** Manages invalid input, multiple photo uploads, and conversation state errors.

TO-DO: image

```python
# filters.py

class ImageDocumentFilter(filters.BaseFilter):
    """Custom filter to match documents that are JPEG or PNG images."""
    def __call__(self, update: Update) -> bool:
        if not update.message or not update.message.document:
            return False
        return update.message.document.mime_type in ("image/jpeg", "image/png")

IMAGE_FILE_FILTER = filters.PHOTO | ImageDocumentFilter()
```
