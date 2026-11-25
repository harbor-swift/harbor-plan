# Harbor backend

An example Swift backend: people register with an email address and a password, verify the address from a
link the system emails them, sign in for an access token and a refresh token, and read their own user.
Two services own the data — **Auth** (accounts, credentials, verification, sessions, token signing) and
**Users** (users and their display names) — behind one public REST **gateway**. Built to
swift-microservices-architect 1.10.0.

## Services

| Service | Owns | Tables (tenant?) | Workflows | Short-lived state |
|---|---|---|---|---|
| Auth | Accounts (id, email address, password hash, verification), registrations awaiting verification, sessions and refresh tokens; issues every access token | `accounts`, `registrations`, `sessions`, `refresh_tokens` (none tenant: every read is anonymous, by email or by a secret's digest) | `RegistrationWorkflow`: sends the verification email, waits for verification or expiry, then creates the user in Users or removes the unverified account | None |
| Users | Users: the id Auth gave the account, and the display name | `users` (tenant, by `id`) | None | None |

## Contracts

| Proto service | RPC | Audience: anyone, own, a role, administrators, or internal (callers) |
|---|---|---|
| `harbor.auth.v1.AuthService` | `Register` | Anyone (an `idempotency_key` makes retries safe) |
| | `VerifyEmail` | Anyone, proved by the emailed token |
| | `SignIn` | Anyone, proved by the password |
| | `RefreshSession` | Anyone, proved by the refresh token |
| `harbor.users.v1.UserService` | `GetUser` | The caller's own |
| `harbor.users.v1.UserInternalService` | `CreateUser` | Internal: `auth-worker` |

## Module maps

### harbor-auth

| Target | Holds |
|---|---|
| `AuthCore` | `Accounts/` (`Account`, `Registration`, `VerificationEmail`, `EmailAddress`, `DisplayName`, policies, repositories and their errors, use cases), `Sessions/`, `Tokens/` (`OpaqueToken`, which verification and refresh tokens share); `Ports/` (`PasswordHasher`, `VerificationEmailSender`, `UsersClient` and `UsersClientError`); `Workflows/RegistrationWorkflow` |
| `AuthPostgres` | `PostgresAuthScope`, repositories, statements, `Extensions/` (enums and validated values as Postgres types), migrations |
| `AuthGRPC` | `AuthService`, conversions, `GRPCUsersClient` over `UserInternalService` |
| `AuthBcrypt` | `BcryptPasswordHasher` |
| `AuthLog` | `LogVerificationEmailSender`, the fake email provider: it writes each email to the log |
| `Auth` | `auth serve`, `auth worker run`, `auth migrate`; readers for the policies, the fake provider, the signing key and the database role |

- Commands: `serve`, `worker run`, `migrate`. No `build-indexes`: no index is built concurrently yet.
- Data: its own Postgres instance, database `harbor_auth`; `auth_owner` migrates; `auth_service` serves and runs the worker's system use cases (no tenant table, so one role).
- Calls out to: Users' `UserInternalService.CreateUser` (worker), Temporal (serve starts and signals, worker runs).

### harbor-users

| Target | Holds |
|---|---|
| `UsersCore` | `Users/` (`User`, `UserRepository`, `UserRepositoryError`, `CreateUserCommand`, `GetUserUseCase`, `CreateUserUseCase`) |
| `UsersPostgres` | `PostgresUsersScope` (`users_service`), `PostgresUsersInternalScope` (`users_internal`), repository, statements, migrations with row-level security |
| `UsersGRPC` | `UserService`, `UserInternalService`, conversions |
| `Users` | `users serve`, `users migrate`; the internal callers' reader and the database role's reader |

- Commands: `serve`, `migrate`.
- Data: its own Postgres instance, database `harbor_users`; `users_owner` migrates; `users_service` runs the caller's own read under `tenant_isolation`; `users_internal` runs `CreateUser`.
- Calls out to: nothing. It verifies tokens with the public key file.

## Shared packages

- harbor-core:
  - `HarborAuthentication`: `UserIdentity`, `UserRole`, `OperationContext.user`, `Logger.MetadataProvider.user`, `JWTIssuer(privateKey:)` and `JWTAuthenticator(publicKeys:)`.
  - `HarborPersistence`: `PostgresSettings.user(_:)`, `UserSettingsInterceptor`, `PostgresClient.Configuration(config:role:)` (password from a file, TLS `prefer`).
  - `HarborTransport`: the mTLS, certificate reloader and HTTP/2 readers, `ServiceConfig`, `JWTAuthenticator(config:)`, `InternalCallerInterceptor`.
  - `HarborTesting`: `MockDatabase`, `MockAuthenticator`.
- harbor-protos: `AuthProtos`, `UsersProtos`.

## Events

None.

## Gateway routes

| Route | Tier | Upstream RPC |
|---|---|---|
| `POST /v1/account` | No caller | `AuthService.Register` |
| `POST /v1/account/verify-email` | No caller | `AuthService.VerifyEmail` |
| `POST /v1/account/sign-in` | No caller | `AuthService.SignIn` |
| `POST /v1/account/refresh` | No caller | `AuthService.RefreshSession` |
| `GET /v1/users/me` | A caller | `UserService.GetUser` |
| `GET /health` | None (before middleware) | — |

## Decisions

- **Two services, as asked.** Credentials and users change for different reasons (security policy vs. what a person shows), and no request needs both in one transaction. Registration crosses the boundary once, after verification, so it is a workflow, not a call on the request path.
- **The service names are `Auth` and `Users`**, not `Authentication`, which would collide with swift-authentication's `Authentication` and `AuthenticationGRPC` modules.
- **Auth owns the email address; Users owns `User` (id, display name).** Weighed with `system-design` and `architecture`: if Users owned the email, every sign-in would ask Users for the account by address, and duplicate-address checks would span two services; the skill keeps parts together when a split needs a call to the other side on every request. Users' model is `User`, not `Profile`, as the user asked, and keeps no copy of the address. Sources: identity services commonly own the login address and other services key by the immutable user id ([Stack Overflow: should my identity server own the user profiles](https://kiwix.lema.org/stackoverflow.com_en_all_2023-11/questions/57521840/should-my-identity-server-own-the-user-profiles), [WorkOS: email changes](https://workos.com/docs/authkit/email-changes)).
- **`GET /v1/users/me`** reads the caller's own user, the alias Microsoft Graph uses for `/users/{id}` of the signed-in user ([Microsoft Graph: get user](https://learn.microsoft.com/graph/api/user-get)).
- **`CreateUser`'s idempotency key is the user's id**, which Auth's worker names: `INSERT … ON CONFLICT ON CONSTRAINT users_pkey DO UPDATE … WHERE users.display_name = EXCLUDED.display_name RETURNING`, so a repeat returns the user and the same id under another name returns no row (`idempotencyConflict`).
- **Idempotent creates are one statement** ([PostgreSQL: INSERT, ON CONFLICT](https://www.postgresql.org/docs/18/sql-insert.html)): `Register` creates the account with `ON CONFLICT ON CONSTRAINT accounts_idempotency_key_key … WHERE accounts.email = EXCLUDED.email`, then, while the account is still pending, its registration with `ON CONFLICT ON CONSTRAINT registrations_pkey … WHERE registrations.display_name = EXCLUDED.display_name`. A key reused for another address or another name is `idempotencyConflict`. The advisory locks (`lockRegistration`, `lockProfile`) and the lookup before the create are gone.
- **Guarded session writes.** Opening a session inserts only for an active account, and adding a refresh token only to an open session; no row back is `SessionRepositoryError.accountInactive` or `.sessionEnded`, so a session ended by reuse detection mid-refresh gets no successor.
- **Registration is one workflow per account** (`auth-registration-<account id>`). The verification token is a secret, so it never enters the workflow's history: `SendVerificationEmailUseCase` makes it, stores its SHA-256 digest and sends it in one activity. `VerifyEmail` makes its change in its own transaction, then signals the run; a run woken by its timer instead still finds the account verified, so correctness never rests on the signal.
- **The display name waits in `registrations`** until the user exists in Users, then the row is deleted: Auth keeps no user data once Users holds it, and the workflow's payloads carry ids only.
- **Fake email provider.** `AuthLog`'s `LogVerificationEmailSender` writes the recipient and the verification token to the log, at `notice`. This is the one place the system logs an email address and a secret, and it exists for local runs only; a real provider (`AuthResend`, …) replaces it behind the same port.
- **Access tokens are signed with a static Ed25519 key file** (`jwt.privateKeyPath`, Auth only) and verified against the public key file (`jwt.publicKeyPath`, Users and the gateway), each read once at startup. A key's `kid` is its RFC 7638 thumbprint ([RFC 7638](https://www.rfc-editor.org/rfc/rfc7638), [RFC 8037](https://www.rfc-editor.org/rfc/rfc8037.html) for the OKP key type), written in every token's header; a verifier holds its keys as non-default JWKs, so a token naming no `kid`, or an unknown one, is refused. Keys are PKCS#8 and SubjectPublicKeyInfo PEM files ([OpenSSL: Ed25519](https://docs.openssl.org/1.1.1/man7/Ed25519/)). The `ListSigningKeys` RPCs, `SigningKeyRefresher` and the `jwt-signingKey` secret are gone.
- **swift-authentication-jwt 1.1.0 adds `JWTIssuer(keys:kid:)`.** 1.0.0's `JWTIssuer` signed with no `kid`, so the skill's issuer couldn't name its key; the user chose to fix the library first rather than depart. ([RFC 8725](https://rfc-editor.org/rfc/rfc8725.html), JWT best current practices.)
- **`JWTAuthenticator(publicKeys:)` is `async throws`**, where the skill shows `async`: adding a key as a non-default JWK, which is what refuses a token naming no `kid`, is jwt-kit's throwing `add(jwk:isDefault:)`.
- **No `/.well-known/jwks.json`**: nothing outside Harbor verifies its tokens (the user's answer).
- **Internal callers by DNS name.** `InternalCallerInterceptor(callers:logger:)` runs first on every service of every server and admits a listed RPC's caller by a DNS subject alternative name, case-insensitively; the common name is never read ([RFC 9525](https://www.rfc-editor.org/rfc/rfc9525.html), which obsoletes RFC 6125). Each executable reads its map under `grpc.server.internalCallers.<rpc>`: Users lists `createUser: [auth-worker]`; Auth serves no internal RPC, so its map is empty.
- **Health.** Every serve process registers `grpc.health.v1` and marks itself serving as it builds the server ([gRPC health checking](https://grpc.io/docs/guides/health-checking/)); the gateway answers `GET /health` with `200`. Nothing waits on a key fetch.
- **Postgres per service.** Auth and Users each have their own Postgres 18 instance ([microservices.io: database per service](https://microservices.io/patterns/data/database-per-service), [AWS: database-per-service](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/database-per-service.html)).
- **Postgres logins: passwords, TLS `prefer`** (the user's answer, postgres-nio's default mode). Each role's password is a file (`postgres.<role>.passwordPath`); `migrate` sets each service role's password from its file after the migrations run. The client verifies the server's certificate against the environment's authority whenever TLS is offered; the server's `pg_hba.conf` takes `hostssl … scram-sha-256` only, so the server, not the client's `prefer`, keeps connections encrypted ([libpq: sslmode](https://www.postgresql.org/docs/current/libpq-ssl.html), [PostgreSQL: pg_hba.conf](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)).
- **Temporal: self-hosted, mutual TLS everywhere, local runs included** (the user's answer). Each process presents a pair under `temporal.tls`, reloaded by its own `TimedCertificateReloader`; the frontend requires client certificates from the environment's authority ([Temporal: self-hosted security](https://docs.temporal.io/self-hosted-guide/security), [Temporal: service configuration](https://docs.temporal.io/temporal-service/configuration)). The CLI's dev server terminates no TLS, so harbor-dev runs `temporalio/auto-setup` with its own Postgres store instead. The plaintext toggle (`TemporalTransport`) is gone.
- **Composition.** Each command builds its `ConfigReader` (environment variables, then defaults) and all its wiring inline; readers take no `Logger`, and the roots set the reloader's `logger` and `onCertificateLoadFailed`. Secrets are file paths, read by the reader that needs them; `DirectoryFilesProvider` and `SECRETS_DIRECTORY` are gone.
- **Repository errors.** Every Core repository has its error beside it (`AccountRepositoryError`, `RegistrationRepositoryError`, `SessionRepositoryError`, `UserRepositoryError`); a stored value a validated type refuses fails as a decoding error, and `PostgresRepositoryError` is gone.
- **Ports.** `Ports/` holds protocols and their errors only: `NewProfile` is gone (`UsersClient.createUser(id:displayName:)`), and `VerificationEmailSender.send(to:displayName:token:expirationDate:idempotencyKey:)` takes values Core already has; `VerificationEmail` is the send use case's result, in `Accounts/`.
- **Typed throws.** Handlers and activities write a plain `do`, which Swift 6.4 infers as throwing the use case's error.
- **No attempt limits (departure).** Per-account sign-in limits need Valkey; this example leaves Valkey out. A production deploy adds `SignInFailureRepository` before going live.
- **Refresh tokens rotate with reuse detection**: a used refresh token presented again ends its whole session.
- **The verification email retries until it lands.** A person waits on it, and a run that gave up would leave the account pending with nothing left to expire it, so `RegistrationWorkflow` sends under the `delivery` options, with no attempt limit, and a provider that stays down shows on the failing-activity alert. Each attempt stores a new token, so a retry after an attempt that did deliver sends a second email whose link replaces the first: the secret stays out of the workflow's history.
- **No `health` subcommand (departure).** grpc-swift-extras keeps its health client `package`-scoped, so a probe command would need the health proto in harbor-protos. Left out of this example.
- **Warnings in swift-temporal-sdk's `@Workflow` (departure).** swift-temporal-sdk 1.5.0, the latest release, expands `@Workflow` through `@_WorkflowState` to `nonisolated(unsafe)` state not marked `unsafe`, which warns "expression uses unsafe constructs but is not marked with 'unsafe'" (`StrictMemorySafety`) three times in the expansion for `RegistrationWorkflow.isEmailVerified`, so harbor-auth's warnings-as-errors build (`scripts/swift_ci.sh`) fails there and nowhere else. Warnings and warnings-as-errors stay on in `Package.swift`; this leaves in the change that raises swift-temporal-sdk to a release without the warning.
- **No CI/CD or images yet (departure).** No package has CI workflows, a `.license_header_template`, `.licenseignore`, `dependabot.yml`, or, for a service, a `Containerfile`, though Build order's first four lines are ticked: the platform, the image registry and the CI host are unanswered (see Questions), and the CI workflows' deploy jobs and image settings depend on them. Until then each change is held by hand to `scripts/swift_ci.sh build` and `test` (`--library` for harbor-core and harbor-protos), `scripts/check_boundaries.sh`, `scripts/check_style.sh` and `swift-format lint --strict`. Once they're answered, each package runs Set up CI/CD, with `linux_host_archs: '["aarch64"]'`.
- **No `XPostgresTests` yet (departure).** The repositories and statements are untested against a real Postgres; the end-to-end demo covers them.
- **The email address is unique and shared.** Registration answers `409` for a registered address, which reveals it is registered, as the skill accepts for one-step sign-up.

## Operations

Production environments are out of scope for this example; `harbor-dev/` runs everything on one machine. These are the answers it runs on:

- Each service's Postgres instance: Postgres 18; roles log in with passwords mounted as files; TLS `prefer` on the client, `hostssl … scram-sha-256` on the server; default `max_connections` (100), each role's pool 10. Only its own service's database is on each instance, so each sizes against its own: Auth's `auth_service` pools in serve and the worker, one replica each, total 20; Users' `users_service` and `users_internal` pools in serve, one replica, total 20; each leaves room for `migrate`'s owner pool and administration.
- Stop timeout: the worker's `temporal.worker.gracefulShutdownPeriodSeconds` is 30, so whatever stops `auth worker run` waits longer than 30 seconds.
- How each process connects to Temporal: self-hosted frontend over mutual TLS, a pair per process (`auth`, `auth-worker`) signed by the environment's authority; namespace `harbor-auth`.
- Each environment's certificate authority: a development authority per machine, made by `harbor-dev/certs.sh`.
- How a developer runs Postgres, Valkey and Temporal: Apple `container` — one Postgres per service, `temporalio/auto-setup` 1.29.7 over its own Postgres store, Temporal's UI; no Valkey. The Swift processes run natively.
- Backups, point-in-time recovery, retention and Temporal's authorizer: unanswered (see Questions).
- The platform each environment deploys to, its servers' CPU architectures, the image registry and the CI host: a developer's machine runs harbor-dev on macOS (arm64); servers run `aarch64` (the user's answer); the platform, the image registry and the CI host are unanswered, with no default (see Questions).

Runbooks:

1. **A signing key's rotation**, in this order, so no verifier meets a token it can't check:
   1. Add the new public key to every verifier's file (`jwt.publicKeyPath`: Users and the gateway) beside the old one, and deploy every process that verifies tokens.
   2. Point Auth's `jwt.privateKeyPath` at the new private key, and deploy Auth: its tokens now name the new `kid`.
   3. Once one access-token lifetime (`sessions.accessTokenLifetimeSeconds`, 15 minutes) has passed, remove the old public key from every file, and deploy again.
2. **Certificate renewal**: mount the renewed pair at the same paths; each process's reloader picks it up within `tls.refreshIntervalSeconds`, with no deploy. A new trust root joins every bundle beside the old one, with a restart of each process, before any pair it signs is mounted.
3. **A Postgres password's rotation**: change the role's password file, run `migrate` (which sets it), then deploy serve and the worker.

## Build order

1. [x] harbor-core and harbor-protos: Start a project
2. [x] harbor-auth (the token issuer): Add a service
3. [x] harbor-users: Add a service
4. [x] harbor-gateway: Add the gateway
5. [ ] Production: Prepare for production

## Questions

- Postgres: how roles log in, the TLS mode, the rest of the setup. **Answered**: passwords from files, TLS `prefer`; one instance per service.
- How Harbor connects to Temporal, and how local runs get it. **Answered**: self-hosted with mutual TLS, local runs included.
- Does anything outside Harbor verify its tokens? **Answered**: no, so no JWK Set route.
- Who owns the user record? **Answered**: Auth keeps the account and its email address; Users owns `User` (id, display name).
- Users' migrations, with nothing deployed. **Answered**: replaced in place (`CreateUsersTable`, `CreateUsersRLSPolicy`); local databases are recreated.
- swift-authentication-jwt couldn't name a `kid`. **Answered**: fix the library first (1.1.0).
- Email provider for production: unanswered; `AuthLog` stands in.
- Backups and point-in-time recovery of each Postgres and Temporal's store, the namespace's retention, and whether Temporal's authorizer confines each identity to its namespace: unanswered.
- Production's certificate authority, objectives and alerts: unanswered (Prepare for production).
- The servers' CPU architecture. **Answered**: `aarch64`.
- The platform each environment deploys to, the image registry and the CI host: unanswered, with no default. Until they're answered, no package has CI/CD or an image (see Decisions).
