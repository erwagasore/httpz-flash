# httpz-flash — Design Document

Flash middleware for [httpz](https://github.com/karlseguin/http.zig).

## Overview

Flash messages are short-lived, one-time notifications passed between HTTP
requests. A handler sets a message ("Item saved"), redirects, and the next
request reads it and it disappears.

## Delivery patterns

| Pattern | How it works |
|---------|--------------|
| **Full-page redirect** | Cookie set on redirect, read on next GET, expired after read |
| **HTMX boosted link/form** | `hx-boost` follows redirects transparently — cookie works as-is |
| **HTMX `HX-Redirect`** | Server sends header, browser does full navigation — cookie works |
| **HTMX partial swap** | No redirect, no page load. Needs `HX-Trigger` header instead of cookie |

The middleware handles cookie lifecycle. A small helper handles the
HTMX partial swap case.

## Architecture

```
                          ┌──────────────────────────────┐
                          │        Flash middleware       │
  Request ───────────────▶│  read cookie → arena entries  │
                          └──────────┬───────────────────┘
                                     │
                          ┌──────────▼───────────────────┐
                          │          Handler              │
                          │                               │
                          │  flash.get(req, "success")    │
                          │  flash.set(res, "info", msg)  │
                          │                               │
                          │  // HTMX partial swap only:   │
                          │  flash.setHxTrigger(res)      │
                          └──────────┬───────────────────┘
                                     │
                          ┌──────────▼───────────────────┐
                          │        Flash middleware       │
                          │  dirty? → write Set-Cookie    │
                          │  consumed? → expire cookie    │
                          │  untouched? → no-op           │
                          └──────────────────────────────┘
```

### Request phase (before `executor.next()`)

1. Read the `_flash` cookie via `req.cookies().get()`.
2. Base64-decode and deserialize into arena-allocated entry slices.
3. Store pointer so handlers can access entries.

### Response phase (after `executor.next()`)

4. If entries were **set** → serialize, encode, call `res.setCookie()`.
5. If entries were **consumed** and none added → expire cookie (`Max-Age=0`).
6. If **untouched** → no-op.

## Cookie format

```
_flash=<base64(payload)>; Path=/; HttpOnly; SameSite=Lax; Max-Age=60
```

Uses httpz's built-in `res.setCookie()` and `CookieOpts`. No custom
cookie serialization needed.

No signing, no encryption. Flash messages are user-facing display
strings. A user tampering with their own cookie only changes what they
see.

### Payload encoding

Length-prefixed binary. Decode requires no allocator beyond the arena:

```
[num_entries: u8]
  [key_len: u8] [key_bytes...]
  [val_len: u16 LE] [val_bytes...]
  ...
```

Hard limit: **4000 bytes** after encoding (within the 4096-byte cookie
ceiling after name, flags, and base64 overhead).

## Handler API

### Cookie-backed (redirect flow)

```zig
const flash = @import("httpz-flash");

fn createItem(req: *httpz.Request, res: *httpz.Response) !void {
    // ... create item ...
    flash.set(res, "success", "Item created!");
    res.status = 303;
    res.header("location", "/items");
}

fn listItems(req: *httpz.Request, res: *httpz.Response) !void {
    if (flash.get(req, "success")) |msg| {
        // render msg in page
    }
    for (flash.getAll(req)) |entry| {
        // entry.key, entry.value
    }
}
```

### HTMX partial swap (no redirect)

```zig
fn updateItem(req: *httpz.Request, res: *httpz.Response) !void {
    // ... update item ...
    flash.set(res, "success", "Item updated!");

    // Emit as HX-Trigger instead of cookie
    // Writes: HX-Trigger: {"showFlash": {"level": "success", "message": "Item updated!"}}
    flash.setHxTrigger(res);

    res.body = "<div>updated content</div>";
}
```

Client-side (any JS toast library):

```js
document.addEventListener("showFlash", (e) => {
    showToast(e.detail.level, e.detail.message);
});
```

`setHxTrigger` marks entries as consumed so the middleware won't also
write a cookie. No double delivery.

## Edge cases

### Flash set but handler returns 200 (no redirect)

The cookie is still written. It will be consumed on whatever the next
request is. This is correct — the handler might be doing a client-side
redirect via JS or `<meta http-equiv="refresh">`. The middleware does
not inspect the status code.

### Multiple flashes per request

The binary format supports up to 255 entries (u8 count). Multiple calls
to `set()` accumulate. `getAll()` returns all of them.

### Duplicate keys

`set()` with the same key overwrites the previous value. Last write
wins. Rationale: if a handler calls `set("error", "A")` then later
`set("error", "B")`, only B matters.

### HTMX boosted navigation

`hx-boost` makes links/forms behave as AJAX but follows redirects
transparently. The browser sends cookies on the follow-up GET. Cookie
flash works without any special handling.

### HTMX `HX-Redirect` / `HX-Location`

The server sets the header, HTMX tells the browser to do a full
navigation. The browser sends cookies on that navigation. Cookie flash
works.

### HTMX partial + redirect race

If a handler calls both `flash.set()` and `flash.setHxTrigger()`, the
HX-Trigger helper marks entries as consumed. The middleware sees them as
consumed and expires the cookie (or does nothing if no prior cookie
existed). No double delivery.

### Multi-tab consumption

Tab A sets a flash and redirects. Tab B makes a request and consumes
the flash instead. This is an inherent limitation of cookie-based
flash. Not solvable without server-side sessions, which is out of scope.
Short `max_age` (default 60s) limits the window.

### Cookie overflow

If the encoded payload exceeds 4000 bytes, `set()` returns an error.
The handler can decide what to do (truncate the message, drop it, etc.).
The middleware never writes an oversized cookie.

### Malformed cookie

If the cookie exists but fails to decode (corrupted, truncated, wrong
format), silently ignore it. No entries loaded, no error propagated.
The middleware expires the bad cookie on the response.

### `Secure` flag in development

Default `secure: true`. For local dev over plain HTTP, configure
`secure: false`. httpz's `CookieOpts` handles this.

## Middleware struct

```zig
pub const Flash = struct {
    cookie_name: []const u8,
    max_age: i32,
    secure: bool,

    pub const Config = struct {
        cookie_name: []const u8 = "_flash",
        max_age: i32 = 60,
        secure: bool = true,
    };

    pub fn init(config: Config) !Flash {
        return .{
            .cookie_name = config.cookie_name,
            .max_age = config.max_age,
            .secure = config.secure,
        };
    }

    pub fn execute(
        self: *const Flash,
        req: *httpz.Request,
        res: *httpz.Response,
        executor: anytype,
    ) !void {
        // 1. Parse cookie, store entries in arena
        // 2. Call executor.next()
        // 3. Write/expire cookie based on state
    }
};
```

`max_age` is `i32` to match httpz's `CookieOpts.max_age`.

## File structure

```
├── build.zig
├── build.zig.zon
├── src/
│   ├── flash.zig       — middleware struct (init, execute)
│   ├── codec.zig       — encode/decode binary payload + base64
│   └── root.zig        — public API (set, get, getAll, setHxTrigger)
├── test/
│   ├── flash_test.zig  — full middleware cycle tests
│   └── codec_test.zig  — encode/decode round-trip tests
├── DESIGN.md
├── README.md
├── AGENTS.md
└── LICENSE
```

Dropped `cookie.zig` — httpz's `req.cookies().get()` and
`res.setCookie()` handle parsing and serialization. No need to
reimplement.

## Decisions

| Decision | Rationale |
|----------|-----------|
| No signing/HMAC | Flash is display-only. Tampering only affects the tamperer. |
| No FlashMap struct | httpz provides a per-request arena. Use arena-allocated slices. |
| Use httpz's cookie API | `req.cookies().get()`, `res.setCookie()` already exist. Don't rewrite. |
| `setHxTrigger` as opt-in helper | Handler knows if it's a partial swap. Don't auto-detect. |
| Binary payload, not JSON | No allocator needed for decode. Parse directly into arena slices. |
| Overwrite on duplicate key | Last write wins. Simplest semantics, matches user expectation. |
| Silent drop on malformed cookie | Don't crash on bad input. Expire it and move on. |

## Open questions

1. Should `setHxTrigger` support custom event names (default `showFlash`)?
   Probably yes — make it a parameter.
2. Should there be `setHxTriggerAfterSwap` / `setHxTriggerAfterSettle`
   variants? Defer unless requested.
