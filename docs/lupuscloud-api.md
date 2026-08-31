# LUPUS Cloud API — unofficial reference

Reverse-engineered notes on the **LUPUS Cloud** IoT platform API (`api.lupusiot.de`), the
backend behind the [lupuscloud.com](https://www.lupuscloud.com) web app and the *LUPUS Cloud*
mobile apps. This is the platform for LUPUS-Electronics' **NB-IoT cellular sensors** (mobile
smoke detector, motion, temperature/humidity, OMS meters) — a different product line from the
local **LupusEC XT** alarm panels covered by Home Assistant's built-in `lupusec` integration.

> ⚠️ **Unofficial and reverse-engineered.** Not affiliated with, endorsed by, or supported by
> LUPUS-Electronics GmbH. Everything here was observed by inspecting the vendor's own public web
> app and a single personal account. There is **no public API contract** — LUPUS may change or
> break any of this at any time, without notice. See [DISCLAIMER](../README.md#disclaimer).
> All example values below are placeholders or fictional; no real account data is included.

Observed: 2026-08-31. Backend is Spring Boot; list responses are Spring `Page` objects.

---

## 1. Base URL & how it was obtained

- **API base:** `https://api.lupusiot.de/api/v1`
- **Web app:** `https://www.lupuscloud.com` — an Angular SPA. Its JavaScript bundle (`main.*.js`
  plus lazy-loaded chunks referenced from `runtime.*.js`) contains the environment config and
  every API path, built as `apiURL + "/" + <resource>`. The config block:

  ```js
  { production: true,
    cognitoPoolId: "eu-central-1_QObZU1E4n",
    cognitoClientId: "3ip8eelf4l2fatq7ficlbk5spq",
    apiGatewayUrl: "https://api.lupusiot.de/api/v1" }
  ```

These identifiers are shipped to every visitor in the public web bundle; they are not secrets
(the Cognito app client is public and has no client secret).

## 2. Authentication (AWS Cognito)

| Field | Value |
|---|---|
| Provider | AWS Cognito User Pools, region `eu-central-1` |
| User pool | `eu-central-1_QObZU1E4n` |
| App client | `3ip8eelf4l2fatq7ficlbk5spq` (public, no secret) |
| Flow | `USER_PASSWORD_AUTH` (also `USER_SRP_AUTH`, `CUSTOM_AUTH` in the bundle) |

Log in with a plain, **unsigned** `InitiateAuth` POST — no reCAPTCHA (the captcha only guards the
web login form, not the token endpoint):

```http
POST https://cognito-idp.eu-central-1.amazonaws.com/
Content-Type: application/x-amz-json-1.1
X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth

{ "AuthFlow": "USER_PASSWORD_AUTH",
  "ClientId": "3ip8eelf4l2fatq7ficlbk5spq",
  "AuthParameters": { "USERNAME": "<email>", "PASSWORD": "<password>" } }
```

**Authorization header quirk:** send the Cognito **ID token** as the raw `Authorization` header
value (no `Bearer ` prefix). The **access token is rejected with `401`.** Renew with the standard
Cognito `REFRESH_TOKEN_AUTH` flow.

## 3. Conventions

- Lists come back as Spring `Page`: `{ "content": [...], "totalPages", "last", "first", "empty", "sort" }`.
- Errors: either `{ "timestamp", "path", "status", "error", "requestId" }` (framework) or
  `{ "timestamp", "action", "error": { "code": "DEVICES_046", "message", "userMessage" } }` (domain).
- **Rate limiting:** the API returns `429` under rapid requests. Poll gently and back off.
- **Account tiers gate access.** A normal (`PERSONAL`) account can read its own devices,
  locations, notifications, and alarm state, but installer/company endpoints return
  `400 DEVICES_601 "User has no system permission"` (e.g. `devices/manage/list`). The OMS /
  consumption / invoicing surface is effectively installer/company-only.

## 4. Read endpoints (confirmed on a personal account)

All `GET` unless noted. Prefix every path with the base URL.

| Endpoint | Returns |
|---|---|
| `account_details/account` | `{ uuid, email, cognitoUsername, accountType, mfa, ... }` |
| `location/tree` | Standort hierarchy — see [§6](#6-location-tree-standort) |
| `location/devices/deviceTree` | **Primary device list** with location + live condition — see [§5](#5-device-record) |
| `location/devices/alarm` | `Page` of devices currently in alarm (empty when none) |
| `location/devices/filteredSearchPaginated` | Paginated device search |
| `devices/filteredSearch`, `devices/` | Device lists |
| `notifications` | `Page` of alert/notification history — see [§7](#7-notifications) |
| `notifications/unreadCount`, `notifications/ws` | Unread count; WebSocket handle |
| `location/statistics`, `location/statistics/tree` | Aggregate stats over the tree |
| `profiles/byAccountId`, `company/byAccount/` | Profile / company info |

## 5. Device record

One entry of `location/devices/deviceTree` (a smoke detector, `deviceType: RWM`). **Values below
are illustrative placeholders.**

```json
{
  "uuid": "<device-uuid>",
  "deviceId": "<iccid>",
  "firmwareVersion": "1.19",
  "status": "ACTIVE",
  "statusReason": "OK",
  "batteryLevel": 2.91,
  "batteryState": "ACTIVE",
  "signalLevel": "WEAK",
  "lastHeartbeatDate": 1788175696850,
  "deviceType": "RWM",
  "nodeRef": "<node-uuid>",
  "nodeName": "Musterstadt > Musterstraße 1 > EG > Flur",
  "nodeParents": "<buildingUuid>/<floorUuid>",
  "condition": { "temperature": 24.0 }
}
```

Notes:
- `deviceId` is the NB-IoT SIM **ICCID**.
- `lastHeartbeatDate` (epoch ms) is an explicit heartbeat — use it (with `status`/`statusReason`)
  to judge liveness, not the last measured value.
- `condition` carries live readings (temperature seen; other fields may appear per device type).
- `nodeRef` + `nodeName` place the device in the Standort tree; `nodeName` is a
  `"City > Street > Floor > Room"` breadcrumb.

**Device types** (`deviceType` enum): `RWM` (smoke / Rauchwarnmelder), `THS`
(temperature/humidity), `MOTION`, `SWAN`, `OMS` (wireless M-Bus consumption meter), `MIOTY`.

## 6. Location tree (Standort)

`GET location/tree` (placeholders):

```json
{
  "privateHouses": [
    {
      "uuid": "<building-uuid>",
      "type": "BUILDING",
      "name": "Home",
      "owned": true,
      "address": {
        "country": "DE", "city": "<city>", "streetAddress": "<street>",
        "number": "<no>", "zipCode": "<zip>", "latitude": "<lat>", "longitude": "<lng>"
      }
    }
  ]
}
```

The tree is `country → city → building → floor → room` nodes. Company/installer accounts also
expose `location/buildings/*` and `location/building_structure/*` for editing the structure.

## 7. Notifications

`GET notifications` → `Page` of:

```json
{
  "uuid": "<uuid>",
  "createdDate": 1788172033450,
  "group": "PLATFORM",
  "eventType": "ALERT_DEVICE",
  "type": "WEB_SOCKET",           // or EMAIL, PUSH
  "detailedMessage": "Warning ... triggered by device ... at location ...",
  "read": true,
  "status": "SENT",
  "accountUuid": "<account-uuid>",
  "resourceType": "DEVICE",
  "resourceUuid": "<device-uuid>"
}
```

Real-time delivery is available over a WebSocket at the API host **minus** `/api/v1`
(`getWebSocketApiUrl()` in the SPA), plus `GET notifications/ws`.

## 8. Outbound webhook & integrations (the useful part)

LUPUS Cloud can **push events to an arbitrary URL** — a supported feature, not a hack. This is
the simplest way to get LUPUS events into another system (e.g. Home Assistant's
`/api/webhook/<id>`), with no polling and no Cognito.

CRUD via `service/webhook` (`GET` / `POST` / `PUT` / `DELETE`) and a test trigger at
`service/webhook/test`. Config fields observed in the SPA:

| Field | Meaning |
|---|---|
| `webhookUrl` | Target URL LUPUS calls |
| `webhookBearer` | Bearer token sent to the target |
| `webhookUsername` / `webhookPassword` | Basic-auth alternative |
| `webhookCustom` | Custom payload |
| `webhookEnabled` | On/off |
| active events | Per-event subscription (see below) |

Built-in integration modules also include **Alamos FE2** (`INTEGRATION.ALAMOS.*`), and this
webhook mechanism is the same one LUPUS documents for Feuersoftware Connect and GroupAlarm
(`https://connectapi.feuersoftware.com/interfaces/lupus/webhook`, bearer token).

**Selectable event types:** `FIRE_ALARM_DEVICE`, `HEAT_ALARM_DEVICE`, `HIGH_TEMPERATURE_ALARM_DEVICE`,
`LOW_TEMPERATURE_ALARM_DEVICE`, `RISING_TEMPERATURE_ALARM_DEVICE`, `MOTION_ALARM_DEVICE`,
`PANIC_ALARM_DEVICE`, and their `*_ACKNOWLEDGED` counterparts.

> The webhook is **event-only** — alarms, not steady-state telemetry. It cannot enumerate
> devices or report current temperature/battery/signal; for that you must poll the API in §4.

## 9. Installer / company surface (permission-gated)

Not accessible to a personal account, documented here for completeness. This is where the
platform's real business — property submetering and billing — lives.

| Area | Endpoints |
|---|---|
| Device management | `devices/manage`, `devices/manage/list`, `devices/manage/rwm/config[/template]`, `devices/preregister`, `devices/name`, `devices/decommissionDevice` |
| Device config | `devices/thsConfig`, `devices/swanConfig`, `devices/dataUsage/`, `devices/extra` |
| OMS / metering | `devices/oms/readings`, `devices/oms/readings/byApartments`, `devices/oms/aggregation`, `devices/oms/device/key`, `devices/oms/plausibility-check/byDevices`, `devices/consumption`, `devices/mioty/log`, `reports/compareConsumption` |
| Location editing | `location/` (POST), `location/buildings[/duplicate]`, `location/building_structure/{addNode,updateNode,duplicateNode}`, `location/templates` |
| Invoicing | `invoice/byAll`, `invoice/byCompany`, `invoice/contract[s]/*`, `invoice/price`, `invoice/markPaid`, `invoice/download` |
| Sharing / roles | `shared-permissions`, `sharing/resource-permissions`, `account-permissions/*`, `permissions/roles/augmented`, `ownership-transfers`, impersonation via an `impersonate-account-uuid` header |

## 10. Why no Home Assistant integration was built

An HA integration was scoped (see `openspec/changes/add-lupuscloud-integration/`, kept as a
decision record) and deliberately shelved:

1. **The webhook already solves the real need.** For "get a smoke alarm into Home Assistant,"
   point `service/webhook` at an HA webhook URL and drive automations — supported, near-zero
   code, and it survives API changes. A reverse-engineered polling component only adds
   telemetry dashboards on top.
2. **The valuable half is locked.** OMS submetering → HA Energy dashboard would be the
   differentiated feature, but it's installer/company-gated and can't even be tested on a normal
   account.
3. **Tiny audience.** LUPUS Cloud is a B2B compliance product (Rauchmelderpflicht /
   Heizkostenverordnung), not a consumer smart-home device — so a community integration has
   little reach.

If you want LUPUS Cloud events in Home Assistant today, use the webhook (§8).
