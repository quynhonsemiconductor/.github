# Local development

This page is the org-wide map: the fixed host ports each product's **dev** stack uses, how an **e2e** stack starts without fixed
ports, and what works today. Each product's README has the commands. Anything marked **pending** names the PR that adds it; anything
marked **not started** has no PR yet. Unmarked entries are on `main` now.

## Dev stacks: fixed ports

Each product owns a block so the products run side by side. Every host port is bound to `127.0.0.1`.

| Product | API | Web (Vite) | Postgres | Valkey / Redis | Object storage | Mail | OTel (OTLP / metrics) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| rova (block pending #742) | 3010 | 5175 | 5420 | 6320 | LocalStack 4620 (Garage: not started) | Mailpit 1120 / 8120 (pending #742) | 4420 / 8420 |
| opshub (block pending #427) | 3001 | 5174 | 5410 | 6310 | LocalStack 4610 (Garage: not started) | Mailpit 1110 / 8110 (pending #431) | 4410 / 8410 |
| knowledge base (qnsc-kb-backend) | 8000 | 5173 | 5432 | 6379 | Garage 4570 | console locally; Graph when deployed (invitations and password resets) | none |

- LocalStack's community edition does **not** keep its data across a restart (its persistence option is a Pro feature). The KB
  uses Garage on 4570, whose volumes survive a restart. rova and opshub still run LocalStack, so a local upload there does not
  survive a restart; their Garage move is not started.
- The web port is strict: Vite refuses to start on a taken port, because the sign-in redirect and the bucket CORS are set for that origin.
- The KB keeps the default ports. Its frontend (qnsc-kb-frontend) uses the same 5173 as its dev server.

## E2E stacks: no fixed ports

The e2e suite writes to the database it points at, and several people and agents run it at once. So an e2e stack:

1. has its **own compose project name**, unique per run;
2. publishes **ephemeral loopback ports** (`127.0.0.1::<container-port>`), never the table above;
3. is found by the harness with `docker compose -p <project> port <service> <container-port>`, and reads the resulting URLs from
   an env file written for that run.

Nothing in the e2e path may assume a port from the table. Stop and remove your own stack when done. If a stack is not yours, ask its
owner first.

## Commands and status

| Product | Dev stack | E2E stack | Checks today |
| --- | --- | --- | --- |
| rova | On `main`: the compose file and the `pnpm` steps in its README (CI's local quickstart job runs them). `make local` / `make check` / `make down`: pending #742. | `scripts/e2e/run.sh`: pending #742 | On `main`: the CI local quickstart. `make check` (readiness, worker heartbeat, sign-in through the web proxy, storage round trip, mail landing in Mailpit): pending #742 |
| opshub | On `main`: the compose file and the `pnpm` steps in its README. `make local`: not started (after #431). | `scripts/e2e/run.sh`: pending #435 | On `main`: the CI local quickstart and the env example spec. `make check`: not started. |
| knowledge base | On `main`: `docker compose up -d --build db redis garage garage-init`, then the README's API steps. Garage replaces LocalStack. | `scripts/e2e/run.sh`: pending #253 | On `main`: the CI quickstart job (migrations, health, login, worker ping, beat, and a presigned PUT/GET through Garage). |

Do not reuse a dev stack for e2e, and do not share an e2e stack between runs.
