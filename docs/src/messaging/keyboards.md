# Inline Keyboards & Reply Markup

`ferogram` ships with high-level keyboard builders: `InlineKeyboard` and `ReplyKeyboard`, so you never have to construct raw TL types by hand.

As of Telegram layer 229, inline-keyboard buttons and reply-keyboard buttons
are two separate TL type trees (they used to share one `KeyboardButton`
schema). `ferogram` mirrors that split: `Button` builds inline-keyboard
buttons, `ReplyButton` builds reply-keyboard buttons. They are not
interchangeable - `InlineKeyboard::row()` takes `Button`, `ReplyKeyboard::row()`
takes `ReplyButton`.

Both builders are in `ferogram::keyboard` and re-exported at the crate root:

```rust
use ferogram::keyboard::{Button, InlineKeyboard, ReplyButton, ReplyKeyboard};
```

---

## `InlineKeyboard`: buttons attached to a message

Inline keyboards appear below a message and trigger `Update::CallbackQuery` when tapped.

```rust
use ferogram::keyboard::{Button, InlineKeyboard};
use ferogram::InputMessage;

let kb = InlineKeyboard::new()
    .row([
        Button::callback("✅ Yes", b"confirm:yes"),
        Button::callback("❌ No",  b"confirm:no"),
    ])
    .row([
        Button::url("📖 Docs", "https://docs.rs/ferogram"),
    ]);

client.send_message(peer.clone(), InputMessage::text("Do you want to proceed?").keyboard(kb)).await?;
```

### `InlineKeyboard` methods

| Method | Description |
|---|---|
| `InlineKeyboard::new()` | Create an empty keyboard |
| `.row(buttons)` | Append a row; accepts any `IntoIterator<Item = Button>` |
| `.into_markup()` | Convert to `tl::enums::ReplyMarkup` |

`InlineKeyboard` implements `Into<tl::enums::ReplyMarkup>`, so you can also pass it directly to `InputMessage::reply_markup()`.

---

## `Button`: inline-keyboard button types

```rust
// Sends data to your bot as Update::CallbackQuery (max 64 bytes)
Button::callback("✅ Confirm", b"action:confirm")

// Opens URL in a browser
Button::url("🌐 Website", "https://example.com")

// Copy text to clipboard (Telegram 10.3+)
Button::copy_text("📋 Copy code", "PROMO2024")

// Login-widget: authenticates user before opening URL
Button::url_auth("🔐 Login", "https://example.com/auth", None, bot_input_user)

// Opens bot inline mode in the current chat with query pre-filled
Button::switch_inline("🔍 Search here", "default query")

// Chat picker so user can choose which chat to use inline mode in
Button::switch_elsewhere("📤 Share", "")

// Telegram Mini App with full JS bridge
Button::mini_app("🚀 Open App", "https://myapp.example.com")

// Launch a Telegram game (bots only)
Button::game("🎮 Play")

// Payment buy button (bots only, used with invoice)
Button::buy("💳 Pay $4.99")
```

There is no inline "simple webview" (no-JS-bridge) button as of layer 229 -
Telegram moved that variant to reply keyboards only. See
`ReplyButton::mini_app_simple` below.

### Escape hatch

```rust
// Get the underlying tl::enums::KeyboardInlineButton
let raw = Button::callback("x", b"x").into_raw();
```

---

## `ReplyButton`: reply-keyboard button types

```rust
use ferogram::keyboard::ReplyButton;

// Plain text
ReplyButton::text("📸 Send photo")

// Shares user's phone number on tap
ReplyButton::request_phone("📞 Share my number")

// Shares user's location on tap
ReplyButton::request_geo("📍 Share location")

// Opens poll creation interface
ReplyButton::request_poll("📊 Create poll")

// Forces quiz mode in poll creator
ReplyButton::request_quiz("🧠 Create quiz")

// Simple webview without JS bridge (reply-keyboard only, layer 229+)
ReplyButton::mini_app_simple("ℹ️ Info", "https://info.example.com")
```

### Escape hatch

```rust
// Get the underlying tl::enums::KeyboardButton
let raw = ReplyButton::text("x").into_raw();
```

---

## `ReplyKeyboard`: replacement keyboard

A reply keyboard replaces the user's text input keyboard until dismissed.
The user's tap arrives as a plain-text `Update::NewMessage`.

