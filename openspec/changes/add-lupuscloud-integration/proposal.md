> **STATUS: PARKED — not being built.** This change was written to evaluate a Home
> Assistant integration for LUPUS Cloud. After exploring all angles it was deliberately
> shelved. Reasons: (1) LUPUS Cloud already offers a *supported* outbound webhook
> (`service/webhook`) that delivers real-time alarms to any URL — the one genuinely useful
> thing — with no code and no fragile reverse-engineered polling; (2) the valuable half of
> the platform (OMS submetering, consumption, invoicing) is permission-gated to
> installer/company accounts and inaccessible to a normal user; (3) the addressable audience
> is tiny (a B2B compliance product, not a consumer device). The lasting deliverable of this
> repo is therefore the **API documentation** in `docs/`, not an integration. This proposal
> and its specs remain as the evaluation/decision record. See `docs/lupuscloud-api.md`.

## Why

LUPUS-Electronics' **LUPUS Cloud** platform (`api.lupusiot.de`) runs the company's
NB-IoT cellular sensors — the mobile smoke detector (RWM), motion, and temperature/humidity
sensors that phone home over Deutsche Telekom's narrowband network. As of 2026-08-31 **no
Home Assistant integration, custom component, or reverse-engineered client library exists
for this platform** (verified via authenticated GitHub code/repo search and the HA
integrations catalog). Every existing "Lupus" HA project — HA core `lupusec`, `Lupusec2Mqtt`,
`lupus-to-mqtt` — targets the *local LupusEC XT alarm panel* on the LAN, a completely
different product line with a different API and auth. Owners of a LUPUS Cloud smoke detector
currently have no way to see it in Home Assistant.

This change adds an **unofficial, community, use-at-your-own-risk** HACS integration that
surfaces LUPUS Cloud devices in Home Assistant: each cellular smoke detector as an HA device
with smoke/temperature/battery/signal/connectivity entities, placed into the HA area that
matches its LUPUS "Standort" (location node).

### Positioning and disclaimer (non-negotiable, must ship in README + manifest + repo)

- **Unofficial.** Not affiliated with, endorsed by, or supported by LUPUS-Electronics GmbH.
  "LUPUS", "LUPUSEC", and "LUPUS Cloud" are trademarks of their owner, used here only to
  describe compatibility.
- **Reverse-engineered.** Built against an undocumented private API observed from the
  vendor's own web app. There is no public API contract; LUPUS may change or break it at
  any time, without notice, and this integration may stop working as a result.
- **No warranty, no liability.** Provided "AS IS", without warranty of any kind. The authors
  accept **no responsibility** for any damage, data loss, missed alarms, false alarms,
  account lockout, or safety consequences arising from its use. **Safety-critical alerting
  (fire, smoke) must not rely on this integration** — the vendor's own app/cloud alerting
  chain remains the system of record.
- **Your credentials, your risk.** The integration authenticates as the user against the
  vendor's Cognito pool; the user is responsible for their own account and terms of use.

## What Changes

- **NEW**: A HACS-installable Home Assistant custom integration (`custom_components/lupuscloud/`)
  that connects to LUPUS Cloud and creates read-only entities for the account's devices.
- **NEW**: Config-flow onboarding — user enters LUPUS Cloud email + password; the integration
  authenticates against AWS Cognito (`USER_PASSWORD_AUTH`, no reCAPTCHA on the token endpoint)
  and stores the refresh token. No local IP, no LAN access (cloud-only device line).
- **NEW**: A DataUpdateCoordinator polls `location/devices/deviceTree` on an interval and fans
  each device out into HA entities:
  - `binary_sensor` smoke (device_class `smoke`)
  - `sensor` temperature (from `condition.temperature`)
  - `sensor` battery voltage + battery state
  - `sensor` signal level
  - `binary_sensor` connectivity (device_class `connectivity`, driven by `status`/`statusReason`
    and `lastHeartbeatDate` as an explicit heartbeat — so a "clear" smoke reading and a dead
    SIM are not indistinguishable)
- **NEW**: Standort → HA area mapping. Each device carries `nodeRef` (room node UUID) and
  `nodeName` (`"City > Street > Floor > Room"`); the integration mirrors the LUPUS location
  tree (`location/tree`) into HA areas and assigns each device to the area of its leaf node.
- **NEW**: Real-time alarm path. Subscribe to the LUPUS WebSocket (`notifications/ws`, base
  host minus `/api/v1`) for `ALERT_DEVICE` events; fall back to polling `location/devices/alarm`.
  Push matters because NB-IoT sensors sleep and interval polling is too slow for a fire alarm.
- **NEW**: Repository docs establishing the unofficial / own-risk / no-liability positioning
  above (README, LICENSE, manifest metadata, HACS config).

Out of scope for this change (see Non-goals in design): writing HA's setup *back* into LUPUS
Cloud, OMS/meter consumption entities, motion/THS sensor types beyond RWM, and multi-account
/ impersonation flows.

## Capabilities

### New Capabilities
- `cloud-connection`: Cognito authentication, config-flow onboarding, token lifecycle
  (refresh + re-auth), the update coordinator, rate-limit backoff, and the unofficial /
  own-risk positioning as an explicit requirement.
- `device-entities`: mapping one LUPUS Cloud device (RWM smoke detector) to its set of Home
  Assistant entities, including the heartbeat-based connectivity/liveness entity.
- `location-areas`: mirroring the LUPUS Standort location tree into Home Assistant areas and
  assigning each device to the area of its location node.
- `alarm-notifications`: real-time alarm delivery via the LUPUS WebSocket with a poll-based
  fallback, and how alarm state maps onto the smoke `binary_sensor`.

### Modified Capabilities
<!-- None — this is a greenfield repo with no existing specs. -->

## Impact

- **New code**: `custom_components/lupuscloud/` (manifest, config_flow, coordinator, auth,
  api client, binary_sensor, sensor platforms), `hacs.json`, README, LICENSE.
- **New runtime dependency**: an AWS Cognito SRP/USER_PASSWORD auth path. Prefer a small pure
  approach (direct `InitiateAuth` JSON calls) over pulling in `boto3` to keep the HA install
  footprint small; `pycognito` is the fallback if SRP is later required.
- **External dependency (uncontrolled)**: the private `api.lupusiot.de` API and its Cognito
  pool. No SLA, no versioning guarantee — this is the standing risk the disclaimer covers.
- **Distribution**: HACS custom repository (not HA core). Unofficial, so it will not be
  submitted to HA core.
- **No impact** on the existing HA `lupusec` (XT panel) integration — different product,
  different code, no shared surface.
