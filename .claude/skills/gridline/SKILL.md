---
name: gridline
description: How to query the gridline Formula One database (1950 to round 12 of 2026) for this dashboard - connecting through dashboard/db.py, what each table and view holds, how they join, the seven known data traps and how to handle each, and the query patterns this repo uses. Use it whenever you write or check SQL, a function in dashboard/metrics.py, a reconciliation test, a chart, or a spec number that touches gridline, or whenever someone asks about F1 races, drivers, constructors, points, standings, qualifying, pit stops, or seasons in this project, even if they don't name the database.
---

# gridline

Formula One World Championship, every session of every race weekend from
13 May 1950 to round 12 of 2026 (run 23 Aug 2026). Schema `public`, plus
`meta.about`. Names below were checked against the live database's
`information_schema` on 2026-11-18.

## Connecting

Use the repo's connection; never build your own or put the password anywhere
but `.streamlit/secrets.toml` (gitignored) or Community Cloud's Secrets box.

- In the app: `db.get_engine()`. In tests and scripts: `db.engine_from_secrets()`
  (tests get it as the `engine` fixture). Both carry the TLS and pool settings.
- Run anything that imports `dashboard` from the repo root in the `dashboard`
  conda env.
- If the password is still the placeholder, `db.py` refuses to connect. Ask the
  person to paste it in; don't guess (wrong passwords count toward a lockout
  shared by the whole class).

## What's in it

**Start with the views.** Each is `race_data` for one session type, with the
session prefix dropped from column names (`race_result.points` is
`race_data.race_points`) and no `type` column. Every view and `race_data` share
these columns: `race_id`, `position_display_order`, `position_number`,
`position_text`, `driver_number`, `driver_id`, `constructor_id`,
`engine_manufacturer_id`, `tyre_manufacturer_id`.

| View (one row = one driver in that session) | Extra columns worth knowing |
|---|---|
| `race_result` | `points`, `laps`, `time_millis`, `gap_millis`, `reason_retired`, `grid_position_number`, `positions_gained`, `pit_stops`, `pole_position`, `fastest_lap`, `grand_slam` |
| `sprint_race_result` | `points`, `laps`, `grid_position_number`, `positions_gained` |
| `qualifying_result`, `sprint_qualifying_result` | `time_millis`, `q1_millis`, `q2_millis`, `q3_millis`, `laps` |
| `starting_grid_position`, `sprint_starting_grid_position` | `qualification_position_number`, `grid_penalty_positions` |
| `pit_stop` (one row = one stop) | `stop`, `lap`, `time_millis` |
| `fastest_lap` | `lap`, `time_millis`, `gap_millis` |
| `driver_of_the_day_result` | `percentage` (fan vote share) |
| `free_practice_1..4_result`, `qualifying_1/2_result`, `pre_qualifying_result`, `warming_up_result` | `time_millis`, `gap_millis`, `laps` |

Times come as text (`1:23.456`) and as `_millis` integers: compute with `_millis`.

**Tables:**

| Table | One row is |
|---|---|
| `race_data` | one driver in one session (or one pit stop); `type` says which, e.g. `RACE_RESULT`, `PIT_STOP`. Use only when you need several sessions at once. |
| `race` | one race weekend: `id` (1 to 1,172 in date order), `year`, `round`, `date`, `grand_prix_id`, `circuit_id`, `circuit_layout_id`, `laps`, `distance` (km), `qualifying_format` |
| `season` | one year (`year` only) |
| `grand_prix`, `circuit`, `circuit_layout` | one event name (`short_name`, e.g. "British GP"); one track (`type`, `country_id`, lat/long); one layout of a track |
| `driver`, `constructor`, `engine_manufacturer`, `tyre_manufacturer` | one driver or company, with career `best_*` and `total_*` columns |
| `season_driver`, `season_constructor`, `season_engine_manufacturer`, `season_tyre_manufacturer` | season totals for one driver or company |
| `season_driver_standing`, `season_constructor_standing` | final championship position and `points` for one season; `championship_won` |
| `race_driver_standing`, `race_constructor_standing` | championship position and points *after* one race (running total) |
| `entrant`, `season_entrant*`, `chassis`, `engine` | the team name a car ran under, and which chassis/engine/tyre/driver each entrant used per year |
| `country`, `continent` | lookups |
| `driver_family_relationship`, `constructor_chronology` | family ties; team lineage (e.g. renamed teams) |

IDs are readable slugs (`lewis-hamilton`, `ferrari`, `monza`), except `race.id`
(integer).

## How tables join

