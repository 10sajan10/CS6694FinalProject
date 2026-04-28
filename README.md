# CS6994 Final Project: Logan River Forecasting

This repository contains the database, feature-building, training, and live
inference workflow for Logan River Observatory forecasting. The final forecasting
targets are:

- `discharge` for stream sites `3` and `4`.
- `air_temperature` for climate sites `1` and `2`.

Each model uses a `168` hour lookback window and predicts the next `24` hourly
values. Live inference uses only the season-specific models: one model per
target variable, site, and season.

## Quick Start — Clone and Run the Demo

Follow these steps to clone the repository and run the live inference demo
from scratch.

### 1. Clone the repository

```bash
git clone https://github.com/10sajan10/CS6994FinalProject.git
cd CS6994FinalProject
```

### 2. Set up the Python environment

```bash
bash temporal_transformer/setup_env.sh
```

This creates `.venv/` with all required packages.

### 3. Start the PostgreSQL database

```bash
docker compose up -d --build
```

Wait ~20 seconds for the container to finish loading the data dump. Verify:

```bash
docker exec postgres psql -U admin -d database -c "\dt"
```

### 4. Start the inference listener (Terminal 1)

Installs the database trigger and prediction table on first run, then waits
for new sensor data:

```bash
.venv/bin/python inference.py --mode listen --install-db-objects
```

You should see:

```
Database inference tables and hourly insert trigger are installed.
Loaded 16 season-specific model bundle(s)
Listening for PostgreSQL notifications on 'datastream_hourly_insert'.
```

Leave this terminal running.

### 5. Run the demo notebook (Terminal 2 / Jupyter)

Open `demo2.ipynb` and run all cells:

```bash
.venv/bin/jupyter notebook demo2.ipynb
```

The notebook will:
1. Re-insert the latest datastream row for Site 1 AirTemp, Site 3 Discharge, and Site 4 Discharge.
2. The PostgreSQL trigger notifies the inference listener automatically.
3. The listener writes 24-step predictions to `model_predictions`.
4. The notebook polls until all predictions arrive, then plots 72-hour history plus the 24-hour forecast.
5. A clean-up cell removes the test rows and their predictions.

---

## End-To-End Workflow

1. **Start PostgreSQL with Docker.**
   The Docker image creates the normalized schema and loads the bundled database
   dump from `init-scripts/`.

2. **Store observations in `datastream`.**
   Environmental observations are normalized into `site`, `variable`, `unit`,
   `method`, `owner`, `qualifier`, `processing_level`, and `datastream`.
   `datastream` is the main time-series table.

3. **Build training parquet files.**
   `build_streamflow_training_parquet_v2_multihorizon.ipynb` converts database
   observations into model-ready rows with lag features and future targets.

4. **Train temporal transformers.**
   The training scripts save model bundles containing the model weights, lag
   columns, target columns, scaler statistics, model hyperparameters, and
   feature metadata needed by inference.

5. **Run live inference.**
   `inference.py` watches new hourly rows in `datastream`, prepares the same
   shape of feature row used during training, selects the correct
   season-specific model, and writes future predictions to `model_predictions`.

6. **Demo the process.**
   `demo2.ipynb` triggers the live inference listener by re-inserting the latest
   sensor rows, then polls and plots the predictions. `demo.ipynb` is a
   self-contained alternative that calls the inference functions directly.

## Database Setup

Start the database:

```bash
docker compose up -d --build
```

Connection settings:

| Parameter | Value |
| --- | --- |
| Host | `localhost` |
| Port | `5433` |
| Database | `database` |
| User | `admin` |
| Password | `password` |

The schema is defined in `init-scripts/01-create-schema.sql`.

Main normalized tables:

- `owner`
- `site`
- `unit`
- `method`
- `variable`
- `processing_level`
- `qualifier`
- `datastream`
- `staging`

The ER diagram is saved as:

- `database_er_diagram.png`

## Training Data

Current parquet files:

- `streamflow_parquet_v2/discharge_training_site_3_lb168_air24avg_precip24avg_h24.parquet`
- `streamflow_parquet_v2/discharge_training_site_4_lb168_air24avg_precip24avg_h24.parquet`
- `air_temperature_parquet_v2/air_temperature_training_site_1_lb168_h24.parquet`
- `air_temperature_parquet_v2/air_temperature_training_site_2_lb168_h24.parquet`

Discharge rows include:

