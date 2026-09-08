## Purpose

Defines real-time alarm delivery from LUPUS Cloud into Home Assistant, with a polling fallback,
and how alarm events drive the smoke binary sensor promptly enough to matter.

## ADDED Requirements

### Requirement: Real-time alarm subscription

The integration SHALL subscribe to the LUPUS Cloud real-time notification channel (WebSocket) and
react to device alert events without waiting for the next poll interval, because NB-IoT sensors
sleep and interval polling is too slow for a smoke alarm.

#### Scenario: Alarm arrives over WebSocket
- **WHEN** an `ALERT_DEVICE` event for a known device arrives over the WebSocket
- **THEN** the integration SHALL update that device's smoke binary sensor to ON promptly, without
  waiting for the polling interval

#### Scenario: WebSocket drops
- **WHEN** the WebSocket connection is lost
- **THEN** the integration SHALL attempt to reconnect and SHALL continue to reflect alarm state
  from the polling fallback in the meantime

### Requirement: Polling fallback for alarm state

The integration SHALL also determine active-alarm state from the alarm endpoint on each poll, so
that alarm state is correct even when the real-time channel is unavailable.

#### Scenario: Alarm present at poll
- **WHEN** the alarm endpoint lists a device as in alarm
- **THEN** that device's smoke binary sensor SHALL be ON

#### Scenario: Alarm cleared
- **WHEN** a device previously in alarm no longer appears in the alarm endpoint and no active
  alert event stands
- **THEN** that device's smoke binary sensor SHALL return to OFF

### Requirement: Alarm state is not treated as system-of-record

The integration SHALL NOT present itself as a reliable safety alerting path. Documentation and
entity/device context SHALL make clear that the vendor's own alerting chain remains authoritative
for fire safety.

#### Scenario: User reads alarm documentation
- **WHEN** a user consults the integration's documentation about the smoke sensor
- **THEN** they SHALL find a statement that this integration must not be relied upon for
  life-safety alerting
