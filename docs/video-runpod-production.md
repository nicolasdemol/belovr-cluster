# Video callbacks in production

The production platform exposes only `/internal/transcoding` on
`https://video.belovr.com`. Traefik terminates TLS using the existing wildcard
certificate and forwards requests to `video:4700`. No CORS rule is needed for
Runpod's server-to-server callbacks. The application requires its shared bearer
token on every callback.

## Prerequisites

Published release verified on 2026-09-16:

- Application images (except `ai`): `87954164fb5e9e3eb41a1ac88b33b690ff6cdef4`.
- Runpod worker: `ghcr.io/nicolasdemol/belovr-video-transcoder:87954164fb5e9e3eb41a1ac88b33b690ff6cdef4`.
- [Service publication](https://github.com/nicolasdemol/belovr/actions/runs/35132280348)
  and [worker publication](https://github.com/nicolasdemol/belovr/actions/runs/35133181982)
  both completed successfully, including their image push steps.

Updating this repository does not change the Runpod endpoint's image. Select
the worker reference above in Runpod when activating this release.

- Use the production cluster/context. Do not replace `belovr-root-dev` just to
  expose video; that would switch all application routes.
- Publish the new `video`, `content`, `image`, `web` and Runpod worker images.
  Pin the application manifests and Runpod endpoint to these tested image SHAs.
- Create the `belovr-video-runpod-secrets` Secret in namespace `belovr` with
  `RUNPOD_API_KEY`, `RUNPOD_ENDPOINT_ID` and `BELOVR_TRANSCODER_TOKEN`.
  Provision it with the same SealedSecrets procedure as the other production
  secrets; never commit unencrypted values. The deployment intentionally does
  not make this Secret optional.
- The worker must use the identical `BELOVR_TRANSCODER_TOKEN` and
  `BELOVR_INTERNAL_API_URL=https://video.belovr.com`, plus its R2 credentials,
  `R2_RAW_BUCKET=belovr-raw` and `R2_TRANSCODED_BUCKET=belovr-dash`.
- Redis must have persistent storage and AOF enabled. It holds BullMQ jobs and
  the pending Runpod registry. PostgreSQL holds the upload dispatch outbox;
  the content release runs an additive migration before accepting uploads.

## DNS and activation

1. Verify the public IP of the production Traefik load balancer. The versioned
   production override currently advertises `137.74.174.36`; confirm this on
   the actual target before changing DNS.
2. Replace the temporary Cloudflare Tunnel record for `video.belovr.com` with
   an A record pointing to that production IP. Do not leave a competing CNAME
   or an incorrect AAAA record. Use Cloudflare Full (strict) TLS if proxied.
3. Allow incoming TCP 443 to Traefik. Do not expose port 4700, Redis or RabbitMQ
   publicly. Cloudflare Access must not require an interactive login on the
   callback path; authentication is provided by the application token.
4. Sync the production secrets, video ConfigMap/Deployment/Service, then the
   production platform. The new ingress is included only in the prod overlay.

## Verification

```sh
kubectl -n belovr rollout status deployment/video
kubectl -n belovr get endpointslices -l kubernetes.io/service-name=video
kubectl -n belovr get ingress belovr-video-ingress
curl -i -X POST https://video.belovr.com/internal/transcoding/check/started \
  -H 'Content-Type: application/json' -d '{}'
```

The unauthenticated POST must return **401**, not 404, 502 or 530. Do not test
`completed` with invented media IDs. Upload a real test video, leave the Vault,
then return: it must move from processing to ready and play. During a callback
outage the video service reconciles Runpod jobs and private R2 completion
receipts. Configure a 30-day expiration for the RawBucket prefix
`transcoding-results/`; do not make it publicly readable. Preserve receipts
longer than the maximum outage/recovery window.

This configuration does not itself update Cloudflare DNS, provision secrets,
switch the active ArgoCD root, or deploy newly built application images.