```rust
use ferogram::keyboard::{ReplyButton, ReplyKeyboard};

let kb = ReplyKeyboard::new()
    .row([
        ReplyButton::text("📸 Photo"),
        ReplyButton::text("📄 Document"),
    ])
    .row([ReplyButton::text("❌ Cancel")])
    .resize()      // shrink to fit content (recommended)
    .single_use(); // hide after one press

client.send_message(peer.clone(), InputMessage::text("Choose file type:").keyboard(kb))
    .await?;
```

### `ReplyKeyboard` methods

| Method | Description |
|---|---|
| `ReplyKeyboard::new()` | Create an empty keyboard |
| `.row(buttons)` | Append a row of buttons |
| `.resize()` | Shrink keyboard to fit button count |
| `.single_use()` | Dismiss after one tap |
| `.selective()` | Show only to mentioned/replied users |
| `.into_markup()` | Convert to `tl::enums::ReplyMarkup` |

---

## Remove keyboard

```rust
use ferogram_tl_types as tl;

let remove = tl::enums::ReplyMarkup::ReplyKeyboardHide(
    tl::types::ReplyKeyboardHide { selective: false }
);
client
    .send_message(peer.clone(), InputMessage::text("Done.").reply_markup(remove))
    .await?;
```

---

## Answer callback queries

Always answer every `CallbackQuery`: Telegram shows a loading spinner until you do.

```rust
Update::CallbackQuery(cb) => {
    let data = cb.data().unwrap_or(b"");
    match data {
        b"confirm:yes" => client.answer_callback_query(cb.query_id, Some("✅ Done!"), false).await?,
        b"confirm:no"  => client.answer_callback_query(cb.query_id, Some("❌ Cancelled"), false).await?,
        _              => client.answer_callback_query(cb.query_id, None, false).await?,
    }
}
```

Pass `alert: true` to show a popup alert instead of a toast:

```rust
client.answer_callback_query(cb.query_id, Some("⛔ Access denied"), true).await?;
```

---

## Legacy raw TL pattern (still works)

If you prefer constructing TL types directly. Note the two-level shape as of
layer 229: the button's `text` lives on the outer `KeyboardInlineButton`,
while type-specific data (like callback `data`) lives on the inner
`InlineButtonType` variant.

```rust
use ferogram_tl_types as tl;

fn inline_kb(rows: Vec<Vec<tl::enums::KeyboardInlineButton>>) -> tl::enums::ReplyMarkup {
    tl::enums::ReplyMarkup::ReplyInlineMarkup(tl::types::ReplyInlineMarkup {
        force_reply: false,
        rows: rows.into_iter().map(|buttons|
            tl::enums::KeyboardInlineButtonRow::KeyboardInlineButtonRow(
                tl::types::KeyboardInlineButtonRow { buttons }
            )
        ).collect(),
    })
}

fn btn_cb(text: &str, data: &str) -> tl::enums::KeyboardInlineButton {
    tl::enums::KeyboardInlineButton::KeyboardInlineButton(tl::types::KeyboardInlineButton {
        style: None,
        text:  text.into(),
        r#type: tl::enums::InlineButtonType::Callback(tl::types::InlineButtonTypeCallback {
            requires_password: false,
            data: data.as_bytes().to_vec(),
        }),
    })
}

fn btn_url(text: &str, url: &str) -> tl::enums::KeyboardInlineButton {
    tl::enums::KeyboardInlineButton::KeyboardInlineButton(tl::types::KeyboardInlineButton {
        style: None,
        text:  text.into(),
        r#type: tl::enums::InlineButtonType::Url(tl::types::InlineButtonTypeUrl {
            url: url.into(),
        }),
    })
}
```

## Send with keyboard

```rust
let kb = inline_kb(vec![
    vec![btn_cb("✅ Yes", "confirm:yes"), btn_cb("❌ No", "confirm:no")],
    vec![btn_url("🌐 Docs", "https://github.com/ankit-chaubey/ferogram")],
]);

let (text, entities) = parse_markdown("**Do you want to proceed?**");
let msg = InputMessage::text(text)
    .entities(entities)
    .reply_markup(kb);

client.send_message(peer, msg).await?;
```

## All button types (raw TL, layer 229)

### Inline keyboard (`InlineButtonType`, wrapped in `KeyboardInlineButton`)

