## Purpose

Defines how the integration authenticates to LUPUS Cloud, onboards through the Home Assistant
config flow, keeps its session alive, polls the cloud within rate limits, and surfaces its
unofficial / use-at-your-own-risk nature to the user.

## ADDED Requirements

### Requirement: Config-flow onboarding with LUPUS Cloud credentials

The integration SHALL onboard through the Home Assistant UI config flow using the user's LUPUS
Cloud account email and password. It SHALL NOT require any local network address, because the
supported devices are cloud-only (NB-IoT).

#### Scenario: Successful login
- **WHEN** the user enters valid LUPUS Cloud email and password
- **THEN** the integration authenticates against the LUPUS Cognito user pool and creates a
  config entry keyed to the account UUID

#### Scenario: Invalid credentials
- **WHEN** the user enters an email/password the Cognito pool rejects
- **THEN** the config flow SHALL show an `invalid_auth` error and SHALL NOT create a config entry

#### Scenario: Duplicate account
- **WHEN** the user tries to add an account that already has a config entry
- **THEN** the config flow SHALL abort with `already_configured`

### Requirement: Token lifecycle

The integration SHALL cache the refresh token and use it to obtain new access/ID tokens without
re-prompting the user. It SHALL send the Cognito **ID token** as the `Authorization` header (the
access token is rejected by the API). When the refresh token is no longer valid, it SHALL raise a
re-authentication flow rather than failing silently.

#### Scenario: Token refresh
- **WHEN** the current ID token is expired or near expiry before a poll
- **THEN** the integration SHALL refresh it using the stored refresh token and proceed without
  user interaction

#### Scenario: Refresh token revoked
- **WHEN** the stored refresh token is rejected by Cognito
- **THEN** the integration SHALL trigger a Home Assistant reauth flow prompting the user to log in again

### Requirement: Polling with rate-limit backoff

The integration SHALL poll cloud endpoints on a configurable interval through a single update
coordinator, and SHALL back off on HTTP 429 rather than retrying immediately, so it does not
trip the API's rate limiter.

#### Scenario: Normal poll
- **WHEN** the update interval elapses
- **THEN** the coordinator SHALL fetch the device tree once and share the result with all entities

#### Scenario: Rate limited
- **WHEN** a request returns HTTP 429
- **THEN** the integration SHALL delay the next request with increasing backoff and SHALL NOT
  mark entities unavailable solely because of a transient 429

### Requirement: Unofficial and own-risk disclosure

The integration SHALL make its unofficial, reverse-engineered, no-warranty status visible to the
user and SHALL NOT present itself as an official LUPUS product.

#### Scenario: Disclosure is discoverable
- **WHEN** a user views the integration's repository, HACS listing, or documentation
- **THEN** they SHALL find a clear statement that it is unofficial, not affiliated with
  LUPUS-Electronics, reverse-engineered against a private API that may break without notice,
  provided AS IS with no warranty or liability, and that safety-critical smoke/fire alerting
  must not depend on it
