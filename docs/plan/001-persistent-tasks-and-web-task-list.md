---
plan_id: 001-persistent-tasks-and-web-task-list
spec: 001-persistent-tasks-and-web-task-list
feature: Keep tasks in Postgres and give the website a task list with a create form
gitops_app: ika100-demo/taskboard27-gitops
status: draft
repos:
  - id: taskboard27-gitops
    shape: gitops-app
    summary: Postgres database for the Task API, two API instances in dev, the API's URL for the website
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.5, AC-001.6]
    order: before-merge
    gitops:
      - {resource: postgres, type: database}
      - {uses: postgres, component: taskboard27-api}
      - {replicas: 2, component: taskboard27-api, instance: dev}
      - {env: {TASKBOARD27_API_URL: 'http://taskboard27-api'}, component: taskboard27-web}
    arguments: |
      `uses: postgres` gives taskboard27-api DATABASE_URL and PGHOST/PGPORT/PGDATABASE/PGUSER/PGPASSWORD from the
      cluster-generated secret (A3). The env change is harmless to the current images: the in-memory API ignores
      DATABASE_URL and the current web ignores TASKBOARD27_API_URL until their PRs merge.
    depends_on: []
    done: false
  - id: taskboard27-api
    shape: service-python
    summary: Persistent tasks in Postgres
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.7]
    criteria:
      - "Given tasks A, B and C were created through `POST /tasks`, when the service is restarted (or a new version started) against the same database, then `GET /tasks` returns `200` with exactly A, B and C, oldest first, each with the same `id`, `title` and `due_date` as returned at creation (system AC-001.1) (replaces AC-001.7)"
      - "Given an empty database reached through `DATABASE_URL`, when the service starts, then it creates its own schema without a manual step and `GET /tasks` returns `200` with `[]`; starting again against an existing schema keeps every stored task (system AC-001.1)"
      - "Given two service processes connected to the same database, when a task is created through `POST /tasks` on one, then each following `GET /tasks` on either process includes that task (system AC-001.2)"
      - "Given 1000 tasks are stored, when a client sends another valid `POST /tasks`, also after a restart and also when two processes receive valid `POST /tasks` at the same time while 999 are stored, then every request beyond the 1000th is answered `409` with `{\"detail\": \"task limit of 1000 reached\"}` and `GET /tasks` returns exactly 1000 tasks (system AC-001.4)"
      - "Given the database is unreachable, when a client sends `POST /tasks` or `GET /tasks`, then the response arrives within 6 seconds as `503` with `{\"detail\": \"task storage unavailable\"}`, no task is stored, and `GET /ready` answers `503`; given the database is reachable again, then `GET /ready` answers `200` and `GET /tasks` returns every task stored before the outage (system AC-001.3)"
      - "Given a running service, when a client sends `POST /tasks` with the title missing, empty or only whitespace after trimming, or longer than 200 characters after trimming, then the response is `422` in the standard body `{\"detail\": [{\"type\", \"loc\", \"msg\", \"input\"}]}` with an item whose `loc` ends in `\"title\"`, whose `type` is the contract rule id (`title-required`, `title-min` or `title-max`) and whose `msg` is non-empty, and no task is stored (system AC-001.7)"
    arguments: |
      Keep POST/GET /tasks, the task fields, the due_date validation and its 422 exactly as spec 001 built them (A1);
      only where tasks are kept and the title errors' `type`/`msg` values change (see the plan's Contract).
      Connection settings come from DATABASE_URL (injected by the gitops `uses: postgres`); no new env var.
    depends_on: []
    done: false
  - id: taskboard27-web
    shape: web-nextjs
    summary: Task list and create form on the start page
    acs: [AC-001.5, AC-001.6, AC-001.7, AC-001.8]
    criteria:
      - "Given the Task API's `GET /tasks` answers `200` with tasks, when a person opens the start page, then below the form it lists every task in the order the API returned (oldest first), each with its title and its `due_date` as `YYYY-MM-DD` or the text `No due date`; with `[]` it shows `No tasks yet`; the list is fetched on every page request, so a reload shows tasks created elsewhere (system AC-001.5)"
      - "Given the start page is open, when a person enters the title `Buy milk`, optionally picks a due date in the date input, and presses `Add task`, then the website sends `POST /tasks` with `{\"title\": <trimmed title>, \"due_date\": \"YYYY-MM-DD\" or null}`, and on `201` the returned task appears at the end of the list without a full page reload and the form is emptied; after reloading the page the task is still listed with the same title and due date (system AC-001.6)"
      - "Given the start page is open, when a person submits the form with an empty title, a title of only spaces, or a title longer than 200 characters after trimming, then `Enter a title of 1 to 200 characters` appears at the title field, the entered values stay in the form, and no request is sent to the Task API (system AC-001.7)"
      - "Given a valid submit, when the Task API answers `422` with a `detail` item whose `loc` ends in `\"title\"`, then that item's `msg` appears at the title field and the entered values stay in the form (system AC-001.7)"
      - "Given the Task API is unreachable, does not answer within 5 seconds, or answers `GET /tasks` with anything but `200` and a JSON array, when a person opens the start page, then the list area shows `Tasks could not be loaded. Try again later.` and the create form is still shown (system AC-001.8)"
      - "Given a valid submit, when the Task API answers `409`, then the form shows `Task limit of 1000 reached.`; when it answers any other non-`201` status not covered above (including `503` and `500`), is unreachable or does not answer within 5 seconds, then the form shows `The task could not be saved. Try again.`; in both cases the entered values stay in the form and the list is unchanged (system AC-001.8)"
    arguments: |
      The browser never calls the Task API directly: the start page (server-rendered, dynamic) and the create action
      run on the web server and call `${TASKBOARD27_API_URL}/tasks` (env var set by the gitops entry; document it in
      docs/env-vars.md). No NEXT_PUBLIC_ variable for the API URL.
    depends_on: []
    done: false
