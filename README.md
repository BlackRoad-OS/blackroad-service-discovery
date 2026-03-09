# 🛣️🔍 blackroad-service-discovery

> **Official service discovery client for the BlackRoad OS platform.**  
> Zero-config, production-ready — resolves microservices, agents, and infrastructure endpoints across the entire BlackRoad ecosystem.

[![npm version](https://img.shields.io/npm/v/@blackroad/service-discovery.svg)](https://www.npmjs.com/package/@blackroad/service-discovery)
[![License: Proprietary](https://img.shields.io/badge/license-Proprietary-red.svg)](./LICENSE)
[![Node.js ≥ 18](https://img.shields.io/badge/node-%3E%3D18-brightgreen.svg)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue.svg)](https://www.typescriptlang.org/)
[![BlackRoad OS](https://img.shields.io/badge/BlackRoad-OS-black.svg)](https://github.com/BlackRoad-OS)

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [API Reference](#api-reference)
   - [Client](#client)
   - [discover()](#discover)
   - [register()](#register)
   - [deregister()](#deregister)
   - [watch()](#watch)
   - [health()](#health)
6. [Stripe Integration](#stripe-integration)
   - [Billing Service Resolution](#billing-service-resolution)
   - [Webhook Endpoint Discovery](#webhook-endpoint-discovery)
7. [Configuration](#configuration)
8. [Environment Variables](#environment-variables)
9. [End-to-End Testing](#end-to-end-testing)
10. [Error Handling](#error-handling)
11. [Security](#security)
12. [Contributing](#contributing)
13. [License](#license)

---

## Overview

`blackroad-service-discovery` is the canonical service registry and lookup client for BlackRoad OS. It enables any service, agent, or application running inside the BlackRoad platform to:

- **Resolve** the live endpoint of any registered service by logical name
- **Register** itself so other services can find it
- **Watch** for real-time endpoint changes without polling
- **Health-check** registered services and surface degraded nodes automatically

It integrates directly with the BlackRoad infrastructure layer, supporting Stripe billing endpoints, GraphQL gateways, webhook routers, email services, and the full portfolio of BlackRoad microservices.

---

## Features

| Feature | Description |
|---|---|
| 🔍 **Service Lookup** | Resolve any registered service by name in < 5 ms |
| 📝 **Service Registration** | Register your service at startup with a single call |
| 👁️ **Real-Time Watching** | Subscribe to endpoint change events via SSE/WebSocket |
| 💳 **Stripe-Aware** | Built-in helpers for resolving Stripe billing and webhook endpoints |
| 🔒 **Auth-Ready** | Passes BlackRoad API keys automatically |
| 🌐 **Multi-Region** | Region-aware resolution with automatic failover |
| 🛡️ **Health Checks** | Integrated health probe and circuit-breaker support |
| 📦 **Zero Dependencies** | Ships with zero mandatory runtime dependencies |
| 🔷 **Full TypeScript** | 100% typed — works equally well from JS or TS |

---

## Installation

```bash
# npm
npm install @blackroad/service-discovery

# yarn
yarn add @blackroad/service-discovery

# pnpm
pnpm add @blackroad/service-discovery
```

> **Requires Node.js ≥ 18** and a valid `BLACKROAD_API_KEY`.

---

## Quick Start

```ts
import { ServiceDiscovery } from '@blackroad/service-discovery';

const sd = new ServiceDiscovery({
  apiKey: process.env.BLACKROAD_API_KEY,
});

// Discover the live URL of the payments service
const payments = await sd.discover('payments');
console.log(payments.url); // https://payments.blackroad.io

// Register the current service
await sd.register({
  name: 'my-service',
  url: 'https://my-service.example.com',
  healthPath: '/health',
});
```

---

## API Reference

### Client

```ts
import { ServiceDiscovery } from '@blackroad/service-discovery';

const sd = new ServiceDiscovery(options);
```

#### Constructor options

| Option | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | `BLACKROAD_API_KEY` env | BlackRoad API key |
| `registryUrl` | `string` | `https://registry.blackroad.io` | Registry base URL |
| `region` | `string` | `'us-east-1'` | Preferred resolution region |
| `timeout` | `number` | `5000` | Request timeout in ms |
| `retries` | `number` | `3` | Automatic retry count |

---

### discover()

Resolve the live endpoint of a registered service.

```ts
const service = await sd.discover(name: string, options?: DiscoverOptions);
```

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Logical service name (e.g. `'payments'`, `'graphql'`) |
| `options.region` | `string` | Override default region |
| `options.tags` | `string[]` | Filter by service tags |

**Returns** `Promise<ServiceRecord>`

```ts
interface ServiceRecord {
  name:      string;
  url:       string;
  region:    string;
  version:   string;
  healthy:   boolean;
  tags:      string[];
  updatedAt: string; // ISO 8601
}
```

**Example**

```ts
const graphql = await sd.discover('graphql');
// { name: 'graphql', url: 'https://blackroad-graphql-gateway.amundsonalexa.workers.dev', ... }
```

---

### register()

Register the current service in the registry.

```ts
await sd.register(payload: RegisterPayload);
```

**Parameters**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | ✅ | Unique service name |
| `url` | `string` | ✅ | Publicly reachable base URL |
| `healthPath` | `string` | — | Health-check path (default: `/health`) |
| `version` | `string` | — | Semver version string |
| `tags` | `string[]` | — | Searchable tags |
| `ttl` | `number` | — | TTL in seconds (default: `30`) |

**Example**

```ts
await sd.register({
  name: 'order-service',
  url: 'https://orders.blackroad.io',
  healthPath: '/health',
  version: '2.1.0',
  tags: ['orders', 'commerce', 'stripe'],
  ttl: 60,
});
```

---

### deregister()

Remove a service from the registry.

```ts
await sd.deregister(name: string);
```

---

### watch()

Subscribe to live changes for a service endpoint. Emits `ServiceRecord` events whenever the registry is updated.

```ts
const watcher = sd.watch('payments');

watcher.on('update', (record: ServiceRecord) => {
  console.log('New endpoint:', record.url);
});

watcher.on('error', (err: Error) => {
  console.error('Watch error:', err.message);
});

// Stop watching
watcher.close();
```

---

### health()

Check the health status of a registered service.

```ts
const status = await sd.health(name: string);
// { healthy: true, latencyMs: 12, checkedAt: '2026-03-01T00:00:00Z' }
```

---

## Stripe Integration

`blackroad-service-discovery` includes first-class helpers for resolving Stripe-related infrastructure.

### Billing Service Resolution

```ts
import { ServiceDiscovery } from '@blackroad/service-discovery';

const sd = new ServiceDiscovery({ apiKey: process.env.BLACKROAD_API_KEY });

// Resolve the active Stripe billing endpoint
const billing = await sd.discover('stripe-billing');
console.log(billing.url); // https://billing.blackroad.io

// Call Stripe billing API through the resolved endpoint
const response = await fetch(`${billing.url}/subscriptions`, {
  headers: { Authorization: `Bearer ${process.env.BLACKROAD_API_KEY}` },
});
```

### Webhook Endpoint Discovery

Stripe webhooks must point to a stable, discoverable URL. Use the registry to route webhook events to the correct handler:

```ts
// Discover the live webhook handler endpoint
const webhooks = await sd.discover('stripe-webhooks');

// Register your local webhook handler for development
await sd.register({
  name: 'stripe-webhooks',
  url: 'https://my-app.example.com/webhooks/stripe',
  tags: ['stripe', 'webhooks', 'payments'],
});
```

> **Stripe Tip**: Set your Stripe webhook endpoint in the [Stripe Dashboard](https://dashboard.stripe.com/webhooks) to the URL returned by `sd.discover('stripe-webhooks')`. The registry handles failover automatically.

---

## Configuration

Create a `blackroad.config.ts` (or `.js`) at your project root for project-level defaults:

```ts
// blackroad.config.ts
import type { ServiceDiscoveryConfig } from '@blackroad/service-discovery';

const config: ServiceDiscoveryConfig = {
  registryUrl: 'https://registry.blackroad.io',
  region: 'us-east-1',
  timeout: 5000,
  retries: 3,
  services: {
    // Pin a service to a specific version tag
    'graphql': { tags: ['stable'] },
    'stripe-billing': { tags: ['production'] },
  },
};

export default config;
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `BLACKROAD_API_KEY` | ✅ | Your BlackRoad API key |
| `BLACKROAD_REGISTRY_URL` | — | Override registry URL |
| `BLACKROAD_REGION` | — | Preferred region (default: `us-east-1`) |
| `BLACKROAD_TIMEOUT` | — | Request timeout in ms (default: `5000`) |
| `BLACKROAD_LOG_LEVEL` | — | Log level: `debug` \| `info` \| `warn` \| `error` |

---

## End-to-End Testing

Run the full E2E test suite against the live BlackRoad registry:

```bash
# Install dev dependencies
npm install

# Run unit tests
npm test

# Run E2E tests (requires BLACKROAD_API_KEY)
BLACKROAD_API_KEY=your-key npm run test:e2e

# Run tests in watch mode
npm run test:watch
```

> **CI/CD**: All pull requests are validated against the live staging registry before merge. E2E tests must pass on every PR.

### Mocking the Registry (for local unit tests)

```ts
import { ServiceDiscovery, MockRegistry } from '@blackroad/service-discovery/testing';

const mock = new MockRegistry();
mock.set('payments', { url: 'http://localhost:3001' });

const sd = new ServiceDiscovery({ registry: mock });
const svc = await sd.discover('payments');
// svc.url === 'http://localhost:3001'
```

---

## Error Handling

```ts
import { ServiceNotFoundError, RegistryUnavailableError } from '@blackroad/service-discovery';

try {
  const svc = await sd.discover('unknown-service');
} catch (err) {
  if (err instanceof ServiceNotFoundError) {
    // Service not registered
    console.error('Service not found:', err.serviceName);
  } else if (err instanceof RegistryUnavailableError) {
    // Registry unreachable — check BLACKROAD_REGISTRY_URL
    console.error('Registry unavailable:', err.message);
  } else {
    throw err;
  }
}
```

---

## Security

- All registry requests are authenticated with your `BLACKROAD_API_KEY`.
- API keys are never logged or included in error messages.
- Communication with the registry is over HTTPS only.
- To report a security vulnerability, email **security@blackroad.io** — do **not** open a public issue.

---

## Contributing

This repository is proprietary. Internal BlackRoad OS contributors should follow the standard PR workflow:

1. Branch from `main` — use the format `feat/`, `fix/`, or `chore/`
2. Run `npm test` and `npm run test:e2e` before opening a PR
3. All PRs require one reviewer approval before merge
4. Squash-merge only

---

## License

**Copyright © 2024–2026 BlackRoad OS, Inc. All Rights Reserved.**

This software is proprietary and confidential. See [LICENSE](./LICENSE) for full terms.

**CEO & Sole Stockholder**: Alexa Louise Amundson

---

<p align="center">
  <strong>BlackRoad OS, Inc.</strong> · blackroad.io · security@blackroad.io
</p>
