# JWKS Authentication Server

A lightweight Python HTTP server implementing a JSON Web Key Set (JWKS) endpoint and JWT authentication service. The server manages RSA key pairs stored in a SQLite database, issues signed JWTs, and exposes a standards-compliant JWKS endpoint for public key discovery.

---

## Overview

This project demonstrates core concepts in modern authentication infrastructure:

- RSA key generation and lifecycle management (valid vs. expired keys)
- JWT issuance and signing using RS256
- JWKS endpoint for public key discovery (`/.well-known/jwks.json`)
- SQLite-backed key persistence with expiration tracking

---

## Project Structure

```
├── main1.py        # Server implementation
├── test_main1.py   # Unit and integration test suite
└── totally_not_my_privateKeys.db   # Auto-generated SQLite database
```

---

## How It Works

### Key Management

On startup, the server initializes a SQLite database and generates two RSA-2048 key pairs:

- **Valid key** — expires 1 hour in the future
- **Expired key** — expired 1 hour in the past

Keys are stored in PEM format in the `keys` table alongside their UNIX expiration timestamps. All timestamp comparisons use timezone-aware UTC datetimes.

### Database Schema

```sql
CREATE TABLE keys (
    kid  INTEGER PRIMARY KEY AUTOINCREMENT,
    key  BLOB NOT NULL,   -- RSA private key in PEM format
    exp  INTEGER NOT NULL  -- UNIX timestamp (UTC)
);
```

---

## Endpoints

### `GET /.well-known/jwks.json`

Returns all currently valid (non-expired) public keys in JWKS format. Clients use this endpoint to retrieve the public keys needed to verify JWTs issued by the `/auth` endpoint.

**Response:**
```json
{
  "keys": [
    {
      "alg": "RS256",
      "kty": "RSA",
      "use": "sig",
      "kid": "1",
      "n": "<base64url-encoded modulus>",
      "e": "<base64url-encoded exponent>"
    }
  ]
}
```

### `POST /auth`

Issues a signed JWT. Accepts an optional `expired` query parameter to request a token signed with an expired key (useful for testing token validation logic).

| Request | Behavior |
|---|---|
| `POST /auth` | Returns a valid JWT signed with a non-expired key |
| `POST /auth?expired` | Returns an expired JWT signed with the expired key |

**Response:** A signed JWT string (RS256)

**Token payload:**
```json
{
  "user": "username",
  "exp": "<1 hour from now, or 1 hour in the past if ?expired>"
}
```

### All other methods and paths → `405 Method Not Allowed`

PUT, PATCH, DELETE, and HEAD are explicitly rejected. Requests to unrecognized paths also return 405.

---

## Running the Server

### Prerequisites

```bash
pip install cryptography pyjwt requests
```

### Start

```bash
python main1.py
```

The server starts on `http://localhost:8080`. On first run it creates `totally_not_my_privateKeys.db` and populates it with an initial valid/expired key pair.

---

## Running the Tests

The test suite requires the server to already be running in a separate terminal.

```bash
# Terminal 1
python main1.py

# Terminal 2
python -m unittest test_main1.py -v
```

---

## Test Coverage

### `test_initialization`
Verifies the database file is created on startup, at least one key is stored, and the JWKS endpoint is accessible and returns a non-empty key set.

### `test_int_to_base64`
Unit tests the `int_to_base64` helper against known inputs including edge cases (0, max single-byte, max two-byte, large 256-bit integers) to ensure correct Base64URL encoding of RSA key components.

### `test_unsupported_methods`
Confirms that PUT, PATCH, DELETE, and HEAD all return `405 Method Not Allowed`.

### `test_jwks_endpoint`
Validates the structure of the JWKS response — checks for correct `alg`, `kty`, `use`, `kid`, `n`, and `e` fields.

### `test_auth_endpoint`
Posts to `/auth`, retrieves the JWKS, locates the matching public key by `kid`, reconstructs it, and fully decodes and verifies the returned JWT. Confirms the `user` claim is present and correct.

### `test_token_expiration`
Posts to `/auth?expired=true` and asserts that decoding the returned token raises `jwt.ExpiredSignatureError`, confirming the server correctly issues expired tokens when requested.

### `test_expired_token`
Additional expiration check using a manually constructed public key, reinforcing that expired tokens cannot be decoded without `options={"verify_exp": False}`.

---

## Concepts Demonstrated

- RSA-2048 key generation with the `cryptography` library
- JWT issuance and RS256 signing with `pyjwt`
- JWKS format and public key serialization (modulus/exponent as Base64URL)
- SQLite database initialization and parameterized queries
- HTTP server implementation with `http.server`
- Expiration-aware key selection using UTC timestamps
- Integration testing with live HTTP requests and token verification