gitops_pin: []
---

## taskboard27-gitops — Postgres database for the Task API, two API instances in dev, the API's URL for the website

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.5, AC-001.6 · **Depends on:** []

The system declares no database yet. This entry adds the `postgres` database resource (run in the cluster by the platform's database operator, credentials generated in the cluster, A3), wires `taskboard27-api` to it (`uses`, which injects `DATABASE_URL`), runs the API with 2 replicas in dev so AC-001.2 is exercised where the smoke runs, and gives `taskboard27-web` the in-cluster URL of the API (`http://taskboard27-api`, the API's Service on port 80). It owns the infrastructure part of AC-001.1 to AC-001.3 (a database that outlives API pods, reachable by every instance) and the wiring part of AC-001.5 and AC-001.6 (the website can reach the API). It is applied before the component PRs merge.

## taskboard27-api — Persistent tasks in Postgres

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.7 · **Depends on:** []

The in-memory `TaskStore` is replaced by a Postgres-backed store reached through `DATABASE_URL`. The API owns durability across restarts and redeploys (AC-001.1), one shared list for every instance (AC-001.2), the `503 "task storage unavailable"` answer and a `/ready` that reflects database reachability (AC-001.3), the 1000-task limit counted over all stored tasks and safe across concurrent instances (AC-001.4), and the server-side guarantee that an invalid title stores nothing (AC-001.7). Its spec 001 criterion AC-001.7 ("tasks are lost on restart") is replaced by the first criterion of this slice. The endpoints, fields, ids and due-date validation stay as built (A1); the title errors' `type` and `msg` values change to name the shared rules so that the contract check can verify them. Tasks held in memory at deploy time are not carried over (A2).

## taskboard27-web — Task list and create form on the start page

**Shape:** `web-nextjs` · **Criteria:** AC-001.5, AC-001.6, AC-001.7, AC-001.8 · **Depends on:** []

The website's start page gets a create form (above) and the task list (below, A6), with the English texts of A9. It owns rendering the list (AC-001.5), creating a task without a full reload (AC-001.6), the client-side title check with the API's rule (AC-001.7, A7) and every failure message (AC-001.8). It reaches the API only from the Next.js server through `TASKBOARD27_API_URL`, because the in-cluster address is not reachable from a browser and the API is not meant to be called cross-origin. It needs no `depends_on`: the contract below fixes everything it calls, and it tests against a stubbed API; its persistence-after-reload criterion is verified end to end once both are in dev.

One decomposition: three entries, all at level 0. The library shape is not used because no type is shared between a Python and a TypeScript repo; the contract is the shared artefact.

## Contract

No authentication (A8). JSON in and out, `Content-Type: application/json`. Provider: `taskboard27-api`; consumer: `taskboard27-web`, which calls it server-side at `${TASKBOARD27_API_URL}` (`http://taskboard27-api` in the cluster).

**Task object** (every task in a response): `{"id": string (UUID), "title": string (trimmed, 1..200 characters), "due_date": "YYYY-MM-DD" | null}`. No other fields.

**`POST /tasks`**

Request body (object; unknown fields ignored):

| Field | Type | Rules |
|---|---|---|
| `title` | string, required | trimmed; after trimming 1..200 characters; stored trimmed |
| `due_date` | string or null, optional (default `null`) | exactly `YYYY-MM-DD` and a real calendar date; past dates accepted |

Responses:

