# belovr-cluster

Repo GitOps du cluster Belovr. Les environnements `dev` et `prod` reposent sur
le même socle afin que `prod` serve de préproduction fidèle, avec seulement
quelques différences explicites.

## Structure

```text
argocd/
  bootstrap/        # Root applications a appliquer une seule fois
  applications/     # Catalogue app-of-apps de production
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
- Le root `prod` ne rend que les six Applications de domaine définies dans
  `argocd/applications/prod/`.
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

Un cluster `prod` existant doit suivre la procédure de migration sans pruning
avant d'appliquer le nouveau root :

- [docs/app-of-apps-migration.md](docs/app-of-apps-migration.md)

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

## Secrets PostgreSQL applicatifs

Les charts PostgreSQL applicatifs utilisent des Secrets Kubernetes externes.
Chaque environnement doit provisionner ses propres Secrets, chiffrés avec le
certificat Sealed Secrets du cluster concerné, avant la synchronisation des
Applications PostgreSQL :

| Application | Secret externe | Clé attendue |
| --- | --- | --- |
| `chat-db` | `chat-db-secrets` | `CHAT_DB_PASSWORD` |
| `content-db` | `content-db-secrets` | `CONTENT_DB_PASSWORD` |
| `payment-db` | `payment-db-secrets` | `PAYMENT_DB_PASSWORD` |
| `user-db` | `user-db-secrets` | `APP_DB_PASSWORD` |

Les SealedSecrets de production restent dans
`services/overlays/prod/secrets/`. Ceux de développement doivent être générés
séparément avec le certificat du cluster dev et ne doivent jamais réutiliser
le ciphertext de prod. Les fichiers DB générés doivent être ajoutés au
`kustomization.yaml` de l'overlay correspondant. En prod, cet overlay applique
la sync-wave `10`, avant les Applications PostgreSQL en wave `20` et les
workloads applicatifs en wave `40`.

Lors d'une migration avec des PVC existants, le Secret externe doit contenir
le mot de passe déjà enregistré dans PostgreSQL. Changer uniquement le Secret
ne réalise pas une rotation du mot de passe dans la base.

Les autres composants partagés utilisent également des Secrets externes
propres à chaque environnement :

| Composant | Namespace | Secret externe | Clés attendues |
| --- | --- | --- | --- |
| PostgreSQL Keycloak et Keycloak | `belovr` | `keycloak-db-secrets` | `KEYCLOAK_DB_USERNAME`, `KEYCLOAK_DB_PASSWORD` |
| Administration Keycloak | `belovr` | `keycloak-admin-secrets` | `KEYCLOAK_ADMIN_PASSWORD` |
| RabbitMQ | `belovr` | `rabbitmq-infra-secrets` | `RABBITMQ_PASSWORD`, `RABBITMQ_ERLANG_COOKIE` |
| Grafana | `observability` | `grafana-admin-secrets` | `GRAFANA_ADMIN_USER`, `GRAFANA_ADMIN_PASSWORD` |

Le Secret applicatif `belovr-rabbitmq-secrets`, qui expose actuellement une
`AMQP_URL` complète aux microservices, reste distinct du Secret infrastructure
de RabbitMQ. Cette duplication chiffrée ne pourra être supprimée qu'après une
évolution des applications permettant de construire l'URL depuis des champs
séparés ou l'adoption d'un contrôleur capable de dériver plusieurs Secrets
depuis une source de credential unique.

En production, les SealedSecrets correspondants doivent rester dans
`services/overlays/prod/secrets/` et sont appliqués en wave `10`. La stack
d'observabilité est créée en wave `15`, les bases et RabbitMQ en wave `20`,
Keycloak en wave `30`, puis les microservices en wave `40`. Les clusters dev
doivent provisionner leurs propres versions de ces Secrets avec leur propre
certificat Sealed Secrets.

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
