# NodeX

Decentralized P2P chat. **Implemented so far: user registration and peer identity only.**

## Registration flow

1. The user enters an email. The backend emails a 6-digit OTP (valid for 10 minutes, 60 s resend cooldown, 5 attempts).
2. After the OTP is verified, the backend issues a single-use registration token (valid for 15 minutes).
3. The user picks a globally unique username (case-insensitive, so `Rahul` blocks `rahul`) and a password.
4. The browser generates a libp2p **Ed25519** key pair and derives the **Peer ID** (`12D3KooW…`) from it.
   - It signs `NodeX identity registration v1\nusername:…\npeer_id:…\nsession:<token>` to prove it holds the key.
   - It encrypts the private key with AES-GCM under a PBKDF2-SHA256 key (600k iterations) derived from the password, and stores it in IndexedDB (`nodex-keystore`). **The private key never leaves the device.**
5. The backend verifies that the Peer ID derives from the public key and checks the signature. It hashes the password with Argon2id and stores the user.

PostgreSQL `users` table: `email`, `email_verified_at`, `username`, `password_hash` (Argon2id), `peer_id`, `public_key` (libp2p protobuf). The Peer ID is an identity, not a login credential.

## Running locally

Requirements: Go 1.27+, Node 20+, Docker.

```sh
# 1. Database (exposed on host port 5433)
docker compose up -d

# 2. Backend on :8080 (applies migrations on start)
cd NodeX-backend
go run ./cmd/server

# 3. Frontend on :3000
cd NodeX-frontend
npm install
npm run dev
```

Open http://localhost:3000/register. When `SMTP_HOST` is unset, the backend doesn't send email. It prints OTP codes to its log instead.

Configuration: the backend reads `NodeX-backend/.env` on start (see [.env.example](NodeX-backend/.env.example)); real environment variables override it. Set the `SMTP_*` values there to send OTP emails, e.g. Gmail with an App Password. In production (`APP_ENV=production`) it requires `OTP_SECRET` and SMTP settings.

## API

| Method | Path | Body |
| --- | --- | --- |
| POST | `/api/v1/auth/register/otp` | `{ email }` |
| POST | `/api/v1/auth/register/verify` | `{ email, otp }` → `{ registration_token }` |
| GET | `/api/v1/users/username-available?username=` | → `{ available }` |
| POST | `/api/v1/auth/register/complete` | `{ registration_token, username, password, confirm_password, peer_id, public_key, signature }` |

Errors look like this: `{ "error": { "code": "username_taken", "message": "…" } }`.

## Tests

```sh
cd NodeX-backend && go test ./...
```
