# Traefik production override

This override makes Traefik publish the OVH VPS address in
`status.loadBalancer.ingress` for Kubernetes Ingress resources. It also uses a
single-node-safe rolling update because the Traefik pod reserves host ports 80
and 443.

The chart is pinned to `37.4.0`. For that version, the advertised IP is passed
through `additionalArguments` because the chart does not render
`providers.kubernetesIngress.ingressEndpoint.ip` yet.

Apply it to the existing Helm release while preserving all other release
values:

```powershell
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm upgrade traefik traefik/traefik `
  --namespace ingress `
  --version 37.4.0 `
  --reuse-values `
  --values infrastructure/traefik/prod/values-patch.yaml
```

Verify that Traefik advertises the production IP:

```powershell
kubectl get ingress -A -w
```

The `ADDRESS` column should become `137.74.174.36`.
