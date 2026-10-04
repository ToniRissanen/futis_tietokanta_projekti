# Football Database: Databricks Data Pipeline

A portfolio project that builds a data pipeline in Databricks (PySpark and Delta Lake) on the Kaggle dataset [football-database](https://www.kaggle.com/datasets/technika148/football-database). The goals are to learn the Databricks platform, to clean and analyse the data, and later to build prediction models on top of it.

**Status:** work in progress. The Bronze layer and data profiling are done. The Silver layer is next.

## Dataset

Seven CSV files covering five top European leagues across seven seasons, starting from 2014/15.

| Table | Rows | Columns | Content |
|---|---|---|---|
| `games` | 12,680 | 36 | One row per match: teams, goals, win probabilities, bookmaker odds |
| `teamstats` | 25,360 | 18 | Two rows per match (home and away): xG, shots, corners, cards, result |
| `appearances` | 356,513 | 21 | One row per player per match: goals, assists, xG, minutes, cards |
| `shots` | 324,543 | 13 | One row per shot: shooter, assister, situation, xG, pitch position |
| `players` | 7,659 | 4 | Player names and IDs |
| `teams` | 146 | 4 | Team names and IDs |
| `leagues` | 5 | 5 | League names and IDs |

Row and column counts are from the Bronze tables and include two metadata columns added during ingestion (`_ingested_at`, `_source_file`).

## Architecture

The project follows the medallion architecture:

- **Bronze (done):** each CSV is loaded as-is into its own Delta table (`bronze_<name>`) with minimal transformation. Only ingestion metadata is added.
- **Silver (next):** type casting, `NA` handling, and joins between tables.
- **Gold (planned):** analysis-ready tables, such as team season statistics and xG versus actual goals.

Environment: Databricks Free Edition, serverless compute, Unity Catalog. Raw files are stored in a Volume (`/Volumes/workspace/default/football_database`).

## Data profiling findings so far

Profiling of the Bronze tables showed no SQL NULLs, but missing values are encoded as the text `NA`, which is why some numeric columns were read as strings.

- **Bookmaker odds (21 columns in `games`):** `NA` marks missing odds. Each bookmaker's home, draw and away columns have identical `NA` counts, so a missing quote is missing for the whole match. At most 20 matches out of 12,680 are affected (under 0.2%). Silver: `TRY_CAST(... AS DOUBLE)`.
- **`teamstats.yellowCards`:** one `NA` row (game 4888), a genuine missing value. Silver: `TRY_CAST(... AS INT)`, left as NULL rather than imputed.
- **`shots.assisterID`:** `NA` on 84,344 of 324,543 shots (26%). The share is 100% for penalties and direct free kicks, so `NA` means "not applicable" here, not missing data. Silver: cast to INT and add a `hasAssist` flag.
- **Card counts:** team yellow card totals match between `teamstats` and `appearances` in about 92% of matches. The differences concentrate in matches with a red-only player (96% of the matches with a +2 difference). This supports the hypothesis that `teamstats` counts a second-yellow dismissal as two yellows while `appearances` records only a red. Silver: use `teamstats` for team-level yellow cards, and do not use `appearances.yellowCard` for player discipline statistics without caution.

## Open questions

- `appearances.substituteIn` and `substituteOut` have 77,560 distinct values, which does not fit substitution minutes. Their meaning needs to be investigated.
- Value ranges and impossible values have not yet been checked systematically (goals, probabilities summing to 1, minutes, xG between 0 and 1, pitch coordinates).
- Referential integrity between tables (for example every `playerID` in `appearances` existing in `players`) has not yet been checked.
- About 1.5% of matches without red cards still show a one-card yellow difference between `teamstats` and `appearances`, unexplained.

## Planned work

1. Finish data quality checks (value ranges, referential integrity, `substituteIn`/`substituteOut`).
2. Build the Silver layer with explicit schemas, `NA` handling and joins.
3. Build Gold tables and exploratory analyses (home advantage, xG versus goals, team and player comparisons).
4. Prediction models for match outcome and goals, tracked with MLflow, and possibly a custom xG model from the shots table.
5. Dashboard and write-up of the process.

## Repository structure

To be filled in as the project grows. Currently:

- `notebooks/`: Databricks notebooks (Bronze ingestion and data profiling)

## How to reproduce

1. Download the dataset from Kaggle.
2. Create a Unity Catalog Volume in Databricks and upload the seven CSV files.
3. Set `raw_path` in the Bronze notebook to the Volume path.
4. Run the notebook top to bottom.

## Data source

Dataset: [football-database](https://www.kaggle.com/datasets/technika148/football-database) by technika148 on Kaggle. Check the dataset page for its license and terms before reuse.