- 168 hourly discharge lags for the target stream site.
- Latest snow depth from site `1`.
- 24-hour average air temperature from site `2`.
- 24-hour average precipitation from site `2`.
- Calendar features.
- 24 hourly discharge targets.

Air-temperature rows include:

- 168 hourly air-temperature lags for the target climate site.
- Calendar features.
- 24 hourly air-temperature targets.

See `parquet.txt` for detailed feature definitions and data-quality rules.

## Training

Set up the Python environment:

```bash
bash temporal_transformer/setup_env.sh
```

Run all-season discharge training:

```bash
bash temporal_transformer/run_discharge_training.sh
```

Run all-season air-temperature training:

```bash
bash temporal_transformer/run_air_temperature_training.sh
```

Run season-specific training:

```bash
bash temporal_transformer/run_season_specific_training.sh
```

The season-specific pipeline trains 16 models:

- 2 target variables: `discharge`, `air_temperature`
- 2 sites per target variable
- 4 seasons: `winter`, `spring`, `summer`, `fall`

Season-specific model folders:

- `season_specific_training/models_discharge_season_specific/site_3/<season>/`
- `season_specific_training/models_discharge_season_specific/site_4/<season>/`
- `season_specific_training/models_air_temperature_season_specific/site_1/<season>/`
- `season_specific_training/models_air_temperature_season_specific/site_2/<season>/`

Each saved `*_best_model.pt` bundle contains everything needed for inference:

- model weights
- target variable
- site id
- season
- lag column order
- target horizon column order
- scaler means and scales
- temporal feature names
- static feature names for discharge models
- model hyperparameters

## Live Inference

The live inference entry point is:

```bash
.venv/bin/python inference.py --mode listen --install-db-objects
```

`inference.py` watches the `datastream` table for new rows, runs the
matching season-specific model, and writes 24 prediction rows per event
to `model_predictions`. Predictions are stored automatically — no extra
flag is needed.

### What Triggers Inference

Inference runs only when a new row satisfies all of these conditions:

1. The row is inserted into `datastream`.
2. The timestamp is an exact hourly timestamp, such as `15:00:00`.
3. The timestamp is newer than the latest existing timestamp for the same
   `site_id + variable_id`.
4. The variable has a target model:
   - `AirTemp`
   - `Discharge`
5. A season-specific model exists for the row's target variable, site, and
   forecast-start season.

Rows such as `15:15:00` are ignored. Historical reloads are ignored because
their timestamps are not newer than the latest existing timestamp for that
site/variable.

### Model Selection

For each accepted row:

1. `inference.py` maps the database variable code to a model target:
   - `AirTemp` -> `air_temperature`
   - `Discharge` -> `discharge`
2. The forecast start time is computed as:
   - `history_end_time + 1 hour`
3. The forecast-start month selects the season:
   - December, January, February -> `winter`
   - March, April, May -> `spring`
   - June, July, August -> `summer`
   - September, October, November -> `fall`
4. The matching season-specific model is loaded by:
   - target variable
   - site id
   - season

Example:

```text
Inserted row:
site_id = 4
variable_code = Discharge
datetime_utc = 2026-03-24 19:00:00

Inference model:
target_variable = discharge
site_id = 4
season = spring
```

### Feature Preparation

The inference row is prepared to match training.

For `air_temperature`:

1. Read the previous 168 hourly air-temperature values for that site.
2. Fill short gaps using the same time interpolation style used in training.
3. Add temporal sequence features for each lag timestamp.
4. Apply the saved lag scaler.
5. Pass the sequence to the season-specific model.

For `discharge`:

1. Read the previous 168 hourly discharge values for the stream site.
2. Build static support features:
   - `snow_depth_latest` from site `1`
   - `air_temp_avg_last_24h` from site `2`
   - `precip_avg_last_24h` from site `2`
3. Add temporal sequence features for each lag timestamp.
4. Apply the saved lag and static scalers.
5. Pass the sequence to the season-specific model.

The model predicts scaled deltas from the latest observed value. `inference.py`
inverse-transforms those deltas and reconstructs absolute future predictions.

### Prediction Output

Predictions are stored in the database table `model_predictions`.
This table is created automatically on first run via `--install-db-objects`.

Important columns:

- `source_datastream_id`
- `target_variable`
- `site_id`
- `season`
- `history_end_time`
- `forecast_start_time`
- `horizon`
- `horizon_bin`
- `target_timestamp`
- `prediction`
- `model_path`
- `feature_row_json`

