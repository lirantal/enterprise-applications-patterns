# Backend Testing Strategy

Responsibilities of an N-tier architecture:

- Controllers:
  _ Serialization & Request Validations
  _ DTOs transformations \* HTTP Responses
- Services: \* Business logic
- Repositories: \* Data access layer, whether ORM, Data Mapper or otherwise.

## Controllers

**How to test?**

- Unit tests
- Integration tests

Guidelines are that the scope of unit tests of controllers is all code branching, negative testing, edge cases, validations and error exceptions being thrown. Integration tests will be focused on the actual endpoint setup properly, request middleware (in terms of Express middleware chains), HTTP transport (headers and return codes) and primarily that the business logic on the endpoint works as expected (returning the correct data).

### Unit Tests

**What to check?**

- Check proper transformations have been made and calls to the proper service are properly made
- Check proper validations are made
- Check proper responses are being sent back from the controller based on the request (i.e.: the relevant status codes are sent, or the relevant error codes based on user input)

**What to mock?**

- Mock service layers
- Mock the actual request/response web application tier as this isn’t necessary to unit test

### Integration Tests

- Use a test utility such as `supertest` to mock the express application middleware.
- These tests should generally depend on data, for that end, each test case should be responsible to setup its own data and cleanup.
  _ In cases where CRUD APIs exist in order to facilitate the setup of data (create and deleted) - make use of them.
  _ In cases where CRUD APIs do not exist, revert to using test utilities that depend on the repository layer to create and delete data. \* If all else fails, instrument the application using specially crafted tools to inject and remove the data as required.
- What are we testing?
  _ Application API routes - ensuring that the routes are setup correctly.
  _ Application middleware - such as the case with Express, we will be testing that middleware in the chain is setup correctly. \* If a middleware in the chain has dependencies on external data sources, for example an up-to-date bearer token for authentication to allow a request then this middleware should be mocked.

**What to check?**

- Check generic routes and the possible actions on them: \* Make requests to GET, POST, PUT, etc routes to confirm these routes are behaving properly (they will test all underlying logic)
- Check API transport configuration such as different HTTP header properties if these have been employed:
  _ If the service is using API versioning that maps to different controllers, be sure to make those requests.
  _ Assert on expected HTTP headers (for example: content-type, or cache control if expected)
- Do not check permutations of validations, input params and exhausting all possible error conditions - these have already been tested through unit testing the controller.

## Services

**How to test?**

- Unit tests

### Unit Tests

**What to check?**

- Assert business logic
- Aim for high code coverage, covering both successful and failure operations.

**What to mock?**

- Any dependencies of the service under test should be mocked (i.e: other services or repositories being accessed)

## Repositories

**How to test?**

- Integration tests \* Use an actual instance of the persistency layer used. If it’s MySQL or otherwise, they can be easily spawned as ephemeral instances using containers.
- Tests must be independently executed and as such they should be responsible for setting up their data as required for the test and clean it up when the test is finished.
- Tests should aim for handling isolated and individual items over mutating a global data set. For example, if a test needs to test getting all resources or deleting all resources then while this test is running, other tests that run in parallel may mutate the data that it is dependent on. To mitigate the problem, aim to create resources of specific type during the test, and then bulk delete all of those specifically to test a “deleting all resources” functionality.

### Integration Tests

**What to check?**

- Assert repository access layer performs as expected:
  _ Bulk creation or updates
  _ Special fetches based on filters

**What to mock?**

- At this lower-level there are usually less mocking being done.
- If a repository access layer is fetching data from a remote service (i.e: S3) then that service should be mocked.

## Avoid the following bad practices:

- Preparing global state for tests - each test should be responsible for its own setup and cleanup of data for the test. \* In extreme cases where there is no way to create the data then a database seeding can take place while the environment is being set up.
- Testing validations in integration or end-to-end tests - these can be easily covered in unit tests, and there’s no need to cover them again in higher stages of the pyramid.
- Asserting for list of items returned - if a test is returning a list of data, assert the expected data being returned (the actual payload or format of the data) and not only the list count itself.
- Asserting only for returned values of a method for integration tests - in unit tests you will be asserting for returned value of a method. In integration tests however it is important to check through methods that rely on the persistency layer that an item was indeed mutated. For example: when testing a `createResource()` method, use another method for the data that is expected which queries from the persistency later instead of asserting on the returned value of the `createResource()` method.
- Asserting too many expectations in tests - setting too many assertions in a test may prove hard to debug and isolate the culprit. Prefer more test cases over big test cases with many assertions.
