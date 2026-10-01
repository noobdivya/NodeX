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
- **Finding people is peer-to-peer.** Your handle is published, signed with your key, into a distributed hash table (DHT) shared by the network's peers. Searches go to the DHT, and every result is verified on your device.
- **Chat is direct and end-to-end encrypted.** Messages travel straight from your browser to your friend's browser over WebRTC. No server stores or forwards them.
- **Contacts, messages and your profile photo are stored on your device**, not on a server.
- **The only server** is a tiny, stateless email verifier used once during sign-up to confirm your email with a one-time code. It has no database and stores no users.

> **Project status:** decentralized identity, P2P contact search and real-time P2P chat are complete. Peer-to-peer profile photos and group chats are next. See the [Roadmap](#roadmap).

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
| ✅ | **Profile photo (shared P2P)** | Cropped and resized on your device; others get it directly from your device when they find or chat with you, and changes are pushed live |
| ✅ | **Chat-style interface** | Dark, WhatsApp-inspired home, profile and logout screens |
| ✅ | **Find people by handle (P2P)** | Signed handle records in a libp2p DHT, verified on your device |
| ✅ | **Local contacts** | Add people you find; saved only on your device |
| ✅ | **Real-time P2P chat** | One-to-one messages, browser to browser over WebRTC, end-to-end encrypted |
| ✅ | **Ticks and read receipts** | 🕓 waiting · ✓ delivered (recipient's device confirmed) · ✓✓ read (recipient opened the chat); unread counts in the chat list |
| ✅ | **Disappearing messages** | Each message is deleted from both devices 48 hours after it's read; unread messages are kept |
| ✅ | **Offline queue** | Messages to an offline friend wait on your device and are delivered automatically when they're back online |

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

### Finding people (P2P contact search)

```
You (browser) ──publish signed record──►  NodeX DHT  ◄──lookup "rahul#7k3m9x"── Someone else (browser)
                                          (held by network peers,
                                           in memory, no accounts)
```

1. **Publish:** when you're online, your browser joins the network and publishes a **handle record** to the DHT, then republishes it every 6 hours while the app is open:
   ```
   key   = /nodex/rahul#7k3m9x                (canonical lowercase handle)
   value = { handle, peer_id, email_commitment, seq, sig }
   ```
   The record is **signed with your private key**.
2. **Search:** type a full handle in any capitalisation, such as `rahul#7k3m9x`. Your browser asks the DHT for that key.
3. **Verify on your device:** a result is shown only if all of these hold:
   - the signature matches the key inside its Peer ID
   - it's stored under its own handle
   - the handle's tag re-derives from that Peer ID
4. **Add:** the contact (handle + Peer ID) is saved on your device.

**Why nobody can take over your handle:** every network node runs the same check before storing a record, so a record for `Rahul#7K3M9X` signed by any other key is rejected. Search sends **no HTTP requests**; it only talks to network peers over libp2p.

> Search is by **exact handle**: a DHT can't do partial ("starts with") search. Share your full handle so people can find you.

### Chat (real-time, peer to peer)

```
Asha's browser ──WebRTC (direct, encrypted)──► Bob's browser
        │                                         │
        └──── handshake only, via a NodeX node ───┘
```

1. **Connecting:** when you open a chat, your browser connects to your friend's browser. The NodeX node relays only the short WebRTC handshake, because browsers can't receive incoming connections on their own. After that the connection is **direct**. If a direct path can't be found, the relay is used instead, still end-to-end encrypted.
2. **Sending:** each message goes over that connection using the `/nodex/chat/1.0.0` protocol, and your friend's device sends back an acknowledgement. Then the 🕓 turns into ✓.
3. **Who sent it:** libp2p connections are authenticated with the sender's key, so the sender's Peer ID is proven. Their handle is checked against your contacts or the DHT. A stranger who claims to be someone else is rejected.
4. **Storage:** messages are kept only on the two devices (IndexedDB). People who message you appear in your chat list automatically.
5. **If your friend is offline:** the message waits on **your** device (🕓) and is delivered automatically when they're next online while your app is open. The app retries every 15 seconds and whenever they connect.
6. **Read receipts (✓✓):** when your friend opens the chat, their device sends a small `{ t: "read", ids }` receipt back to yours over the same direct connection. A receipt is only accepted from the person the messages were sent to. If you're offline at that moment, the receipt waits on their device, like messages do.
7. **Disappearing messages:**
   - The 48-hour timer starts when a message is read. For the reader that's when they open the chat; for the sender it's when the read receipt arrives.
   - After 48 hours, each device deletes its own copy. The app checks when it opens and every minute.
   - Unread messages are never deleted.

> **Both people need the app open at the same time** for a message to move. There's no server to hold messages for offline users.

### Profile photos (peer to peer)

- **Your photo stays on your device.** When someone finds you in search, opens a chat with you, or connects to you as a contact, their app asks yours for it over `/nodex/profile/1.0.0`. They only download it again if it has changed (compared by SHA-256).
- **Changes reach people straight away:** when you change or remove your photo, it's pushed to everyone currently connected. Others get the update the next time they connect.
- **Checks on every photo received:**
  - only WebP, JPEG or PNG, up to 256 KB
  - the file contents must really be that image type
  - the SHA-256 hash must match
  - unasked-for photo pushes are accepted only from contacts
- **Who can see it:** anyone who finds you can see your photo, like WhatsApp's "Everyone" setting.

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
| Handle record (handle, Peer ID, email commitment, signature) | The P2P network's DHT, so others can find you | Public by design; never on a server |
| Contacts | Your device | ❌ Never |
| Chat messages | Your device and your friend's device | ❌ Never (sent directly between the two browsers, encrypted) |
| Profile photo | Your device, plus copies on the devices of people who found or chatted with you | ❌ Never (sent directly device to device) |

> The email commitment in your public handle record is a **slow, salted hash** of your email, not the email itself. It's needed so anyone can check your handle belongs to your key. Someone who already suspects your exact email could test that guess (slowly), so treat your handle as linked to your email.

---

## Architecture

```
┌──────────────────────────── Your browser ────────────────────────────┐
│  Next.js app (React + TypeScript)                                      │
│   • identity generation, login, recovery  (lib/identity.ts)            │
│   • encrypted key vault, contacts, photo   (IndexedDB)                 │
│   • js-libp2p node: publish + look up handle records (lib/p2p/)        │
│   • chat: WebRTC straight to other browsers (lib/p2p/chat.ts)          │
└──────────┬──────────────────────────────────────────┬────────────────┘
           │ sign-up only: send / check code          │ libp2p (WebSockets, Noise, Yamux)
           ▼                                          ▼
┌──── Email verifier (Go) ────┐      ┌──────── NodeX P2P nodes (Go) ────────┐
│ • no database, no users      │      │ • DHT /nodex/kad/1.0.0 (server mode) │
│ • HMAC-signed code tokens    │      │ • holds signed handle records, in    │
│ • rate limits, 5 attempts    │      │   memory; validates every record     │
│ • SMTP (e.g. Gmail)          │      │ • circuit relay for browsers         │
└──────────────────────────────┘      │ • no database, no accounts; anyone   │
                                      │   can run one, link with NODE_PEERS  │
                                      └──────────────────────────────────────┘
```

**Why the network needs nodes like `NodeX-node`:** browsers can't accept incoming connections, so they can't hold a DHT by themselves. They join the network by dialling always-on libp2p peers. These peers hold only public, signed handle records, validate every record before storing it, and have no accounts or database. This is the same role bootstrap and DHT-server nodes play in IPFS. Any number of independently run nodes can form the network.

### Tech stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 16, React 19, TypeScript |
| Cryptography | `@libp2p/crypto`, `@libp2p/peer-id` (Ed25519, Peer IDs), `@scure/bip39` (recovery phrase), Web Crypto API (PBKDF2, AES-GCM) |
| Local storage | IndexedDB |
| Email verifier | Go 1.27, standard library `net/smtp` |
| P2P (browser) | `libp2p` (js), `@libp2p/kad-dht`, `@libp2p/websockets`, `@libp2p/webrtc`, `@libp2p/circuit-relay-v2`, Noise, Yamux |
| P2P (node) | `go-libp2p`, `go-libp2p-kad-dht` |

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

### 3. Start a NodeX P2P node

In a second terminal:

```bash
cd NodeX-node
go run .
```

- **First run:** the node creates `node.key`, its identity, which keeps its address the same across restarts. Keep the file private; it's git-ignored.
- **Address:** it prints the address browsers need, for example:

```
browsers (local dev): NEXT_PUBLIC_BOOTSTRAP_PEERS=/ip4/127.0.0.1/tcp/4002/ws/p2p/12D3KooW…
```

### 4. Start the app

In a third terminal, create `NodeX-frontend/.env.local` from [.env.example](NodeX-frontend/.env.example), paste the node address into it, then:

```bash
cd NodeX-frontend
npm install
npm run dev
```

Open **http://localhost:3000** and create your identity. The home screen shows *Online · discoverable on the NodeX network* once your handle is published. To try search, register a second identity in another browser profile and look it up by its handle.

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
| `NEXT_PUBLIC_BOOTSTRAP_PEERS` | – | NodeX P2P node addresses to join the network through (comma-separated multiaddrs) |
| `NEXT_PUBLIC_STUN_SERVERS` | – | Optional STUN servers (e.g. `stun:stun.l.google.com:19302`) that help browsers on different home networks find a direct path. STUN only tells a browser its public address and never sees messages. Not needed on one machine or network. |

**P2P node** (`NodeX-node`, environment variables):

| Variable | Default | Description |
|---|---|---|
| `NODE_LISTEN` | `/ip4/0.0.0.0/tcp/4001,/ip4/0.0.0.0/tcp/4002/ws` | Listen addresses (TCP for nodes, WebSockets for browsers) |
| `NODE_KEY_FILE` | `node.key` | Where the node's identity is kept (created on first run) |
| `NODE_KEY` | – | Node identity as 64 hex characters (overrides the key file) |
| `NODE_PEERS` | – | Other NodeX nodes to link with (comma-separated multiaddrs) |

> **Production:** browsers on an `https://` site can only use **secure** WebSockets. Put the node's WebSocket port behind TLS (for example a reverse proxy serving `wss://`) and list its `/dns4/…/tcp/443/wss/p2p/…` address in `NEXT_PUBLIC_BOOTSTRAP_PEERS`.

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
├── NodeX-node/                    Go libp2p P2P node (DHT server + relay)
│   ├── main.go                    Host, WebSockets/TCP, DHT, relay, node key
│   └── record.go                  Handle-record validation (signature + handle binding)
│
└── NodeX-frontend/                Next.js app
    ├── app/                       Pages: /, /register, /login, /home
    ├── components/
    │   ├── register/              Email → code → name → phrase → identity
    │   ├── login/                 Log in / recover
    │   └── home/                  Chat list, chat screen, P2P search, profile, menu
    └── lib/
        ├── p2p/node.ts            Browser libp2p node: join, publish, look up, WebRTC
        ├── p2p/chat.ts            P2P chat protocol: send, receive, acks, offline queue
        ├── messages.ts            Local message storage (IndexedDB)
        ├── p2p/record.ts          Create/validate handle records (mirrors record.go)
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
cd NodeX-backend && go test ./...
cd ../NodeX-node && go test ./...
```

- **Email verifier tests:** code sending and checking, single use, the 5-attempt lockout, expiry, the resend cooldown and tamper-proof tokens.
- **P2P node tests:** valid records, case-insensitive keys, and rejecting forgeries. Rejected cases include a handle claimed by another key, a swapped Peer ID, a wrong key, a future-dated record, garbage and oversized values. "Newest record wins" is also tested.
- **End-to-end tests in a real browser:**
  - **Identity:** sign-up, handle format, recovery, case-insensitive login, logout, and checking that no secrets or extra requests leave the device.
  - **P2P search:** two users publish, then find and add each other (handle typed in lowercase), with no HTTP requests during search. An attacker's forged record for someone else's handle is rejected by the node, and lookups still return the real user.
  - **P2P chat:**
    - Two browsers chat both ways in real time over a confirmed direct WebRTC connection.
    - ✓ appears once the recipient's device confirms; unread badges and previews update live.
    - 5 rapid messages arrive in order.
    - A message to an offline friend is queued and delivered automatically when they return.
    - History survives a reload, and chatting makes zero HTTP requests.
    - A stranger who claims to be someone else is rejected.

---

## Security notes

- **Keep your 12 words safe.** Anyone who has them, plus your email and handle, can restore your identity. If you lose them and your device, the identity can't be recovered, by design, because nobody else ever had your key.
- **No password.** Anyone with access to your unlocked browser profile is logged in, as with most messaging web apps.
- **Handle tags are 6 characters.** The tag calculation is deliberately slow, but a very determined attacker with significant computing power could, in principle, create a different key with a matching handle. Contacts you've already added are tied to the real Peer ID. Longer tags are planned as an option.
- **Being discoverable needs you online now and then.** Network nodes keep handle records in memory for up to 48 hours. Your app republishes yours when you open NodeX and every 6 hours while it's open. If you've been offline for a long time, or all nodes restarted, people can find you again as soon as you next open the app.
- **Disappearing messages rely on each device's NodeX app.** In a decentralized app nobody can force-delete data on someone else's device: each NodeX app deletes its own copy after 48 hours, but a modified app, a screenshot or a copy-paste can keep a message. The same is true of disappearing messages in any messenger.
- **Chat needs both people online.** Without a server, nothing can hold a message for someone who's offline. Messages wait on the sender's device until both are online together.
- **Across different home networks, browsers may need STUN** (`NEXT_PUBLIC_STUN_SERVERS`) to find a direct path. If none is found, chat still works through a NodeX node's relay (encrypted end to end), but relayed connections are time- and size-limited, so the app reconnects as needed.
- **Running more nodes makes the network more resilient.** With a single node, search depends on that node being up. Independently run nodes linked with `NODE_PEERS` share the DHT.

---

## Roadmap

- [x] Email OTP verification (stateless verifier)
- [x] Cryptographic identity with `Name#TAG` handle
- [x] 12-word recovery phrase and on-device recovery
- [x] Passwordless local login and logout
- [x] Case-insensitive handles
- [x] Local contacts and profile photo
- [x] libp2p node in the browser, plus P2P node (DHT server + relay)
- [x] Find people by handle over the DHT, verified on device
- [x] Share profile photos peer to peer
- [x] Real-time, end-to-end encrypted one-to-one chat (direct WebRTC)
- [x] Delivery acknowledgements, unread badges, on-device offline queue
- [ ] Group chats
- [x] Read receipts (✓✓) and disappearing messages (48 h after reading)
- [ ] Typing indicators
- [ ] Link a new device by QR code

---

## Contributing

Issues and pull requests are welcome. Please open an issue first to discuss larger changes.

## License

No license has been chosen yet. Until one is added, all rights are reserved by the author.
