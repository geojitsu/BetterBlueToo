---
title: 'Architecture'
description: 'As-built architecture of the fork: data flow, key layers, SwiftData models, and how the fork extends the upstream design.'
icon: 'i-heroicons-cpu-chip'
---

## Overview

The fork follows upstream's layered architecture while adding its own SwiftUI features
and the Keychain credential boundary. The current merge also adopts BetterBlueKit's
EU/Canada API fixes, the capability-driven trip history API, the cached API client, and
the shared App Intent command flow.

The upstream architecture is documented in depth in the repository's `CLAUDE.md`.

## Data Flow

```
SwiftUI Views (BetterBlue app)
  | @Query / SwiftData
  v
BBAccount / BBVehicle (SwiftData models)
  | password, PIN, serialized auth token -> KeychainService
  | account.sendCommand() / account.refreshStatus()
  v
CachedAPIClient -> APIClientFactory -> regional APIClient
  | URLSession HTTPS
  v
Hyundai / Kia BlueLink cloud API
  | JSON response
  v
SwiftData persistence -> WidgetKit / ActivityKit refresh
```

Routine re-authentication retains the `rememberMeToken` / `deviceId` trust anchors; only
an explicit session reset clears them. See
[Auth Session Recovery](/better-blue-too/api/guides/auth-session-recovery) for the full behavior.

## Key Patterns Used by This Fork

### Adding a new command (preconditioning shortcut)

Per the upstream CLAUDE.md convention:
1. `VehicleCommand.startClimate(ClimateOptions)` already exists in BetterBlueKit — no new API work.
2. Add a `BBAccount` convenience method if needed (see `lockVehicle()`, `startClimate()` as references).
3. Create new SwiftUI view component (`PreconditioningButton.swift`).
4. Wire into `VehicleCardView.swift`.
5. Status-wait must use a **60-second hard timeout** (not unbounded).

### Location access (Find My Car)

`BBVehicle.location` (`VehicleStatus.Location?`) already provides GPS coordinates from the
last status refresh. Always access via `safeLocation` guard; never force-unwrap.

### Credential security (Keychain migration)

`BBAccount` exposes `password`, `pin`, and `serializedAuthToken` as computed properties
backed by `KeychainService`, migrated from plain SwiftData fields. See
[Pitfall 4](/better-blue-too/troubleshooting#pitfall-4-keychain-migration-timing-resolved-in-task-001)
for the full migration mechanics and caveats.

### Capability-driven trip history

`BBAccount` delegates trip-history support to BetterBlueKit. The UI first checks
`supportedEVTripTypes`, then requests a summary or date-specific trip information through
`fetchEVTripSummary` and `fetchEVTripInfo`. This keeps regional API differences in the
client package rather than branching in the views. See
[Account Auth and Trip History](/better-blue-too/api/account-auth-and-trip-history) for
the API reference.

## Targets

| Target | Description |
|---|---|
| `BetterBlue` | Main iOS app — primary target for all fork changes |
| `BetterBlueWatch Watch App` | watchOS companion — out of scope for initial milestone |
| `Widget` | WidgetKit / ActivityKit extensions — battery meter changes may affect this |
| `BetterBlueKit` (submodule) | API package — no changes planned for initial milestone |
