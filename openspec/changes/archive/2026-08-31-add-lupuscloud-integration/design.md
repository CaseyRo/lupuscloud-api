## Context

See `proposal.md` — Why. This section records the reverse-engineering facts the implementation
depends on, all confirmed live against a real account on 2026-08-31. The API is private and
undocumented; these are observations, not a contract.

**Base:** `https://api.lupusiot.de/api/v1` — a Spring Boot backend. List responses are Spring
`Page` objects (`content[]`, `totalPages`, `last`, …).

**Auth:** AWS Cognito.
- Region `eu-central-1`, user pool `eu-central-1_QObZU1E4n`, app client `3ip8eelf4l2fatq7ficlbk5spq`
  (public client, no secret).
- `USER_PASSWORD_AUTH` via a plain `InitiateAuth` JSON POST to
  `https://cognito-idp.eu-central-1.amazonaws.com/` works and requires **no reCAPTCHA** (the
  captcha is only on the web SPA login form, not the token endpoint).
- The API authorizer accepts the **ID token** as the raw `Authorization` header value. The
  access token is rejected with 401. Standard Cognito refresh-token flow for renewal.
- WebSocket base = API base minus `/api/v1`; the app also calls `GET notifications/ws`.

**Endpoints that matter here** (all GET unless noted):

| Endpoint | Use |
|---|---|
| `location/tree` | Standort hierarchy (`privateHouses[]` → building → nodes) |
| `location/devices/deviceTree` | Primary poll: array of devices with location + live condition |
| `location/devices/alarm` | Devices currently in alarm (Page; empty when none) |
| `notifications` / `notifications/ws` | Alert history (Page) and real-time channel |
| `account_details/account` | Account UUID, email, type |

**Device record shape** (from `deviceTree`, one real RWM smoke detector):
```json
{ "uuid": "<device-uuid>", "deviceId": "<iccid>", "firmwareVersion": "1.19",
  "status": "ACTIVE", "statusReason": "OK",
  "batteryLevel": 2.91, "batteryState": "ACTIVE", "signalLevel": "WEAK",
  "lastHeartbeatDate": 1788175696850, "deviceType": "RWM",
  "nodeRef": "<node-uuid>", "nodeName": "Musterstadt > Musterstraße 1 > EG > Flur",
  "nodeParents": "<buildingUuid>/<floorUuid>",
  "condition": { "temperature": 24.0 } }
```

## Goals / Non-Goals

**Goals**
- Read-only surfacing of the account's LUPUS Cloud devices in HA with stable entity IDs.
- Single-account, single-coordinator, minimal-dependency implementation.

**Non-Goals**
- Writing HA's setup *back* into LUPUS Cloud. The API has write endpoints (`POST location/`,
  `location/building_structure/addNode`, `POST devices`, `devices/name`, …) but they only
  register LUPUS's own NB-IoT hardware; arbitrary HA entities cannot be pushed. Deferred /
  likely never.
- OMS/mioty meter consumption entities (`devices/oms/*`, `devices/consumption`).
- Device types beyond RWM (motion, THS) in the first cut — the record shape generalizes, but
  scope is the smoke detector first.
- Multi-account and impersonation (`impersonate-account-uuid` header) support.
- Submission to HA core — it is unofficial and stays in HACS.

## Decisions

- **Direct Cognito `InitiateAuth` over `boto3`/`pycognito`.** `USER_PASSWORD_AUTH` is a single
  unsigned JSON POST; implementing it directly keeps the HA dependency footprint tiny. Fallback:
  `pycognito` if LUPUS ever disables `USER_PASSWORD_AUTH` and forces SRP (the SRP math is not
  worth hand-rolling). Alternative rejected: `boto3` — heavyweight for one call.
- **`deviceTree` as the single poll source.** It already carries location (`nodeRef`/`nodeName`),
  liveness (`status`/`lastHeartbeatDate`), and the live `condition` in one call, so one request
  per interval feeds every entity. `alarm` is polled additionally only for alarm state.
- **Heartbeat drives connectivity, not value-change.** `lastHeartbeatDate` + `status` are the
  liveness signal; a steady "clear" smoke reading must never be mistaken for a live feed
  (known HA trap — and this device already emits "Sensor inaktiv"). Staleness threshold is a
  tunable knob, seeded from the device's own reporting cadence.
- **WebSocket for alarms, poll as fallback.** Push is required for a fire alarm to be timely;
  the poll keeps state correct when the socket is down.
- **HACS custom repository distribution** with `zip_release` handled per the fleet's HACS
  packaging notes (zip the contents of `custom_components/lupuscloud/`, files at archive root).

## Risks / Trade-offs

- **Private API changes/breaks without notice** → the disclaimer sets expectations; pin nothing
  we can avoid, fail soft (entities unavailable, reauth flow), and keep the endpoint map in one
  module so a shape change is a localized fix.
- **Rate limiting (observed 429 during probing)** → single coordinator, conservative interval,
  exponential backoff on 429, never hammer.
- **Safety-critical misuse** → explicit, repeated "not for life-safety" statements in README,
  manifest, and alarm docs; never market it as an alarm system.
- **Flaky NB-IoT link (real device shows `signalLevel: WEAK`)** → connectivity entity makes the
  flakiness visible instead of hiding it behind a stale "clear" reading.
- **Credentials / ToS** → user authenticates as themselves; document that they own their account
  usage. Store only the refresh token via HA's config-entry storage.

## Migration Plan

Greenfield; nothing to migrate. Ship as a HACS custom repo. Rollback = uninstall the
integration; it holds no server-side state beyond what the LUPUS app already shows. Follow the
fleet HACS packaging sequence (release workflow first, flip `zip_release` after testing against a
throwaway custom-repository entry).

## Open Questions

- Exact real-time `ALERT_DEVICE` WebSocket frame shape (envelope/subscribe handshake) — needs one
  captured live frame; does not change specs or task breakdown, only the parser in the alarm module.
- Whether `condition` ever carries humidity/other fields for RWM, or only temperature — additive
  if so; measurement entities already spec "present in the record" semantics.
