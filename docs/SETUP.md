# Setup: running Delineo locally

This guide takes you from an empty laptop to all four services running, with
one zone created and one simulation run. Plan on about an hour, not counting the
data download.

The commands are written for macOS. On Linux they're the same, except that you
install packages with your distribution's package manager instead of Homebrew.
On Windows, see [Windows](#windows).

If you just want the big picture first, read [Start Here](START_HERE.md).

## 0. Install the tools

| Tool | Version | Why this version |
| --- | --- | --- |
| Git | any recent | |
| Python | **3.13** | Production images use 3.13. macOS's built-in `python3` is too old for the pinned numpy. |
| [uv](https://docs.astral.sh/uv/) | any recent | Optional, but the easiest way to get Python 3.13 and create venvs. |
| Node.js | **22** | Production uses `node:22`. Next.js 16 needs at least 20.9. |
| pnpm | **10.29.1** | Pinned in `Fullstack/package.json`. Installed through corepack. |
| PostgreSQL | **16** | Production uses `postgres:16-alpine`. |

On macOS with Homebrew:

```bash
brew install git uv node@22 postgresql@16
```

```bash
brew services start postgresql@16
```

```bash
corepack enable pnpm
```

You also need read access to the
[Delineo-Disease-Modeling](https://github.com/Delineo-Disease-Modeling) GitHub
organization. Ask your team lead to add you.

## 1. Clone the four repos side by side

The scripts in `Deploy/` expect all four repos in the same parent folder.

```bash
mkdir -p ~/Code/delineo && cd ~/Code/delineo
```

```bash
for repo in Algorithms Deploy Fullstack Simulation; do git clone https://github.com/Delineo-Disease-Modeling/$repo.git; done
```

## 2. Create the Python environments

Each Python repo gets its own `.venv`. `run_servers.sh` looks for
`Algorithms/.venv` and `Simulation/.venv` by those exact names.

```bash
cd ~/Code/delineo/Algorithms && uv venv --python 3.13 && uv pip install -r requirements.txt pytest
```

```bash
cd ~/Code/delineo/Simulation && uv venv --python 3.13 && uv pip install -r requirements.txt pytest
```

`pytest` isn't in `requirements.txt` (the production images don't need it), so
it's added here for running the tests.

Without uv, the equivalent is
`python3.13 -m venv .venv && .venv/bin/pip install -r requirements.txt pytest`
in each folder.

## 3. Get the Algorithms data

Algorithms needs a large set of input data in `Algorithms/server/data/`. That
folder is in `.gitignore` because it's too big for git. **Without it, zone
generation fails** with missing-file errors.

> **TODO:** say where students get this folder (shared drive link, a copy on
> the lab server, or who to ask) and roughly how big it is.

What the code expects to find there:

| Path under `server/data/` | What it is |
| --- | --- |
| `patterns/<STATE>/<YYYY-MM>-<STATE>.parquet` | SafeGraph monthly patterns, one file per state per month |
| `shapefiles_2016/tl_2016_<FIPS>_bg/` | Census TIGER block-group shapes |
| `cbg_b01.csv` | Census population by age and sex for each CBG |
| `neighbor-states.json` | Which states border which (for zones near state lines) |
| `zip_to_cbg.json` | ZIP code to CBG lookup. Optional: a bundled copy in `server_app/bundled_seed_data/` is used if it's missing. |
| `TEST/test.csv` | Small test patterns file, used when a request asks for test data |

The shapefiles are public, so you can download them yourself instead of copying
them. For example, for Maryland and Oklahoma:

```bash
cd ~/Code/delineo/Algorithms && .venv/bin/python server/scripts/download_tiger_2016_bg.py --states MD OK --delete-zips
```

The SafeGraph patterns are licensed data, so they have to come from the team.

## 4. Create the database

Create a Postgres role and database, both named `delineo`. Pick your own
password; this is only your local database.

```bash
psql postgres -c "CREATE ROLE delineo WITH LOGIN CREATEDB PASSWORD 'choose-a-local-password';"
```

```bash
createdb -O delineo delineo
```

`CREATEDB` is there because `prisma migrate dev` creates a temporary "shadow"
database when you change the schema later.

## 5. Write the two env files

You need two env files, and **their database credentials and auth secret must
match**. `run_servers.sh` loads `Deploy/.env.local` and builds `PRISMA_DB_URL`
from its `POSTGRES_*` values. Anything it exports overrides `Fullstack/.env`
while the stack is running. Prisma commands you run by hand inside `Fullstack/`
read `Fullstack/.env` instead.

Generate one auth secret and use it in both files:

```bash
openssl rand -base64 32
```

**`Deploy/.env.local`:** copy `Deploy/.env.local.example`, then set
`POSTGRES_PASSWORD` and `BETTER_AUTH_SECRET`. The other defaults are fine.

```bash
cd ~/Code/delineo/Deploy && cp .env.local.example .env.local
```

**`Fullstack/.env`:** copy `Fullstack/.env.example`, then set:

```text
PRISMA_DB_URL=postgresql://delineo:choose-a-local-password@localhost:5432/delineo
BETTER_AUTH_SECRET=<the same secret as in Deploy/.env.local>
```

```bash
cd ~/Code/delineo/Fullstack && cp .env.example .env
```

Leave `DB_FOLDER=./db/` and the `NEXT_PUBLIC_*` URLs at their defaults.
`DELINEO_ADMIN_EMAILS` can stay empty unless you want to test admin deletion
locally.

Both env files are gitignored. Never commit them.

## 6. Install Fullstack and set up the tables

```bash
cd ~/Code/delineo/Fullstack && pnpm install
```

`pnpm install` also generates the Prisma client.

```bash
pnpm exec prisma migrate deploy
```

This applies every migration in `prisma/migrations/` to your empty database.
It's the same command production runs.

> **Don't run `pnpm db:push` here.** It's `prisma db push --force-reset`, which
> **deletes everything** in the database. Only use it on a database you're happy
> to wipe.

## 7. Start everything

```bash
cd ~/Code/delineo/Deploy && ./run_servers.sh
```

This starts all four services in one terminal. Each log line is prefixed with
the service name (`[fullstack]`, `[algorithms]`, `[simulation]`, `[dmp]`).
**Ctrl+C stops all of them.** If a port is already in use, the script skips that
service and says so.

| Service | URL | Quick check |
| --- | --- | --- |
| Fullstack | <http://localhost:3000> | The home page loads |
| Algorithms | <http://localhost:1880> | `[algorithms]` log shows it's running on port 1880 |
| Simulation | <http://localhost:1870> | Returns `{"service": "delineo-simulation", "status": "ok"}` |
| DMP | <http://localhost:8000/docs> | FastAPI's interactive API docs |

Use `localhost`, not `127.0.0.1`. The CORS lists in Algorithms and Simulation
only allow `http://localhost:3000`.

To run a single service on its own instead, see that repo's README.

## 8. Smoke test: one zone, one run

1. Open <http://localhost:3000> and sign up. The account lives only in your
   local database.
2. Go to **CZ Generator**. Search for a **small town** in a state you have
   pattern data for, and generate a zone with the default algorithm. The zone's
   size comes from the town's movement, so a small town gives a small zone that
   generates quickly. Watch the `[algorithms]` logs while it runs.
3. Go to **Simulator**, pick your zone, and run a short simulation (a day or
   two). Watch the `[simulation]` logs.
4. When the results page opens, you're done. To see where everything went, look
   in the `ConvenienceZone` and `SimData` tables in Postgres and in
   `Fullstack/db/`.

## Troubleshooting

**`ModuleNotFoundError`, or pip tries to compile numpy and fails.** The venv is
using the wrong Python. Check `.venv/bin/python --version` in that repo; it should
say 3.13. If not, delete `.venv` and redo step 2.

**Zone generation fails with "No such file" or "Expected files like ... .parquet".**
The Algorithms data is missing, or there's no patterns file for that state and
month. See step 3.

**Prisma error `P1000: Authentication failed`.** The password in `PRISMA_DB_URL`
doesn't match the Postgres role, or `Deploy/.env.local` and `Fullstack/.env`
disagree. See step 5.

**A port is already in use.** Find what's holding it with
`lsof -nP -iTCP:3000 -sTCP:LISTEN` (change the port number as needed).

**The browser console shows CORS errors.** You're probably on
`127.0.0.1:3000`. Use `http://localhost:3000`.

**Simulations are much slower than expected.** The simulator may be falling back
to HTTP calls to the DMP for every infection. This happens if the Simulation
server is started as `app:app`. Always start it from `server.py` (or as
`server:app` under gunicorn/waitress). See the
[Simulation README](https://github.com/Delineo-Disease-Modeling/Simulation#readme).

## Windows

`Deploy/run_servers.ps1` is the PowerShell version of `run_servers.sh`:

```powershell
.\run_servers.ps1
```

The rest of the steps are the same, except for the Windows venv paths
(`.venv\Scripts\python.exe`).

> **TODO:** have someone on Windows walk through this guide and fill in the
> differences.

## Docker (how production runs)

Production runs every service in Docker from `Deploy/docker-compose.yml`. You
can run the same stack locally with Docker Desktop:

```bash
cd ~/Code/delineo/Deploy && docker compose --env-file .env.local up -d --build
```

Two caveats for local use:

- The compose file mounts **hard-coded production paths** for the Postgres
  data and for Fullstack and Algorithms output. On a Mac those paths don't
  exist, so change them, or use a compose override file, before you start.
- `.env.local` also needs `ALGORITHMS_DATA_DIR` pointing at your
  `Algorithms/server/data` folder. The example file doesn't include it.

The Fullstack container applies database migrations itself when it starts.

For day-to-day development, the native setup above is faster and easier to
debug. How the production server uses this file is documented in the private
Deploy repo, in `docs/INTERNAL.md`.
