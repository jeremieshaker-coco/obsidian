---
tags:
---
# Unit Testing NestJS Services

Best practices and common pitfalls discovered while writing unit tests for NestJS services in the `delivery-platform`.

## Dependency Mocking
All external dependencies of a service must be mocked using the `Test.createTestingModule`.

- **Incomplete Mocks**: A frequent source of errors is an incomplete mock. Ensure that all methods on a mocked service that are called by the service-under-test have mock implementations (e.g., using `jest.fn()`).
- **Prisma Enums**: Prisma enums are not automatically available in the Jest test environment if the client hasn't been generated correctly. While manual mocking is possible, it is brittle. The preferred approach is to ensure the client is generated via `yarn workspace <service> run prisma:generate`. See [[Prisma Client Generation in Monorepo]].

  If manual mocking is absolutely required (e.g. for speed in isolated tests), it can be done like this:
  ```typescript
  jest.mock('@prisma/client', () => {
    const originalModule = jest.requireActual('@prisma/client');
    return {
      ...originalModule,
      LidEvent: {
        LID_OPENED: 'LID_OPENED',
        LID_CLOSED: 'LID_CLOSED',
      },
    };
  });
  ```

## Testing Decorated Methods
Decorators (e.g., from `nestjs-cls`) can interfere with test execution, as they wrap the original method with additional logic that may not be initialized in a test environment.

A reliable pattern to handle this is to make the decorated method `public` (or keep its core logic in a separate public method). This allows the test to call the method directly, bypassing the decorator and focusing solely on the method's internal logic.

## Table-Driven Tests
Follow the table-driven test pattern to keep tests clean, DRY, and easy to extend. Refer to the `delivery-platform/unit-testing` Cursor Rule for an example.
