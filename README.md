<p align="center">
  <img src="NodeX-frontend/app/icon.svg" alt="NodeX logo" width="88" height="88">
</p>

<h1 align="center">NodeX</h1>

<p align="center">
  <strong>A decentralized, peer-to-peer chat application.</strong><br>
  Your identity lives on your device, not on a server.
</p>

<p align="center">
  <img alt="Status" src="https://img.shields.io/badge/status-in%20development-orange">
  <img alt="Frontend" src="https://img.shields.io/badge/frontend-Next.js%2016%20%2B%20TypeScript-black">
  <img alt="Backend" src="https://img.shields.io/badge/backend-Go%201.27-00ADD8">
  <img alt="P2P" src="https://img.shields.io/badge/P2P-libp2p-2dd4bf">
</p>

---

## What is NodeX?

NodeX is a messaging app built on a simple principle: **no central server owns your account.**

- **Your identity is created on your device.** It's a cryptographic key pair, with a human-friendly handle like **`Rahul#7K3M9X`**.
- **Your private key never leaves your device.** A 12-word recovery phrase is the only way to restore it, and that phrase is never stored anywhere.
- **Logging in and recovering your account happen entirely on your device**, with zero network requests.
- **Contacts and your profile photo are stored on your device**, not on a server.
- **The only server** is a tiny, stateless email verifier used once during sign-up to confirm your email with a one-time code. It has no database and stores no users.

