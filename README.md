# Habit Tracker with Streaks

Node 20 + Postgres + Redis, containerised, tested, and built in CircleCI.

## Run it

```bash
docker compose up --build
curl localhost:8080/health
# {"status":"ok","postgres":true,"redis":true}
curl localhost:8080/habits
```

`docker compose up` starts Postgres, Redis, a one-shot `migrate` service (runs `node src/migrate.js`, which applies `migrations/*.sql` in filename order and records them in `schema_migrations`, so restarts never duplicate the seed data), and then the app.

## Endpoints

| | |
|---|---|
| `GET /health` | Postgres and Redis reported separately |
| `GET /habits` | every habit with `streak`, `longest`, `tickedToday`, and its local `today` |
| `POST /habits` | `{"name": "Read", "schedule": [1,3,5], "timezone": "Asia/Kolkata"}` (`schedule`, `freezes`, `timezone` optional; timezone defaults to `UTC`) |
| `POST /habits/:id/tick` | mark today done (optional `{"day": "YYYY-MM-DD"}` to backfill; future days rejected) |
| `DELETE /habits/:id/tick` | undo today (optional `?day=`) |

## Tests and CI

- `npm test`: unit tests for `src/streaks.js` and `src/time.js` (no DB, no network, 100% coverage)
- `npm run test:integration`: real Postgres + Redis (CircleCI service containers; run `npm run migrate` first)
- `.circleci/config.yml`: lint → unit → integration → secret scan (gitleaks) → image build. The image is pushed to Docker Hub as `:1.0` and `:latest` from `main` only.
- CircleCI project variables needed: `DOCKERHUB_USER`, `DOCKERHUB_TOKEN` (a Docker Hub access token, not the password).

Hostnames differ by environment: in compose the host is the service name (`postgres`, `redis`); in CircleCI service containers it is `localhost`.

## Timezones: where "today" is defined

**Decision: "today" is defined by the habit's own IANA timezone (`habits.timezone`), never by the server's clock.**

The starter computed today in three places, all UTC: the `today()` helper in the server, the cache expiry (seconds until *UTC* midnight), and the conversion of Postgres `DATE` values into JS `Date` objects. For someone in Bangalore (UTC+5:30), UTC midnight is 05:30 local, so the server "rolls over" in the middle of their morning. For someone in Los Angeles it is 17:00 the previous afternoon, which is worse:

> A Los Angeles user ticked on 11 and 12 Oct. At 2026-10-14 03:00 UTC it is still the 13th in Los Angeles and they have all evening to tick. A UTC server thinks it is the 14th, sees two unticked days, and reports a streak of **0**. (This exact case is a unit test in `tests/unit/time.test.js`.)

How it is fixed:

1. **Store a timezone on each habit** (migration `003_add_timezone.sql`, default `UTC`), validated against the IANA database when a habit is created. There are no user accounts in this app, so the habit owns the timezone.
2. **Ticks are stored as a calendar `DATE`** (`'2026-10-12'`), not a timestamp. A tick means "I did this on this local day", so a UTC rollover can never move it to another day.
3. **`todayIn(tz, now)` is the single place "today" is computed** (`src/time.js`, using `Intl.DateTimeFormat`). The server calls it at the edge and passes the resulting `'YYYY-MM-DD'` string into the pure functions in `streaks.js`, which never read the clock. That keeps streak logic trivially unit-testable and keeps timezone handling out of it.
4. **The cache expires at the habit's local midnight** (`secondsUntilLocalMidnight`), and each cached value records the `today` it was computed for and is discarded if that no longer matches.
5. **`DATE` columns are read back as plain strings** (`types.setTypeParser(1082, ...)` in `db.js`), so a server running in a non-UTC zone cannot shift a day.
6. **Ticking a future day is rejected**, judged against the habit's local today.

Edge cases and trade-offs:

- A streak stays alive until the end of the user's local day, because an unticked "today" never counts as a miss.
- Days not in the schedule are skipped, and weekdays are evaluated on the local calendar date.
- DST: days are calendar dates, not 24-hour blocks, so a 23- or 25-hour day does not matter. The only imprecision is that the cache TTL can be off by an hour on a DST-change day, which is harmless because stale entries are also discarded by the `today` check.
- Travel: the timezone is fixed per habit until changed, so a trip does not silently reshuffle history. The cost is that someone who moves countries has to update it (there is no `PATCH` endpoint yet).
- Why not just use UTC? It is simpler, but it is wrong for most of the world for part of every day. A per-habit timezone costs one column and one conversion function.
CI/CD workflow: main -> stage -> prod