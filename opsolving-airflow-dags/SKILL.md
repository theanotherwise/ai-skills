---
name: opsolving-airflow-dags
description: Create and refactor Airflow 3 DAG repositories using explicit instances, concrete processes, and reusable libraries. Use when adding DAGs, reorganizing Airflow code, separating orchestration from providers or crawlers, or introducing adjacent and group-shared runtime configuration. Do not use for deploying the Airflow platform or generating tests.
---

# Build layered Airflow DAG repositories

## Goal

Keep Airflow definitions small and make the rest of the code usable without importing Airflow. Organize repositories into three top-level layers:

```text
instances/   Airflow definitions and runtime config
processes/   concrete use cases invoked by tasks
libs/        reusable technical libraries
```

The dependency direction is `instances -> processes -> libs`. An instance may also use `libs.config` directly. Libraries must never depend on processes or instances, and processes must never depend on instances.

## Required workflow

1. Read the governing repository instructions and inspect the existing DAG loading, import path, config, and deployment assumptions.
2. Identify the target Airflow instance, DAG ID, concrete process, external providers, crawler adapters, schedule semantics, retry behavior, and expected output.
3. Read [references/architecture.md](references/architecture.md) before creating a repository layout, moving existing code, sharing a process across DAGs, or adding crawlers or providers.
4. Read [references/dag-template.md](references/dag-template.md) before writing or substantially changing `dag.py`.
5. Keep each move scoped: preserve working behavior, credentials, config precedence, stable DAG IDs, schedules, and task IDs unless the user requests a behavioral change.
6. Validate syntax, file discovery, config paths, and Airflow imports without generating tests or starting an Airflow service.

## Layer contract

Place each DAG in `instances/<instance-name>/<dag-id>/` with exactly the files it needs. A normal DAG owns `dag.py` and adjacent `config.yaml`. `dag.py` is the only layer allowed to import Airflow and must explicitly own decorators, task topology, schedule, start date, catchup, concurrency, retries, timeouts, tags, and instance gating.

Place each concrete use case in `processes/<process-name>/`. A process loads runtime config during task execution, orchestrates libraries, applies business rules, logs a useful summary, and returns a bounded serializable result for XCom. It must not import Airflow.

Place reusable technical behavior in `libs/`: config loading, provider clients, crawler infrastructure, source adapters, transport, authentication, shared technical models, and security primitives. A provider handles one external service contract. A crawler acquires and parses source data. Neither decides task topology, scheduling, business completion, alert timing, or XCom contents.

## DAG definitions

Keep `dag.py` readable without opening another file. Do not hide `@dag`, `@task`, metadata, or task dependencies behind a shared DAG factory. Resolve only the adjacent config path during module import; do not read YAML, authenticate, call APIs, discover inventory, or execute processing while Airflow parses the file.

Import the process inside the task body:

```python
@task(...)
def execute():
    from processes.example import run

    return run(config_path=CONFIG_PATH, instance=TARGET_INSTANCE)
```

Use Airflow data intervals for scheduled windows. Do not derive a scheduled processing interval from wall-clock time. Keep retries and execution timeout explicit on tasks, and keep DAG-level concurrency explicit when overlapping runs would be unsafe.

## Runtime configuration

Resolve the local file with `CONFIG_PATH = adjacent_config(__file__)` and pass that single path to the process. Keep DAG metadata in `dag.py`, not YAML.

When a family of DAGs in one Airflow instance shares runtime values, place the base file at `instances/<instance-name>/_<group-name>/config.yaml`. A DAG-local config may declare `extends: ../_<group-name>/config.yaml`; it must not inherit another DAG's config. Resolve inherited paths relative to the declaring file, reject cycles, merge mappings recursively, and let the more specific file replace scalar and list values.

Do not invent global config directories or implicit group discovery. Preserve the target repository's secret-management rules and never log resolved secrets or credentials.

## Processes, providers, and crawlers

A process may be used by one DAG or a family of DAGs. Create optional modules such as `reporting.py`, `schedule.py`, `models.py`, or `transformations.py` only when they own real behavior. Export the process entry point from `processes/<process-name>/__init__.py`.

Organize providers as `libs/providers/<provider-name>/`. Organize crawler mechanics under `libs/crawlers/core/` and concrete crawler adapters under `libs/crawlers/<namespace>/<crawler-name>/`. A namespace describes a coherent source or technical family and must not be hardcoded to one current use case. Do not create a global crawler registry; a process that selects crawlers dynamically owns its registry.

## Existing repositories

Do not reorganize an existing repository merely because this skill was selected. Apply the layered layout when the user asks to create, adopt, or refactor toward it. During migration, update import paths, Airflow ignore rules, runtime `PYTHONPATH`, git-sync or packaged DAG roots, documentation, and existing path-based checks together. Avoid a partial layout where old DAG factories or duplicate source trees remain active.

## Validation

Do not create unit or integration tests as part of this skill. Preserve existing tests and update path references only when a requested refactor would otherwise leave them broken.

Run the narrowest available non-mutating checks, normally Python compilation and `git diff --check`. When an existing Airflow runtime is available and the user requested runtime verification, confirm DAG listing, file locations, and import errors. Do not start Airflow, deploy, trigger DAGs, install dependencies, or mutate external systems without explicit authorization.
