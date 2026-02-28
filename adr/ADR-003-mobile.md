# ADR-003: Mobile Approach — React Native vs Flutter vs Native

**Date:** 2026-02-28
**Status:** Accepted
**Author:** Frank Reynolds

## Context

MedSales is mobile-first. Requirements:
- iOS (16+) and Android (12+) simultaneously
- GPS location services for nearby search and mapping
- Offline mode (cached profiles, offline call logging, sync on reconnect)
- Push notifications for task reminders
- Map view with 500+ physician pins
- App Store and Play Store compliant (NFR-029)
- Camera access not required for v1
- No AR, Bluetooth, or hardware-intensive features

Team context: Spring Boot backend team (Java). No existing mobile team.

Options:
1. **React Native** — JavaScript/TypeScript, shared codebase, native bridge
2. **Flutter** — Dart, shared codebase, custom rendering engine
3. **Native (Swift + Kotlin)** — Two separate codebases

## Decision

**React Native** for iOS and Android from a single codebase.

### Comparison

| Criterion | React Native | Flutter | Native (Swift/Kotlin) |
|---|---|---|---|
| Code sharing | ~90% shared | ~95% shared | 0% shared |
| Development speed | ✅ Fast, single team | ✅ Fast, single team | ❌ Two teams needed |
| Hiring pool | ✅ Large (JS/TS devs) | ⚠️ Growing (Dart devs) | ⚠️ Specialized per platform |
| Maps / geo support | ✅ react-native-maps (mature) | ✅ google_maps_flutter | ✅ Native MapKit / Google Maps |
| Offline / SQLite | ✅ WatermelonDB, Realm, SQLite | ✅ Hive, sqflite | ✅ Core Data, Room |
| Push notifications | ✅ Firebase + native | ✅ Firebase + native | ✅ Native |
| GPS location | ✅ react-native-geolocation | ✅ geolocator | ✅ Native |
| Performance (500 pins) | ✅ Adequate (clustered) | ✅ Good | ✅ Best |
| App Store compliance | ✅ Proven | ✅ Proven | ✅ Native |
| Hot reload / dev speed | ✅ Yes | ✅ Yes | ❌ No |
| Backend integration | ✅ REST/JSON natural | ✅ REST/JSON natural | ✅ REST/JSON natural |
| Long-term maintenance | ✅ 1 codebase | ✅ 1 codebase | ❌ 2 codebases |

### Why React Native Over Flutter

1. **Hiring.** JavaScript/TypeScript developers are everywhere. Dart developers are not. For a startup/small team, this matters. A lot.

2. **Ecosystem maturity.** React Native has been in production at Meta, Microsoft, Shopify, and Discord since 2015. The mapping, offline, and geo libraries are battle-tested. Flutter is mature too, but RN's ecosystem for our specific needs (maps, offline sync, forms) is deeper.

3. **Web companion synergy.** Phase 3 includes a web companion app for managers. React Native + React Web share component patterns, state management (Redux/Zustand), and developer knowledge. Flutter web exists but is less mature for business apps.

4. **Spring Boot team alignment.** The backend team knows Java, not Dart. JavaScript is a shorter learning curve if backend devs need to contribute to mobile.

### Why Not Native

1. **Cost.** Two codebases means two teams, double the bugs, double the testing, double the release cycles. For a v1 product without hardware-intensive features, this is waste.

2. **Speed.** React Native ships both platforms in one development cycle. Native means iOS ships first, Android catches up 3–6 months later, or vice versa.

3. **MedSales doesn't need native performance.** We're displaying lists, maps, forms, and profiles. This is not a game or video editor. React Native handles this workload easily.

## Key Libraries

| Need | Library |
|---|---|
| Navigation | React Navigation |
| Maps | react-native-maps (Google Maps provider) |
| GPS | @react-native-community/geolocation |
| Offline storage | WatermelonDB (SQLite-backed, sync support) |
| Push notifications | @react-native-firebase/messaging |
| State management | Zustand or Redux Toolkit |
| HTTP client | Axios or React Query |
| Forms | React Hook Form |
| UI components | React Native Paper or NativeBase |

## Consequences

### Positive
- Single codebase, single team, ship both platforms simultaneously
- Large hiring pool (JS/TS)
- Proven at scale (Meta, Shopify, Discord)
- Synergy with Phase 3 React web app
- Faster v1 delivery

### Negative
- Not truly native — occasional platform-specific bugs or UX differences
- Map performance with 500+ pins requires clustering optimization
- Offline sync (WatermelonDB) adds complexity vs simple native Core Data/Room
- React Native major version upgrades can be painful (though much improved since New Architecture)
- Some native modules may require Swift/Kotlin bridge code

### Mitigations
- Use React Native New Architecture (Fabric + TurboModules) from day one
- Implement map pin clustering to handle 500+ pins (supercluster library)
- Budget 2 weeks for offline sync architecture with WatermelonDB
- Set up Detox for E2E testing on both platforms

---

*React Native. One codebase. Two platforms. I'm not paying for two teams when one team can do the job. That's just business. — Frank*
