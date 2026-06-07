# Runbook: Secret Rotation

## Purpose

Rotate the platform's secrets on a defined cadence (Requirements 18.1-18.5).
Secrets are stored in HashiCorp Vault under `secret/courtbooking/{env}/` and
projected into the cluster by the External Secrets Operator
(`kubernetes/base/secrets/external-secrets.yaml`).

## Rotation cadence

| Secret | Cadence | ESO refresh | Application hook |
| --- | --- | --- | --- |
| DB credentials | 90d | 2160h | Hikari picks up new password on pool refresh |
| JWT signing keys | 90d | 2160h | `JwtKeyRotationService` (multi-key validation, Task 29) |
| Internal API key | 30d | 720h | `INTERNAL_API_CREDENTIAL_ROTATED` consumer (Task 29) |
| Stripe API key | annual | 8760h | Restart picks up new key |
| Stripe webhook secret | annual | 8760h | `STRIPE_WEBHOOK_SECRET_ROTATED` consumer (Task 29) |
| Redis / Kafka credentials | 90d | 2160h | Reconnect on credential refresh |

## General procedure

1. Generate the new secret value (see per-secret notes below).
2. Write the new value to Vault, keeping the previous value where overlap is
   required (JWT, webhook secret):

   ```bash
   vault kv put secret/courtbooking/production/jwt \
     active-key-id="$NEW_KID" active-private-key=@new_key.pem \
     previous-public-key=@old_pub.pem
   ```

3. Wait for ESO to sync (or force it):

   ```bash
   kubectl -n production annotate externalsecret jwt-signing-keys \
     force-sync="$(date +%s)" --overwrite
   ```

4. Trigger the application rotation hook where applicable:

   - JWT: `POST /api/admin/security/rotate-jwt-keys`
   - Internal API key / Stripe webhook secret: publish the corresponding
     `platform-commands` event (handled automatically by the rotation jobs).

5. Verify (see verification below), then remove the previous value from Vault
   after the overlap window (JWT: after max token TTL; webhook secret: after
   Stripe dashboard shows the new secret active).

## Per-secret notes

- **JWT keys** — dual-key overlap is mandatory so in-flight tokens signed with
  the previous key still validate. Never delete `previous-public-key` before
  the access-token TTL elapses.
- **Stripe webhook secret** — add the new endpoint secret in the Stripe
  dashboard first; the service validates against both old and new during the
  overlap.
- **DB credentials** — rotate the DigitalOcean managed-DB user password, then
  update Vault; Hikari reconnects with the new password.

## Rollback

1. Restore the previous Vault value:

   ```bash
   vault kv rollback -version=<n> secret/courtbooking/production/<secret>
   ```

2. Force ESO re-sync (step 3 above).
3. For JWT, re-run the rotation endpoint to restore the prior active key.

## Verification

- Auth still works: obtain a token via `/api/auth/login` and call a protected
  endpoint.
- No spike in `401`/`403` rates in Grafana after rotation.
- For webhook secret: send a Stripe test event and confirm `200`.
