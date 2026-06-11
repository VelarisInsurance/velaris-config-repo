# velaris-config-repo

Centralised configuration backend for the Velaris Spring Cloud Config server
(`velaris-config-server`). Every Spring Boot service in the platform pulls its
runtime configuration from this repo on startup and on refresh.

## Layout

```
velaris-config-repo/
  application.yml                 # shared defaults applied to ALL services
  <service-name>/
    application.yml               # base config for that service (all envs)
    application-dev.yml           # dev-only overrides   (optional)
    application-staging.yml       # staging-only overrides (optional)
    application-prod.yml          # prod-only overrides  (optional)
```

Current services tracked here:

- `claims-api`
- `commercial-api`
- `notification`
- `onboarding-api`
- `orders-api`
- `partner-api`
- `policy-api`
- `properties-api`
- `refundable-api`
- `reservations-api`
- `velaris-flags`
- `velaris-organizations`

**Intentionally NOT tracked here:** the SCDF one-shot tasks
`velaris-connector-batch` and `velaris-connector-launcher`. They are
configured entirely via environment variables passed at SCDF task launch
(plus their bundled `application-aws.yml`) and never contact the config
server — a config-server fetch would add startup latency and an
availability dependency to every batch run. Do not add folders for them;
the copies would silently diverge from the values the tasks actually use.

The folder name MUST match the client's *effective* `spring.application.name`.
Note: no Velaris service has `spring-cloud-starter-bootstrap` on its
classpath, so `bootstrap.yml` files are ignored — the name that counts is the
one set in `application.yml` / `application-{profile}.yml` (most services set
it in `application-aws.yml`, next to the `spring.config.import` line).

## Branching strategy: folder overlay, not branch-per-environment

We use a **single `main` branch** with environment-specific YAML files
(`application-{env}.yml`) rather than a branch per environment.

**Why folder overlay wins here**

- Promotion is a PR diff against `main`, not a cross-branch merge — easier to
  review, easier to revert.
- One source of truth. No drift between `dev`/`staging`/`prod` branches.
- Spring profile activation (`SPRING_PROFILES_ACTIVE=dev,prod,...`) does the
  overlay at runtime, so the mechanism is already battle-tested.
- Branch-per-env shines when you need *per-env access control* enforced by Git
  hosting. We get that via GitHub branch protection + CODEOWNERS on `main`
  instead.

**When we'd revisit**

If we ever need an environment to pin to an older config snapshot
independently of `main` (e.g., a long-lived QA branch frozen at a release
boundary), Spring Cloud Config's `label` parameter still lets us point that
client at a specific Git ref without changing the layout above.

## Adding configuration for a new service

1. Create a directory matching the client's `spring.application.name`:
   ```
   mkdir <service-name>
   ```
2. Add base config that applies in every environment:
   ```
   <service-name>/application.yml
   ```
3. Add per-environment overrides only for keys that actually differ:
   ```
   <service-name>/application-dev.yml
   <service-name>/application-prod.yml
   ```
4. On the client side, the service's `spring.config.import` should already be:
   ```yaml
   spring:
     config:
       import: "optional:configserver:${CONFIG_SERVER_URL}"
   ```
5. Open a PR. Once merged to `main`, trigger a refresh (see below).

## Per-environment overrides

Spring's profile mechanism merges files in this order (later wins):

1. `application.yml` (root — global defaults across all services)
2. `<service>/application.yml` (service base)
3. `<service>/application-{profile}.yml` (env override)

So to change the Mongo URI just for prod:

```yaml
# velaris-organizations/application-prod.yml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI_PROD}
```

Keep secrets out of this repo. Reference them via `${ENV_VAR}` placeholders
and inject the actual values from the deployment environment (ECS task
definition, GitHub Actions secret, etc.).

## Triggering a config refresh after a change

After your change is merged to `main`:

```bash
# Refresh every client subscribed to the Spring Cloud Bus (Kafka).
curl -u $CONFIG_CLIENT_USERNAME:$CONFIG_CLIENT_PASSWORD \
     -X POST https://config.velarisinsurance.com/actuator/busrefresh
```

This publishes a `RefreshRemoteApplicationEvent` to the `springCloudBus`
Kafka topic. Every client with `spring-cloud-starter-bus-kafka` on the
classpath receives it and re-evaluates `@RefreshScope` beans without a
restart.

To refresh a single instance only (rarely useful):

```bash
curl -X POST https://<service-host>/actuator/refresh
```

If a property is read into a non-`@RefreshScope` bean (e.g., a static field or
a connection pool created at startup), the bean will keep the old value until
the pod is restarted. When in doubt, restart.

## Audit log

**The git history of this repo IS the audit log.** Every change is a commit
with author, timestamp, and diff. Use `git log -p <path>` to review the
history of a specific file.

For prod-impacting changes:

- Open a PR — never push directly to `main`.
- At least one reviewer from the owning service team.
- Include the reason for the change in the commit message body, not just the
  subject. Future-you (and the on-call engineer at 3am) will thank you.
- Tag the PR with the affected service and environment for quick searching.

GitHub branch protection on `main` enforces the PR-only rule.
