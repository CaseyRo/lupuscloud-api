# lupuscloud — unofficial LUPUS Cloud API notes

Reverse-engineered documentation of the **LUPUS Cloud** IoT platform API (`api.lupusiot.de`),
the backend behind [lupuscloud.com](https://www.lupuscloud.com) and the *LUPUS Cloud* mobile
apps — the platform for LUPUS-Electronics' **NB-IoT cellular sensors** (mobile smoke detector,
motion, temperature/humidity, OMS meters).

**This repo is documentation, not an integration.** A Home Assistant integration was evaluated
and deliberately parked — the short version is that LUPUS Cloud already has a supported outbound
webhook that covers the one genuinely useful case, and the valuable half of the platform is
locked behind installer/company accounts. See
[`docs/lupuscloud-api.md` §10](docs/lupuscloud-api.md#10-why-no-home-assistant-integration-was-built).

## What's here

- **[`docs/lupuscloud-api.md`](docs/lupuscloud-api.md)** — the API reference: Cognito auth, the
  ID-token quirk, endpoint map, device/location/notification schemas, event types, the outbound
  webhook (`service/webhook`), and the permission-gated installer surface.
- **`openspec/changes/add-lupuscloud-integration/`** — the (parked) evaluation of an HA
  integration, kept as a decision record.

## Want LUPUS Cloud events in Home Assistant?

You don't need this repo's blessing or any custom code. LUPUS Cloud can **push** alarm events to
any URL via its built-in webhook — point it at a Home Assistant webhook and drive automations.
See [`docs/lupuscloud-api.md` §8](docs/lupuscloud-api.md#8-outbound-webhook--integrations-the-useful-part).

## Not this: LupusEC XT panels

If you have a local **LupusEC XT1/XT2** alarm panel (a box on your LAN), that is a different
product with its own support — use Home Assistant's built-in
[`lupusec`](https://www.home-assistant.io/integrations/lupusec/) integration instead. This repo
is only about the **cloud** NB-IoT platform.

## Disclaimer

- **Unofficial.** Not affiliated with, endorsed by, or supported by LUPUS-Electronics GmbH.
  "LUPUS", "LUPUSEC", and "LUPUS Cloud" are trademarks of their respective owner, used here
  only to describe compatibility.
- **Reverse-engineered.** Compiled by inspecting the vendor's own public web app. There is no
  public API contract; the API is undocumented and may change or break at any time without
  notice.
- **No warranty, no liability.** Provided **AS IS**, without warranty of any kind. The authors
  accept **no responsibility** for any damage, data loss, missed or false alarms, account
  lockout, or other consequences arising from use of this information. **Do not rely on anything
  derived from these notes for life-safety alerting** (fire/smoke) — the vendor's own app and
  alerting chain remain the system of record.
- **Your account, your responsibility.** Accessing the API uses your own LUPUS Cloud
  credentials; you are responsible for complying with LUPUS-Electronics' terms of use.
- No real account data (credentials, addresses, device IDs) is included in this repo; all
  example values are placeholders.

## License

[MIT](LICENSE).
