# ADR-IdP-001: Tech Stack Choice for Identity Provider

**Date:** 2026-02-28  
**Status:** Accepted

## Context

We need to choose the tech stack for the MedSales IdP microservice. The main platform API is Spring Boot 3.x on Java 21. The candidates are:

1. **Spring Boot 3.x + Java 21** — Same as the main API
2. **Node.js (Express/Fastify)** — Lightweight, fast startup
3. **Go** — Fast, small binary, strong crypto stdlib

## Decision

**Spring Boot 3.x + Java 21** using Spring Authorization Server.

## Rationale

| Factor | Spring Boot | Node.js | Go |
|--------|------------|---------|-----|
| **Team knowledge** | ✅ Primary stack | ❌ No Java devs switching | ❌ Nobody writes Go |
| **OAuth2 server framework** | ✅ Spring Authorization Server (official, maintained) | ⚠️ node-oidc-provider (community) | ⚠️ Manual or fosite (less mature) |
| **Crypto maturity** | ✅ Bouncy Castle, JCA — decades of hardening | ⚠️ node:crypto adequate but less battle-tested for IdP | ✅ Go crypto stdlib is solid |
| **Spring Security integration** | ✅ Native | ❌ N/A | ❌ N/A |
| **Operational consistency** | ✅ Same JVM, same Docker base, same Gradle, same CI | ❌ New runtime, new build, new monitoring | ❌ New everything |
| **Debugging** | ✅ Same tools (IntelliJ, JFR, VisualVM) | ❌ Different debugger, profiler | ❌ Different everything |
| **Startup time** | ⚠️ 3-8 seconds | ✅ < 1 second | ✅ < 1 second |
| **Memory footprint** | ⚠️ 256-512 MB | ✅ 50-100 MB | ✅ 20-50 MB |

**Startup time and memory don't matter here.** The IdP is a long-running service — it starts once and stays up. We're not doing serverless functions. A few hundred MB of RAM is nothing compared to the operational cost of maintaining a separate tech stack.

**Spring Authorization Server** is the official Spring project for building OAuth 2.0 authorization servers. It handles the protocol complexity (PKCE, token rotation, JWKS) and lets us focus on our multi-tenant and MFA customizations.

## Consequences

### Positive
- Zero onboarding cost — same stack the team already uses
- Shared Gradle build, Docker base image, CI/CD pipeline patterns
- Spring Security's password encoding, CSRF, session management out of the box
- Easy to share domain classes (User, Organization) between API and IdP if we go monorepo

### Negative
- Higher memory footprint than Go/Node alternatives (~512 MB vs ~50 MB)
- Slower cold start (irrelevant for a long-running service)
- Spring Authorization Server has a learning curve (different from Spring Security's resource server)

### Risks
- Spring Authorization Server is relatively newer than Keycloak — smaller community for troubleshooting
- Mitigation: The Spring team actively maintains it, and we have the source code if we need to debug