| Type | Constructor | Description |
|---|---|---|
| Callback | `InlineButtonTypeCallback` | Triggers `CallbackQuery` with custom data |
| URL | `InlineButtonTypeUrl` | Opens a URL in the browser |
| URL Auth | `InputInlineButtonTypeUrlAuth` | Login-widget style authenticated URL |
| WebView | `InlineButtonTypeWebView` | Opens a Telegram Mini App (JS bridge) |
| Switch Inline | `InlineButtonTypeSwitchInline` | Opens inline mode with a query |
| User Profile | `InputInlineButtonTypeUserProfile` | Opens a user's profile |
| Game | `InlineButtonTypeGame` | Opens a Telegram game |
| Buy | `InlineButtonTypeBuy` | Purchase button for payments |
| Copy | `InlineButtonTypeCopy` | Copies text to clipboard |
| Disabled | `InlineButtonTypeDisabled` | Renders greyed out, not tappable |

### Reply keyboard (`ButtonType`, wrapped in `KeyboardButton`)

| Type | Constructor | Description |
|---|---|---|
| Text | `ButtonTypeDefault` | Plain text button |
| Request Phone | `ButtonTypeRequestPhone` | Requests the user's phone number |
| Request Location | `ButtonTypeRequestGeoLocation` | Requests location |
| Request Poll | `ButtonTypeRequestPoll` | Opens poll creator |
| Request Peer | `InputButtonTypeRequestPeer` | Requests peer selection |
| Simple WebView | `ButtonTypeSimpleWebView` | Opens a webview without a JS bridge |

### Switch Inline button (raw)

Opens the bot's inline mode in the current or another chat:

```rust
tl::enums::KeyboardInlineButton::KeyboardInlineButton(tl::types::KeyboardInlineButton {
    style: None,
    text:  "🔍 Search with me".into(),
    r#type: tl::enums::InlineButtonType::SwitchInline(tl::types::InlineButtonTypeSwitchInline {
        same_peer:  false, // false = let user pick any chat
        query:      "default query".into(),
        peer_types: None,
    }),
})
```

### Simple WebView button (raw, reply-keyboard only)

```rust
tl::enums::KeyboardButton::KeyboardButton(tl::types::KeyboardButton {
    style: None,
    text:  "Open App".into(),
    r#type: tl::enums::ButtonType::SimpleWebView(tl::types::ButtonTypeSimpleWebView {
        url: "https://myapp.example.com".into(),
    }),
})
```

## Reply keyboard (replaces user's keyboard)

```rust
let reply_kb = tl::enums::ReplyMarkup::ReplyKeyboardMarkup(
    tl::types::ReplyKeyboardMarkup {
        resize:      true,       // shrink to fit buttons
        single_use:  true,       // hide after one tap
        selective:   false,      // show to everyone
        persistent:  false,      // don't keep after message
        force_reply: false,      // layer 229; not covered elsewhere in this doc
        placeholder: Some("Choose an option…".into()),
        rows: vec![
            tl::enums::KeyboardButtonRow::KeyboardButtonRow(
                tl::types::KeyboardButtonRow {
                    buttons: vec![
                        tl::enums::KeyboardButton::KeyboardButton(tl::types::KeyboardButton {
                            style: None,
                            text: "🍕 Pizza".into(),
                            r#type: tl::enums::ButtonType::Default(tl::types::ButtonTypeDefault {}),
                        }),
                        tl::enums::KeyboardButton::KeyboardButton(tl::types::KeyboardButton {
                            style: None,
                            text: "🍔 Burger".into(),
                            r#type: tl::enums::ButtonType::Default(tl::types::ButtonTypeDefault {}),
                        }),
                    ]
                }
            ),
            tl::enums::KeyboardButtonRow::KeyboardButtonRow(
                tl::types::KeyboardButtonRow {
                    buttons: vec![
                        tl::enums::KeyboardButton::KeyboardButton(tl::types::KeyboardButton {
                            style: None,
                            text: "❌ Cancel".into(),
                            r#type: tl::enums::ButtonType::Default(tl::types::ButtonTypeDefault {}),
                        }),
                    ]
                }
            ),
        ],
    }
);
```

The user's choices arrive as plain text `NewMessage` updates.

## Remove keyboard

```rust
let remove = tl::enums::ReplyMarkup::ReplyKeyboardHide(
    tl::types::ReplyKeyboardHide { selective: false }
);
let msg = InputMessage::text("Keyboard removed.").reply_markup(remove);
```

## Button data format

Telegram limits callback button data to **64 bytes**. Use compact, parseable formats:

```rust
// Good: structured, compact
"vote:yes"
"page:3"
"item:42:delete"
"menu:settings:notifications"

// Bad: verbose
"user_clicked_the_settings_button"
```
