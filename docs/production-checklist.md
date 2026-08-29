# Production checklist

Ce repo est maintenant structure pour la prod, mais il reste quelques verrous avant un go-live serein.

## 1. Secrets

- Generer les SealedSecrets propres a chaque environnement avant la
  synchronisation des workloads qui les consomment.
- Planifier la rotation des credentials historiques apres validation de la
  migration vers les Secrets externes.

## 2. Images et versions

- Remplacer les tags flottants comme `latest` par des versions immuables.
- Verrouiller les charts Helm sur des versions validees en pre-prod.

## 3. Ingress production

- Remplacer les backends temporaires `belovr-dev-service` par les vrais Services applicatifs.
- Valider les noms DNS finaux et la politique de certificats associee.

## 4. Donnees stateful

- Definir des `StorageClass` adaptees a la prod.
- Mettre en place backups, snapshots et tests de restauration.
- Valider les tailles de PVC par service.

## 5. Securite

- Ajouter des `NetworkPolicy`.
- Ajouter des `PodDisruptionBudget` et des `resource quotas` la ou c'est critique.
- Limiter l'exposition publique des interfaces d'admin.

## 6. Observabilite

- Verifier la stack `kube-prometheus-stack`, `loki` et `otel-collector`.
- Integrer le contrat applicatif de [nest-observability.md](nest-observability.md).
- Verifier que `grafana-admin-secrets` est provisionne dans le namespace
  `observability` du cluster cible.
- Valider les dashboards et alertes avec [observability-runbook.md](observability-runbook.md).

## 7. Exploitation

- Ecrire la procedure de bootstrap cluster.
- Documenter le rollback et le disaster recovery.
- Definir les responsabilites entre plateforme et applicatif.
