# Thermone V1 Environments

## Purpose

Thermone uses separate environments for development, staging, and production.

The purpose of environment separation is to ensure that:

- Development work does not affect real customers
- Test firmware does not reach production devices
- Staging can be used for realistic release testing
- Production secrets stay isolated
- Database changes can be validated safely
- API changes can be tested before release
- Devices always communicate with the correct backend
- OTA firmware channels remain controlled

The three primary environments are:

```text
development
staging
production
```

---

# Environment Overview

```text
                    THERMONE
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   DEVELOPMENT        STAGING        PRODUCTION
        │               │               │
        ▼               ▼               ▼
   Dev Devices      Test Devices    Customer Devices
        │               │               │
        ▼               ▼               ▼
     Dev API         Staging API       Prod API
        │               │               │
        ▼               ▼               ▼
     Dev DB          Staging DB        Prod DB
        │               │               │
        ▼               ▼               ▼
  Dev Firmware      Staging FW        Prod FW
```

---

# 1. Development Environment

The development environment is used for active engineering work.

It may change frequently and may contain incomplete or experimental features.

## Example URLs

```text
https://api.dev.thermone.com
https://app.dev.thermone.com
```

Additional services may use:

```text
https://firmware.dev.thermone.com
https://setup.dev.thermone.com
```

---

## Development Database

Development must use a separate database from staging and production.

Example:

```text
Thermone Development Database
```

The development database may contain:

- Test users
- Test devices
- Simulated telemetry
- Temporary schema changes
- Fake tank records
- Automated test data

Production data must never be copied into development unless a future approved process sanitizes it first.

---

## Development Devices

Physical development ESP32 controllers should be clearly marked.

Example:

```text
DEV-THV1-0001
DEV-THV1-0002
```

Development devices must connect only to development services.

Example device configuration:

```text
ENVIRONMENT=development
API_BASE_URL=https://api.dev.thermone.com
FIRMWARE_CHANNEL=development
```

---

# 2. Staging Environment

Staging is the final test environment before production.

Staging should behave as closely as possible to production.

## Example URLs

```text
https://api.staging.thermone.com
https://app.staging.thermone.com
```

Optional supporting services:

```text
https://firmware.staging.thermone.com
https://setup.staging.thermone.com
```

---

## Purpose of Staging

Staging is used to test:

- Production-like API behavior
- Database migrations
- OTA firmware updates
- Device registration
- User claiming
- Alert flows
- Authentication
- Dashboard releases
- Provisioning
- Recovery flows
- Hardware revisions

---

## Staging Devices

Staging devices should be separate from development and production hardware.

Example:

```text
STG-THV1-0001
STG-THV1-0002
```

Recommended device configuration:

```text
ENVIRONMENT=staging
API_BASE_URL=https://api.staging.thermone.com
FIRMWARE_CHANNEL=staging
```

---

# 3. Production Environment

Production is used by real Thermone customers and deployed hardware.

Production must prioritize:

- Stability
- Security
- Reliability
- Backward compatibility
- Monitoring
- Controlled releases

## Example URLs

```text
https://api.thermone.com
https://app.thermone.com
```

Additional production services may use:

```text
https://firmware.thermone.com
https://setup.thermone.com
```

---

## Production Database

Production must use a dedicated database.

Production contains:

- Real customer accounts
- Real devices
- Real tank assignments
- Real telemetry
- Real alert rules
- Real firmware assignments
- Real ownership records

No development or staging system should connect directly to the production database.

---

# Environment Isolation

Each environment must have separate:

- API deployment
- Database
- Authentication configuration
- Secrets
- API keys
- Device credentials
- Firmware manifests
- Firmware storage
- Logging
- Monitoring
- Alerting
- Rate limits where appropriate

The environments should not share credentials.

---

# Environment Selection on Device

The Thermone firmware must know which environment it belongs to.

Possible compile-time configuration:

```cpp
#define THERMONE_ENVIRONMENT "development"
```

or:

```cpp
#define THERMONE_API_BASE_URL "https://api.dev.thermone.com"
```

Production devices should not allow users to casually switch environments.

---

# Recommended Firmware Environment Strategy

