## Quick orientation for AI coding agents

This repo is a small Dockerized Spring Boot demo with a Postgres init image and an Apache HTTPD proxy. Keep guidance short and specific to discoverable patterns, commands, and files so a developer (or another agent) can be productive immediately.

### Big-picture architecture
- Orchestration: `docker-compose.yml` defines three services: `initdb` (Postgres), `backend` (Spring Boot app), and `httpd` (Apache reverse proxy). Services sit on the `app-network` and the `backend` is exposed to `httpd` at `http://backend:8080/`.
- Database bootstrap: SQL schema + seed live in `initdb/01-CreateSchema.sql` and `initdb/02-InsertData.sql`; `initdb/Dockerfile` copies them into `/docker-entrypoint-initdb.d/` so Postgres initializes on first run.
- Backend: Java 21, Spring Boot 3.4.x app under `backend/`. Maven is the build tool (`backend/pom.xml`). The Dockerfile is multi-stage: it installs Maven in an Alpine image, runs `mvn clean package -DskipTests`, then copies the jar into a runtime image.

### Key files and where to look
- Service wiring: `docker-compose.yml` (env vars, service names used as hostnames)
- Backend build: `backend/pom.xml`, `backend/Dockerfile`, `backend/src/` (Java source)
- DB init: `initdb/*sql`, `initdb/Dockerfile`
- Proxy: `httpd/httpd.conf`, `httpd/Dockerfile` (reverse-proxy to `backend:8080`)
- Ansible inventory placeholder: `Ansible/inventaires/setup.yml` (contains `ansible_user` and SSH key path variables)
- Test artifacts & reports: `backend/target/surefire-reports`, `backend/target/failsafe-reports`, `backend/target/site/jacoco`

### Concrete developer workflows (commands & examples)
- Build the backend (with tests):
  - `mvn -f backend/pom.xml clean package`
  - This produces `backend/target/*.jar` and test reports under `target/`.
- Run unit/integration tests locally:
  - Unit tests: `mvn -f backend test`
  - Integration (failsafe) / full lifecycle: `mvn -f backend verify` (note: the project includes the failsafe plugin entry — check `pom.xml` for custom config if added later)
  - Many tests use Testcontainers (see `pom.xml`). Running these requires a running Docker daemon on the machine where tests run.
- Run backend in dev mode:
  - `mvn -f backend spring-boot:run` or run from your IDE (Java 21). Environment variables in `docker-compose.yml` show canonical property names, e.g. `SPRING_DATASOURCE_URL`.
- Build & run the full stack with Docker Compose:
  - `docker-compose up --build` (from repo root). This will build `initdb`, `backend`, and `httpd` and start the services on the `app-network`.

### Project-specific conventions & gotchas
- Java & runtime: Uses Java 21 (`pom.xml` and Dockerfiles use Eclipse Temurin 21). Ensure local toolchains match when building/running outside Docker.
- Dockerized build: `backend/Dockerfile` runs `mvn clean package -DskipTests`. CI or local builds that want tests should run Maven directly before building the image.
- Service hostnames: Compose uses service names as DNS entries (e.g., the app connects to the DB using `jdbc:postgresql://initdb:5432/db`). When changing service names, update `docker-compose.yml` and any environment variables.
- Tests + Docker: Because tests include Testcontainers dependencies, a Docker daemon is necessary for many integration tests — CI runners must allow Docker or provide alternatives.
- SQL seeds: The canonical seed and schema live in `initdb/`; do not duplicate inserts in `backend/test/resources` without confirming intent.

### Integration points & external dependencies
- Postgres image and SQL scripts: `initdb/Dockerfile` relies on the official Postgres image init mechanism.
- HTTP reverse proxy: `httpd/httpd.conf` contains a ProxyPass to `http://backend:8080/` — changing backend port requires updating this file and `docker-compose.yml` ports.
- Maven artifacts: The project uses Spring Boot parent POM and a few test dependencies (Testcontainers). Network access to Maven Central may be needed during builds (unless dependencies are already cached in CI).

### Useful patterns and examples found in the repo
- Multi-stage Docker build for Java (see `backend/Dockerfile`): dependency go-offline, copy `src`, `mvn clean package -DskipTests`, then copy jar into runtime image.
- DB init via `/docker-entrypoint-initdb.d/` (see `initdb/Dockerfile`).
- Proxy mapping in `httpd/httpd.conf` that forwards all requests to `backend:8080` (explicit example you can reuse).

### What to ask the maintainers if unclear
- Should integration tests run in CI using Testcontainers, or is there an alternative (embedded DB or a hosted DB)?
- Are there intended Maven profiles for dev / prod (none are present in the POM today)?
- Confirmation of the production-grade HTTPD proxy configuration (currently a simple reverse proxy in `httpd.conf`).

If any section is unclear or you'd like more detail (example: expand the troubleshooting section for Testcontainers on Windows, or add a CI job template), tell me which area to expand and I will iterate.
