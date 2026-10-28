# Data Engineering Stack

A ready-to-run Apache Airflow environment for developing and testing DAGs before they go to production. It is used to run the metrics ETL of the udata-based open data portal: tracking events are aggregated from MongoDB, written to PostgreSQL, exposed through an API and fed back into udata.

It is based on [etalab/data-engineering-stack](https://github.com/etalab/data-engineering-stack). The executor is `LocalExecutor`, so the scheduler, webserver and worker run in the same container.

## Getting started

Requirements: Docker and Docker Compose.

```bash
# Create the directories Airflow needs
./1_prepareDirs.sh

# Create the .env file, then edit it
./2_prepare_env.sh
nano .env
# Set POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, AIRFLOW_ADMIN_MAIL,
# AIRFLOW_ADMIN_FIRSTNAME, AIRFLOW_ADMIN_NAME and AIRFLOW_ADMIN_PASSWORD

# On macOS with Apple silicon
# export DOCKER_DEFAULT_PLATFORM="linux/amd64"

# Start the services
docker compose up --build -d
```

Once the containers are up, sign in to the Airflow web UI with the admin credentials from `.env`. The published port is defined in `docker-compose.yml` (the configuration guide uses http://localhost:18080).

New DAGs can take a moment to appear. Force a refresh with:

```bash
./refreshBagDags.sh
```

## The `metrics_etl` DAG

`dags/metrics_etl.py` builds the portal metrics in five tasks:

```text
extract_tracking_events -> send_to_metrics_db -> refresh_materialized_views -> update_udata_metrics -> save_to_mongodb
```

| Task | What it does |
|---|---|
| `extract_tracking_events` | Aggregates daily views and downloads from the udata `tracking_events` collection in MongoDB |
| `send_to_metrics_db` | Upserts daily visits per dataset and downloads per resource into the `metric` schema in PostgreSQL |
| `refresh_materialized_views` | Refreshes the materialized views served by PostgREST |
| `update_udata_metrics` | Reads the totals from the metrics API and writes them back to the udata MongoDB documents |
| `save_to_mongodb` | Stores a log of each run |

The DAG is scheduled every minute for testing. Change the `schedule` before using it in production (for example `@hourly` or `@daily`).

## Connections and variables

Airflow connections (PostgreSQL, MongoDB and HTTP APIs) can be created in the UI, with the Airflow CLI, or with the `createConn.py` scripts that live with each project. `docs/connections.json` and `docs/variables.json` list the entries in use. Add your SSH key to the `ssh/` folder if the container needs to reach remote hosts.

Keep real hosts, passwords and tokens out of the repository: they belong in `.env` and in Airflow connections.

## Repository layout

| Path | Purpose |
|---|---|
| `dags/` | Airflow DAGs (`metrics_etl.py`) |
| `docs/` | Step-by-step configuration guide (in Portuguese) plus connection and variable exports |
| `scripts/` | Container entrypoint scripts |
| `docker-compose.yml`, `Dockerfile` | Container definitions |
| `airflow.cfg` | Airflow configuration |
| `requirements.txt` | Extra Python dependencies |
| `1_prepareDirs.sh`, `2_prepare_env.sh`, `refreshBagDags.sh`, `getDagExamples.sh` | Helper scripts |
