## Purpose

Defines how the LUPUS Cloud "Standort" location tree is mirrored into Home Assistant areas and
how each device is assigned to the area matching its location node.

## ADDED Requirements

### Requirement: Device assigned to its location-node area

Each device SHALL be assigned to a Home Assistant area corresponding to its LUPUS location leaf
node, identified by the device's `nodeRef` (node UUID). The area name SHALL be derived from the
node so a user recognizes it (e.g. the leaf room name from the `nodeName` path
`"City > Street > Floor > Room"`).

#### Scenario: Device placed in its room
- **WHEN** a device reports `nodeName` `"Musterstadt > Musterstraße 1 > EG > Flur"` with a stable `nodeRef`
- **THEN** the device SHALL be assigned to an HA area representing that leaf node ("Flur")

#### Scenario: Stable across polls
- **WHEN** the same device is seen on a later poll with the same `nodeRef`
- **THEN** it SHALL remain in the same HA area and SHALL NOT create a duplicate area

### Requirement: Respect user area overrides

The integration SHALL NOT overwrite an area a user has manually assigned to a device on
subsequent polls. Automatic assignment applies when the device has no user-set area.

#### Scenario: User moved the device
- **WHEN** a user has manually placed the device in a different HA area
- **THEN** later polls SHALL leave the user's assignment untouched

### Requirement: Node change moves the device

The integration SHALL move a device to the area of its new location node when the cloud reports
the device's `nodeRef` has changed.

#### Scenario: Device relocated in LUPUS Cloud
- **WHEN** a device's `nodeRef` changes between polls and the user had not overridden its area
- **THEN** the device SHALL be reassigned to the area of the new node
