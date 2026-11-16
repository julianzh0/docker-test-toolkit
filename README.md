<p align="center">
  <img src=".github/logo.svg" alt="docker-test-toolkit" width="240">
</p>

<h1 align="center">docker-test-toolkit</h1>

<p align="center"><strong>A waiting line for Docker-backed tests: many projects and coding agents share one machine, nothing collides, and nothing is left behind.</strong></p>

<p align="center">
  <a href="https://github.com/juzhxng/docker-test-toolkit/releases/latest"><img src="https://img.shields.io/github/v/release/juzhxng/docker-test-toolkit" alt="Release"></a>
  <a href="https://github.com/juzhxng/docker-test-toolkit/actions/workflows/ci.yml"><img src="https://github.com/juzhxng/docker-test-toolkit/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="https://www.python.org/downloads/"><img src="https://img.shields.io/badge/python-3.12+-blue.svg" alt="Python 3.12+"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-blue.svg" alt="License: Apache-2.0"></a>
</p>

Picture three coding agents on one laptop, each starting its project's integration tests at the same moment. Each
suite starts a database, a cache and a search engine. Together they run out of memory, a test run is killed halfway,
and its containers keep running. Someone types `docker system prune` and takes the other projects' volumes with it.

docker-test-toolkit prevents that. A test run tells it how much memory and CPU it needs. If the machine has room, the
run starts at once; if not, it waits its turn. When the run ends, the toolkit removes exactly the containers,
networks, volumes and images that run created, and nothing else. If a run is killed, the next one cleans up after it.

