# Local development

Every product runs its local stack from its own repository's compose file. This page is the org-wide map: which fixed host ports
each **dev** stack uses, and how an **e2e** stack is started without fixed ports. The product READMEs carry the commands.

## Dev stacks: fixed ports

Each product owns a block, so the products run side by side on one machine. Every host port is bound to `127.0.0.1`.

| Product | API | Web (Vite) | Postgres | Valkey / Redis | Object storage | Mail (SMTP / UI) | OTel (OTLP / metrics) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| rova | 3010 | 5175 | 5420 | 6320 | LocalStack 4620 | Mailpit 1120 / 8120 | 4420 / 8420 |
| opshub | 3001 | 5174 | 5410 | 6310 | LocalStack 4610 | Mailpit 1110 / 8110 | 4410 / 8410 |
| knowledge base (qnsc-kb-backend) | 8000 | 5173 | 5432 | 6379 | Garage 4570 | none (sends no mail) | none |

- The knowledge base keeps the default ports; its frontend (qnsc-kb-frontend) uses the same 5173 as its dev server.
- A web port is strict: Vite refuses to start on a taken port, because the sign-in redirect and the bucket CORS are set for that origin.
- Local mail is `EMAIL_PROVIDER=smtp` (rova, opshub): it goes to Mailpit, nothing leaves the machine.

## E2E stacks: no fixed ports

The e2e suite truncates the database it points at, and several people and agents test the same product at once. So an e2e stack:

1. has its **own compose project name** (`-p <product>-e2e-<who>`, unique per run);
2. publishes **ephemeral loopback ports** (`127.0.0.1::<container-port>`, the host port is picked by Docker), never the table above;
3. is found by the harness with `docker compose -p <project> port <service> <container-port>`, or through the database and
   cache URLs it writes to its env file.

Nothing in the e2e path may assume a port from the table. A test run on a fixed port can collide with another run and truncate its
data.

Stop and remove your own stack when you are done: `docker compose -p <project> down -v`. Leave other stacks alone; if a stack is not
yours, ask its owner first.

### Commands, per product

| Product | Dev stack | E2E stack |
| --- | --- | --- |
| rova | `make local` (then `make check`, `make down`) | `scripts/e2e/run.sh` (own project, its e2e port block) |
| opshub | `make local` (then `make check`, `make down`) | rollout in progress: see its README |
| knowledge base | `docker compose up -d --wait db redis garage garage-init` (README "Run it locally") | rollout in progress: see its README |

Until a product's e2e rollout lands, its README is the source for the e2e command. Do not reuse a dev stack for e2e.

## Checks

Each product's `check` command (rova and opshub: `make check`) proves the dev stack works end to end: readiness, the worker, sign-in
through the web proxy, a storage round trip with persistence across runs, and a mail that lands in Mailpit. The knowledge base proves
its ingest and ask path through its quickstart job.
