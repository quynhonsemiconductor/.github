# Local development

This page is the org-wide map: which fixed host ports each product's **dev** stack uses, how an **e2e** stack is started without
fixed ports, and what each product can do today. A product's README has the commands. Items marked **pending** are not on `main` yet;
the PR is named.

## Dev stacks: fixed ports

Each product owns a block, so the products run side by side. Every host port is bound to `127.0.0.1`.

| Product | API | Web (Vite) | Postgres | Valkey / Redis | Object storage | Mail (SMTP / UI) | OTel (OTLP / metrics) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| rova | 3010 | 5175 | 5420 | 6320 | LocalStack 4620 (Garage: pending) | Mailpit 1120 / 8120 (pending #742) | 4420 / 8420 |
| opshub | 3001 | 5174 | 5410 | 6310 | LocalStack 4610 (Garage: pending) | Mailpit 1110 / 8110 (pending #431) | 4410 / 8410 |
| knowledge base (qnsc-kb-backend) | 8000 | 5173 | 5432 | 6379 | Garage 4570 (pending #239) | none: the KB sends no mail | none |

- LocalStack's community edition does **not** keep its data across a restart. Only the KB's Garage node (once #239 lands) keeps
  uploads. Do not expect a local upload to survive a LocalStack restart.
- The KB keeps the default ports. Its frontend (qnsc-kb-frontend) uses the same 5173 as its dev server.
- A web port is strict: Vite refuses to start on a taken port, because the sign-in redirect and the bucket CORS are set for that origin.
- Mail in dev: rova and opshub send through Mailpit once their smtp PRs land (`EMAIL_PROVIDER=smtp`); nothing leaves the machine.

## E2E stacks: no fixed ports

The e2e suite truncates the database it points at, and several people and agents test at once. So an e2e stack:

1. has its **own compose project name**, unique per run;
2. publishes **ephemeral loopback ports** (`127.0.0.1::<container-port>`), never the table above;
3. is found by the harness with `docker compose -p <project> port <service> <container-port>`, or through the URLs it writes to
   its env file.

Nothing in the e2e path may assume a port from the table. Stop and remove your own stack when done
(`docker compose -p <project> down`, add `-v` only for a disposable stack). If a stack is not yours, ask its owner first.

## Commands and status, per product

| Product | Dev stack | E2E stack | Checks that exist today |
| --- | --- | --- | --- |
| rova | `make local` · `make check` · `make down` (pending #742) | `scripts/e2e/run.sh`: own project, ephemeral ports (pending #742) | `make check` (pending #742): readiness, worker heartbeat, sign-in through the web proxy, storage round trip with persistence across runs, mail landing in Mailpit |
| opshub | `make local` (pending #431) | rollout pending: see its README | none on `main`; the e2e bucket is created by the bootstrap (pending #431) |
| knowledge base | `docker compose up -d --wait db redis garage garage-init`, then the README's API steps (Garage pending #239) | rollout pending: see its README | the CI quickstart job: migrations, health, login, worker ping and beat; the presigned PUT/GET step lands with #239 |

On `main` today, the only dev command that exists is the KB's `docker compose` quickstart. The rova and opshub commands become
available when their PRs merge; until then use the README of the branch you are on.

Do not reuse a dev stack for e2e, and do not share an e2e stack between runs.