**Purpose and non-goals.** docker-test-toolkit is a queue for Docker-backed test runs on one machine: runs go one
after another, or side by side when their declared memory fits, and later arrivals wait. Every Docker resource a run
creates is recorded and removed exactly, a killed run is cleaned up by the next one, and a guard keeps agents from
changing Docker around it. It is not a scheduler (it does not place, prioritise or pre-empt work), it does not police
live processes (a run's host processes are neither measured nor limited), and it is not a container runtime, a CI
system or a cluster tool.

## What it does

- **Waits for room.** A run says what it needs. It starts when the machine has that much memory, CPU and disk free,
  and runs that fit together run side by side.
- **Cleans up exactly.** Every Docker resource a run creates is written down before it exists. At the end, only those
  are removed. It never prunes.
- **Recovers.** A run killed at any moment is cleaned up by the next run, or by `cleanup project --execute`.
- **Protects the machine.** If memory runs critically short or the machine starts swapping hard, the toolkit ends the
  run that holds it, instead of letting everything stall.
- **Explains a wait.** When a run waits or is refused, its record says why, with the numbers that decided it, such as
  the memory it needed and the memory free.
- **Keeps agents in line.** A hook for Claude Code, Codex and other agents refuses direct Docker changes, so
  everything goes through the toolkit.

Your project keeps its own compose files, services, readiness checks, migrations and test selection. The toolkit
never interprets them.

## Highlights

- **One file** // a single `.pyz` with nothing but Python's standard library, installed once per machine
- **No project copy** // projects hold no copy, and a new release never asks a project to change
- **Same everywhere** // the same toolkit on your laptop, on GitHub-hosted runners and on your own CI server
- **Exact** // removes only what it recorded, so other projects on the same Docker are never touched

## Set it up with your agent

Paste this into your coding agent, in the project you want to connect:

```text
Set up docker-test-toolkit for this project, on this machine, in my CI, or both. Ask me which if I
did not say. Read and follow
https://raw.githubusercontent.com/juzhxng/docker-test-toolkit/main/docs/agent-setup.md
Ask me before anything that changes my machine, my CI or my harness configuration.
```

The agent installs the toolkit where you choose, connects your tests and proves it with one real run.
[Read the steps](docs/agent-setup.md) first if you like. To do it by hand, read on.

## Requirements

- macOS or Linux, with CPython 3.12 or newer
- The Docker CLI with the Compose and Buildx plugins, and one of these:
  - on macOS: [Docker Desktop](https://docs.docker.com/desktop/), [Colima](https://github.com/abiosoft/colima) or a
    [Podman](https://podman.io) machine
  - on Linux: Docker Engine

Other runtimes, OrbStack among them, are refused. Setup notes for each runtime are in the
[operations guide](docs/operations.md#host-telemetry-limits).

## Install

Download `docker-test-toolkit.pyz` from the [latest release](https://github.com/juzhxng/docker-test-toolkit/releases/latest),
check its SHA-256 against the release notes, then:

```sh
python3.12 -I docker-test-toolkit.pyz install --json
docker-test-toolkit readiness inspect --json
```

Use any CPython 3.12 or newer outside a virtual environment. On Linux the toolkit lives in `/opt/docker-test-toolkit`
and belongs to the user who runs the tests, so create that folder first and link the command afterwards:

```sh
sudo install -d -o "$(id -un)" -g "$(id -gn)" -m 755 /opt/docker-test-toolkit
python3 -I docker-test-toolkit.pyz install --json
sudo ln -sf /opt/docker-test-toolkit/bin/docker-test-toolkit /usr/local/bin/docker-test-toolkit
```

## Where it runs

- **Your machine:** installed once. Every project and every agent session on the machine shares it, so their test runs
  wait for each other instead of colliding.
- **GitHub-hosted runners:** each job starts on a fresh machine and installs the toolkit first.
- **Your own CI server:** install it once. Runners that share one Docker daemon share the installation, so their jobs
  wait for each other too, and for runs on the same machine outside CI.

The [CI guide](docs/ci.md) has the steps for each.

## Use it in your tests

**Python projects** copy [`docs/loader/docker_test_toolkit_loader.py`](docs/loader/docker_test_toolkit_loader.py) into
their tree once, then open a run around their tests:

```python
import subprocess
from pathlib import Path

from docker_test_toolkit_loader import load_toolkit

toolkit, _archive = load_toolkit()
GIB = 1024**3

spec = toolkit.RunSpec(
    project_root=Path.cwd(),
    entrypoint="integration",
    purpose="integration-tests",
    services=("redis",),
    closure=("redis",),
    profile="default",
    phases={
        "tests": toolkit.PhaseBudget(container_bytes=2 * GIB, host_process_bytes=1 * GIB, daemon_cpus=2, host_threads=2)
    },
    initial_phase="tests",
)

run = toolkit.open_run(spec).enter()  # waits here until the machine has room
outcome = "failed"
try:
    docker = run.docker_context  # run your compose commands and tests through this

    def run_docker(*arguments: str) -> None:
        subprocess.run([str(docker.executable), *arguments], env=docker.environment, check=True)

    run_docker("pull", "redis@sha256:<digest>")
    run_docker("compose", "up", "--detach", "--wait", "--wait-timeout", "60")
    ...  # your tests
    outcome = "passed"
finally:
    run.leave(outcome)  # removes exactly what the run created
```

Your `compose.yaml` uses images the run pulled by digest or built. To run side by side with other runs, give every
service a hard memory limit that fits the budget:

```yaml
services:
  redis:
    image: redis@sha256:<digest>
    mem_limit: 256m
    cpus: 0.5
```

**Other languages** talk to `docker-test-toolkit bridge lease` over NDJSON on standard input and output.

The [adopter guide](docs/adopters.md) covers both, plus test batches, sharing a run and CI.

## Documentation

- [Agent setup](docs/agent-setup.md): the steps a coding agent follows to connect a project
- [CI guide](docs/ci.md): hosted and self-hosted runners
- [Adopter guide](docs/adopters.md): connecting a project, and what it can rely on
- [Operations guide](docs/operations.md): installing, cleanup, recovery and troubleshooting
- [Compatibility](docs/compatibility.md): what stays the same between releases

## Development

You also need [`uv`](https://docs.astral.sh/uv/) and `gitleaks` 8.30.1.

```sh
uv sync
uv run poe qa
```

The release archive is built from committed files, and the same commit always gives the same bytes:

```sh
uv run python scripts/build_zipapp.py build --repository . --output-directory dist
```

## License

Copyright 2026 juzhxng. Licensed under [Apache-2.0](LICENSE).
