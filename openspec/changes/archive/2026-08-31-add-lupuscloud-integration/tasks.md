## 1. Scaffold & distribution

- [ ] 1.1 Create `custom_components/lupuscloud/` with `manifest.json` (domain `lupuscloud`,
  `iot_class: cloud_polling`, version, codeowners, `config_flow: true`); verify HA loads the
  integration stub with no manifest errors in the log.
- [ ] 1.2 Add `hacs.json` and `README.md` carrying the unofficial / not-affiliated /
  reverse-engineered / no-warranty / not-for-life-safety disclosure; verify the disclosure text
  is present in README, `hacs.json`/manifest metadata, and LICENSE header (`cloud-connection`
  disclosure requirement).
- [ ] 1.3 Add release workflow that builds the HACS zip from the contents of
  `custom_components/lupuscloud/`; verify the zip has files at its root (per fleet HACS packaging).

## 2. Auth & API client

- [ ] 2.1 Implement Cognito `USER_PASSWORD_AUTH` login via direct `InitiateAuth` POST (pool
  `eu-central-1_QObZU1E4n`, client `3ip8eelf4l2fatq7ficlbk5spq`); unit test asserts a valid
  credential yields ID + refresh tokens against a mocked Cognito response.
- [ ] 2.2 Implement token refresh + "refresh rejected → reauth" signalling; test that an expired
  ID token triggers a refresh and a rejected refresh raises the reauth error.
- [ ] 2.3 Implement the API client sending the **ID token** as `Authorization`, with 429 backoff;
  test that a 429 defers the next call and does not raise, and that the access token is not used.
- [ ] 2.4 Add one runnable check (`__main__`/`test_`) that parses the recorded `deviceTree`,
  `location/tree`, and `alarm` fixtures from recon into typed objects.

## 3. Config flow

- [ ] 3.1 Implement the user config flow (email/password → login → entry keyed to account UUID);
  test success creates an entry, bad creds show `invalid_auth`, duplicate aborts `already_configured`.
- [ ] 3.2 Implement the reauth flow triggered by a revoked refresh token; test it re-prompts and
  updates stored tokens.

## 4. Coordinator & entities

- [ ] 4.1 Implement a DataUpdateCoordinator polling `location/devices/deviceTree` on a
  configurable interval; test entities share one fetch per interval.
- [ ] 4.2 Implement one HA device per cloud device (unique_id from device UUID) with smoke
  `binary_sensor`; test smoke reads OFF with no alarm and the device carries type/firmware metadata.
- [ ] 4.3 Implement measurement sensors (temperature from `condition`, battery voltage + state,
  signal level); test temperature 24.0 → 24.0 °C and a missing measurement yields unavailable,
  not a fabricated value.
- [ ] 4.4 Implement heartbeat-based connectivity `binary_sensor` from `status`/`statusReason` +
  `lastHeartbeatDate`; test a stale heartbeat reads disconnected even when smoke/temp are unchanged.
- [ ] 4.5 Mark entities unavailable when a device drops out of the device tree; test the transition.

## 5. Location → areas

- [ ] 5.1 Mirror `location/tree` + device `nodeRef`/`nodeName` into HA areas and assign each
  device to its leaf-node area; test a device with a known `nodeName` lands in the "Flur" area.
- [ ] 5.2 Respect user area overrides and reassign only on `nodeRef` change; test a user-set area
  survives a poll and a changed `nodeRef` moves an un-overridden device.

## 6. Real-time alarms

- [ ] 6.1 Capture one live `ALERT_DEVICE` WebSocket frame (resolves design Open Question) and
  record it as a fixture.
- [ ] 6.2 Implement the WebSocket subscription with reconnect; test an `ALERT_DEVICE` frame flips
  the smoke sensor ON promptly without waiting for the poll.
- [ ] 6.3 Implement the `location/devices/alarm` poll fallback; test alarm present → smoke ON and
  cleared (absent + no standing event) → smoke OFF.

## 7. Release

- [ ] 7.1 End-to-end verify against the real account: add via config flow, confirm the RWM device,
  its five entities, and area assignment appear correctly; confirm no 429 storms in the log.
- [ ] 7.2 Tag a release and flip `zip_release` after testing against a throwaway HACS
  custom-repository entry; verify a clean install from HACS.
