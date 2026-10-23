# Model assumptions and limitations

This page lists the assumptions built into Delineo's model, which parts are
calibrated and which aren't, and what a run can and can't be used to claim.
Read it before you present results, write a paper section, or change a model
parameter.

Technical details are in the deep dives:
[Clustering](https://github.com/Delineo-Disease-Modeling/Algorithms/blob/main/docs/CLUSTERING.md),
[Population and movement](https://github.com/Delineo-Disease-Modeling/Algorithms/blob/main/docs/POPULATION_AND_MOVEMENT.md),
and [Simulation engine](https://github.com/Delineo-Disease-Modeling/Simulation/blob/main/docs/SIMULATION_ENGINE.md).

Everything here describes the code as of October 2026.

## What the model is good for


Based on what's calibrated today, results are most trustworthy as **relative
comparisons**: the same zone, the same disease, and the same seeds, with and
without an intervention. Absolute numbers, such as "1,240 people will be
infected", depend on parameters that aren't calibrated yet (see below), so
treat them as illustrative.

A single run is **one random outcome**. Compare scenarios with several runs
each, using different `random_seed` values, and look at the spread, not just
one line. `Simulation/scripts/equivalence_ensemble.py` shows how to run an
ensemble.

## Calibration status

| Parameter | Value | Where | Status |
| --- | --- | --- | --- |
| Quanta emission rate `q` | 20 per hour | `Simulation/simulator/runner.py` (`_precompute_location_quanta`) | Fixed. **The same for every disease and variant.** |
| Breathing rate `p` | 0.5 m³/hour | same | Fixed |
| Ventilation coefficient | 9.0 m³/hour per m² of floor | `DELINEO_VENTILATION_COEFF` | **Not calibrated.** `config.py` describes it as a physical estimate. |
| Household ventilation | 150 m³/hour | `transmission.py` | Fixed |
| Movement scale | ≈ 19.7 / 30 (from the data) | `DELINEO_MOVEMENT_SCALE` | **Not calibrated.** The code logs a warning on every run. |
| External prevalence | 0 | `DELINEO_EXTERNAL_PREVALENCE` | Not calibrated. External infection is off by default. |
| Mask effect | 70% less emitted, 50% less inhaled | `transmission.py` | Fixed |
| Vaccine effect | 65% against infection, 40% against transmission | `infection_models/v6_wells_riley.py` | Fixed. No waning (see below). |
| Employment rate | 65% (18-64), 10% (65+) | `Algorithms/server/worker_assignment.py` | Fixed. Not taken from local data. |
| Disease progression | DMP state machines | `Simulation/dmp/app/state_machine/state_machines.db` | **TODO:** document where the transition probabilities and durations come from. |

The database holds 17 state machines: 10 for COVID-19 (a default, Delta, and
Omicron) and 7 for measles (a default, vaccinated, and unvaccinated). None of
them records its source.

## Assumptions, by stage

### The zone is a closed world

- Only residents of the zone exist. Visitors from outside never appear, and
  residents never leave (except students in a zone with no school, who go to a
  placeholder "Out of Zone School"). Instead, POI crowds are scaled down to the residents'
  share (`catchment_fj`). That share is often small: the movement redesign doc
  measured about 85% of visitors to one zone's POIs living outside it.
- The **external force of infection** setting can approximate infections
  brought in by outsiders, but it's off by default and uncalibrated.
- A zone is a starting point chosen by an algorithm and then edited by hand.
  Different algorithms, capture targets, or months give different zones, and
  so different results.

### The population is synthetic

- Households and ages follow Census statistics. Household makeup comes from
  county-level ACS 2023 tables, and age/sex from 2016 CBG-level tables. No real
  person is represented.
- The population is full scale: about one synthetic person per resident.
- **Data vintages differ:** 2016 CBG shapes and age/sex tables, 2023 ACS
  household tables, and SafeGraph patterns from whichever month the zone uses.
  CBG boundaries were redrawn for the 2020 Census. 
- A zone that crosses a state line uses one state's household statistics for
  all its counties.

### Movement is statistical, not behavioral

- How busy each POI is each hour comes from SafeGraph, which samples phones.
  **Who** fills each POI is random: any resident who's at home is equally
  likely, regardless of age, distance, or habit.
- Every worker works at one in-zone POI from 9 to 5 on every weekday, and every
  student is at school from 8 to 3 on every weekday. There are no weekends at
  work, no holidays, and no school breaks.
- One month of SafeGraph data defines the pattern for the whole run, and
  seasonal change isn't modeled.
- Interventions change movement in simple ways. Lockdown cancels a share of
  trips; self-isolation keeps a share of symptomatic people home; capacity
  limits turn people away at the door. Nobody changes *where* they go in
  response.

### Transmission is well-mixed and airborne

- Everyone in a room shares the same air (Wells-Riley). Distance, room layout,
  and close contact inside a room aren't modeled.
- **Transmissibility doesn't depend on the disease.** The emission rate is
  the same for COVID-19, measles, and every variant. The disease only changes
  how a person progresses after infection, through the DMP. Measles is usually
  considered far more transmissible than COVID-19, so this choice matters for
  measles runs.
- The time step is one hour. Shorter contacts are only captured through
  fractional dwell.
- Homes use a fixed ventilation rate, regardless of size.
- **No reinfection:** anyone ever infected is immune for the rest of the run.

### Disease progression

- Each infected person gets one random path through the matching DMP state
  machine, chosen by age, sex, vaccination status, and variant.
- If no state machine matches, the default `dmp_mode: "auto"` silently uses a
  simple fallback with **no hospitalization or death**. Check
  `timeline_source_counts` in a run's metadata, or use `dmp_mode: "required"`.
- Hospitalized, recovered, and deceased people stop taking part in
  transmission, but **they keep following their movement schedule**, so they
  still appear at places on the maps.
- Vaccine waning is implemented but never applied, because nothing advances a
  person's days-since-vaccination.

## Known modeling questions

These are open items worth turning into issues or student projects:

1. Calibrate `ventilation_coeff` and `movement_scale` against an observed
   outbreak.
2. Give each disease and variant its own transmissibility, at least the quanta
   emission rate.
3. Find and document the sources for the DMP state machines.
4. Check how well the 2016 CBG boundaries match recent SafeGraph months.
5. Decide whether hospitalized and deceased people should be removed from the
   movement schedule.
6. Make visitor choice depend on age and distance, not just who's at home.
7. Calibrate external infection for zones with low catchment.

If you change one of these, update this page in the same PR, and say in the PR
description whether results from earlier runs are still comparable.
