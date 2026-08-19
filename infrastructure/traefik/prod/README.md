# Traefik shared configuration and VPS overrides

`../base/values.yaml` is shared by dev and prod-like clusters. It enables the
Kubernetes Ingress and CRD providers, creates the `traefik` IngressClass, binds
host ports 80/443, redirects HTTP to HTTPS, and uses a single-node-safe
DaemonSet rolling update. The service stays internal (`ClusterIP`) because the
VPS exposes Traefik through `hostPort`.

`values-patch.yaml` only publishes the OVH prod-like VPS address in
`status.loadBalancer.ingress`. The chart is pinned to `37.4.0`; for that
version, the IP is passed through `additionalArguments` because the chart does
not render `providers.kubernetesIngress.ingressEndpoint.ip` yet.

Apply the shared values and the prod-like IP override to the existing release:

```powershell
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm upgrade traefik traefik/traefik `
  --namespace ingress `
  --version 37.4.0 `
  --reset-values `
  --values infrastructure/traefik/base/values.yaml `
  --values infrastructure/traefik/prod/values-patch.yaml
```

For dev, use the same shared values and provide that VPS public IP at deploy
time instead of committing an unknown address:

```powershell
$DevPublicIp = "203.0.113.10"

helm upgrade traefik traefik/traefik `
  --namespace ingress `
  --version 37.4.0 `
  --reset-values `
  --values infrastructure/traefik/base/values.yaml `
  --set-string "additionalArguments[0]=--providers.kubernetesingress.ingressendpoint.ip=$DevPublicIp"
```

Replace the documentation-only example address before running the dev command.
`--reset-values` is intentional: it removes compatibility IngressClasses,
Gateway API objects, and providers previously inherited from the MicroK8s
Traefik addon.

Verify that Traefik advertises the production IP:

```powershell
kubectl get ingress -A -w
```

For prod-like, the `ADDRESS` column should become `137.74.174.36`.
