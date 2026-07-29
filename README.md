# velaris-config-repo

Centralised configuration backend for the Velaris Spring Cloud Config server
(`velaris-config-server`). Every Spring Boot service in the platform pulls its
runtime configuration from this repo on startup and on refresh.

## Layout

```
velaris-config-repo/
  application.yml                 # shared defaults applied to ALL services
  <service-name>/
    application.yml               # base config for that service (ALL envs)
```

> **There are no per-environment files here, and adding one would do nothing.**
> Every deployed service runs `SPRING_PROFILES_ACTIVE=aws` in *every* cluster —
> dev, staging and prod alike (see `SPRING_PROFILES_ACTIVE` in each
> `k8s/base/*/deployment.yml`; no overlay overrides it). Spring Cloud Config
> only serves `application-{profile}.yml` for profiles the client actually
> requests, so `application-dev.yml` / `application-prod.yml` would never be
> read. See [Per-environment values](#per-environment-values) for the mechanism
> that does work.

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

## Branching strategy: single branch, no per-environment config here

We use a **single `main` branch**, and this repo holds only values that are
identical in every environment. Everything that differs per environment lives
in `velaris-infrastructure` (see below), not here.

- Promotion is a PR diff against `main`, not a cross-branch merge — easier to
  review, easier to revert.
- One source of truth. No drift between `dev`/`staging`/`prod` branches.
- Branch-per-env shines when you need *per-env access control* enforced by Git
  hosting. We get that via GitHub branch protection + CODEOWNERS on `main`
  instead.

If we ever need an environment to pin to an older config snapshot
independently of `main` (e.g., a long-lived QA branch frozen at a release
boundary), Spring Cloud Config's `label` parameter lets us point that client
at a specific Git ref without changing the layout above.

## Per-environment values

Because every environment runs the same `aws` profile, a per-env value can only
come from one of two places, both outside this repo:

1. **An env var set by the k8s overlay** —
   `velaris-infrastructure/k8s/overlays/{dev,staging,prod}/<service>/kustomization.yml`.
   The convention is a placeholder in `k8s/base/<service>/deployment.yml` that
   each overlay replaces, e.g. `velaris-claims-files-SECRET_ENV`.
2. **A key in that environment's Secrets Manager secret** —
   `velaris/{env}/<service>/config`. Each service's `main()` reads every key in
   that JSON and sets it as a **JVM system property** before Spring starts, so
   `${SOME_KEY}` placeholders resolve from it.

### The rule that matters when reviewing this repo

> **Every `${VAR:default}` in this repo IS the production value** unless that
> environment's k8s overlay or Secrets Manager secret sets `VAR`. There is no
> profile that quietly swaps it out.

So a dev-flavoured default here — a `-dev` bucket name, a `localhost` URL, a
`local-dev-*` shared secret, a feature toggle that should only be on in dev —
is a production defect, not a placeholder. Two live examples that this repo
caused: prod `claims-api` resolved `S3_CLAIMS_BUCKET` to `velaris-claims-files-dev`
(a bucket in the *dev* AWS account), and all four internal service-to-service
tokens resolved to the literal `local-dev-internal-token` on internet-facing
endpoints (VEL-615 / VEL-616).

**When you add a `${VAR:default}` here, either** make the default safe for prod
and let dev opt in via its overlay, **or** give it no default at all so a
missing value fails startup loudly instead of silently doing the dev thing.

## Property precedence (highest wins)

Learned the hard way during the flags-api h2c incident (VEL-598), where a
service-local override of a fleet-wide default here silently lost for weeks:

1. JVM system properties — i.e. **Secrets Manager** keys, loaded in `main()`
2. Container env vars — set by the k8s base/overlay
3. **This repo, service file** — `<service>/application.yml`
4. **This repo, shared file** — root `application.yml`
5. The service's own `src/main/resources/application-aws.yml`
6. The service's own `src/main/resources/application.yml`

**Config-server values (3 and 4) outrank the service's own YAML (5 and 6).** A
service cannot override a default set here by editing its own
`application-aws.yml`; the only working seam is the service's folder in this
repo.

The corollary bites in the other direction too: the `spring.config.import` is
`optional:` with `fail-fast: false`, so if config-server is unreachable at boot
the service starts anyway on layers 5-6. Those local files are the silent
fallback for a config-server outage, so they need prod-safe values as well —
keep them in sync with what this repo sets.

## Adding configuration for a new service

1. Create a directory matching the client's `spring.application.name`:
   ```
   mkdir <service-name>
   ```
2. Add base config that applies in every environment:
   ```
   <service-name>/application.yml
   ```
3. For anything that differs per environment, do **not** add a file here — wire
   an env var in `velaris-infrastructure` or a key in
   `velaris/{env}/<service>/config`. See [Per-environment values](#per-environment-values).
4. On the client side, the service's `spring.config.import` should already be:
   ```yaml
   spring:
     config:
       import: "optional:configserver:${CONFIG_SERVER_URL}"
   ```
5. Open a PR. Once merged to `main`, trigger a refresh (see below).

## Secrets

Keep secrets out of this repo. Reference them via `${ENV_VAR}` placeholders and
put the real values in `velaris/{env}/<service>/config` in Secrets Manager,
which the service loads as system properties at startup.

A placeholder with an empty default (`${SOME_API_KEY:}`) means the feature is
inert until the key is set — that is a deliberate, safe pattern. A placeholder
with a *usable* dummy default (`local-dev-internal-token`,
`dev-only-...-change-in-prod`) is not: it silently makes the dummy the
production credential. Use an empty default and let the code fail closed.

## Triggering a config refresh after a change

**A merge to `main` does not reach running pods. Restart them.**

The Spring Cloud Bus is disabled fleet-wide — the shared `application.yml` here
sets `spring.cloud.bus.enabled: false`, because services carrying
`spring-cloud-starter-bus-kafka` default the broker to `localhost:9092` and the
AdminClient reconnect-spams past the liveness deadline until MSK SASL/IAM is
wired up. So `POST /actuator/busrefresh` fans out to nobody, and clients pick up
a change only on their next boot:

```bash
kubectl -n velaris rollout restart deploy/<service>
```

This bit real work once: a config-repo fix for the flags-api h2c leak (VEL-598)
was merged and assumed live for days before anyone restarted the pods.

Even with the bus enabled, a property read into a non-`@RefreshScope` bean (a
static field, a connection pool, a constructor-injected `@Value`) keeps its old
value until restart. When in doubt, restart.

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
