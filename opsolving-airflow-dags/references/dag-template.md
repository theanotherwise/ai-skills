# Minimal Airflow DAG pattern

Replace angle-bracket placeholders with repository-specific values. Keep the Airflow definition local even when many DAGs invoke the same process.

## DAG entrypoint

```python
"""Airflow DAG for <purpose>."""

import os
from datetime import timedelta

import pendulum
from airflow.sdk import dag, task

from libs.config import adjacent_config


CONFIG_PATH = adjacent_config(__file__)
TARGET_INSTANCE = "<instance-name>"


if os.environ.get("AIRFLOW_INSTANCE_NAME") == TARGET_INSTANCE:

    @dag(
        dag_id="<dag-id>",
        description="<description>",
        schedule="<schedule>",
        start_date=pendulum.datetime(<year>, <month>, <day>, tz="UTC"),
        catchup=False,
        max_active_runs=1,
        tags=["<tag>", TARGET_INSTANCE],
    )
    def workflow():
        @task(
            task_id="<task-id>",
            retries=<retry-count>,
            retry_delay=timedelta(minutes=<retry-delay-minutes>),
            execution_timeout=timedelta(minutes=<timeout-minutes>),
        )
        def execute():
            from processes.<process-name> import run

            return run(
                config_path=CONFIG_PATH,
                instance=TARGET_INSTANCE,
            )

        execute()

    workflow_dag = workflow()
```

Do not copy placeholder values mechanically. Choose schedule, catchup, concurrency, retries, timeout, timezone, tags, and task topology from the requested behavior and repository policy. Omit the instance gate only when the repository is guaranteed to load in one Airflow installation.

## Scheduled data interval

For a task that processes the preceding scheduled interval, derive boundaries from Airflow context:

```python
from airflow.sdk import get_current_context


context = get_current_context()
interval_start = context["data_interval_start"]
interval_end = context["data_interval_end"]

if interval_start is None or interval_end is None:
    raise ValueError("Airflow data interval is required")
```

Pass both values to the process. If the timetable produces point intervals but the use case requires a fixed preceding window, derive that window explicitly from `data_interval_end`; do not use wall-clock time.

## Process entrypoint

```python
from __future__ import annotations

from pathlib import Path
from typing import Any

from libs.config import load_config


def run(
    *,
    config_path: str | Path,
    instance: str,
) -> dict[str, Any]:
    config = load_config(config_path)

    # Invoke crawlers and providers, apply process rules, and publish results.

    return {
        "instance": instance,
        "processed_count": 0,
    }
```

Export it from `processes/<process-name>/__init__.py`:

```python
from processes.<process-name>.handler import run


__all__ = ["run"]
```

## Group configuration

```text
instances/<instance-name>/
|-- _<group-name>/
|   `-- config.yaml
`-- <dag-id>/
    |-- dag.py
    `-- config.yaml
```

The DAG-local file references its group base:

```yaml
extends: ../_<group-name>/config.yaml

source:
  name: "<source-name>"
```

The process still receives only the adjacent `CONFIG_PATH`. `libs.config.load_config` resolves inheritance during task execution; `dag.py` does not read or merge YAML while Airflow parses it.
