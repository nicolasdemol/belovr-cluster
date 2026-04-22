# Observabilite NestJS

Ce contrat rend les services Nest compatibles avec la stack du cluster:

- logs JSON collectes par OpenTelemetry Collector et envoyes a Loki;
- `requestId` stable sur HTTP et RabbitMQ;
- contexte propage entre services;
- endpoint `/metrics` scrape par Prometheus;
- champs de logs normalises entre services.

## Variables d'environnement

Chaque service doit exposer au minimum:

```bash
SERVICE_NAME=payment-service
SERVICE_VERSION=1.0.0
NODE_ENV=production
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector-gateway.observability.svc.cluster.local:4318
```

## Annotations Kubernetes

Le `Service` Kubernetes applicatif doit etre scrape par Prometheus:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: /metrics
    prometheus.io/port: "3000"
```

## Dependances Node

```bash
npm i nestjs-pino pino pino-http prom-client uuid
npm i @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http @opentelemetry/exporter-metrics-otlp-http
npm i @opentelemetry/resources @opentelemetry/semantic-conventions @opentelemetry/instrumentation-http @opentelemetry/instrumentation-express @opentelemetry/instrumentation-amqplib
```

## Champs de logs normalises

Tous les services doivent logger ces champs quand ils sont connus:

| Champ | Exemple | Source |
| --- | --- | --- |
| `timestamp` | `2026-04-22T10:00:00.000Z` | logger |
| `level` | `info`, `warn`, `error` | logger |
| `message` | `payment credited` | logger |
| `service` | `payment-service` | env |
| `env` | `production` | env |
| `version` | `1.0.0` | env |
| `request_id` | `01HV...` | middleware |
| `trace_id` | `0af765...` | OpenTelemetry |
| `span_id` | `b7ad6b...` | OpenTelemetry |
| `user_id` | `usr_123` | auth |
| `route` | `/payments/:id` | HTTP |
| `method` | `POST` | HTTP |
| `status_code` | `200` | HTTP |
| `duration_ms` | `42` | HTTP/RabbitMQ |
| `transport` | `http`, `rabbitmq` | caller |
| `rabbitmq_exchange` | `payments` | RabbitMQ |
| `rabbitmq_routing_key` | `payment.succeeded` | RabbitMQ |
| `message_id` | `msg_123` | RabbitMQ |
| `error_name` | `PaymentCreditError` | exceptions |
| `error_message` | `wallet not found` | exceptions |

## Contexte requestId

```ts
// observability/context.ts
import { AsyncLocalStorage } from 'node:async_hooks';

export type RequestContext = {
  requestId: string;
  userId?: string;
  traceId?: string;
  spanId?: string;
};

export const requestContext = new AsyncLocalStorage<RequestContext>();

export function getRequestContext(): RequestContext | undefined {
  return requestContext.getStore();
}
```

```ts
// observability/request-id.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { randomUUID } from 'node:crypto';
import { requestContext } from './context';

const REQUEST_ID_HEADER = 'x-request-id';

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: any, res: any, next: () => void): void {
    const requestId = req.headers[REQUEST_ID_HEADER] ?? randomUUID();
    req.requestId = requestId;
    res.setHeader(REQUEST_ID_HEADER, requestId);

    requestContext.run({ requestId }, next);
  }
}
```

Dans `AppModule`:

```ts
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { RequestIdMiddleware } from './observability/request-id.middleware';

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer.apply(RequestIdMiddleware).forRoutes('*');
  }
}
```

## Logger JSON

```ts
// observability/logger.ts
import { Params } from 'nestjs-pino';
import { getRequestContext } from './context';

export function loggerConfig(): Params {
  return {
    pinoHttp: {
      level: process.env.LOG_LEVEL ?? 'info',
      messageKey: 'message',
      timestamp: () => `,"timestamp":"${new Date().toISOString()}"`,
      formatters: {
        level(label) {
          return { level: label };
        }
      },
      customProps() {
        const ctx = getRequestContext();
        return {
          service: process.env.SERVICE_NAME,
          env: process.env.NODE_ENV,
          version: process.env.SERVICE_VERSION,
          request_id: ctx?.requestId,
          user_id: ctx?.userId
        };
      },
      serializers: {
        req(req) {
          return {
            method: req.method,
            route: req.route?.path ?? req.url,
            url: req.url
          };
        },
        res(res) {
          return {
            status_code: res.statusCode
          };
        },
        err(err) {
          return {
            error_name: err.name,
            error_message: err.message,
            stack: err.stack
          };
        }
      }
    }
  };
}
```

Dans `AppModule`:

```ts
import { LoggerModule } from 'nestjs-pino';
import { loggerConfig } from './observability/logger';

@Module({
  imports: [LoggerModule.forRoot(loggerConfig())]
})
export class AppModule {}
```

## Propagation HTTP

Avec `HttpService`/Axios, injecter les headers avant chaque appel sortant:

```ts
// observability/http-propagation.ts
import { trace, propagation, context } from '@opentelemetry/api';
import { getRequestContext } from './context';

