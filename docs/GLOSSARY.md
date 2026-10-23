# Glossary

The words that show up across all four Delineo repos, roughly in the order
you'll run into them. The names in `code font` are the ones used in the code.

## Geography and input data

- **Census Block Group (CBG)**: The smallest area the US Census publishes sample data for, usually 600 to
  3,000 people. CBGs are Delineo's basic unit of geography: zones are built from
  them, and every person's home belongs to one.
- **GEOID**: A CBG's 12-digit ID: 2 digits of state FIPS, 3 of county, 6 of tract, and 1 of
  block group. For example, `240430001001` is in Maryland (`24`). **Leading zeros
  matter.** California CBGs start with `06`. If a GEOID gets stored as a number,
  that zero is lost and you get an 11-digit string. Algorithms pads 11-digit
  inputs back to 12, but keep GEOIDs as strings everywhere.
- **FIPS code**: The numeric code for a state (2 digits) or county (3 digits). Shapefiles and
  pattern files are organized by state FIPS or state abbreviation.
- **SafeGraph patterns**: Foot-traffic data for points of interest (POIs): how many visits each place
  got, which CBGs the visitors came from (`visitor_daytime_cbgs`), how busy it
  was each hour (`popularity_by_hour`), and how long people stayed
  (`median_dwell`). Algorithms reads monthly files per state from
  `server/data/patterns/<STATE>/<YYYY-MM>-<STATE>.parquet`. Both clustering and
  movement generation depend on this data.
- **POI (point of interest)**: A place people visit: a store, school, church, clinic. Each POI is identified
  by its SafeGraph `placekey`.
- **Dwell time**: How long a visit lasts. With **fractional dwell** (on by default in both
  Algorithms and Simulation), visits can last part of an hour instead of always
  being rounded to whole hours.
- **TIGER shapefiles**: Census boundary files. Algorithms uses the 2016 block-group shapes in
  `server/data/shapefiles_2016/` to draw zones and to place homes on the map.
  `server/scripts/download_tiger_2016_bg.py` downloads them.
- **ACS (American Community Survey)**: Census survey data on age, sex, and household makeup. Algorithms reads it from
  `cbg_b01.csv` and from the Census API to make the synthetic population
  realistic.

## Zones and populations

- **Convenience zone (CZ)**: A group of CBGs chosen so that most of the residents' everyday movement stays
  inside the group. Delineo simulates one zone at a time. A zone is a
  `ConvenienceZone` row in Postgres, and its ID is `czone_id` everywhere.
- **Seed CBG**: The CBG the user started from. Clustering grows the zone outward from it (or
  from several seed CBGs).
- **Clustering algorithm**: The method used to grow a zone. Each has a name in
  `VALID_CLUSTER_ALGORITHMS` (`Algorithms/server/server_app/constants.py`),
  for example `czi_balanced`, `greedy_weight`, or `mobility_prune`. The
  `czcode_modules/` package implements them.
