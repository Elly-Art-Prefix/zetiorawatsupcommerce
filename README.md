# WhatsApp service (worker)

This small program keeps your WhatsApp number logged in 24/7, sends every
incoming customer message to your app, and replies with the answer the app
gives back.

It must run somewhere that is always on — a small VPS, Railway, Render, Fly.io
or a home server. It cannot run inside the app itself, because WhatsApp
requires a long-lived connection.

## 1. Requirements

- Node.js 20 or newer
- A machine or hosting account that stays online

## 2. Setup

```bash
cd whatsapp-worker
cp .env.example .env      # then open .env and fill in the values
npm install
npm start
```

In `.env`:

| Setting | What to put |
| --- | --- |
| `PORT` | The port to listen on (default 8088) |
| `WORKER_TOKEN` | The shared password. Must match the one saved in the app exactly |
| `APP_URL` | Your app address, e.g. `https://project--<id>.lovable.app` |
| `SESSIONS_DIR` | Folder for the WhatsApp login files. Keep it between restarts |

When it starts you should see: `WhatsApp worker listening on :8088`.

Check it: `curl http://localhost:8088/health` → `{"ok":true,"sessions":0}`

## 3. Make it reachable from the app

The app calls this service over the internet, so it needs a public HTTPS
address.

- Hosting (recommended): deploy this folder to Railway / Render / Fly.io and use
  the URL they give you, e.g. `https://whatsapp-worker.up.railway.app`.
- Quick test from your own computer: `npx cloudflared tunnel --url http://localhost:8088`
  and use the `https://...trycloudflare.com` address it prints.

Then give that address to the app as `WHATSAPP_WORKER_URL` (no trailing slash),
and the same `WORKER_TOKEN` as `WHATSAPP_WORKER_TOKEN`.

## 4. Connect a number

In the app, open WhatsApp in the dashboard and choose QR or pairing code. The
service will produce it, and once the phone is linked the status turns
Connected. From then on, customer messages arrive in the Inbox and AI replies
go out automatically.

## 5. Keeping it alive

With systemd:

```ini
[Unit]
Description=WhatsApp worker
After=network.target

[Service]
WorkingDirectory=/opt/whatsapp-worker
EnvironmentFile=/opt/whatsapp-worker/.env
ExecStart=/usr/bin/node server.js
Restart=always

[Install]
WantedBy=multi-user.target
```

Or with pm2: `pm2 start server.js --name whatsapp-worker && pm2 save`.

## Endpoints (used by the app, all need the token)

```
GET  /whatsapp/status?businessId=&accountId=
POST /whatsapp/connect         { businessId, accountId, phoneNumber?, method }
POST /whatsapp/disconnect      { businessId, accountId }
POST /whatsapp/reconnect       { businessId, accountId }
GET  /whatsapp/pairing-code?businessId=&accountId=&phoneNumber=
GET  /whatsapp/qr?businessId=&accountId=
GET  /health                   (no token)
```

It calls back into the app at `/api/public/whatsapp/connection-state` and
`/api/public/whatsapp/inbound-message` with the same token.

## Notes

- One WhatsApp login per account id; sessions survive restarts.
- If you log out on the phone, the session files are deleted and the app is told.
- Never share `WORKER_TOKEN` — anyone with it can control your WhatsApp session.
