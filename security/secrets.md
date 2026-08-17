# Thermone V1 Secrets Management

## Purpose

This document defines how Thermone stores, accesses, rotates, and protects secrets.

Thermone secrets include:

- Device credentials
- Factory credentials
- Claim tokens
- Supabase keys
- Database credentials
- Firmware signing keys
- GitHub deployment secrets
- Email provider credentials
- Storage credentials
- Internal API secrets
- Wi-Fi credentials stored on devices

The core rule is:

```text
Secrets must be stored only where they are required.
```

No secret should be shared more broadly than necessary.

---

# Secret Categories

Thermone V1 uses several secret classes.

```text
User Secrets
Device Secrets
Factory Secrets
Cloud Secrets
CI/CD Secrets
Firmware Signing Secrets
Local Development Secrets
```

Each category has different storage and access requirements.

---

# 1. User Secrets

Examples:

```text
User passwords
User session tokens
Password-reset tokens
Magic-link tokens
Refresh tokens
```

User authentication should be handled by a trusted authentication provider such as:

```text
Supabase Auth
```

Thermone application code should never store plaintext user passwords.

---

# User Password Rules

Never store:

```text
password
plaintext_password
raw_password
```

inside:

- Thermone database tables
- Logs
- Analytics
- GitHub
- Device firmware
- Support tools

---

# User Session Tokens

User JWTs and refresh tokens are sensitive.

They may exist temporarily in:

```text
Browser session
Authentication provider
Server request context
```

They must not be:

- Printed in logs
- Sent to ESP32 devices
- Stored in source code
- Embedded in QR codes

---

# 2. Device Runtime Secrets

Each registered Thermone controller receives a unique runtime device credential.

Example conceptual value:

```text
dt_prod_xxxxxxxxxxxxxxxxxxxxx
```

This token authenticates:

- Telemetry
- Heartbeats
- Configuration requests
- Command polling
- Firmware checks
- Device events

---

# Runtime Token Rules

Every controller must have its own unique credential.

Never use:

```text
one shared fleet token
```

for every device.

---

# Device Token Storage

On the ESP32, store the runtime token in persistent local storage.

Initial implementation may use:

```text
NVS
```

Future hardened production versions should evaluate:

```text
Encrypted NVS
Flash Encryption
Secure Boot
```

---

# Device Token Server Storage

Where possible, the backend should store a secure verifier or hash.

Preferred:

```text
device_token_hash
```

Avoid storing plaintext runtime tokens indefinitely.

---

# Device Token Exposure

Runtime device credentials must never appear in:

- Dashboard responses
- Device labels
- QR codes
- Support screenshots
- Serial logs
- GitHub
- Public APIs

---

# Device Token Rotation

Runtime device credentials must support rotation.

Conceptual flow:

```text
Old Token
   │
   ▼
Request Rotation
   │
   ▼
Generate New Token
   │
   ▼
Store New Token
   │
   ▼
Confirm New Token
   │
   ▼
Revoke Old Token
```

---

# 3. Factory Credentials

Every manufactured Thermone controller receives a unique factory credential.

Purpose:

```text
Initial registration
Approved recovery
```

The factory credential is more sensitive than the public serial number.

---

# Factory Credential Storage

On the device:

```text
Factory-specific persistent storage
```

On the backend:

```text
factory_credential_hash
```

where possible.

---

# Factory Credential Restrictions

A factory credential must never be:

- Printed on the enclosure
- Included in QR codes
- Visible to customers
- Used for routine telemetry
- Shared between devices

---

# Factory Credential Lifecycle

```text
Generated
   │
   ▼
Flashed to Device
   │
   ▼
Used for Registration
   │
   ▼
Restricted / Recovery Only
```

---

# 4. Claim Tokens

Claim tokens connect a physical Thermone unit to a user account.

Example:

```text
https://app.thermone.com/claim/<token>
```

The raw token is intentionally present in the QR code.

Because the QR may be physically visible, the token is treated as a possession secret rather than a permanent high-privilege device credential.

---

# Claim Token Storage

Backend:

```text
claim_token_hash
```

Raw value may exist temporarily during:

- Generation
- QR generation
- Label printing

After that, it should not be routinely retrievable.

---

# Claim Token Rules

Claim tokens must be:

- Unique
- Random
- High entropy
- Single-purpose
- Revocable
- Invalid after successful claim

---

# 5. Customer Wi-Fi Credentials

Customer Wi-Fi credentials are stored only on the Thermone controller.

Flow:

```text
User Phone
   │
   ▼
Local Setup Portal
   │
   ▼
ESP32
```

