# ADR-IdP-002: JWT vs Opaque Tokens and Signing Strategy

**Date:** 2026-02-28  
**Status:** Accepted

## Context

We need to decide:
1. Whether access tokens are JWTs (self-contained) or opaque (require introspection)
2. The signing algorithm and key rotation strategy

## Decision

- **Access tokens: JWT** signed with **RS256** (RSA + SHA-256)
- **Refresh tokens: Opaque** (random 256-bit, stored hashed)
- **Signing keys: 2048-bit RSA, rotated every 90 days** with overlap period

## Rationale

### JWT Access Tokens

| Approach | Pros | Cons |
|----------|------|------|
| **JWT (chosen)** | No network call to validate; API can verify locally using JWKS | Cannot be instantly revoked; larger token size (~800 bytes) |
| **Opaque** | Instant revocation; smaller token | Every API request requires introspection call to IdP (latency + coupling) |

With 10,000+ concurrent users making API calls, the introspection overhead of opaque tokens is unacceptable. JWTs let the API validate tokens locally by fetching the public key once and caching it.

**Short TTL (15 minutes) mitigates the revocation gap.** If a token is compromised, the window of exposure is 15 minutes max. For critical revocation (e.g., user deactivated), the API checks a Redis-cached revocation list — a lightweight check against `idp:revoked:{jti}` keys.

### Opaque Refresh Tokens

Refresh tokens are opaque because:
- They're only sent to the IdP (not to every API endpoint) — no network overhead concern
- We need server-side state for rotation tracking and family-based revocation
- They should be revocable instantly

### RS256 Signing

| Algorithm | Type | Key Management | Performance |
|-----------|------|---------------|-------------|
| **RS256 (chosen)** | Asymmetric | Public key shared via JWKS; private key stays in IdP | ~1ms sign, ~0.1ms verify |
| HS256 | Symmetric | Shared secret between IdP and API | ~0.01ms both — but requires secret distribution |
| ES256 | Asymmetric (ECDSA) | Smaller keys, faster — but less library support | ~0.5ms sign, ~1ms verify |

RS256 wins because:
- **The API never needs the private key.** It fetches the public key from JWKS and verifies. Clean separation.
- **Universal support.** Every JWT library supports RS256. No compatibility surprises.
- **Key rotation is straightforward.** Add new key to JWKS, start signing with it, old keys still verify existing tokens.

### Key Rotation Strategy

```
Day 0:   Generate key-2026-Q1, set as ACTIVE
Day 1-89: Sign all new tokens with key-2026-Q1
Day 80:  Generate key-2026-Q2, set as ACTIVE (overlap period begins)
Day 90:  Start signing with key-2026-Q2
Day 90:  Set key-2026-Q1 to ROTATED (still in JWKS for verification)
Day 105: Remove key-2026-Q1 from JWKS (all its tokens have expired)
```

Both keys appear in the JWKS endpoint during the overlap period. The `kid` (Key ID) header in each JWT tells the verifier which key to use.

## Consequences

### Positive
- API validates tokens with zero network calls in the happy path
- Clean key separation — API only has public keys
- Standard JWKS endpoint works with any OAuth2 client library
- 90-day rotation limits blast radius of key compromise

### Negative
- Cannot instantly revoke individual access tokens (15-min window)
- Mitigation: Redis revocation list for emergency revocation
- JWT payload is larger than opaque tokens (~800 bytes vs ~64 bytes)
- Must handle key rotation carefully — overlapping validity period required
