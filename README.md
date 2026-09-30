# NodeX

A decentralized, peer-to-peer chat application. **Identities, login, recovery, contacts and profiles live on users' devices.** There are no user accounts on any server.

**Implemented so far:** decentralized identity (registration, handle, recovery phrase, re-login/recovery) and a local profile.
**Next:** the libp2p P2P network (finding people by handle, profile sharing, chat).

## Architecture

```
Browser (Next.js) ── all identity logic, keys, contacts, profile ── IndexedDB on the device
      │
      └── only during registration ──► Email verifier (Go, stateless, no database)
```

The **email verifier** is the only server-side piece, used only while registering.
- It only emails a 6-digit OTP and checks it.
- It has no database. Its tokens are HMAC-signed, and its only state is short-lived, in-memory anti-abuse data (resend cooldown, attempt counts).
- It never receives Peer IDs, keys, recovery phrases or display names. It never stores emails.

## Identity

```
12-word phrase ──BIP39──► Ed25519 key ──► Peer ID
email + Peer ID ──PBKDF2 (200k)──► email commitment
display name + Peer ID + commitment ──PBKDF2 (600k)──► 6-char tag
handle = "<DisplayName>#<TAG>"      e.g. Rahul#7K3M9X
```

- **The handle is the user's only username**, and it's shown everywhere.
- **The display name is used only to form the handle.** It isn't stored or sent separately.
- **The tag uses Crockford base32** (no I/L/O/U). On input, `I`/`L` are read as `1` and `O` as `0`.
- **Handles are case-insensitive.** Every comparison, lookup and validation uses the canonical form `handleKey` (lowercase with look-alikes fixed), so `Rahul#7K3M9X`, `rahul#7k3m9x` and `RAHUL#7K3M9X` are the same handle. The helpers are in `lib/handle.ts` (`normalizeHandle`, `handlesEqual`), and `handleKey` is stored with the identity and each contact.
- **Uniqueness comes from the key.** The tag is bound to the user's key, so another key can't produce the same handle without brute-forcing a deliberately slow hash.

## Registration

1. **Email and OTP:** the user enters their email, the verifier sends an OTP, and the user enters it.
2. **Display name and identity:** the user enters a display name. The browser generates the phrase, key, Peer ID, email commitment and handle. Nothing is sent to any server.
3. **Storage:** the private key is encrypted with a **non-extractable device AES key** and saved in IndexedDB, together with the handle and `handleKey`.
4. **Recovery phrase:** the 12 words are shown once and never stored.
5. **Identity screen:** the handle is shown with *"This is your username/handle. Use this identity to find other users and to identify yourself on NodeX."*

## Re-login and recovery

Logging out removes the identity from the device. To log in again, on this or any device:

1. The user enters their **email**, **handle** and **12-word phrase**.
2. The phrase recreates the same Ed25519 key and Peer ID.
3. The email and handle are re-derived, and the result must equal the entered handle (compared case-insensitively). Otherwise nothing is restored and no new identity is created.
4. The identity is saved on the device again.

All of this happens on the device with **zero network requests**.

## Running locally

Requirements: Go 1.27+, Node 20+.

```sh
# Email verifier on :8080
cd NodeX-backend
go run ./cmd/server

# Frontend on :3000
cd NodeX-frontend
npm install
npm run dev
```

Open http://localhost:3000.
- **Email:** without `SMTP_HOST`, OTP codes are printed to the verifier's log. The verifier reads `NodeX-backend/.env` (see [.env.example](NodeX-backend/.env.example)), and in production it requires `OTP_SECRET` and SMTP settings.
- **No database:** NodeX doesn't use one any more. An old `nodex-postgres-1` Docker container from earlier versions can be removed with `docker rm -f nodex-postgres-1`.

## Email verifier API

| Method | Path | Body → Response |
| --- | --- | --- |
| POST | `/api/v1/otp/send` | `{ email }` → `{ otp_token, expires_in_seconds, resend_in_seconds }` |
| POST | `/api/v1/otp/verify` | `{ otp_token, otp }` → `{ verified: true }` |

## Tests

```sh
cd NodeX-backend && go test ./...
```