They should not pass through Thermone Cloud.

---

# Wi-Fi Storage

Initial prototype:

```text
ESP32 NVS
```

Future production:

```text
Encrypted NVS
```

recommended.

---

# Wi-Fi Password Rules

Never:

```text
POST Wi-Fi password to Thermone Cloud
```

Never log:

```text
SSID password
```

Never include it in:

- Crash reports
- API telemetry
- Support diagnostics
- Cloud backups

---

# 6. Supabase Secrets

Thermone may use Supabase for:

- PostgreSQL
- Authentication
- Storage
- Realtime

Different Supabase keys have different privilege levels.

---

# Public / Client Key

A browser-safe public client key may be used by the dashboard when appropriate.

This key must still be paired with:

```text
Row Level Security
```

It must not be treated as a secret that provides administrative access.

---

# Supabase Service Role Key

The service-role key is highly privileged.

It must only exist on trusted backend infrastructure.

Never place it in:

```text
ESP32 firmware
browser JavaScript
mobile client
QR code
public GitHub repository
```

---

# Service Role Environment Variables

Example:

```text
SUPABASE_SERVICE_ROLE_KEY
```

Store using:

- Hosting provider secret manager
- GitHub Actions environment secrets
- Local `.env` during development

---

# 7. Database Credentials

If direct PostgreSQL access is used, database passwords are highly sensitive.

Example:

```text
DATABASE_URL
DATABASE_PASSWORD
```

They must only exist in trusted server environments.

---

# Database Access Principle

Use the least privileged database role necessary.

Example:

```text
API service
```

should not automatically receive:

```text
database superuser
```

permissions.

---

# 8. Firmware Signing Keys

Firmware signing keys are among Thermone's most sensitive secrets.

The private signing key can authorize firmware that thousands of deployed devices may trust.

Protection level:

```text
CRITICAL
```

---

# Signing Private Key

Never store the private signing key in:

```text
firmware repository
docs repository
developer laptop source tree
ESP32 firmware
dashboard application
normal CI logs
```

---

# Preferred Signing Storage

As Thermone matures, prefer:

```text
Cloud KMS
HSM
Dedicated signing service
```

For the earliest prototype, an encrypted restricted secret store may be acceptable.

---

# Signing Public Key

The public verification key is not secret.

It may be embedded in firmware.

Example:

```text
ESP32 contains public key
```

and uses it to verify firmware signatures.

---

# Signing Separation

Preferred flow:

```text
Build Firmware
     │
     ▼
Generate Artifact
     │
     ▼
Restricted Signing Step
     │
     ▼
Signed Firmware
```

Normal build jobs should not automatically have unrestricted signing privileges.

---

# 9. GitHub Secrets

Thermone GitHub Actions may require credentials for:

- Deployment
- Firmware storage
- Supabase
- Cloud hosting
- Package publishing
- Signing workflow

Store these under:

```text
GitHub Environments
```

and:

```text
GitHub Actions Secrets
```

---

# GitHub Environments

Recommended:

```text
development
staging
production
```

Each environment contains only its own secrets.

Example:

```text
DEV_SUPABASE_SERVICE_ROLE_KEY
STAGING_SUPABASE_SERVICE_ROLE_KEY
PROD_SUPABASE_SERVICE_ROLE_KEY
```

---

# Production GitHub Secrets

Production secrets should require tighter controls.

Recommended:

- Environment approval
- Restricted reviewers
- Protected branches
- Minimal workflow permissions

---

# GitHub Token Permissions

Use the smallest permissions required.

Avoid:

```text
read/write everything
```

when a workflow only needs:

```text
read repository
write release asset
```

---

# 10. Local Development Secrets

Developers may require secrets locally.

Use:

```text
.env
.env.local
```

files.

---

# Example `.env.example`

Safe to commit:

```env
THERMONE_ENVIRONMENT=development

SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

DATABASE_URL=

DEVICE_API_SECRET=

FIRMWARE_STORAGE_BUCKET=
```

Do not put actual secret values into `.env.example`.

---

# `.gitignore`

Every applicable repository should ignore:

```text
.env
.env.local
.env.*.local
*.pem
*.key
*.p12
*.pfx
credentials.json
secrets.json
```

unless a specific public test fixture is intentionally committed.

---

# Example `.gitignore`

```gitignore
# Environment files
.env
.env.local
.env.*.local

# Secrets / certificates
*.pem
*.key
*.p12
*.pfx

# Credentials
credentials.json
secrets.json

# Local provisioning output
provisioning-output/
factory-secrets/
```

