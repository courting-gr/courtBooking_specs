# Runbook: Stripe Webhook IP Allowlist Refresh

## Purpose

The `/api/webhooks/stripe` endpoint is protected at the NGINX Ingress
perimeter by a source-IP allowlist containing Stripe's published webhook IP
ranges. This is **defense-in-depth only** (Requirements 11.3, 7.6).

> **Primary protections** for the webhook endpoint are Stripe **signature
> verification** and the **5-minute timestamp tolerance** (Task 11.x). The IP
> allowlist is an additional perimeter layer and may be removed without
> weakening any other control.

## Refresh cadence

- **Quarterly** as routine maintenance, AND
- **On-demand** whenever Stripe issues a security advisory or changes its
  published IP ranges.

Do **not** schedule this daily — the list changes rarely and unattended
automation against an external source adds risk.

## Components

| Resource | Location | Purpose |
| --- | --- | --- |
| `stripe-webhook-ips` ConfigMap | `kubernetes/base/security/stripe-webhook-ips-configmap.yaml` | Canonical record of allowlisted CIDRs |
| `court-booking-ingress-webhooks` Ingress | `kubernetes/base/security/ingress-webhooks.yaml` | Applies the allowlist via `whitelist-source-range` |
| `stripe-webhook-ip-refresh` Job | `kubernetes/base/security/stripe-webhook-ip-refresh-job.yaml` | On-demand refresh |
| ServiceAccount / Role / RoleBinding | `kubernetes/base/security/stripe-webhook-ip-refresh-rbac.yaml` | `configmaps: get,update,patch`; `ingresses: get,patch` |

## Procedure (on-demand refresh)

1. Confirm the latest published list:
   <https://stripe.com/files/ips/ips_webhooks.txt>
2. Run the refresh Job in the target namespace:

   ```bash
   kubectl -n production create job stripe-webhook-ip-refresh-$(date +%Y%m%d) \
     --from=job/stripe-webhook-ip-refresh
   # or, for a fresh apply of the manifest:
   kubectl -n production delete job stripe-webhook-ip-refresh --ignore-not-found
   kubectl -n production apply -f kubernetes/base/security/stripe-webhook-ip-refresh-job.yaml
   ```

3. Verify the Job completed and inspect logs:

   ```bash
   kubectl -n production logs job/stripe-webhook-ip-refresh
   ```

4. Confirm the ConfigMap and Ingress annotation were updated:

   ```bash
   kubectl -n production get configmap stripe-webhook-ips -o jsonpath='{.data.ip-ranges}'
   kubectl -n production get ingress court-booking-ingress-webhooks \
     -o jsonpath='{.metadata.annotations.nginx\.ingress\.kubernetes\.io/whitelist-source-range}'
   ```

5. Send a Stripe test webhook (dashboard → Developers → Webhooks → Send test
   event) and confirm a `200` response and an event recorded in the
   transaction service.

## Rollback

If a refresh breaks webhook delivery (e.g. fetched an empty or wrong list):

1. Restore the previous annotation from the manifest in version control:

   ```bash
   kubectl -n production apply -f kubernetes/base/security/ingress-webhooks.yaml
   ```

2. Re-run validation step 5 above.

The refresh Job aborts without making changes if it fetches an empty list, so
a transient network failure leaves the existing allowlist intact.

## Disabling the allowlist

If Stripe stops publishing IP ranges, or the operational team decides the
maintenance overhead outweighs the benefit:

1. Remove the allowlist annotation:

   ```bash
   kubectl -n production annotate ingress court-booking-ingress-webhooks \
     nginx.ingress.kubernetes.io/whitelist-source-range-
   ```

2. Set `enabled: "false"` in the `stripe-webhook-ips` ConfigMap for the record.

Signature verification and timestamp tolerance continue to protect the
endpoint. No other control is weakened.