```
race_result / any view / race_data / race_*_standing
   .race_id ─────────────► race.id
                             .year ────────► season.year
                             .grand_prix_id ► grand_prix.id
                             .circuit_id ───► circuit.id ── circuit_layout.circuit_id
   .driver_id ───────────► driver.id
   .constructor_id ──────► constructor.id
   .engine_manufacturer_id ► engine_manufacturer.id
   .tyre_manufacturer_id ──► tyre_manufacturer.id

season_* tables join on year + the *_id column.
*.country_id, driver.nationality_country_id, driver.country_of_birth_country_id
   ──► country.id ── country.continent_id ──► continent.id
```

No foreign keys are declared on the views or `race_data`; join on these columns anyway.

## The seven known issues

| # | Trap | Handle it by |
|---|---|---|
| 1 | `race_*_standing.points` is the season total *so far*, so summing across races double-counts. | Points per race: `race_result.points`. Final totals: `season_*_standing.points`, or the standing at the season's last race. |
| 2 | `race` lists all 23 rounds of 2026, but only 12 have results (11 unrun, last result 2026-08-23; `race.date` runs to 2026-12-06). | "Latest" or "season so far" means races that have rows in `race_result`: filter with `EXISTS` (example below). Never use `max(race.date)` as the latest result. |
| 3 | `points` is NULL for non-scorers (8,532 of 27,555 `race_result` rows have points; none are 0). | `COALESCE("points", 0)` before summing or averaging per race; `count("points")` counts scoring results, not races. |
| 4 | About 1,780 `race_result` rows never started: `position_text` in `DNQ`, `DNPQ`, `DNS`, `DNP`, `EX`. `position_number` is NULL for these and for `DNF`, `NC`, `DSQ`. | For starts, exclude those five codes. Don't count rows as starts or races. Classified finishes: `position_number IS NOT NULL`. |
| 5 | Recording starts at different years: practice 1986, pit stops 1994, driver of the day 2016, sprints 2021. `fastest_lap` has about one row per race up to 1982, about 20 to 25 after. | Limit charts on these to the years they exist, and say so on screen. A jump in counts is a recording change, not a sport change. |
| 6 | Points scales differ by era: a win's top haul was 9 or 10 points for decades, 25 from 2010 (26 with a fastest-lap bonus), 50 at the 2014 finale. `total_championship_points` ≠ `total_points` for some, since not every result counted. | Don't compare raw points across eras. Use wins, podiums, finishing position, or points as a share of the season's maximum or leader. |
| 7 | `qualifying_result.time` is filled through 2005 and empty from 2006, when knockout qualifying put times in `q1`, `q2`, `q3`. | For a best qualifying time across eras: `COALESCE("q3_millis", "q2_millis", "q1_millis", "time_millis")`. Note the format change on any chart spanning 2006. |

## Working with the data

- Prefer a view (`race_result`) to `race_data` filtered by `type`: shorter SQL,
  and the column names match what you'll show.
- Double-quote every identifier: `"race"`, `"time"`, `"interval"` and `"date"`
  are column names that clash with SQL words.
- Aggregate in SQL; queries stop at 30 seconds. Pass filter values as `:name`
  parameters, never pasted into SQL text.
- Put each query in `dashboard/metrics.py` as one function per figure, using
  `read_sql`, and load it on screen through a cached wrapper:

```python
# dashboard/metrics.py
def wins_by_driver(engine, year: int) -> pd.DataFrame:
    return read_sql(engine, """
        SELECT d."name" AS driver, count(*) AS wins
        FROM "race_result" rr
        JOIN "race" r   ON r."id" = rr."race_id"
        JOIN "driver" d ON d."id" = rr."driver_id"
        WHERE r."year" = :year AND rr."position_number" = 1
        GROUP BY d."name"
        ORDER BY wins DESC""", {"year": year})

# streamlit_app.py
@st.cache_data(ttl=600)
def load_wins(year):
    return metrics.wins_by_driver(db.get_engine(), year)
```

More patterns:

```sql
-- Races run so far in a season (issue 2)
SELECT r."round", gp."short_name", r."date"
FROM "race" r JOIN "grand_prix" gp ON gp."id" = r."grand_prix_id"
WHERE r."year" = :year
  AND EXISTS (SELECT 1 FROM "race_result" x WHERE x."race_id" = r."id")
ORDER BY r."round";

-- Points and starts per constructor in a season (issues 3 and 4)
SELECT c."name", sum(COALESCE(rr."points", 0)) AS race_points,
       count(*) FILTER (WHERE rr."position_text" NOT IN ('DNQ','DNPQ','DNS','DNP','EX')) AS starts
FROM "race_result" rr
JOIN "race" r ON r."id" = rr."race_id"
JOIN "constructor" c ON c."id" = rr."constructor_id"
WHERE r."year" = :year
GROUP BY c."name" ORDER BY race_points DESC;
-- Sprint points are separate, in sprint_race_result; the final title total is season_*_standing.points.
```
