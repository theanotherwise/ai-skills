# Airflow DAG repository architecture

## Canonical layout

```text
repository/
|-- instances/
|   `-- <instance-name>/
|       |-- _<group-name>/
|       |   `-- config.yaml
|       `-- <dag-id>/
|           |-- dag.py
|           `-- config.yaml
|-- processes/
|   `-- <process-name>/
|       |-- __init__.py
|       |-- handler.py
|       |-- reporting.py
|       |-- schedule.py
|       |-- models.py
|       `-- transformations.py
`-- libs/
    |-- config.py
    |-- crawlers/
    |   |-- core/
    |   |   |-- runner.py
    |   |   |-- protocol.py
    |   |   |-- models.py
    |   |   |-- errors.py
    |   |   |-- http.py
    |   |   |-- browser.py
    |   |   `-- security.py
    |   `-- <namespace>/
    |       `-- <crawler-name>/
    |           |-- __init__.py
    |           |-- crawler.py
    |           |-- parser.py
    |           |-- models.py
    |           `-- mapping.py
    `-- providers/
        `-- <provider-name>/
            |-- __init__.py
            |-- client.py
            |-- auth.py
            |-- models.py
            `-- errors.py
```

This tree shows possible files, not required placeholders. A simple process may contain only `__init__.py` and `handler.py`; a simple crawler or provider may contain only `__init__.py` and one implementation file.

## Dependency rules

```text
instances  --->  processes  --->  libs
     |                              ^
     `------------------------------|
```

`instances` owns Airflow-facing code. `processes` owns application orchestration. `libs` owns reusable technical capabilities. A library cannot import a process or instance, and a process cannot import an instance.

Only instance-local `dag.py` files import Airflow. Processes and libraries must remain importable in a plain Python runtime.

## Instances

An instance directory represents one Airflow installation, not its Kubernetes cluster, cloud environment, or business application. Keep the identifier stable if the installation moves between infrastructure environments.

Every DAG directory is named after its stable DAG ID and contains its own complete Airflow definition. Shared factories must not create DAG objects on behalf of many entrypoints. Local entrypoints may import pure schedule lookup functions or constants, provided those imports perform no I/O and do not hide the Airflow definition.

An instance gate normally compares `AIRFLOW_INSTANCE_NAME` with a local `TARGET_INSTANCE`. Use it only when the same repository is loaded by multiple installations. A repository loaded by exactly one installation does not need a synthetic gate unless its deployment contract requires one.

## Processes

A process is a concrete use case such as collection, synchronization, publication, reconciliation, notification, or cleanup. It receives explicit inputs from the task, loads runtime config, invokes crawlers and providers, applies completion and error rules, and returns a compact result.

Process-specific scheduling rosters, transformations, reporting payloads, and domain models belong with the process. A process may expose one `run` function through `__init__.py` while keeping internal modules private.

Do not put processes below `libs/`. The fact that several DAGs reuse a process does not make it a generic library; it remains a shared concrete use case.

## Libraries

Libraries provide technical capabilities reusable across processes. `libs/config.py` resolves and validates runtime configuration. `libs/providers` contains external service clients. `libs/crawlers` contains source acquisition infrastructure and adapters. Other technical packages may be added when several processes genuinely share them.

Provider code owns transport, authentication, pagination, provider response parsing, provider models, bounded timeouts, and provider-specific errors. It does not own business filtering, process completion, alert timing, task topology, or XCom shape.

Crawler code owns source acquisition, source pagination, parsing, completeness signals, stable source identifiers, and neutral crawl results. It does not own target publication, process finalization, DAG scheduling, notifications, or XCom shape.

## Generic crawler organization

Put shared mechanics in `libs/crawlers/core/`: runner protocols, neutral request/result models, HTTP or browser transport, retry primitives, URL safety, bounded downloads, and crawler-level errors.

Put adapters under `libs/crawlers/<namespace>/<crawler-name>/`. The namespace groups a coherent technical or source family without assuming that every future crawler belongs to the current business case. If no useful namespace exists, use a small neutral namespace rather than inventing a misleading domain.

Each crawler exports a single public callable, normally `crawl(request) -> CrawlResult`. Optional parser, mapping, and source model modules appear only when the adapter is large enough to justify them. A dynamically selecting process owns its crawler registry; `libs/crawlers` does not maintain a global registry.

## Configuration scopes

The adjacent DAG config is always the entry point passed to the process. A local file can be self-contained or inherit one group base:

```text
instances/<instance-name>/
|-- _<group-name>/
|   `-- config.yaml
`-- <dag-id>/
    |-- dag.py
    `-- config.yaml
```

```yaml
extends: ../_<group-name>/config.yaml

source:
  option: value
```

Resolve `extends` relative to the child file. Reject missing parents, inheritance cycles, and paths outside the intended repository/config scope. Merge mappings recursively; child scalars and lists replace parent values. Do not use another DAG's config as a base because that makes one independently owned DAG an undeclared dependency of another.

Configuration contains runtime settings, not Airflow metadata. DAG ID, schedule, start date, catchup, retries, timeouts, tags, and task topology remain visible in `dag.py`.

## Migration checklist

When adopting this layout, first inventory every current DAG entrypoint, factory, process, external client, crawler, config file, runtime path, and import. Move definitions into `instances`, orchestration into `processes`, and reusable technical code into `libs`.

Split mixed modules by responsibility rather than moving the monolith under a new name. Remove central DAG factories only after every local `dag.py` owns the equivalent metadata and topology. Preserve stable IDs, schedules, task IDs, config precedence, and credential boundaries unless explicitly changing behavior.

Update the Airflow source link or packaged DAG root so recursive discovery reaches `instances/`. Put the repository root on `PYTHONPATH` so `processes` and `libs` import normally. Ignore `libs/**`, `processes/**`, underscore-prefixed group directories, cache directories, and non-DAG Python files during DAG discovery.
