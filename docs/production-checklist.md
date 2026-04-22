# Production checklist

Ce repo est maintenant structure pour la prod, mais il reste quelques verrous avant un go-live serein.

## 1. Secrets

- Sortir tous les mots de passe hardcodes des manifests Helm.
- Brancher un mecanisme type External Secrets, SOPS ou Vault.
- Remplacer les credentials admin par des secrets geres hors Git.

## 2. Images et versions

- Remplacer les tags flottants comme `latest` par des versions immuables.
- Verrouiller les charts Helm sur des versions validees en pre-prod.

## 3. Ingress production

- Ajouter un ingress prod qui route vers les vrais services applicatifs, pas vers le hub de dev.
- Definir les noms DNS finaux et la politique de certificats associee.

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
- Remplacer le password Grafana hardcode par un secret gere hors Git.
- Valider les dashboards et alertes avec [observability-runbook.md](observability-runbook.md).

## 7. Exploitation

- Ecrire la procedure de bootstrap cluster.
- Documenter le rollback et le disaster recovery.
- Definir les responsabilites entre plateforme et applicatif.
