# Runbook: GeoIP Database Refresh

## Purpose

The platform uses a GeoIP database to enrich security alerts with the
geographic origin of suspicious requests and to support impossible-travel
detection (Requirement 5.4). The database must be refreshed regularly so
lookups stay accurate as IP allocations change.

## Refresh cadence

- **Monthly** routine refresh (MaxMind publishes GeoLite2 updates weekly;
  monthly is sufficient for security enrichment).
- **On-demand** if alert geolocation looks systematically wrong.

## Procedure

1. Obtain the latest GeoLite2-City database (MaxMind account license key
   required).
2. Update the Vault entry / object storage location the service reads from:

   ```bash
   # Example: upload to the Spaces bucket the service mounts at startup.
   s3cmd put GeoLite2-City.mmdb s3://court-booking-geoip/GeoLite2-City.mmdb
   ```

3. Trigger a refresh. The platform service reloads the database on a schedule;
   to apply immediately, restart the platform-service pods:

   ```bash
   kubectl -n production rollout restart deployment/platform-service
   ```

## Rollback

Keep the previous `.mmdb` file versioned in object storage. To roll back,
re-upload the prior version and restart the platform-service pods.

## Verification

- Trigger a known-origin request and confirm the resulting `SecurityAlert`
  metadata reports the expected country/city.
- Check platform-service logs for `GeoIP database loaded` with the new build
  timestamp.

## Notes

- GeoIP is enrichment/heuristic only; never use it as the sole basis for
  blocking. Combine with the IP blocklist and rate-limiting controls.