There is one row per horizon. A single accepted insert produces 24 prediction
rows.

## Inference Commands

Install prediction table and trigger, then start listener:

```bash
.venv/bin/python inference.py --mode listen --install-db-objects
```

Listener mode (trigger already installed):

```bash
.venv/bin/python inference.py --mode listen
```

Polling mode (no trigger required):

```bash
.venv/bin/python inference.py --mode poll --poll-interval 5
```

Dry-run a specific row without writing predictions:

```bash
.venv/bin/python inference.py --process-datastream-id 428591 --dry-run
```

Check stored predictions:

```bash
docker exec postgres psql -U admin -d database -c "
SELECT target_variable, site_id, season, horizon, target_timestamp, prediction
FROM model_predictions
ORDER BY target_variable, site_id, horizon
LIMIT 30;
"
```

## Demo Notebooks

### `demo2.ipynb` — live demo (recommended)

Requires `inference.py --mode listen` running in a separate terminal.

1. Finds the latest datastream row for Site 1 AirTemp, Site 3 Discharge, Site 4 Discharge.
2. Deletes all rows at that exact timestamp (clears duplicates from prior demo runs).
3. Re-inserts the same row — the PostgreSQL `AFTER INSERT` trigger fires `pg_notify`,
   inference.py receives it, runs the season-specific model, and writes 72 prediction
   rows (3 sites × 24 horizons) to `model_predictions` automatically.
4. Polls `model_predictions` every 3 seconds until all predictions arrive.
5. Displays the prediction table and plots 72-hour history + 24-hour forecast.
6. Clean-up cell deletes the test datastream rows and their predictions.

### `demo.ipynb` — self-contained demo

No separate inference.py process is needed. Calls `inference.process_event`
internally, inserts hypothetical next-hour rows, plots predictions, and cleans up.

## Main Files

- `README.md` — project overview and sequential workflow.
- `Dockerfile` — builds the PostgreSQL image.
- `docker-compose.yml` — runs the local PostgreSQL database on port `5433`.
- `.dockerignore` — keeps the Docker build context limited to database init files.
- `init-scripts/01-create-schema.sql` — creates the normalized database schema.
- `init-scripts/02-data-dump.sql.gz` — database dump loaded into the Postgres image.
- `ingest_csv.py` — CSV-to-staging ingestion helper.
- `normalize_data.sql` — moves staging rows into normalized tables.
- `build_streamflow_training_parquet_v2_multihorizon.ipynb` — builds training parquet files.
- `temporal_transformer/train_season_specific_transformers.py` — trains season-specific models.
- `inference.py` — live inference worker and utility functions.
- `demo2.ipynb` — recommended live demo (triggers inference.py, polls predictions, plots).
- `demo.ipynb` — self-contained inference demo (no separate process needed).
- `visualizations.ipynb` — evaluation figures: seasonal bar, horizon lines, RMSE heatmaps.
- `figures/` — saved publication figures (seasonal_bar.png, horizon_lines.png, heatmap.png).
- `temporal_transformer_architecture.png` — model architecture diagram.
- `database_er_diagram.png` — rendered database entity relationship diagram.

## Saved Outputs

Model artifacts:

- `models_discharge/`
- `models_air_temperature/`
- `season_specific_training/models_discharge_season_specific/`
- `season_specific_training/models_air_temperature_season_specific/`

Training logs:

- `lightning_logs_discharge/`
- `lightning_logs_air_temperature/`
- `season_specific_training/lightning_logs_discharge_season_specific/`
- `season_specific_training/lightning_logs_air_temperature_season_specific/`

Validation metrics and predictions:

- `season_specific_training/metrics/`
- `season_specific_training/predictions/`

Live inference output is stored in the database table `model_predictions`.

## Project Structure

```text
CS6994FinalProject/
+-- README.md
+-- Dockerfile
+-- docker-compose.yml
+-- .dockerignore
+-- init-scripts/
+-- ingest_csv.py
+-- normalize_data.sql
+-- inference.py
+-- demo.ipynb
+-- demo2.ipynb
+-- visualizations.ipynb
+-- figures/
+-- temporal_transformer_architecture.png
+-- database_er_diagram.png
+-- build_streamflow_training_parquet_v2_multihorizon.ipynb
+-- temporal_transformer/
+-- streamflow_parquet_v2/
+-- air_temperature_parquet_v2/
+-- models_discharge/
+-- models_air_temperature/
+-- season_specific_training/
```
