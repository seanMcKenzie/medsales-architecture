# ADR-IdP-003: TOTP Implementation Library Choice

**Date:** 2026-02-28  
**Status:** Accepted

## Context

We need a TOTP (Time-based One-Time Password, RFC 6238) implementation for MFA. The IdP needs to:
1. Generate TOTP secrets and `otpauth://` URIs for QR codes
2. Verify 6-digit codes with a configurable time-step window
3. Work with standard authenticator apps (Google Authenticator, Authy, 1Password, Microsoft Authenticator)

## Decision

Use **[java-totp](https://github.com/samdjstevens/java-totp)** (`dev.samstevens.totp`) — a lightweight, focused Java TOTP library.

## Options Evaluated

| Library | Verdict | Notes |
|---------|---------|-------|
| **java-totp (chosen)** | ✅ Simple, focused, well-tested | Single purpose: TOTP. Does one thing well. MIT license. |
| **aerogear-otp-java** | ⚠️ Adequate but less maintained | Red Hat project; last meaningful update was 2020 |
| **GoogleAuth** | ⚠️ Adequate | Google's reference implementation; minimal API, works fine but less ergonomic |
| **Spring Security TOTP** | ❌ Doesn't exist yet | Spring Security doesn't have built-in TOTP. You'd still need a library. |
| **Roll our own** | ❌ Never | RFC 6238 is simple but getting timing, window drift, and edge cases right matters. Don't reinvent crypto. |

## Rationale

`java-totp` provides:

```java
// Generate secret
SecretGenerator secretGenerator = new DefaultSecretGenerator();
String secret = secretGenerator.generate(); // Base32-encoded 160-bit secret

// Generate QR code URI
QrData data = new QrData.Builder()
    .label("MedSales:" + userEmail)
    .secret(secret)
    .issuer("MedSales")
    .algorithm(HashingAlgorithm.SHA1)
    .digits(6)
    .period(30)
    .build();

// Verify code
TimeProvider timeProvider = new SystemTimeProvider();
CodeGenerator codeGenerator = new DefaultCodeGenerator();
CodeVerifier verifier = new DefaultCodeVerifier(codeGenerator, timeProvider);
verifier.setAllowedTimePeriodDiscrepancy(1); // ±1 time step = ±30 seconds

boolean valid = verifier.isValidCode(secret, userProvidedCode);
```

That's it. Generate, display, verify. No framework dependencies, no XML config, no abstraction layers.

### Security Considerations

| Concern | Approach |
|---------|----------|
| **Secret storage** | AES-256 encrypted in database; encryption key in AWS Secrets Manager |
| **Time window** | Allow ±1 time step (±30 seconds) to handle clock drift |
| **Replay prevention** | Track last used time step per user; reject same code twice |
| **Backup codes** | 8 one-time backup codes, bcrypt hashed, generated at enrollment |
| **Brute force** | 5 failed MFA attempts → 30-minute account lock |
| **Recovery** | Admin can reset MFA; triggers audit log entry + new enrollment required |

## Consequences

### Positive
- Simple dependency, small footprint (~50 KB)
- Compatible with all major authenticator apps
- No vendor lock-in — standard RFC 6238 TOTP
- Easy to test — inject a `TimeProvider` for deterministic testing

### Negative
- Library is maintained by a single developer (Sam Stevens)
- Mitigation: TOTP is a stable RFC; the implementation is ~500 lines of code. If abandoned, we fork it or inline it. The algorithm doesn't change.
- No built-in QR code image generation (we use a separate QR library or return the URI for client-side rendering)
