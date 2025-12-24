---
tags:
  - prisma
  - typescript
  - monorepo
  - troubleshooting
---
# Prisma Client Generation in Monorepo

In the `delivery-platform` monorepo, generating the Prisma client correctly is critical for avoiding runtime errors in tests and the application, specifically regarding `undefined` enums or models.

## The Issue

When running tests for a service (e.g., `state`), you might encounter errors like:

```
TypeError: Cannot read properties of undefined (reading 'LidEvent')
```

This occurs when the test runner imports `@prisma/client`, but the client hasn't been generated or is not being resolved correctly in the workspace context.

## The Fix

The robust solution is to run the generation command via the workspace manager (`yarn workspace`) rather than relying on `npx prisma generate` locally within the directory or custom output paths.

### Command

From the root of the repository:

```bash
yarn workspace state run prisma:generate
```

*Replace `state` with the name of the specific workspace.*

### Why this works

1.  **Workspace Context**: `yarn workspace ... run` ensures the command runs with the correct node environment and path resolution for the specific package.
2.  **Standard Location**: It generates the client into `node_modules/@prisma/client` (or wherever the service is configured to look), which matches the standard import `import { ... } from '@prisma/client'`.
3.  **No Config Changes**: You typically do not need to modify `tsconfig.json` `paths` or `jest.config.ts` `moduleNameMapper` if you use this method, as the node resolution algorithm will find the correctly generated client in `node_modules`.

## Common Pitfalls

*   **Custom Output Paths**: Manually changing the `output` in `schema.prisma` to a local directory (e.g., `../src/generated`) often requires corresponding updates to `tsconfig.json` and `jest.config.ts` to map `@prisma/client` to that new path. This adds complexity and maintenance overhead.
*   **Running `npx prisma generate` directly**: Running this inside the service directory might not always pick up the correct environment variables or `node_modules` structure required by the monorepo tooling.

## Related
- [[State Service]]
- [[Unit Testing NestJS Services]]
- [[Troubleshooting Prisma Enum Errors in Tests]]

