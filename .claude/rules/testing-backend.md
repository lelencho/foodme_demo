---
paths:
  - "apps/backend/src/test/**"
  - "apps/backend/src/main/resources/application-test.properties"
---

# Backend testing

Run from `apps/backend`:
- `./gradlew test`: runs all tests (JUnit 5).
- `./gradlew test --tests am.foodme.backend.OrderControllerTest`: runs one class.
- `./gradlew test --tests "am.foodme.backend.OrderControllerTest.methodName"`: runs one method.

## How the tests work
- They are integration tests: `@SpringBootTest` + `@AutoConfigureMockMvc` + `@ActiveProfiles("test")`, exercising the real controllers, services and security through `MockMvc`. They don't mock the service layer. There is one test class per controller in `src/test/java/am/foodme/backend/`.
- The `test` profile (`src/main/resources/application-test.properties`):
  - Uses H2 in memory with `MODE=PostgreSQL` and `ddl-auto=create-drop`, creating the `foodme` schema.
  - **Disables Flyway.** Fixture data comes from `src/test/resources/data.sql` (`spring.sql.init.mode=always`). When an entity changes, update `data.sql` too.
  - Turns off image seeding, simulated latency (0 ms), HTTP logging, Sentry and the flaky heartbeat job.
  - Uses its own JWT secret and internal API key.
- Authenticated tests get a token through the real API: they call `POST /api/auth/register` with a unique email (UUID or counter), then send `Authorization: Bearer <token>`. `spring-security-test` is also available.
- The Spring context and H2 database are shared across test classes, so make test data unique rather than relying on a clean database. Some classes use `@TestMethodOrder(OrderAnnotation)`.
- Because H2 isn't Postgres, Postgres-only SQL or migrations won't be caught here.