---

# 11. Provisioning Workstation Secrets

Factory provisioning stations may require:

```text
Factory API credential
```

These credentials should be limited to provisioning operations.

Do not give factory stations broad production admin access.

---

# Factory Service Account

Recommended permissions:

```text
Create factory device
Reserve serial
Write factory test result
Complete provisioning
```

Not:

```text
Read all customer data
Change billing
Access user passwords
Deploy backend
```

---

# Provisioning Secret Storage

Factory workstation credentials should be stored using:

- OS keychain
- Secret manager
- Environment injection

Avoid plaintext config files where possible.

---

# 12. Internal API Secrets

Some Thermone backend services may authenticate to other internal services.

Examples:

```text
ALERT_SERVICE_TOKEN
FIRMWARE_SERVICE_TOKEN
PROVISIONING_API_TOKEN
```

These should be:

- Unique per service
- Revocable
- Environment-specific
- Least privilege

---

# 13. Email Provider Secrets

If Thermone uses an email provider, the API key is server-side only.

Example:

```text
RESEND_API_KEY
```

Do not expose it in the dashboard.

---

# 14. Firmware Storage Credentials

Firmware storage write credentials should only exist in:

```text
release pipeline
```

Devices should not receive storage master credentials.

ESP32 devices should receive:

```text
temporary signed download URLs
```

instead.

---

# 15. Secrets by Repository

## `firmware`

May contain:

```text
public verification keys
non-secret API hostnames
hardware constants
```

Must not contain:

```text
production device token
factory credential
Wi-Fi password
signing private key
service role key
```

---

## `api`

May contain:

```text
environment variable names
secret-loading code
```

Must not contain actual:

```text
database password
Supabase service-role key
email API key
```

---

## `dashboard`

May contain:

```text
public client config
```

Must not contain:

```text
service role keys
device credentials
admin secrets
```

---

## `provisioning`

May contain logic to:

```text
generate secrets
inject secrets
```

Must not contain:

```text
real production factory credentials
```

---

## `infrastructure`

May contain:

```text
secret references
secret resource definitions
```

Must not contain plaintext secret values.

---

## `docs`

Must never contain real credentials.

Documentation examples use clearly fake values.

---

# 16. Secret Classification

Recommended classification:

## Critical

```text
Firmware signing private key
Production database admin credentials
Supabase service-role key
Production provisioning master credential
```

---

## High

```text
Runtime device token
Factory credential
Claim token
User session token
Wi-Fi password
Internal service token
```

---

## Medium

```text
Public API keys
Serial number
Device ID
Hardware ID
Sensor ROM ID
```

---

# 17. Rotation Policy

Every long-lived secret should have a rotation strategy.

Examples:

```text
Device token
Factory service credential
Database password
Supabase keys
Email provider API key
Signing key
```

---

# Device Token Rotation

May happen:

- Periodically
- On compromise
- On ownership transfer
- During hardware replacement

---

# Service Credential Rotation

Recommended:

```text
Generate new secret
      │
      ▼
Deploy services supporting new secret
      │
      ▼
Verify
      │
      ▼
Revoke old secret
```

Avoid abrupt rotation that causes production outage.

---

# 18. Secret Revocation

A secret must be revocable without deleting the entire account or device.

Example:

```text
device token compromised
```

Response:

```text
revoke token
issue replacement
```

---

# 19. Secret Expiration

Some secrets should expire.

Good candidates:

```text
Claim sessions
Temporary download URLs
Password reset links
Magic links
Transfer tokens
```

Runtime device tokens may initially be long-lived but rotatable.

---

# 20. Temporary Signed URLs

OTA download URLs should expire.

Example:

```text
15 minutes
```

They should only allow access to:

```text
specific firmware object
```

not the entire storage bucket.

---

# 21. Logging Policy

All services must redact sensitive values.

Never log:

```text
Authorization headers
device tokens
factory credentials
claim tokens
Wi-Fi passwords
service keys
database passwords
private keys
```

---

# Good Logging Example

```text
request_id=req_123
device_id=dev_456
endpoint=/telemetry
status=200
```

---

# Bad Logging Example

```text
Authorization: Bearer dt_prod_abc123...
```

---

# 22. Error Responses

API errors must never include:

- Raw SQL connection strings
- Secrets
- Environment variables
- Authorization headers
- Internal stack data containing credentials

---

# 23. Crash Reporting

Before sending device or server crash logs, sanitize:

```text
tokens
passwords
headers
secret config
```

---

# 24. Backups

Database backups may contain sensitive data.