- **Containment** (also called the CZI): The share of movement involving the
  zone that stays inside it. Most clustering algorithms try to keep it high
  while reaching a population target (`min_pop`). The recommended Mobility Prune
  algorithm instead uses **seed capture** (the share of the seed's own movement
  kept inside the zone) and ignores `min_pop`. See
  [Clustering](https://github.com/Delineo-Disease-Modeling/Algorithms/blob/main/docs/CLUSTERING.md).
- **Synthetic population (popgen)**: Made-up people, households, and places that match a zone's real Census
  statistics. Nobody in it is a real person. `Algorithms/server/popgen.py`
  generates it.
- **papdata**: Short for "people and places data": the synthetic population as JSON, with
  `people` (age, sex, home), `homes`, and `places` (POIs). Algorithms produces it,
  Fullstack stores it, and Simulation reads it.
- **Movement patterns** (or just "patterns"): For each timestep, where each person is: at home or at a particular POI.
  `Algorithms/server/patterns.py` generates them. Careful: "patterns" can mean
  this output, or the SafeGraph input data above. Check which one the code is
  talking about.
- **DLNOPAT**: The compact binary file format for movement patterns, used by default instead
  of JSON. The `patterns_codec.py` files in Algorithms and Simulation encode and
  decode it.

## Simulation

- **Run** (also `simdata` or simulation): One simulation of one zone with one set of settings. It is a `SimData` row in
  Postgres, plus output files in Fullstack's `DB_FOLDER`. "Saved" runs are kept;
  unsaved runs can be cleaned up.
- **Timestep**: One step of simulated time, 60 minutes by default. **Watch the
  units of `length`:** a zone's `length` (`ConvenienceZone.length`) is in
  **hours**, but the `length` in a simulation request and on a run
  (`SimData.length`) is in **minutes**. Fullstack sends `hours * 60`.
- **Wells-Riley model**: The airborne transmission model used to decide who gets infected. Infectious
  people emit **quanta** (doses of infectious particles) into a room, and
  ventilation removes them. The longer a susceptible person shares the air, the
  likelier infection becomes. The code is in
  `Simulation/simulator/transmission.py` and `simulator/infection_models/`.
- **Ventilation / area-aware ventilation**: How fast a place clears infectious air. By default, ventilation scales with
  each POI's floor area, so crowded small rooms are riskier than large ones.
- **Interventions**: Policies applied from a given time onward, each a fraction from 0 to 1:
  `mask`, `vaccine`, `capacity`, `lockdown`, `selfiso` (self-isolation). An
  intervention's `time` is in **hours**. See
  [Simulation engine](https://github.com/Delineo-Disease-Modeling/Simulation/blob/main/docs/SIMULATION_ENGINE.md)
  for exactly how each one works.
- **Disease model / variant**: Which disease (`disease_name`) and strain (`variants`) a run simulates. Each
  variant can have its own DMP transition matrices.
- **External force of infection (external FOI)**: An optional setting for infections that come from outside the zone. It is off
  by default.

## The DMP (Disease Modeling Platform)

- **DMP**: The part of Simulation that decides what happens to a person after they are
  infected. It lives in `Simulation/dmp/`. The simulator usually runs it
  in-process; it can also run as a separate FastAPI service on port 8000.
- **Disease states**: The stages a person can move through: `Infected`,
  `Infectious_Asymptomatic`, `Infectious_Symptomatic`, `Hospitalized`, `ICU`,
  `Recovered`, `Deceased`.
- **State machine**: The DMP's model of a disease: the states, plus the possible moves between
  them and how long each stage lasts.
- **Transition matrix**: The table of probabilities for moving from each state to each other state. A
  different matrix can apply to each demographic group (age, sex, vaccination
  status, variant). `demographic_mapping.csv` says which group uses which matrix
  set. Each row of a matrix must sum to 1, and a test checks this. Users can
  upload their own matrices, which are stored as `DmpMatrix` rows.

## Fullstack and infrastructure

- **`DB_FOLDER`**: The Fullstack folder (default `./db/`) where large files live: papdata,
  movement patterns, simulation output, and map caches. Postgres only stores
  their IDs.
- **Prisma**: The database toolkit Fullstack uses. The schema is
  `Fullstack/prisma/schema.prisma`, and every schema change needs a migration in
  `prisma/migrations/`.
- **Better Auth**: The library that handles sign-up, login, and sessions in Fullstack.
- **Guest zones / claim token**: People can create a zone without logging in. The browser keeps a claim token,
  and after they sign in, the zone is moved to their account.
- **Admin**: A user whose email is listed in `DELINEO_ADMIN_EMAILS`. Admins can delete any
  zone or run.
- **SSE (Server-Sent Events)**: A one-way stream from server to browser. Simulation uses it to send progress
  updates while a run is going.
- **Webhook / auto-deploy**: When anything is pushed to `main` in a repo, GitHub notifies the production
  server, and it rebuilds that service. See
  [How We Work](HOW_WE_WORK.md#deploys).
