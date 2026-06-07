# Runbook: IP Blocklist Management

## Purpose

The platform blocks abusive source IPs at two layers (Requirements 7.4, 8.6):

1. **Application layer** — `IpBlocklistEnforcer` filter in both services reads
   the blocklist from Redis on each request (Tasks 5, 25.2).
2. **Perimeter layer** — the `ip-blocklist-nginx-sync` CronJob mirrors the
   Redis blocklist into NGINX `deny` rules every minute
   (`kubernetes/base/security/ip-blocklist-nginx-sync-cronjob.yaml`).

## Redis data model

| Key | Type | Contents |
| --- | --- | --- |
| `ip:blocklist` | SET | Individual blocked IPs (IPv4) |
| `ip:blocklist:cidr` | ZSET | Blocked CIDR ranges (member = CIDR, score = expiry epoch) |

The application enforces a hard cap of 100,000 entries (Task 50).

## Routine operations

### Block an IP or CIDR

Preferred: use the Admin Web IP Blocklist page (PLATFORM_ADMIN) or the API:

```
POST /api/admin/security/ip-blocklist  { "value": "203.0.113.7", "reason": "..." }
```

Emergency CLI (bypasses audit logging — record manually):

```bash
redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" -a "$REDIS_PASSWORD" --tls \
  SADD ip:blocklist 203.0.113.7
```

### Unblock

```
DELETE /api/admin/security/ip-blocklist/203.0.113.7
```

The perimeter CronJob removes the corresponding `deny` rule within one minute.

## Rollback

If a legitimate IP/range is blocked in error:

1. Remove it via the API or `SREM ip:blocklist <ip>` / `ZREM ip:blocklist:cidr <cidr>`.
2. Confirm the NGINX deny rule clears:

   ```bash
   kubectl -n production get configmap ip-blocklist-deny -o jsonpath='{.data.deny\.conf}'
   ```

3. If the CronJob is unhealthy, manually clear the perimeter annotation:

   ```bash
   kubectl -n production annotate ingress court-booking-ingress \
     nginx.ingress.kubernetes.io/server-snippet-
   ```

## Incident response

- Sudden surge in blocked traffic → check `SecurityAlert` records of type
  `BRUTE_FORCE` / `SCRAPING` and the auto-response config (Task 25).
- CronJob failing → inspect `kubectl -n production logs job/<sync-job>`;
  verify the `ip-blocklist-nginx-sync` ServiceAccount still has
  `configmaps: update` and `ingresses: patch`.
- Approaching the 100k cap → review and prune expired CIDR entries.