export function injectOutboundHeaders(headers: Record<string, string>): Record<string, string> {
  const ctx = getRequestContext();

  if (ctx?.requestId) {
    headers['x-request-id'] = ctx.requestId;
  }

  propagation.inject(context.active(), headers);

  const span = trace.getSpan(context.active());
  const spanContext = span?.spanContext();
  if (spanContext) {
    headers['x-trace-id'] = spanContext.traceId;
    headers['x-span-id'] = spanContext.spanId;
  }

  return headers;
}
```

```ts
// main.ts
app.get(HttpService).axiosRef.interceptors.request.use((config) => {
  config.headers = injectOutboundHeaders(config.headers ?? {});
  return config;
});
```

## Propagation RabbitMQ

Chaque publish doit porter le contexte dans les headers:

```ts
// observability/rabbitmq-propagation.ts
import { propagation, context } from '@opentelemetry/api';
import { randomUUID } from 'node:crypto';
import { getRequestContext, requestContext } from './context';

export function buildRabbitHeaders(headers: Record<string, string> = {}): Record<string, string> {
  const ctx = getRequestContext();
  const nextHeaders = { ...headers };

  if (ctx?.requestId) {
    nextHeaders['x-request-id'] = ctx.requestId;
  }

  propagation.inject(context.active(), nextHeaders);
  return nextHeaders;
}

export function runWithRabbitContext<T>(
  headers: Record<string, string> | undefined,
  handler: () => Promise<T>
): Promise<T> {
  const requestId = headers?.['x-request-id'] ?? randomUUID();
  return requestContext.run({ requestId }, handler);
}
```

Regle d'equipe: aucun publish RabbitMQ sans `buildRabbitHeaders()`, aucun consumer sans `runWithRabbitContext()`.

## Metriques applicatives

```ts
// observability/metrics.ts
import { collectDefaultMetrics, Counter, Histogram, Registry } from 'prom-client';

export const registry = new Registry();

registry.setDefaultLabels({
  service: process.env.SERVICE_NAME,
  namespace: 'belovr'
});

collectDefaultMetrics({ register: registry });

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['service', 'method', 'route', 'status_code'],
  registers: [registry]
});

export const httpRequestDurationSeconds = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['service', 'method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [registry]
});

export const paymentPaidTotal = new Counter({
  name: 'belovr_payment_paid_total',
  help: 'Paid payment events',
  registers: [registry]
});

export const paymentCreditedTotal = new Counter({
  name: 'belovr_payment_credited_total',
  help: 'Credited payment events',
  registers: [registry]
});

export const paymentCreditFailuresTotal = new Counter({
  name: 'belovr_payment_credit_failures_total',
  help: 'Payment credit failures',
  registers: [registry]
});

export const authInvalidTokenTotal = new Counter({
  name: 'belovr_auth_invalid_token_total',
  help: 'Invalid token events',
  labelNames: ['service', 'reason'],
  registers: [registry]
});

export const chatMessagesReceivedTotal = new Counter({
  name: 'belovr_chat_messages_received_total',
  help: 'Chat messages received',
  registers: [registry]
});

export const chatMessagesBroadcastTotal = new Counter({
  name: 'belovr_chat_messages_broadcast_total',
  help: 'Chat messages broadcast',
  registers: [registry]
});

export const chatBroadcastFailuresTotal = new Counter({
  name: 'belovr_chat_broadcast_failures_total',
  help: 'Chat broadcast failures',
  registers: [registry]
});
```

Endpoint `/metrics`:

```ts
import { Controller, Get, Header } from '@nestjs/common';
import { registry } from './observability/metrics';

@Controller()
export class MetricsController {
  @Get('/metrics')
  @Header('Content-Type', registry.contentType)
  metrics(): Promise<string> {
    return registry.metrics();
  }
}
```

## Middleware metriques HTTP

```ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { httpRequestDurationSeconds, httpRequestsTotal } from './metrics';

@Injectable()
export class HttpMetricsMiddleware implements NestMiddleware {
  use(req: any, res: any, next: () => void): void {
    const start = process.hrtime.bigint();

    res.on('finish', () => {
      const durationSeconds = Number(process.hrtime.bigint() - start) / 1e9;
      const service = process.env.SERVICE_NAME ?? 'unknown';
      const route = req.route?.path ?? req.path ?? 'unknown';
      const labels = {
        service,
        method: req.method,
        route,
        status_code: String(res.statusCode)
      };

      httpRequestsTotal.inc(labels);
      httpRequestDurationSeconds.observe(labels, durationSeconds);
    });

    next();
  }
}
```

## OpenTelemetry SDK

```ts
// observability/otel.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express';
import { AmqplibInstrumentation } from '@opentelemetry/instrumentation-amqplib';
import { resourceFromAttributes } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

const endpoint = process.env.OTEL_EXPORTER_OTLP_ENDPOINT;

export const otelSdk = new NodeSDK({
  resource: resourceFromAttributes({
    [ATTR_SERVICE_NAME]: process.env.SERVICE_NAME ?? 'unknown-service',
    [ATTR_SERVICE_VERSION]: process.env.SERVICE_VERSION ?? '0.0.0'
  }),
  traceExporter: new OTLPTraceExporter({
    url: `${endpoint}/v1/traces`
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: `${endpoint}/v1/metrics`
    })
  }),
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
    new AmqplibInstrumentation()
  ]
});
```

Charger avant Nest:

```ts
import { otelSdk } from './observability/otel';

otelSdk.start();
```
