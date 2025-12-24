---
tags:
  - testing
  - state-service
  - turbo
---
# State Service Testing

The `state` service uses `turbo` for running tests in the `delivery-platform` monorepo.

## Unit Tests

To run unit tests for the `state` service:

```bash
npx turbo run test --filter=state
```

This runs the `test` script defined in the `state` package's `package.json`, which typically executes `jest`.

## Integration Tests

To run integration tests:

```bash
npx turbo run test:integration --filter=state
```

This runs the `test:integration` script, which often includes setting specific environment variables (e.g., `dotenv -e .env.integration`) and pointing Jest to a specific config (e.g., `jest.config.integration.ts`).

## Prerequisites

If you encounter errors related to Prisma Client (e.g., `LidEvent` undefined), ensure you have generated the client first:

```bash
yarn workspace state run prisma:generate
```

## Related
- [[State Service]]
- [[Prisma Client Generation in Monorepo]]

