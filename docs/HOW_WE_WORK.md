# How We Work

Read this before you open your first pull request.

## The one rule: `main` is production

Every push to `main` in Algorithms, Fullstack, or Simulation **automatically
deploys that service** to <https://covidmod.isi.jhu.edu>. A push to `main` in
Deploy redeploys everything. Nobody presses a deploy button, so merging a PR
is the same as shipping it to users.

So:

- Never commit directly to `main`.
- Do all work on a short-lived branch and merge it through a pull request.
- Only merge once the checks below pass and someone has reviewed it.

## Branches and pull requests

1. Start from the latest `main`:

   ```bash
   git switch main && git pull
   ```

2. Create a branch named `<your-name>/<what-it-does>`, for example
   `ryad/fractional-dwell` or `maya/fix-zone-legend`:

   ```bash
   git switch -c <your-name>/<what-it-does>
   ```

3. Keep each PR to one change. A focused PR is easier to review and easier to
   revert.
4. In the PR description, say what changed, why, and how you tested it. For UI
   changes, add a screenshot.
5. Ask for a review. After merging, delete the branch.

> **TODO:** say who reviews PRs for each repo, and whether any repo needs more
> than one approval.

## Checks to run before you ask for review

| Repo | Run from | Commands |
| --- | --- | --- |
| Fullstack | `Fullstack/` | `pnpm lint`, `pnpm typecheck`, `pnpm test`, `pnpm build` |
| Algorithms | `Algorithms/` | `.venv/bin/python -m pytest tests` |
| Simulation | `Simulation/` | `.venv/bin/python -m pytest` |
| Deploy | `Deploy/` | `bash -n deploy.sh && bash -n run_servers.sh` (syntax check), then test the change by hand |

None of these run in CI yet, so it's on you to run them. Also try the change in
the running app. Most bugs here show up in the handoff between services, and
unit tests won't catch them.

## Changes that touch more than one repo

Some data passes between services: the papdata and movement patterns format,
the zone and run API routes, and the simulation request body. If you change
one of these, you'll be changing more than one repo, and each repo deploys
separately. [Data formats](DATA_FORMATS.md) describes each one exactly.

- **Ship the reader before the writer.** Make the consumer accept both the old
  and new formats, deploy it, and only then change the producer. The binary
  patterns format was rolled out this way: Simulation learned to detect and
  read both formats before Algorithms started sending the binary one.
- Link the related PRs to each other and say which one merges first.

## Feature flags

Risky behavior changes go behind an environment variable, usually named
`DELINEO_<SOMETHING>`, so they can be turned off on the server without a code
change. Examples include `DELINEO_PATTERNS_BINARY`, `DELINEO_AGG_TRANSMISSION`,
and `DELINEO_FRACTIONAL_DWELL`. Read the comment where the flag is defined
before changing its default.

## Database changes (Fullstack)

1. Edit `prisma/schema.prisma`.
2. Run `pnpm db:migrate` and give the migration a descriptive name. This creates
   a new folder in `prisma/migrations/` and applies it to your local database.
3. Commit the schema change and the migration folder together.

Production applies new migrations with `prisma migrate deploy` before it starts
the new Fullstack container. **Never edit or delete a migration that's already
on `main`.** Make a new migration instead.

## Environment variables

When you add one:

- add it to the repo's `.env.example` (if it has one) with a comment,
- mention it in the repo's README,
- add it to `Deploy/docker-compose.yml`, or production won't see it,
- if it's a secret, add a placeholder to `Deploy/.env.production.example` and
  tell whoever manages the server.

Never commit a real `.env`, `.env.local`, or `.env.production` file.

## Documentation

If your change makes a doc wrong, fix the doc in the same PR. That includes
setup steps, env vars, API routes, and anything in the [Glossary](GLOSSARY.md).
For bigger investigations, write up what you found. `Algorithms/docs/MOVEMENT_MODEL_REDESIGN.md`
is a good example: what was wrong, the evidence, and the fix that was chosen.

## Working with AI coding tools

Much of this codebase was written with Claude Code and Codex, and you're
welcome to use them. A few habits keep that safe:

- **You own the change.** Read every line of the diff before you commit it. If
  you can't explain a change in review, it's not ready.
- **Give the tool the context.** Point it at this folder, the relevant README,
  and the design docs. It will follow the same conventions you do.
- **Make it prove the change works.** Ask it to run the checks above and to
  exercise the change in the running app, not just to write code.
- **Keep it off `main`.** Have it work on a branch, and don't let it push or
  merge for you.
- **Watch for confident mistakes.** AI tools will happily invent a function,
  config flag, or API route that doesn't exist. Check anything surprising
  against the code.

## Deploys

When a PR is merged into `main`, production rebuilds that service on its own,
usually within a few minutes. A merge into Deploy rebuilds every service.

- **Check the result on GitHub.** The merged commit gets a `deploy` status
  check. Green means it's live. Red means the deploy failed: tell the project
  lead right away, because the service may be down.
- **To roll back**, revert the bad commit on `main` with a PR. Merging the
  revert redeploys the previous code.

How the production server works is documented in the private Deploy repo, in
`docs/INTERNAL.md`. You need to be in the org to read it.