> **Project status:** the decentralized identity system is complete. The peer-to-peer network (finding people by handle, chat, sharing profile photos) is the next milestone. See the [Roadmap](#roadmap).

---

## Features

| | Feature | Details |
|---|---|---|
| ✅ | **Email verification** | A 6-digit OTP is sent to your email during sign-up |
| ✅ | **Cryptographic identity** | An Ed25519 key pair and libp2p Peer ID, generated on your device |
| ✅ | **Human-friendly handle** | `DisplayName#TAG`, for example `Rahul#7K3M9X`. It's your only username. |
| ✅ | **12-word recovery phrase** | BIP39. It restores the exact same identity on any device. |
| ✅ | **Passwordless local login** | Email + handle + recovery phrase, checked on your device |
| ✅ | **Case-insensitive handles** | `Rahul#7K3M9X` = `rahul#7k3m9x` = `RAHUL#7K3M9X` |
| ✅ | **Secure key storage** | The private key is encrypted with a non-exportable browser key |
| ✅ | **Profile photo** | Cropped and resized on your device, and stored locally |
| ✅ | **Chat-style interface** | Dark, WhatsApp-inspired home, profile and logout screens |
| 🔜 | **Find people by handle** | Over the P2P network (DHT) |
| 🔜 | **End-to-end encrypted chat** | Direct peer-to-peer messaging |

---

## How it works

### Your identity

Everything below runs in your browser:

```
12-word recovery phrase ──BIP39──►  Ed25519 private key ──► Peer ID (12D3KooW…)
        email + Peer ID ──PBKDF2 (200k)──► email commitment
display name + Peer ID + commitment ──PBKDF2 (600k)──► 6-character tag

handle  =  DisplayName # TAG          →   Rahul#7K3M9X
```

- **The same 12 words always recreate the same key and Peer ID.** That's what makes recovery possible without any server.
- **The tag is tied to your key and email**, so another person can't produce your handle without brute-forcing a deliberately slow hash.
- **The tag uses unambiguous characters** (`0-9 A-Z` without `I L O U`). When you type a tag, `I`/`L` are read as `1` and `O` as `0`.
- **The display name is used only to create the handle.** It's never stored or shown on its own.

### Sign up

```
Email ──► 6-digit code ──► Display name ──► Identity created on device ──► Save 12 words ──► "Your NodeX identity"
```

1. **Verify your email:** enter your email and type the 6-digit code you receive.
2. **Choose a display name** (for example `Rahul`). Your key, Peer ID and handle are generated on your device.
3. **Save your 12-word recovery phrase.** It's shown once and never stored.
4. **See your handle**, with the message: *"This is your username/handle. Use this identity to find other users and to identify yourself on NodeX."*

### Log in and recover

Log in on the same device after logging out, or on a new device, by entering:

- your **email**
- your **handle** (in any capitalisation)
- your **12-word recovery phrase**

Your device recreates the key from the phrase and re-derives the handle from your email and name. If everything matches, the **same identity** is restored. If anything is wrong, nothing is restored and no new identity is ever created. **No request is sent to any server.**

### Log out

Logging out removes your identity from the device. To come back you'll need your email, handle and 12 words.

---

## Privacy: what is stored where

| Data | Where it lives | Sent to a server? |
|---|---|---|
| 12-word recovery phrase | Only with you (written down) | ❌ Never |
| Private key | Your device, encrypted with a non-exportable device key | ❌ Never |
| Display name | Only as part of your handle | ❌ Never |
| Handle, Peer ID, public key | Your device | ❌ No |
| Email | Nowhere on your device; the verifier sees it only to send the code | Only during sign-up, never stored |
| Contacts | Your device | ❌ Never |
| Profile photo | Your device | ❌ Never |

---

## Architecture

```
┌──────────────────────────── Your browser ────────────────────────────┐
│  Next.js app (React + TypeScript)                                      │
│   • identity generation, login, recovery  (lib/identity.ts)            │
│   • encrypted key vault, contacts, photo   (IndexedDB)                 │
└───────────────────────┬──────────────────────────────────────────────┘
                        │  sign-up only: send code / check code
                        ▼
┌──────────── Email verifier (Go) ────────────┐
│  • no database, stores no users              │
│  • codes travel inside HMAC-signed tokens    │
│  • rate limits + 5 attempts per code         │
│  • sends email via SMTP (e.g. Gmail)         │
└──────────────────────────────────────────────┘
```

### Tech stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 16, React 19, TypeScript |
| Cryptography | `@libp2p/crypto`, `@libp2p/peer-id` (Ed25519, Peer IDs), `@scure/bip39` (recovery phrase), Web Crypto API (PBKDF2, AES-GCM) |
| Local storage | IndexedDB |
| Email verifier | Go 1.27, standard library `net/smtp` |
| P2P network (next) | libp2p, DHT, WebRTC / WebTransport |

---

## Getting started

### Prerequisites

- [Go](https://go.dev/dl/) 1.27 or newer
- [Node.js](https://nodejs.org/) 20 or newer

### 1. Clone the repository

```bash
git clone https://github.com/noobdivya/NodeX.git
cd NodeX
```

### 2. Start the email verifier

```bash
cd NodeX-backend
cp .env.example .env        # then edit .env (see below)
go run ./cmd/server
```

It listens on **http://localhost:8080**.

> **No email setup needed for local testing.** If `SMTP_HOST` is empty, the 6-digit codes are printed in the verifier's terminal instead of being emailed.

### 3. Start the app

In a second terminal:

```bash
cd NodeX-frontend
npm install
npm run dev
```

Open **http://localhost:3000** and create your identity.

### Sending real emails (optional)

To send codes by email with Gmail:

1. Turn on [2-Step Verification](https://myaccount.google.com/security) for the Gmail account.
2. Create an **App Password** at <https://myaccount.google.com/apppasswords>.
3. Fill in `NodeX-backend/.env`:

```env
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=you@gmail.com
SMTP_PASSWORD=your-16-character-app-password
SMTP_FROM=NodeX <you@gmail.com>
```

Restart the verifier. **Never commit `.env`**; it's already listed in `.gitignore`.

### Configuration

**Email verifier** (`NodeX-backend/.env`):

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | Port to listen on |
| `CORS_ORIGINS` | `http://localhost:3000` | Allowed frontend origins (comma-separated) |
| `RATE_LIMIT_PER_MINUTE` | `20` | Requests per minute per IP |
| `OTP_SECRET` | random per run | HMAC key for code tokens, at least 32 characters. **Required in production.** |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM` | – | Email sending. **Required in production.** |
| `APP_ENV` | `development` | Set to `production` to enforce the required settings |

**Frontend** (`NodeX-frontend/.env.local`):

| Variable | Default | Description |
|---|---|---|
| `NEXT_PUBLIC_API_URL` | `http://localhost:8080` | Address of the email verifier |

---

## Email verifier API

| Method | Endpoint | Request | Response |
|---|---|---|---|
| `POST` | `/api/v1/otp/send` | `{ "email": "…" }` | `{ "otp_token", "expires_in_seconds", "resend_in_seconds" }` |
| `POST` | `/api/v1/otp/verify` | `{ "otp_token": "…", "otp": "123456" }` | `{ "verified": true }` |
| `GET` | `/healthz` | – | `{ "status": "ok" }` |

**Errors** look like `{ "error": { "code": "otp_invalid", "message": "…" } }`.

**Limits:**
- Codes expire after **10 minutes** and work once.
- A code allows **5 attempts**.
- A new code can be requested every **60 seconds**.

---

## Project structure

```
NodeX/
├── NodeX-backend/                 Go email verifier (no database)
│   ├── cmd/server/main.go         Startup and routes
│   └── internal/
│       ├── otp/                   Send and verify codes (HMAC-signed tokens)
│       ├── mailer/                SMTP email sending
│       ├── httpx/                 JSON helpers, CORS, rate limiting
│       └── config/                Settings and .env loading
│
└── NodeX-frontend/                Next.js app
    ├── app/                       Pages: /, /register, /login, /home
    ├── components/
    │   ├── register/              Email → code → name → phrase → identity
    │   ├── login/                 Log in / recover
    │   └── home/                  Contacts, profile, menu, logout
    └── lib/
        ├── identity.ts            Keys, recovery phrase, handle derivation
        ├── handle.ts              Handle parsing, case-insensitive comparison
        ├── keystore.ts            Encrypted identity vault (IndexedDB)
        ├── contacts.ts            Local contacts
        ├── profile.ts             Local profile photo
        ├── image.ts               Photo crop and resize
        └── api.ts                 Email verifier calls
```

---

## Testing

```bash
cd NodeX-backend
go test ./...
```

The backend tests cover code sending and checking, single use, the 5-attempt lockout, expiry, the resend cooldown and tamper-proof tokens. The full flow (sign-up, handle format, recovery, case-insensitive login, logout, and checking that no secrets or extra requests leave the device) was also tested end to end in a real browser.

---

## Security notes

- **Keep your 12 words safe.** Anyone who has them, plus your email and handle, can restore your identity. If you lose them and your device, the identity can't be recovered, by design, because nobody else ever had your key.
- **No password.** Anyone with access to your unlocked browser profile is logged in, as with most messaging web apps.
- **Handle tags are 6 characters.** The tag calculation is deliberately slow, but a very determined attacker with significant computing power could, in principle, create a different key with a matching handle. Contacts you've already added are tied to the real Peer ID. Longer tags are planned as an option.

---

## Roadmap

- [x] Email OTP verification (stateless verifier)
- [x] Cryptographic identity with `Name#TAG` handle
- [x] 12-word recovery phrase and on-device recovery
- [x] Passwordless local login and logout
- [x] Case-insensitive handles
- [x] Local contacts and profile photo
- [ ] libp2p node in the browser, plus relay/bootstrap node
- [ ] Find people by handle over the DHT
- [ ] Share profile photos peer to peer
- [ ] End-to-end encrypted one-to-one chat
- [ ] Offline message delivery
- [ ] Link a new device by QR code

---

## Contributing

Issues and pull requests are welcome. Please open an issue first to discuss larger changes.

## License

No license has been chosen yet. Until one is added, all rights are reserved by the author.
