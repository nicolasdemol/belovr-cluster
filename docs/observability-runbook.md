# Observability runbook

## Deploiement

La stack est incluse dans `environments/dev` et `environments/prod`.

```bash
kubectl apply -f argocd/bootstrap/root-dev.yaml
kubectl -n argocd get applications
```

Applications attendues:

- `kube-prometheus-stack`
- `loki`
- `otel-collector-gateway`
- `otel-collector-agent`

Verification rapide:

```bash
kubectl -n observability get pods
kubectl -n observability get svc
kubectl -n observability port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Grafana: `http://localhost:3000`, user `admin`, password `adminpass`.

## Dashboards

Les dashboards sont provisionnes automatiquement dans le dossier `Belovr`:

- `Belovr - Overview`
- `Belovr - Payments`
- `Belovr - Auth`
- `Belovr - Chat`
- `Belovr - Cluster SLO`

## Alertes critiques

Les alertes sont livrees par `kube-prometheus-stack`:

- `BelovrDeploymentUnavailable`
- `PaymentNotCredited`
- `InvalidTokenCascade`
- `ChatBroadcastFailure`
- `RabbitMqBacklogCritical`

## Queries utiles

Prometheus:

```promql
sum by (service) (rate(http_requests_total{namespace="belovr"}[5m]))
histogram_quantile(0.95, sum by (service, le) (rate(http_request_duration_seconds_bucket{namespace="belovr"}[5m])))
sum(increase(belovr_payment_paid_total[15m])) - sum(increase(belovr_payment_credited_total[15m]))
sum by (service) (rate(http_requests_total{namespace="belovr",status_code=~"401|403"}[5m]))
sum by (queue) (rabbitmq_queue_messages_ready{namespace="belovr"})
```

Loki:

```logql
{k8s_namespace_name="belovr"}
{k8s_namespace_name="belovr"} |= "payment"
{k8s_namespace_name="belovr"} |~ "token|jwt|auth|401|403"
{k8s_namespace_name="belovr"} |= "chat"
```

## Simulation: paiement non credite

But: verifier l'alerte `PaymentNotCredited`, les logs JSON et le dashboard `Belovr - Payments`.

Pre-requis applicatif:

- le service paiement expose `belovr_payment_paid_total`;
- le credit wallet expose `belovr_payment_credited_total`;
- les echecs incrementent `belovr_payment_credit_failures_total`;
- un mode de simulation non-prod peut traiter un paiement sans le crediter.

Commande type en staging/dev:

```bash
kubectl -n belovr port-forward svc/payment-service 3001:3000
curl -X POST http://localhost:3001/ops/simulations/payment-not-credited \
  -H "content-type: application/json" \
  -H "x-request-id: sim-payment-not-credited-001" \
  -d '{"paymentId":"sim-payment-not-credited-001","amount":990}'
```

Signal attendu:

- log `level=error`, `request_id=sim-payment-not-credited-001`, `service=payment-service`;
- `belovr_payment_paid_total` augmente;
- `belovr_payment_credited_total` ne suit pas ou `belovr_payment_credit_failures_total` augmente;
- `PaymentNotCredited` passe en firing apres 5 minutes.

Rollback:

```bash
curl -X POST http://localhost:3001/ops/simulations/reset
```

## Simulation: token invalide en cascade

But: verifier l'alerte `InvalidTokenCascade` et detecter une rupture Keycloak/JWT.

Commande type:

```bash
kubectl -n belovr port-forward svc/user-service 3002:3000
for i in $(seq 1 500); do
  curl -s -o /dev/null \
    -H "authorization: Bearer invalid-token" \
    -H "x-request-id: sim-invalid-token-cascade-001" \
    http://localhost:3002/me &
done
wait
```

Signal attendu:

- logs contenant `request_id=sim-invalid-token-cascade-001`;
- hausse de `http_requests_total{status_code=~"401|403"}`;
- hausse de `belovr_auth_invalid_token_total`;
- `InvalidTokenCascade` passe en firing apres 5 minutes.

## Simulation: message chat non diffuse

But: verifier l'alerte `ChatBroadcastFailure`, le dashboard chat et la propagation RabbitMQ.

Commande type:

```bash
kubectl -n belovr port-forward svc/chat-service 3003:3000
curl -X POST http://localhost:3003/ops/simulations/chat-broadcast-failure \
  -H "content-type: application/json" \
  -H "x-request-id: sim-chat-broadcast-failure-001" \
  -d '{"roomId":"ops-sim","messageId":"sim-chat-message-001"}'
```

Signal attendu:

- `belovr_chat_messages_received_total` augmente;
- `belovr_chat_messages_broadcast_total` ne suit pas;
- `belovr_chat_broadcast_failures_total` augmente;
- logs RabbitMQ et chat partagent le meme `request_id`;
- `ChatBroadcastFailure` passe en firing apres 2 minutes.

## Backlog RabbitMQ

Si `RabbitMqBacklogCritical` fire:

1. Ouvrir le dashboard `Belovr - Chat`.
2. Identifier la queue avec:

```promql
sum by (queue) (rabbitmq_queue_messages_ready{namespace="belovr"})
```

3. Verifier les consumers:

```bash
kubectl -n belovr get pods
kubectl -n belovr logs deploy/chat-service --tail=200
```

4. Redemarrer uniquement le consumer bloque si necessaire:

```bash
kubectl -n belovr rollout restart deploy/chat-service
```

## Deployment indisponible

Si `BelovrDeploymentUnavailable` fire:

```bash
kubectl -n belovr get deploy
kubectl -n belovr describe deploy <deployment>
kubectl -n belovr get events --sort-by=.lastTimestamp
kubectl -n belovr logs deploy/<deployment> --tail=200
```

