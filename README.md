# belovr-cluster

Repo GitOps du cluster Belovr. Les environnements `dev` et `prod` reposent sur
le même socle afin que `prod` serve de préproduction fidèle, avec seulement
quelques différences explicites.

## Structure

```text
argocd/
  bootstrap/        # Root applications a appliquer une seule fois
  project/          # AppProject Argo CD
environments/
  base/             # Composition Kubernetes commune
  dev/              # Overlay de développement
  prod/             # Overlay de préproduction nommé prod
infrastructure/
  namespace/        # Namespace et ressources cluster/shared
  network/          # TLS, issuer, ingress et proxies
  observability/    # Prometheus, Loki, Grafana, OpenTelemetry Collector
  dev/              # Hub de dev et service multi-ports
  argocd/           # Exposition HTTP interne d'Argo CD via Traefik
  traefik/          # Valeurs Helm communes et override IP par VPS
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
- `environments/base` assemble le socle identique aux deux environnements.
- Les overlays `dev` et `prod` ne portent que leurs différences réelles.
- Traefik termine le TLS public et route en HTTP vers les Services internes.

## Bootstrap

Une fois Argo CD installé sur le cluster, applique une seule root application
selon la cible. Les deux root applications sont mutuellement exclusives sur un
même cluster :

```bash
kubectl apply -f argocd/bootstrap/root-dev.yaml
kubectl apply -f argocd/bootstrap/root-prod.yaml
```

Les root apps pointent vers ce repo GitHub :

- `https://github.com/nicolasdemol/belovr-cluster.git`

## Socle commun

`dev` et `prod` déploient tous les deux :

- le namespace `belovr` et le `AppProject` Argo CD ;
- le `ClusterIssuer` Cloudflare DNS-01 et les certificats TLS ;
- `belovr-main-ingress`, l'Ingress Argo CD et l'Ingress OTEL ;
- les Middlewares Traefik CORS et limites d'upload ;
- Prometheus, Loki, Grafana et OpenTelemetry Collector ;
- toutes les applications définies dans `services/base` ;
- le hub multi-ports temporaire `belovr-dev-service`.

Le hub reste un placeholder basé sur une image `pause`. Avant un vrai go-live,
les routes publiques devront viser les Services applicatifs réels.

## Différences d'environnement

| Point | `dev` | `prod` (préproduction) |
| --- | --- | --- |
| Label | `belovr.io/environment: dev` | `belovr.io/environment: prod` |
| UI RabbitMQ publique | Activée sur `rabbitmq.lab.belovr.com` | Désactivée |
| IP publiée par Traefik | Fournie lors du déploiement Helm | `137.74.174.36` via l'override versionné |

La configuration Traefik commune se trouve dans
`infrastructure/traefik/base/values.yaml`. Les commandes d'installation et la
gestion de l'IP sont documentées dans `infrastructure/traefik/prod/README.md`.

## Conventions mises en place

- `sync-wave` explicites pour ordonner le bootstrap.
- `finalizers` sur les `Application` pour garder un prune propre.
- socle Kustomize commun et overlays minces par environnement.
- AppProject dedie au scope Belovr.

## Suite logique

La configuration `prod` reproduit maintenant la composition de `dev`. Les
travaux restant avant une vraie production sont listés ici :

- [docs/production-checklist.md](docs/production-checklist.md)
- [docs/nest-observability.md](docs/nest-observability.md)
- [docs/observability-runbook.md](docs/observability-runbook.md)