Use different firmware builds for each environment.

Example:

```text
thermone-dev.bin
thermone-staging.bin
thermone-production.bin
```

Each build contains the correct API endpoint and environment identifier.

Example:

```text
Development
API: api.dev.thermone.com

Staging
API: api.staging.thermone.com

Production
API: api.thermone.com
```

---

# Firmware Channels

Thermone OTA should support controlled firmware channels.

Initial channels:

```text
development
staging
production
```

Potential future channels:

```text
alpha
beta
canary
stable
```

---

# Development Firmware Channel

Used for experimental builds.

Characteristics:

- Frequent updates
- May contain unfinished features
- May be unstable
- Only development devices should receive it

---

# Staging Firmware Channel

Used for release candidate builds.

Characteristics:

- Expected to be near production quality
- Used for OTA validation
- Used for regression testing
- Used before production rollout

---

# Production Firmware Channel

Used for customer hardware.

Characteristics:

- Stable builds only
- Signed firmware
- Controlled rollout
- Rollback supported where possible
- Must pass staging validation

---

# Firmware Promotion Flow

Recommended release flow:

```text
Code Change
    │
    ▼
Development Build
    │
    ▼
Development Testing
    │
    ▼
Staging Build
    │
    ▼
Staging Hardware Testing
    │
    ▼
Release Approval
    │
    ▼
Production Build
    │
    ▼
Production Rollout
```

The same firmware artifact may eventually be promoted between environments if configuration is externalized safely.

For V1, separate builds are acceptable.

---

# Dashboard Environments

The dashboard should also use separate environments.

## Development

```text
app.dev.thermone.com
```

Used by developers.

---

## Staging

```text
app.staging.thermone.com
```

Used for pre-production testing.

---

## Production

```text
app.thermone.com
```

Used by customers.

---

# API Environments

The API follows the same separation.

```text
Development:
api.dev.thermone.com

Staging:
api.staging.thermone.com

Production:
api.thermone.com
```

---

# Supabase Environment Strategy

If Thermone uses Supabase, each environment should use a separate Supabase project.

Example:

```text
Thermone Dev
Thermone Staging
Thermone Production
```

Each project should have its own:

- PostgreSQL database
- Authentication configuration
- API keys
- Service role key
- Storage buckets
- Edge Functions if used
- Row Level Security policies
- Realtime configuration

---

# Secret Separation

Secrets must never be reused across environments.

Example:

```text
DEV_SUPABASE_SERVICE_KEY
STAGING_SUPABASE_SERVICE_KEY
PROD_SUPABASE_SERVICE_KEY
```

Similarly:

```text
DEV_DEVICE_API_SECRET
STAGING_DEVICE_API_SECRET
PROD_DEVICE_API_SECRET
```

Production secrets must only be available to production services.

---

# GitHub Environments

GitHub Actions should use GitHub Environments.

Recommended environments:

```text
development
staging
production
```

Each GitHub environment should contain only the secrets needed for that environment.

---

# GitHub Actions Deployment Flow

Example:

```text
Push to feature branch
        │
        ▼
CI Tests
        │
        ▼
Merge to develop
        │
        ▼
Deploy Development
        │
        ▼
Test
        │
        ▼
Merge / Promote
        │
        ▼
Deploy Staging
        │
        ▼
Validation
        │
        ▼
Production Approval
        │
        ▼
Deploy Production
```

---

# Production Deployment Protection

Production deployment should require additional protection.

Recommended controls:

- Protected `main` branch
- Pull request review
- Passing CI
- Manual production approval
- Tagged releases
- Restricted production secrets

---

# Branch Strategy

Initial recommended branch model:

```text
main
develop
feature/*
fix/*
release/*
```

Example:

```text
feature/device-registration
        │
        ▼
      develop
        │
        ▼
   development
        │
        ▼
release/v1.0.0
        │
        ▼
     staging
        │
        ▼
       main
        │
        ▼
   production
```

The exact branching strategy may evolve later.

---

# Device Registration Isolation

A device registered in one environment must not automatically exist in another.

Example:

```text
DEV-THV1-0001
```

may exist in development but not staging or production.

