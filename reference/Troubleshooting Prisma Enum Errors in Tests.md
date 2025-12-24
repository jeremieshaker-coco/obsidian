---
tags:
  - testing
  - troubleshooting
  - prisma
  - state-service
---
# Troubleshooting Prisma Enum Errors in Tests

## Error

When running unit or integration tests in the `state` service, you may see:

```
TypeError: Cannot read properties of undefined (reading 'LidEvent')
```
or
```
TypeError: Cannot read properties of undefined (reading 'LID_OPENED')
```

## Cause

This error usually indicates that the `PrismaClient` or its exported enums (like `LidEvent`) are `undefined` at runtime.

This happens when:
1.  The Prisma Client has not been generated for the current environment.
2.  The test runner (Jest) is resolving `@prisma/client` to an empty or uninitialized package.
3.  The generation output location does not match where the code expects to import it from.

## Solution

**Do not** manually mock the enums if you can avoid it. The preferred fix is to ensure the client is properly generated using the workspace command.

Run:
```bash
yarn workspace state run prisma:generate
```

This generates the client into the standard `node_modules` location, allowing `import { LidEvent } from '@prisma/client'` to work correctly in tests without extra configuration.

## Related
- [[Prisma Client Generation in Monorepo]]
- [[State Service Testing]]