Backups should use:

- Provider encryption
- Restricted access
- Retention policy
- Separate production permissions

---

# 25. Secret Scanning

GitHub repositories should enable secret scanning where available.

CI may also run tools to detect accidentally committed:

- API keys
- Private keys
- JWTs
- Passwords

---

# Pre-Commit Checks

Future developer tooling may include:

```text
gitleaks
trufflehog
```

or equivalent secret scanners.

---

# 26. If a Secret Is Committed

Never assume deleting it from the latest Git commit makes it safe.

Response:

```text
1. Revoke secret immediately
2. Generate replacement
3. Update affected services
4. Remove secret from repository history if appropriate
5. Audit usage
```

Rotation comes first.

---

# 27. Secret Naming

Environment variables should use clear names.

Example:

```text
THERMONE_ENVIRONMENT

SUPABASE_URL
SUPABASE_SERVICE_ROLE_KEY

DATABASE_URL

RESEND_API_KEY

FIRMWARE_STORAGE_TOKEN
```

Avoid vague names like:

```text
SECRET1
KEY2
```

---

# 28. Environment Prefixing

If one deployment system stores multiple environment secrets together, use prefixes.

Example:

```text
DEV_
STAGING_
PROD_
```

But where environment isolation already provides separate secret stores, duplicate prefixes may not be necessary.

---

# 29. Principle of Least Privilege

Every secret should have only the permissions it needs.

Example:

Firmware upload pipeline credential:

```text
write firmware bucket
```

should not also have:

```text
delete customer database
```

---

# 30. Separation of Duties

Highly sensitive capabilities should be separated where practical.

Example:

```text
Build firmware
```

does not necessarily mean:

```text
Sign production firmware
```

and:

```text
Deploy dashboard
```

does not necessarily mean:

```text
Access production database admin credentials
```

---

# 31. Production Access

Production secrets should be accessible to as few people and systems as possible.

Developers should use:

```text
development
```

or:

```text
staging
```

for routine work.

---

# 32. Device Credential Prefixes

Opaque tokens may optionally include non-secret prefixes.

Example:

```text
dt_dev_
dt_stg_
dt_prod_
```

Benefits:

- Easier environment identification
- Reduced accidental cross-environment use

The random secret portion must still carry sufficient entropy.

---

# 33. Token Hashing

Opaque random credentials can be stored using a one-way hash.

Conceptual flow:

```text
Raw token
   │
   ▼
Hash
   │
   ▼
Store Hash
```

During authentication:

```text
Provided token
   │
   ▼
Hash
   │
   ▼
Compare with stored hash
```

---

# Hash Requirements

Use a secure cryptographic hash or token-verification approach suitable for high-entropy random tokens.

Do not invent custom cryptography.

---

# 34. Password Hashing vs Token Hashing

Human passwords require password-specific hashing algorithms.

Random 256-bit tokens are different.

User password handling should remain with Supabase Auth or another trusted auth provider.

---

# 35. Secret Generation

Use cryptographically secure randomness.

Never use:

```text
Math.random()
timestamp
serial number
incrementing IDs
```

for secret generation.

---

# Recommended Token Entropy

At least:

```text
128 bits
```

Preferred:

```text
256 bits
```

for:

- Factory credentials
- Device runtime tokens
- Claim tokens

---

# 36. Secret Output During Provisioning

Provisioning software may temporarily need raw:

```text
factory credential
claim token
```

Do not print raw credentials unnecessarily.

Example preferred output:

```text
Factory credential generated ✓
Claim token generated ✓
QR generated ✓
```

rather than displaying full values.

---

# 37. QR Generation

The claim token is intentionally embedded in the QR.

The QR file itself therefore becomes sensitive until the device is claimed.

Factory output directories containing QR codes should be protected.

---

# 38. Device Labels

Device labels may display:

```text
Serial
Model
Hardware revision
Setup instructions
QR claim code
```

They must never display:

```text
factory credential
runtime token
private key
```

---

# 39. Setup Wi-Fi Password

If Thermone uses a unique setup AP password, it is a local onboarding secret.

It may be printed on the device label.

It must remain separate from:

```text
claim token
factory credential
runtime credential
```

---

# 40. Secret Ownership Table

