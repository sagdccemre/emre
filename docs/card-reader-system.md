# Card Reader System Design

This document summarizes the requirements and a reference design for a closed card reader platform with SQL-backed state, device-specific keys, and a dedicated SDK for device telemetry and control.

## Requirements
- Each device receives an automatically generated, 30-character device-specific key.
- All transactional state and queries run on SQL; keep device, event, and audit history in the database.
- Only authorized clients can call the APIs; requests must be authenticated and authorized per device/user role.
- A separate SDK surfaces device state, events, and configuration management.
- PDSK (event/status logging) keeps a full audit of device activity, firmware, and key lifecycle.

## Key Management
- Generate keys with a cryptographically secure RNG using a URL-safe/Base32 alphabet to avoid casing issues.
- Store only hashed keys (e.g., HMAC-SHA256 + per-key salt) in `device_keys(device_id, key_hash, created_at, rotated_at, status)` with `device_id` unique.
- Provide `rotate_device_key(device_id)` to issue new keys, mark old keys `retired`, and log rotations in an audit table.
- Include timestamp + nonce on signed requests to prevent replay; maintain a short-lived nonce cache or table.

## Authentication and Authorization
- Require each API call to include `device_id` and an HMAC signature derived from the device key; verify against stored hashes.
- Enforce RBAC for operators and service users with `users`, `roles`, `user_roles`, and `api_tokens` tables.
- Use mTLS or IP allowlists to restrict network access to trusted environments.

## Device SDK/Agent
- Lightweight agent captures card read events and batches them to the server over HTTPS with HMAC signing.
- Buffer events locally (e.g., SQLite/LevelDB) when offline; flush on reconnect.
- Attach trace identifiers to every request to correlate device operations with server logs.
- Support config push/pull, firmware version reporting, and status snapshots from the control plane.

## PDSK Logging and Query Model
- Suggested tables:
  - `events(device_id, type, payload_json, created_at)`
  - `status_snapshots(device_id, status, battery, firmware, created_at)`
  - `audits(actor, action, target, meta_json, created_at)`
- Index by `(device_id, created_at)` and `(type, created_at)` for fast filtering.
- Split hot (OLTP) and cold (object storage/OLAP) paths; move aged data via scheduled ETL.
- Provide APIs for event query (filter by device/time/type), current status fetch, and audit export.

## Observability and Runbooks
- Emit metrics for request volume, error rate, latency, queue depth, and signature verification failures; instrument crypto and DB spans for tracing.
- Alerts: signature verification failure ratio >2% over 10m, key rotation failures, queue saturation, and elevated DB latency.
- Runbooks: key rotation verification, agent log collection, and release rollback per `docs/observability.md` to keep response steps consistent across services.
