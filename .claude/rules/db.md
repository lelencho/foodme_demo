---
paths:
  - "apps/backend/src/main/resources/db/**"
  - "apps/backend/src/main/java/am/foodme/backend/model/**"
  - "apps/backend/src/main/java/am/foodme/backend/repository/**"
  - "apps/backend/src/main/resources/application*.properties"
---

# Database (Postgres + JPA + Flyway)

- Postgres, with all tables in schema `foodme` (the Hibernate default schema and the Flyway schema).
- Migrations live in `apps/backend/src/main/resources/db/migration` (`V1__init.sql`, `V2__images_in_db.sql`, `V3__more_menu_items.sql`, …).
  - Add a new `V<n+1>__<desc>.sql`. Never edit a migration that has already been applied.
  - Hibernate runs with `ddl-auto=validate`, so every entity or column change needs a matching migration, or the app fails at startup.
- Timestamps are stored in UTC (`hibernate.jdbc.time_zone=UTC`). `open-in-view` is off, so lazy relations must be loaded inside the service or transaction.
- Connection settings, in priority order:
  1. `SPRING_DATASOURCE_URL` (JDBC).
  2. `DATABASE_URL` (`postgresql://…`, as Render/Neon provide it). `config/DatabaseUrlEnvironmentPostProcessor` converts it to JDBC; it's registered in `META-INF/spring.factories`.
  3. `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER` / `DB_PASSWORD`. Defaults: `localhost:5432/foodme`, user and password `foodme`.
- Flyway retries the connection 10 times at 10 s intervals, because cloud databases can be slow to wake.
- Images are stored in the database (`foodme.image`). `service/ImageSeedRunner` seeds them on first start from `resources/img-seed/chefs/<chefId>/...` when `FOODME_IMAGES_SEED_ENABLED=true` (the default), and `/api/images/**` serves them.
- Dish search uses a custom repository fragment (`DishSearchRepository` / `DishSearchRepositoryImpl`).
- Tests do **not** run migrations: they use H2 with `create-drop` and `src/test/resources/data.sql` (see `testing-backend.md`). Keep SQL in migrations Postgres-specific only where necessary, and update `data.sql` when the schema changes.
