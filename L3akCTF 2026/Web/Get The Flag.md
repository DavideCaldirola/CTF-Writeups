# Get The Flag

<img width="1133" height="616" alt="image" src="https://github.com/user-attachments/assets/2e791458-7911-43f5-9645-a6eea7347256" />

## Writeup

### Overview

"Get The Flag" is a web/misc challenge built with Express + EJS. Normal users can:

- Register / login (`/register`, `/login`)
- Upload arbitrary static HTML pages, hosted on the **same origin** as the app (`/pages/upload` → served from `/pages/:file`)
- Ask an admin-controlled Puppeteer bot to visit one of those pages (`/report`)
- Change their own password (`/account/change-password`)
- View the flag, but only if `user.role === "admin"` (`/flag`)

<img width="1920" height="932" alt="Screenshot From 2026-08-02 16-27-45" src="https://github.com/user-attachments/assets/05299e27-1c46-4fb4-a5bb-f6566ce28dce" />


The bot (`bot.js`) logs into the real app with the actual `admin` account (credentials only known server-side) and then navigates to whatever `/pages/*` URL was submitted through `/report`. This is the classic "self-XSS-style page + admin bot" primitive: get arbitrary JS/HTML to run inside the admin's authenticated browser session.

The catch is that uploaded pages are served with a restrictive CSP:

```
Content-Security-Policy: sandbox allow-scripts allow-forms allow-same-origin; connect-src 'none'; frame-src 'none'; object-src 'none'
```

- `connect-src 'none'` → no `fetch`/`XHR`/`WebSocket`/`EventSource`
- `frame-src 'none'` → no `<iframe>`
- `object-src 'none'` → no `<object>`/`<embed>`
- No `allow-popups` → no `window.open`

So the usual "steal data via fetch/iframe and exfiltrate to an external server" approach is blocked. However, `allow-scripts` **and** `allow-forms` are both granted, and `allow-same-origin` keeps the page on the app's real origin. Crucially, `<form>` submissions are **not** governed by `connect-src`/`frame-src`/`object-src` at all - they are plain navigations. This means the attacker doesn't need to *read* anything back via JS; it's enough to make the admin's browser silently *submit a form* that performs a state-changing action.

### The vulnerability: CSRF-token bypass via `method-override` ordering

`app.js` registers middleware in this order:

```js
app.use(express.urlencoded({ extended: false }));   // (1) body parsed using the REAL HTTP method

app.use(
  methodOverride((req) => {
    if (typeof req.query._method === "string") {
      return req.query._method.toUpperCase();        // (2) req.method overwritten AFTER body parsing
    }
  })
);
```

And the **CSRF** guard used on the sensitive endpoint:

```js
function csrfOnPostOnly(req, res, next) {
  if (req.method !== "POST") {
    return next();                                    // skipped entirely if method isn't POST
  }
  if (!req.body.csrf || req.body.csrf !== req.session.csrf) {
    return res.status(403).send("Invalid CSRF token");
  }
  next();
}
```

applied to:

```js
app.all(
  "/account/change-password",
  requireLogin,
  csrfOnPostOnly,
  (req, res) => {
    if (!["GET", "POST"].includes(req.method)) {
      return res.sendStatus(405);
    }
    if (req.method === "GET" && !req.body?.password) {
      return res.render("change-password", { csrf: req.session.csrf });
    }
    const { password, confirm } = req.body ?? {};
    if (!password || password !== confirm || password.length < 8) {
      return res.status(400).send("Password and confirmation must match (min 8 chars)");
    }
    changePassword(req.session.userId, password);
    res.send("Password changed successfully");
  }
);
```

Because `express.urlencoded()` runs **before** `methodOverride`, the request body is parsed based on the *real* HTTP method, while `req.method` itself gets rewritten *afterwards* based on the `?_method=` query parameter.

So a genuine `POST` request to:

```
/account/change-password?_method=GET
```

with body `password=...&confirm=...` results in:

1. `express.urlencoded()` parses the real POST body → `req.body = { password, confirm }`.
2. `methodOverride` rewrites `req.method` to `"GET"`.
3. `csrfOnPostOnly` sees `req.method !== "POST"` → **skips the CSRF check entirely**.
4. In the handler, `req.method === "GET"` but `req.body.password` is present → the "render the form" branch is skipped, and execution falls straight through to the password-change logic.
5. `changePassword(req.session.userId, password)` runs — **no CSRF token required**.

This is a pure server-side logic bug (middleware ordering) — it works from *any* origin capable of submitting a POST form to that URL, no XSS or CSP bypass is strictly needed for the bug itself. What the bot/page-hosting mechanism provides is the delivery: a way to make the *admin's own authenticated browser* perform that POST silently.

### Why it survives the sandboxed CSP

- The malicious page is uploaded to `/pages/<uuid>.html`, i.e. it lives on the **same origin** as the app.
- The session cookie is set with `sameSite: "lax"`. `Lax` only restricts *cross-site* requests; a form submitted from a page hosted on the app's own origin to another endpoint on the same origin is **same-origin**, so the cookie is sent normally.
- Form submission is a navigation, not a `fetch`/`XHR`/`iframe`/`object` load, so `connect-src 'none'`, `frame-src 'none'`, and `object-src 'none'` never come into play. `allow-forms` explicitly permits it.

### Exploit chain

1. Register a normal account and log in.
2. Host the following payload via `/pages/create` (`POST /pages/upload`):

    ```html
    <form id="pwn" method="POST" action="/account/change-password?_method=GET">
      <input type="hidden" name="password" value="Pwned1234!">
      <input type="hidden" name="confirm" value="Pwned1234!">
    </form>
    <script>document.getElementById('pwn').submit();</script>
    ```

   This returns a URL like `/pages/<uuid>.html`.

3. Submit that path through `/report` (`POST /report`, field `url` = `/pages/<uuid>.html`).
4. The admin bot logs in as `admin`, navigates to the uploaded page inside its own authenticated session, and the hidden form auto-submits. The CSRF bypass fires and the `admin` account's password is silently changed to `Pwned1234!`.
5. Log out of the low-privilege account, then log in as `admin` / `Pwned1234!` through the normal login form.
6. Visit `/flag`: the session now belongs to a user with `role === "admin"`, so the flag is rendered.

<img width="1920" height="932" alt="Screenshot From 2026-08-02 16-50-51" src="https://github.com/user-attachments/assets/1a0405fd-daf4-4b16-94d9-9b2520d8829a" />

### Root causes

- **CSRF middleware bypassable via HTTP method override**: `csrfOnPostOnly` trusts `req.method` *after* it has already been rewritten by `method-override`, while the body parser used the true method. Fix: run `methodOverride` before body parsing (or don't allow query-string method override on state-changing routes at all), and/or check CSRF regardless of the (possibly spoofed) method.
- **Same-origin arbitrary HTML hosting + an authenticated bot** turns any same-origin logic flaw (like the CSRF bypass above) into a full account/session takeover, since `SameSite=Lax` and CSP `connect-src`/`frame-src`/`object-src` restrictions don't protect against same-origin form submissions.
- Secondary hardening gaps worth noting: `/report` and `/pages/upload` have no CSRF protection at all (not required for this exploit, but a bad pattern).

Flag: **L3AK{Meth0d_overR1d3_Csrf_Byp455_g0_brRrR}**

