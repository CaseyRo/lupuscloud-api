## Purpose

Defines how one LUPUS Cloud device (initially the RWM cellular smoke detector) is represented in
Home Assistant as a device with a fixed set of entities derived from the cloud device record.

## ADDED Requirements

### Requirement: One HA device per cloud device

The integration SHALL create one Home Assistant device per LUPUS Cloud device, identified by the
cloud device UUID as its unique identifier, and SHALL carry the cloud `deviceType`, firmware
version, and human name as device metadata. Entity unique IDs SHALL be derived from the immutable
device UUID so they remain stable across restarts and renames.

#### Scenario: Device appears
- **WHEN** the device tree contains a device with type RWM
- **THEN** the integration SHALL register one HA device with entities for smoke, temperature,
  battery, signal, and connectivity

#### Scenario: Device removed from account
- **WHEN** a previously present device no longer appears in the device tree
- **THEN** its entities SHALL become unavailable rather than reporting a stale last value

### Requirement: Smoke binary sensor

Each RWM device SHALL expose a `binary_sensor` with device class `smoke` whose ON state reflects
an active smoke/fire alarm for that device.

#### Scenario: No alarm
- **WHEN** the device has no active alarm
- **THEN** the smoke binary sensor SHALL report OFF

### Requirement: Measurement entities

Each device SHALL expose sensor entities for the measurements present in its cloud record:
temperature (°C, from the device `condition`), battery voltage and battery state, and signal
level. A measurement absent from the record SHALL yield an unavailable entity, not a fabricated value.

#### Scenario: Temperature present
- **WHEN** the device record includes a condition temperature of 24.0
- **THEN** the temperature sensor SHALL report 24.0 °C

#### Scenario: Weak signal
- **WHEN** the device record reports signal level WEAK
- **THEN** the signal sensor SHALL report WEAK

### Requirement: Heartbeat-based connectivity

Each device SHALL expose a `binary_sensor` with device class `connectivity` whose state is
derived from the device `status`/`statusReason` and its `lastHeartbeatDate`, NOT from whether a
measured value has changed. A device the cloud reports as inactive or whose last heartbeat is
older than a staleness threshold SHALL read as disconnected.

#### Scenario: Healthy device
- **WHEN** the device status is ACTIVE/OK and its last heartbeat is recent
- **THEN** the connectivity sensor SHALL report connected

#### Scenario: Stale heartbeat
- **WHEN** the device's last heartbeat is older than the staleness threshold, or the cloud
  reports the device inactive
- **THEN** the connectivity sensor SHALL report disconnected even if the last smoke/temperature
  values are unchanged
