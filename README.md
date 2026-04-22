# belovr-cluster

Repo GitOps du cluster Belovr, restructuré pour préparer une montée en production sans mélanger le socle partagé, le spécifique dev et le bootstrap Argo CD.

## Structure

```text
argocd/
  bootstrap/        # Root applications a appliquer une seule fois
  project/          # AppProject Argo CD
environments/
  dev/              # Composition complete pour l'environnement de dev
  prod/             # Composition minimale et safe pour la prod
infrastructure/
  namespace/        # Namespace et ressources cluster/shared
  network/          # TLS, issuer, ingress et proxies
  observability/    # Prometheus, Loki, Grafana, OpenTelemetry Collector
  dev/              # Hub de dev et service multi-ports
  argocd/           # Exposition d'Argo CD propre a l'environnement
services/
  base/             # Applications Argo CD communes
  overlays/         # Variantes par environnement
docs/
  production-checklist.md
  nest-observability.md
  observability-runbook.md
```

## Principes

- Les `Application` Argo CD vivent dans `services/`.
- Le socle cluster et reseau vit dans `infrastructure/`.
- Les points d'entree de bootstrap vivent dans `argocd/bootstrap/`.
- Les environnements assemblent tout via Kustomize dans `environments/`.
- Le routage local/developpement est isole du chemin production.

## Bootstrap

Une fois Argo CD installe sur le cluster, applique une seule root application selon la cible :

```bash
kubectl apply -f argocd/bootstrap/root-dev.yaml
kubectl apply -f argocd/bootstrap/root-prod.yaml
```

Les root apps pointent vers ce repo GitHub :

- `https://github.com/nicolasdemol/belovr-cluster.git`

## Environnements

### `dev`

Inclut :

- le namespace `belovr`
- le `AppProject` Argo CD
- le wildcard TLS et le `ClusterIssuer`
- le hub multi-services de dev
- l'ingress public qui route vers le hub local
- l'ingress Argo CD en `nip.io`
- la stack observability Prometheus/Loki/Grafana/Otel
- toutes les applications de socle
- l'exposition RabbitMQ reservee au dev

### `prod`

Inclut :

- le namespace `belovr`
- le `AppProject` Argo CD
- le socle TLS partage
- la stack observability Prometheus/Loki/Grafana/Otel
- toutes les applications de socle

N'inclut pas volontairement :

- le hub de dev
- l'ingress public branche sur le proxy local
- l'ingress Argo CD de dev
- l'UI RabbitMQ en `nip.io`

## Conventions mises en place

- `sync-wave` explicites pour ordonner le bootstrap.
- `finalizers` sur les `Application` pour garder un prune propre.
- overlays par environnement au lieu de manifests a plat.
- AppProject dedie au scope Belovr.

## Suite logique

Le repo est maintenant propre pour accueillir une vraie couche prod. La check-list a fermer ensuite est ici :

- [docs/production-checklist.md](docs/production-checklist.md)
- [docs/nest-observability.md](docs/nest-observability.md)
- [docs/observability-runbook.md](docs/observability-runbook.md)
