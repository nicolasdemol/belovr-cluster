# Migration production vers le modèle app-of-apps

Cette procédure fait adopter les ressources existantes par les nouvelles
Applications sans supprimer ni recréer les Deployments, StatefulSets, Services
ou PVC. Elle ne concerne que `belovr-root-prod`.

## Architecture cible

```text
belovr-root-prod
├── belovr-prod-platform
├── belovr-prod-secrets
├── belovr-prod-data
├── belovr-prod-identity
├── belovr-prod-services
│   └── ApplicationSet belovr-prod-microservices
│       ├── belovr-prod-ai
│       ├── belovr-prod-auth
│       ├── belovr-prod-chat
│       ├── belovr-prod-content
│       ├── belovr-prod-image
│       ├── belovr-prod-payment
│       ├── belovr-prod-realtime
│       ├── belovr-prod-social
│       ├── belovr-prod-user
│       └── belovr-prod-video
└── belovr-prod-observability
```

Les Applications Helm existantes conservent leurs noms et leurs
`helm.releaseName`. La migration ne modifie donc pas les noms des StatefulSets,
Services, Secrets Helm ou PVC.

## Pourquoi le basculement se fait en deux temps

Le root actuel possède directement les annotations de suivi des ressources.
Activer immédiatement le pruning après avoir changé son chemin pourrait créer
une course entre la suppression par l'ancien propriétaire et l'adoption par les
nouvelles Applications.

Le manifest `root-prod-migration.yaml` désactive temporairement le pruning.
Après adoption et vérification des UID, `root-prod.yaml` réactive la politique
finale.

## 1. Préflight avant publication

```bash
kubectl kustomize environments/dev >/dev/null
kubectl kustomize environments/prod >/dev/null
kubectl kustomize argocd/applications/prod >/dev/null
kubectl kustomize environments/prod/platform >/dev/null
kubectl kustomize environments/prod/secrets >/dev/null
kubectl kustomize environments/prod/data >/dev/null
kubectl kustomize environments/prod/identity >/dev/null
kubectl kustomize environments/prod/observability >/dev/null
kubectl kustomize argocd/applications/prod/services >/dev/null
```

Prendre un relevé des UID avant le basculement :

```bash
kubectl get deployments,statefulsets,services,persistentvolumeclaims \
  -n belovr \
  -o custom-columns='KIND:.kind,NAME:.metadata.name,UID:.metadata.uid' \
  --sort-by=.metadata.name
```

Conserver également l'état des PVC :

```bash
kubectl get pvc -n belovr -o wide
```

## 2. Installer le health check des Applications enfants

Argo CD ne propage pas automatiquement la santé d'une ressource `Application`
à son parent. Le patch Lua versionné restaure ce comportement. Il s'applique en
merge afin de préserver toutes les autres clés déjà présentes dans `argocd-cm` :

```bash
kubectl patch configmap argocd-cm \
  -n argocd \
  --type merge \
  --patch-file argocd/bootstrap/application-health-patch.yaml
```

Vérifier que la clé est présente :

```bash
kubectl get configmap argocd-cm -n argocd \
  -o jsonpath='{.data.resource\.customizations\.health\.argoproj\.io_Application}'
```

Ce patch doit être conservé dans la procédure d'installation ou dans les
valeurs Helm d'Argo CD si son installation devient elle-même gérée par Helm.

## 3. Publier sans basculer le root

Le fichier bootstrap n'est pas auto-géré par `belovr-root-prod`. Après le push,
le root live continue donc de cibler `environments/prod`. Attendre qu'il soit
`Synced` et `Healthy` avant la suite :

```bash
kubectl get application belovr-root-prod -n argocd
```

## 4. Créer les Applications enfants sans pruning

```bash
kubectl apply -f argocd/bootstrap/root-prod-migration.yaml
```

Attendre la création et la santé des six domaines :

```bash
kubectl wait application -n argocd \
  -l 'belovr.io/environment=prod' \
  --for=jsonpath='{.status.health.status}'=Healthy \
  --timeout=15m
```

Vérifier les dix Applications générées :

```bash
kubectl get applications -n argocd \
  -l 'belovr.io/environment=prod,belovr.io/tier=services'
```

Les avertissements temporaires de ressource partagée sont attendus pendant le
changement d'annotation de suivi. En revanche, ne pas réactiver le pruning tant
qu'une Application enfant est absente, `OutOfSync` ou `Degraded`.

## 5. Vérifier l'adoption et les UID

Les annotations doivent maintenant référencer les nouveaux propriétaires :

```bash
kubectl get deployment ai -n belovr \
  -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}{"\n"}'

kubectl get application chat-db -n argocd \
  -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}{"\n"}'
```

Valeurs attendues :

```text
belovr-prod-ai:apps/Deployment:belovr/ai
belovr-prod-data:argoproj.io/Application:argocd/chat-db
```

Rejouer ensuite le relevé des UID de l'étape 1. Tous les UID existants, en
particulier ceux des StatefulSets et PVC, doivent être identiques.

## 6. Réactiver la politique finale

Uniquement après validation de toutes les Applications et des UID :

```bash
kubectl apply -f argocd/bootstrap/root-prod.yaml
kubectl wait application belovr-root-prod -n argocd \
  --for=jsonpath='{.status.health.status}'=Healthy \
  --timeout=15m
```

Contrôles finaux :

```bash
kubectl get applications -n argocd \
  -l 'belovr.io/environment=prod' \
  -L belovr.io/tier -L belovr.io/service
kubectl get pods -n belovr
kubectl get deployments -n belovr
kubectl get pvc -n belovr
kubectl get events -n belovr --sort-by=.lastTimestamp
```

## Rollback avant réactivation du pruning

Tant que l'étape 6 n'a pas été exécutée, le rollback est non destructif :

```bash
kubectl patch application belovr-root-prod -n argocd --type merge \
  -p '{"spec":{"source":{"path":"environments/prod"},"syncPolicy":{"automated":{"prune":false,"selfHeal":true}}}}'
```

Attendre le retour de l'ancien rendu puis analyser les annotations de suivi
avant de réactiver le pruning.
