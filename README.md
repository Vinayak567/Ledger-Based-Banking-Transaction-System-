# Backend Ledger

A REST API for a ledger-based banking prototype. It manages users and accounts, records money movement as immutable double-entry ledger records, and calculates an account balance from those records instead of storing a mutable balance field.

## What it does

- Registers, logs in, and logs out users with three-day JWTs (returned in the response and set in a `token` cookie).
- Creates INR accounts for authenticated users and lists only the caller's accounts.
- Transfers funds between active accounts using a unique idempotency key.
- Creates one immutable `DEBIT` entry and one immutable `CREDIT` entry for each completed transfer.
- Computes balance as total credits minus total debits.
- Lets a `systemUser` fund an account through a restricted initial-funds endpoint.
- Sends registration and successful-transfer emails through Gmail OAuth2 when configured.

## Stack

Node.js, Express 5, MongoDB/Mongoose, JSON Web Tokens, bcrypt, Nodemailer, and Gmail OAuth2.

## Project layout

```text
server.js                    Application entry point (port 3000)
src/app.js                   Express configuration and route mounting
src/config/db.js             MongoDB connection
src/routes/                  HTTP route definitions
src/controllers/             Request handling and transfer workflow
src/models/                  User, account, transaction, ledger, token blacklist schemas
src/middleware/              JWT and system-user authorization
src/services/email.service.js Email notifications
```

## Prerequisites

- Node.js 18 or newer
- MongoDB 4.0+ configured as a replica set. Transfers use MongoDB sessions and transactions, which do not work on a standalone MongoDB deployment.
- A Gmail OAuth2 application if email delivery is required

## Setup

```bash
npm install
cp .env.example .env
npm run dev
```

The API starts at `http://localhost:3000`; `GET /` returns `Ledger Service is up and running`.

Create `.env` with the following values:

```dotenv
MONGO_URI=mongodb://127.0.0.1:27017/backend-ledger?replicaSet=rs0
JWT_SECRET=replace-with-a-long-random-secret

# Required for Gmail notifications
EMAIL_USER=your-address@gmail.com
CLIENT_ID=your-google-oauth-client-id
CLIENT_SECRET=your-google-oauth-client-secret
REFRESH_TOKEN=your-google-oauth-refresh-token
```

`npm start` runs the server normally. `npm run dev` uses `npx nodemon`; it may download Nodemon if it is not already available locally.

## Authentication

Protected routes accept either the cookie created on login or a bearer token:

```http
Authorization: Bearer <token>
```

Tokens expire after three days. Logout adds the presented token to a MongoDB blacklist with a matching three-day TTL.

## API reference

All request bodies are JSON.

| Method | Endpoint | Auth | Body / purpose |
| --- | --- | --- | --- |
| `POST` | `/api/auth/register` | No | `{ "name", "email", "password" }` — create a user and return a JWT |
| `POST` | `/api/auth/login` | No | `{ "email", "password" }` — return a JWT |
| `POST` | `/api/auth/logout` | Token optional | Blacklist the supplied token and clear the cookie |
| `POST` | `/api/accounts` | User | Create an active INR account for the caller |
| `GET` | `/api/accounts` | User | List the caller's accounts |
| `GET` | `/api/accounts/balance/:accountId` | User | Get balance for one of the caller's accounts |
| `POST` | `/api/transactions` | User | `{ "fromAccount", "toAccount", "amount", "idempotencyKey" }` — transfer money |
| `POST` | `/api/transactions/system/initial-funds` | System user | `{ "toAccount", "amount", "idempotencyKey" }` — fund an account from the system user's account |

### Example workflow

Register two users, create accounts, then use an initial-funds transaction to place money into the sender account before transferring. Store the token returned by login/register and supply it as shown below.

```bash
# Register a user
curl -X POST http://localhost:3000/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada","email":"ada@example.com","password":"secret123"}'

# Create an account (replace TOKEN)
curl -X POST http://localhost:3000/api/accounts \
  -H 'Authorization: Bearer TOKEN'

# Transfer (replace the account IDs and TOKEN)
curl -X POST http://localhost:3000/api/transactions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer TOKEN' \
  -d '{"fromAccount":"SENDER_ACCOUNT_ID","toAccount":"RECIPIENT_ACCOUNT_ID","amount":250,"idempotencyKey":"transfer-2026-001"}'

# Read a balance
curl http://localhost:3000/api/accounts/balance/ACCOUNT_ID \
  -H 'Authorization: Bearer TOKEN'
```

To use the initial-funds endpoint, create a user with `systemUser: true` directly in the database and create an account for that user. The field is immutable and not exposed by public registration, deliberately preventing ordinary users from granting themselves this role.

## Ledger model and transfer behavior

Each successful transfer has a transaction record and two ledger entries:

```text
sender account     -- DEBIT  amount --> transaction
recipient account  -- CREDIT amount --> transaction
```

Balances are derived by aggregation: `sum(CREDIT) - sum(DEBIT)`. Ledger fields are immutable and Mongoose middleware rejects update/delete operations on ledger records.

The transfer endpoint uses `idempotencyKey` as a globally unique key. Repeating a completed request with the same key returns the original transaction rather than moving funds a second time. Both accounts must exist and be `ACTIVE`; the sender must have sufficient derived balance.

## Notes and current limitations

- The transfer controller intentionally contains a 15-second delay between debit and credit creation, likely for transaction/atomicity testing; this makes successful transfers take at least 15 seconds.
- There is no automated test suite yet (`npm test` currently exits with an error placeholder).
- Email configuration is initialized when the application loads. Missing or invalid Gmail OAuth credentials cause email connection/send errors in logs, while the API continues running.
- Currency defaults to INR, and the API does not currently validate that transferring accounts use the same currency.
- Transaction amounts are numeric values; production financial systems should use integer minor units or a decimal library to avoid floating-point precision issues.
- The normal transfer route validates that accounts exist, but it does not currently verify that `fromAccount` belongs to the authenticated user. Add that ownership check before using this API outside a controlled environment.
- The initial-funds route does not check the system account balance, allowing it to act as a source of funds. This is suitable only for a tightly controlled bootstrap/funding account.

## Security notes for production

Set a strong `JWT_SECRET`, use HTTPS, mark authentication cookies as `httpOnly`, `secure`, and `sameSite`, validate request bodies, rate-limit auth endpoints, and keep MongoDB and Gmail credentials outside version control. Do not expose a way for untrusted users to set `systemUser`.

## Scripts

| Command | Description |
| --- | --- |
| `npm start` | Run the API with Node.js |
| `npm run dev` | Run the API with Nodemon |
| `npm test` | Placeholder; no tests are configured yet |
