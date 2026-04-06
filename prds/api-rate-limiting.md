---
title: API Rate Limiting Service
status: draft
owner: security-team
tags: [infrastructure, security, API]
---

# API Rate Limiting Service

## Overview
Design and implement a centralized rate limiting service for all TinkerSystems public APIs. This service must prevent abuse, ensure fair resource allocation across tenants, and protect backend systems from traffic spikes.

## Security Requirements

### Authentication & Authorization
- Rate limit policies must be configurable per-tenant and per-API-key
- Admin endpoints for managing rate limit rules must require elevated permissions
- Rate limit state must be isolated between tenants — one tenant's limits must not affect another

### Data Protection
- Rate limit counters must not leak information about other tenants' usage patterns
- Rejected request logs must be retained for 90 days for audit purposes
- API keys used for rate limit bypass must be stored encrypted at rest

### Abuse Prevention
- The service must detect and block distributed brute-force patterns across multiple source IPs
- Graduated response: warning headers at 80% → throttle at 100% → block at 150%
- IP reputation scoring must be integrated to adjust limits for known-bad actors

### Infrastructure Security
- The rate limiting service must be deployed behind mTLS
- Redis cluster used for counter storage must require authentication
- Failover mode: if the rate limiter is unavailable, default to DENY (fail-closed)

## API Specification
- `GET /api/v1/rate-limits/{tenant_id}` — current usage and remaining quota
- `PUT /api/v1/rate-limits/{tenant_id}/policies` — update rate limit policies (admin only)
- `POST /api/v1/rate-limits/check` — real-time rate check (internal, sub-1ms SLA)

## Non-Functional Requirements
- P99 latency for rate check: < 1ms
- Must handle 100K+ checks/second per cluster
- Zero-downtime policy updates