Likewise:

```text
THV1-000481
```

may exist only in production.

---

# Claim Token Isolation

Claim tokens are environment-specific.

A development claim URL:

```text
https://app.dev.thermone.com/claim/<token>
```

must not work against production.

A production claim URL:

```text
https://app.thermone.com/claim/<token>
```

must not work against development.

---

# API Credential Isolation

Device API credentials must only be valid in the environment that issued them.

Example:

```text
Development device token
```

must fail against:

```text
https://api.thermone.com
```

This prevents accidental cross-environment access.

---

# Database Migration Flow

Database changes should follow:

```text
Write Migration
     │
     ▼
Apply Development
     │
     ▼
Test
     │
     ▼
Apply Staging
     │
     ▼
Validate
     │
     ▼
Apply Production
```

Production schema changes should never be manually invented directly in production when a migration can be used.

---

# Test Data

Development and staging may use generated data.

Examples:

```text
Test User
test1@example.invalid

Test Controller
DEV-THV1-0001

Test Tank
Development Betta Tank
```

Do not use real customer information unnecessarily in test environments.

---

# Logging

Each environment should have separate logging.

Example:

```text
development logs
staging logs
production logs
```

Logs should include environment identifiers.

Example:

```json
{
  "environment": "staging",
  "service": "device-api",
  "device_id": "example",
  "event": "telemetry_received"
}
```

---

# Monitoring

Production requires the strongest monitoring.

Monitor:

- API availability
- Database health
- Device ingestion failures
- OTA failures
- Authentication failures
- Error rates
- Queue backlog
- Storage usage
- Firmware rollout health

Staging should also have monitoring so production issues can be detected before release.

---

# Local Development

Developers may run services locally.

Example:

```text
Dashboard:
http://localhost:3000

API:
http://localhost:4000
```

Local services should connect only to development resources.

Never point a local development environment directly at production unless a documented emergency process explicitly requires it.

---

# Environment Configuration Files

Repositories may contain templates such as:

```text
.env.example
```

Example:

```env
THERMONE_ENVIRONMENT=development
API_BASE_URL=https://api.dev.thermone.com
SUPABASE_URL=
SUPABASE_ANON_KEY=
```

Real secrets must not be committed.

---

# Forbidden Files

Never commit:

```text
.env
.env.local
.env.production
credentials.json
private keys
device secrets
database passwords
OTA signing private keys
```

Add sensitive local files to `.gitignore`.

---

# Hardware Marking

During development, physical hardware should be labeled.

Example:

```text
THERMONE
DEV UNIT

DEV-01
NOT FOR PRODUCTION
```

Staging hardware:

```text
THERMONE
STAGING UNIT

STG-01
NOT FOR CUSTOMER USE
```

This helps prevent accidentally shipping development hardware.

---

# Production Device Rules

Production devices must:

- Use production API endpoints
- Use production credentials
- Receive only approved production firmware
- Use production claim URLs
- Be registered in production inventory
- Pass manufacturing validation
- Never depend on development services

---

# Environment Failure Protection

If a production device cannot reach production services, it must not automatically fall back to staging or development.

Example:

```text
Production API unavailable
        │
        ▼
Remain Offline
        │
        ▼
Continue Local Monitoring
        │
        ▼
Retry Production API
```

Never:

```text
Production failed
      ↓
Try development
```

Cross-environment fallback is prohibited.

---

# DNS Strategy

Recommended eventual DNS structure:

```text
thermone.com
www.thermone.com

app.thermone.com
api.thermone.com
firmware.thermone.com
setup.thermone.com

app.staging.thermone.com
api.staging.thermone.com

app.dev.thermone.com
api.dev.thermone.com
```

Exact domain structure may change before production.

---

# Summary

Thermone uses three isolated environments:

| Environment | Purpose | Devices |
|---|---|---|
| Development | Active engineering | Developer hardware |
| Staging | Production-like validation | Test hardware |
| Production | Real customer use | Customer hardware |

The core rule is:

```text
Development stays in Development.

Staging stays in Staging.

Production stays in Production.
```

No device, credential, firmware release, database, or secret should cross environments unintentionally.