- `201` → the created Task object; it is stored and every instance lists it from then on.
- `422` → validation failed, nothing stored. FastAPI's standard body: `{"detail": [{"type": string, "loc": ["body", "<field>"], "msg": string, "input": any}]}`. For the title rules below, `type` is the rule id (`title-required`, `title-min`, `title-max`) and `msg` is `"Enter a title of 1 to 200 characters"`. A title that is not a string, and every `due_date` failure, keep pydantic's own `type`/`msg`. Validation runs before the limit and storage checks.
- `409` → `{"detail": "task limit of 1000 reached"}`: 1000 tasks are already stored; nothing stored. The limit holds across instances (the count and the insert are serialised in the database).
- `503` → `{"detail": "task storage unavailable"}`: the database cannot be reached or does not answer in time; nothing stored.

**`GET /tasks`**

- `200` → JSON array of every stored Task object, oldest first (creation order, stable across instances and restarts); `[]` when none. Unpaginated, at most 1000 entries.
- `503` → `{"detail": "task storage unavailable"}`.

**`GET /ready`** → `200` when the database answers a trivial query, else `503` (the pod leaves the Service until it is reachable again). **`GET /health`** stays `200` while the process runs, independent of the database, so an outage does not restart pods.

**Website start page** (`GET /` on taskboard27-web): form above the list; title field, date input (`type="date"`, sends `YYYY-MM-DD` or `null`), button `Add task`; texts exactly as in A9.

### Rules

```yaml
rules:
  - id: title-required
    endpoint: POST /tasks
    field: title
    kind: required
    value: true
    provider: taskboard27-api
    consumers: [taskboard27-web]
  - id: title-min
    endpoint: POST /tasks
    field: title
    kind: min_length
    value: 1
    provider: taskboard27-api
    consumers: [taskboard27-web]
  - id: title-max
    endpoint: POST /tasks
    field: title
    kind: max_length
    value: 200
    provider: taskboard27-api
    consumers: [taskboard27-web]
```

Prose rules no tool checks: lengths count characters after trimming surrounding whitespace, so a title of only spaces violates `title-min`; `due_date` is a calendar date `YYYY-MM-DD` or `null`, which the website guarantees by its date input (A7), and the API keeps rejecting anything else with `422` on `due_date`.

### Errors

```yaml
errors:
  status: 422
  body:
    list: detail
    field: loc
    rule: type
    message: msg
  unlisted:
    message: "The task could not be saved. Try again."
```

`loc` is a path (`["body", "title"]`); a caller matches it by its last element. The website treats responses as follows: `422` with a `detail` item whose `loc` ends in `title` shows that item's `msg` at the title field; `409` shows `Task limit of 1000 reached.` on the form; every other answer of `POST /tasks` (`503`, `500`, a `422` without a title item, a body that is not JSON, any unlisted status) shows `The task could not be saved. Try again.` on the form. For `GET /tasks`, anything but `200` with a JSON array shows `Tasks could not be loaded. Try again later.` in the list area. In every failure case the entered form values stay.

### Timeouts

- Website → `GET /tasks` and `POST /tasks`: 5 seconds per call (connect plus response), no retry. On timeout or connection failure the website shows the load message (list) or the save message (form), as for any unlisted answer. A `POST` that timed out may still have been stored; the person sees it after a reload (no retry, so no duplicate is created by the website).
- API → Postgres: connect timeout 3 seconds, statement timeout 3 seconds; on either the request is answered `503 "task storage unavailable"`, so the API answers well inside the website's 5 seconds (within 6 seconds in the worst case of connect plus statement). `/ready` uses the same limits. When the database comes back, the next request reconnects; no restart of the API is needed.
- Website when the API is down or slow: the start page still renders (form plus load message) within about 5 seconds; `/api/ready` of the website does not depend on the API.

## gitops-app PR

The entry PR (`taskboard27-gitops`, before-merge) adds the `postgres` resource, `uses: postgres` and 2 dev replicas for taskboard27-api, and `TASKBOARD27_API_URL` for taskboard27-web; `devbox run render` regenerates the overlays. After Argo reconciles dev: the database cluster is healthy and its app secret exists, both taskboard27-api pods are running with `DATABASE_URL` set, and taskboard27-web has `TASKBOARD27_API_URL=http://taskboard27-api`. Until the API PR merges, the two in-memory API pods each hold their own list; that state lasts only until the persistent image rolls out in dev.

No `gitops_pin` entries: the components run only in dev today (`instances: dev` in `system.yaml`), and dev takes each component's image when its PR merges. After both component PRs merge and dev is verified (`/instance:verify dev`: create a task on the website, restart both API pods, reload, the task is still there), staging and prod come through `/instance:promote taskboard27-api --to staging` and `/instance:promote taskboard27-web --to staging` (A10); promoting the API needs the `postgres` resource and 2 replicas in that environment too.