| Secret | Stored On Device | Stored In Cloud | Browser Access | Printed |
|---|---:|---:|---:|---:|
| User Password | No | Auth provider only | User entry only | No |
| User JWT | No | Auth/session system | Yes | No |
| Factory Credential | Yes | Hash/verifier | No | No |
| Runtime Device Token | Yes | Hash/verifier | No | No |
| Claim Token | No | Hash/verifier | Via QR flow | QR only |
| Wi-Fi Password | Yes | No | During local setup | No |
| Supabase Service Key | No | Server secret store | No | No |
| Firmware Signing Private Key | No | Signing system | No | No |
| Firmware Verification Public Key | Yes | Yes | May be public | No |
| Serial Number | Yes | Yes | Yes | Yes |

---

# 41. Secret Lifecycle

Every secret should have:

```text
Generation
Storage
Usage
Rotation
Revocation
Deletion
```

documented.

---

# 42. Factory Credential Lifecycle

```text
Generate
   │
   ▼
Flash to Device
   │
   ▼
Store Server Verifier
   │
   ▼
Register Device
   │
   ▼
Restrict
   │
   ▼
Revoke if compromised
```

---

# 43. Runtime Device Token Lifecycle

```text
Issue
 │
 ▼
Store on Device
 │
 ▼
Authenticate
 │
 ▼
Rotate
 │
 ▼
Revoke
```

---

# 44. Claim Token Lifecycle

```text
Generate
 │
 ▼
Encode in QR
 │
 ▼
Customer Scans
 │
 ▼
Claim Device
 │
 ▼
Invalidate
```

---

# 45. Signing Key Lifecycle

```text
Generate securely
      │
      ▼
Protect
      │
      ▼
Sign firmware
      │
      ▼
Rotate when required
      │
      ▼
Retire old key carefully
```

Public verification-key rotation must be planned before replacing a production signing key.

---

# 46. Secrets During Development

Development should use:

```text
development-only credentials
```

Never copy production secrets into `.env.local` just to make local development easier.

---

# 47. Test Tokens

Automated tests should use:

```text
fake/test credentials
```

never live production credentials.

---

# 48. Documentation Examples

All documentation tokens should be obviously fake.

Example:

```text
dt_prod_example_not_real
```

Do not paste actual tokens into Markdown.

---

# 49. Support Operations

Support staff should not need raw credentials for routine troubleshooting.

Provide metadata such as:

```text
credential status
last used
created date
rotation status
```

instead of the token itself.

---

# 50. Admin UI

The admin dashboard should never provide a button like:

```text
Show Device Secret
```

if the system can function without revealing it.

Prefer:

```text
Rotate Device Credential
Revoke Device Credential
```

---

# 51. Customer Export

User data exports must not include:

- Runtime device tokens
- Factory credentials
- Claim-token hashes
- Service secrets
- Wi-Fi passwords

---

# 52. Decommissioned Devices

When a device is decommissioned:

```text
runtime credential → revoked
```

Factory/recovery credentials may also be disabled depending on policy.

Historical customer data may remain separately.

---

# 53. Returned / Refurbished Devices

Before resale:

```text
old runtime credential revoked
old claim token invalidated
new claim token generated
runtime state cleared
```

Factory serial remains the same unless hardware replacement policy requires otherwise.

---

# 54. Incident Response

If a secret is suspected to be compromised:

```text
Identify
   │
   ▼
Revoke
   │
   ▼
Rotate
   │
   ▼
Audit
   │
   ▼
Restore Service
```

Do not wait for proof of abuse before revoking a critical exposed credential.

---

# 55. Fleet-Wide Credential Incident

Because Thermone uses unique per-device tokens:

```text
one leaked device token
```

should require:

```text
rotate one device
```

not:

```text
rotate entire fleet
```

This isolation is a major security requirement.

---

# 56. Critical Secret Incident

If a critical fleet-wide secret is exposed, such as:

```text
firmware signing private key
```

Thermone must have a dedicated incident-response and key-rotation process.

This may require firmware trust migration across the fleet.

---

# 57. V1 Secrets Checklist

Before production deployment verify:

- [ ] No `.env` files committed
- [ ] No private keys committed
- [ ] No device tokens committed
- [ ] No factory credentials committed
- [ ] No claim tokens in documentation
- [ ] Supabase service-role key is server-side only
- [ ] GitHub production secrets are environment protected
- [ ] Production and development secrets are different
- [ ] Logs redact authorization headers
- [ ] Wi-Fi password never reaches cloud
- [ ] Every controller has a unique runtime credential
- [ ] Device credentials are revocable
- [ ] Claim tokens are single-use
- [ ] Firmware signing private key is restricted

---

# Core Secrets Principle

Thermone should always ask:

```text
Does this component actually need this secret?
```

If the answer is:

```text
No
```

then that component should never receive it.

The safest secret is one that does not exist in a place where it is not required.