(managed-worker-environments)=

# Isolate plugin dependencies in managed worker environments

A napari plugin can provide a user interface in the napari process while running functions that need additional packages in separate processes.
Managed plugin environments give each plugin its own dependency set without changing the environment that runs napari.

## The host and worker model

A plugin that uses managed environments has two parts:

- The **main plugin package** is the Python distribution installed alongside napari.
  Its **host code** runs in the napari process.
  It owns widgets, reads viewer state, calls napari and Qt APIs, and updates the user interface.
- **Worker code** runs in a separate process created from a napari-managed environment.
  It imports packages not supplied by napari and performs computation on ordinary Python values and NumPy arrays.

The plugin manifest connects the two parts.
It declares the environment recipe and associates a command identifier with a qualified Python target such as `napari_example_worker:segment`.

Napari owns provisioning, worker startup, command dispatch, cancellation, failures, reuse, and shutdown.
Plugin code uses napari's public API and does not import the execution backend.

```{important}
Plugin code that uses napari or Qt APIs must remain in the host process.
Worker modules must not import napari GUI APIs or manipulate a viewer, layer, or widget.
Pass layer data into a worker as a NumPy array and apply the returned value to the viewer in host code.
```

## Follow the dependency rule

Code in the napari process may rely only on Python's standard library, the plugin's own modules, napari, and packages in napari's direct base requirements for the current platform.
Here, **direct base requirements** means the packages listed by napari's own runtime package metadata for that platform, without napari's optional extras; a transitive package or another package that happens to be installed does not qualify.
If a function needs any other runtime package, that function and its import belong in worker code and the package belongs in the worker environment declaration.
This is a placement rule, not a performance recommendation: it applies to every additional package, even one that is small, pure Python, or unlikely to conflict today.

The main plugin package must still declare the allowed packages that its host code imports directly.
For example, host code that imports napari, NumPy, and qtpy declares all three, even though installing napari also supplies NumPy and qtpy.
This keeps standard Python package metadata accurate while limiting host requirements to packages that napari already needs.
Each declared version requirement must accept the exact version already installed with napari.

For a napari-managed installation to enforce this contract, it must inspect the selected wheel's runtime requirements before changing the environment.
It must reject an active requirement unless its package is napari itself or a direct base requirement of napari for the current platform and the installed version satisfies the plugin's constraint.
It must then install that same inspected wheel without dependency resolution.
Checking package names alone is insufficient: two plugins could both require NumPy while placing incompatible constraints on its version.
Active direct URL requirements and dependency extras not already supplied by napari must also be rejected.
A managed installation must install only the main package and must not request one of its optional extras.

With that validation, accepting a plugin does not add, upgrade, or downgrade packages in napari's environment.
The error should identify the rejected requirement and direct the author to move it, and the code that imports it, to a managed environment.
Installations performed directly with `pip`, Conda, or another external tool remain outside that guarantee because napari does not control their dependency resolution.

## Create an embedded worker distribution

The worker target must be importable inside the managed environment.
Ship a small Python distribution for the worker inside the main plugin distribution.

This is the supported project layout for an embedded, single-module worker:

```text
napari-example/
├── pyproject.toml
└── src/
    └── napari_example/
        ├── __init__.py
        ├── _widget.py
        ├── napari.yaml
        └── worker/
            ├── pyproject.toml
            └── napari_example_worker.py
```

The inner `worker` directory is not another plugin and does not need an `__init__.py`, README, manifest, entry point, or separate release.
It is a small buildable Python distribution embedded in the one plugin wheel.
Napari finds it through `local_packages` in the manifest, and the environment backend installs it into the isolated environment without installing anything into the napari environment.

This layout uses a flat module because a small worker rarely needs another `src` directory.
If the worker grows into several modules, replace `napari_example_worker.py` with a package and configure the inner build accordingly.

### Include the worker in the main plugin wheel

The `pyproject.toml` at the repository root describes the main plugin package that users install.
Its relevant sections can look like this:

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "napari-example"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "napari",
    "numpy",
    "qtpy",
]

[project.entry-points."napari.manifest"]
napari-example = "napari_example:napari.yaml"

[tool.setuptools]
include-package-data = true

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-data]
napari_example = [
    "napari.yaml",
    "worker/pyproject.toml",
    "worker/*.py",
]
```

The example host code imports napari, NumPy, and qtpy, so it declares all three as direct dependencies.
They are permitted host dependencies because they are also direct base requirements of napari.
Do not add the example segmentation library, or any other package not supplied by napari, to this `dependencies` list.
Declare every such package in an environment in `napari.yaml`.

An external `pip install` resolves the main package's metadata normally, which keeps the distribution usable outside napari's installer.
The napari-managed installation design requires validation of the selected wheel before installation, as described in the dependency rule above.

This example deliberately leaves the napari requirement unbounded while the managed-environment API is under coordinated pre-release development.
Before publishing a plugin that uses this API, replace `napari` with a lower bound on the first released napari version that provides it.
During pre-release development, install the coordinated napari and `npe2` development checkouts explicitly rather than guessing a future release number in published metadata.

The `package-data` entry is essential.
It places the inner `pyproject.toml` and worker source in the built plugin wheel so that napari can prepare the environment from the installed plugin.

### Define the inner `pyproject.toml`

Create `src/napari_example/worker/pyproject.toml`:

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "napari-example-worker"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = []

[tool.setuptools]
py-modules = ["napari_example_worker"]
```

Each line has a narrow purpose:

- `[build-system]` starts the standard table that tells an installer how to build the embedded distribution.
- `requires = ["setuptools>=68"]` selects setuptools as its build requirement.
  This is a build tool, not a worker runtime dependency.
- `build-backend = "setuptools.build_meta"` selects setuptools' standard PEP 517 build backend.
- `[project]` starts the distribution metadata.
- `name = "napari-example-worker"` gives the embedded distribution its own package-manager name.
  It does not create a second napari plugin because it declares no `napari.manifest` entry point.
- `version = "0.1.0"` supplies the version required to build the distribution.
  Keeping it aligned with the main plugin version is simple, but napari uses the main plugin version and the declared environment recipe when deciding whether to rebuild.
- `requires-python = ">=3.10"` documents which Python versions can import the worker source.
  It must be compatible with the environment's `python` constraint.
- `dependencies = []` is intentionally empty.
  The napari manifest is the single authoritative list of worker runtime dependencies.
  Napari rejects an embedded worker project that declares runtime dependencies here, including dynamically declared dependencies.
- `[tool.setuptools]` starts the setuptools-specific configuration.
- `py-modules = ["napari_example_worker"]` tells setuptools to install the adjacent `napari_example_worker.py` file as an importable top-level module.

Napari does not install this inner project while installing the host plugin.
It reads the installed plugin manifest, resolves `worker` relative to that manifest, validates the inner project, and passes that local project to the managed environment backend during provisioning.

The initial backend requires Wetlands 2.2 or later, but Wetlands is an internal napari implementation dependency rather than part of the plugin API.
Plugin manifests and Python code use napari's environment and task abstractions and must not import Wetlands.

## Write a worker target

Create `src/napari_example/worker/napari_example_worker.py`:

```python
from __future__ import annotations

from typing import Protocol

import numpy as np


class WorkerContext(Protocol):
    @property
    def cancel_requested(self) -> bool: ...

    def update(
        self,
        message: str,
        *,
        current: int | None = None,
        maximum: int | None = None,
    ) -> None: ...


def segment(
    image: np.ndarray,
    threshold: float,
    *,
    napari_context: WorkerContext,
) -> np.ndarray | None:
    napari_context.update("Loading the segmentation library", current=0, maximum=2)
    if napari_context.cancel_requested:
        return None

    import example_segmentation

    napari_context.update("Segmenting image", current=1, maximum=2)
    labels = example_segmentation.segment(image, threshold=threshold)
    if napari_context.cancel_requested:
        return None

    napari_context.update("Returning labels", current=2, maximum=2)
    return np.asarray(labels)
```

Keep expensive or dependency-sensitive imports inside the worker module.
Keeping the import near the function that uses it also makes it clear that the dependency is unavailable to host code.

The optional context protocol is defined locally for typing only.
The worker does not import napari or the environment backend.
When the command declares `accepts_worker_context: true`, napari supplies the reserved keyword-only `napari_context` argument.

Call `napari_context.update()` to report execution progress.
Check `napari_context.cancel_requested` at useful interruption points and return promptly when it becomes true.
Cancellation cannot safely interrupt every third-party function, so divide long work into cancellable steps when the underlying library permits it.

Worker targets must be qualified import targets, not lambdas, closures, bound methods, or serialized callable objects.
Module-level functions make the code importable and its execution contract reviewable.

## Declare the environment and command

Add an environment and a worker command to `napari.yaml`:

```yaml
schema_version: 0.3.0
name: napari-example
display_name: Example
contributions:
  environments:
    - id: napari-example.segmentation
      display_name: Segmentation runtime
      provision: on_install
      python: "3.10.*"
      conda:
        - numpy
      pypi:
        - example-segmentation==2.4.0
      channels:
        - conda-forge
      local_packages:
        - path: worker
  commands:
    - id: napari-example.segment
      title: Segment image
      python_name: napari_example_worker:segment
      environment: napari-example.segmentation
      accepts_worker_context: true
    - id: napari-example.make_widget
      title: Make example widget
      python_name: napari_example._widget:ExampleWidget
  widgets:
    - command: napari-example.make_widget
      display_name: Example segmentation
```

Environment and command identifiers must start with the plugin's manifest name followed by a dot.
Each environment identifier must be unique within the manifest.
A worker command can reference only an environment owned and declared by the same plugin.

The environment fields mean:

- `display_name` is the name shown to users in environment-management interfaces.
- `provision` controls when a managed installation prepares the persistent environment.
  Use `on_install` for functionality that is central to the plugin and should be ready after installation.
  Use `on_demand` for large or optional functionality that users might never invoke.
- `python` constrains the Python interpreter in the isolated environment.
- `conda` lists Conda requirements.
- `pypi` lists PEP 508 Python requirements.
- `channels` lists Conda channels in resolution order and must contain at least one channel.
- `local_packages` identifies embedded worker distributions relative to the manifest.
- `lockfile` can identify an optional Pixi lockfile relative to the manifest when reproducible resolution requires one.

Include a declared lockfile in the main plugin package's data just like the worker files.

Dependencies may be split between `conda` and `pypi`, but do not declare the same distribution in both lists.
Pin dependencies as tightly as the worker needs and no tighter.
Two plugins can declare incompatible versions of the same library because napari provisions their environments separately.

The command's `python_name` identifies the callable that the worker process imports.
The `environment` field changes the command from an in-process command into a managed worker command.
The `accepts_worker_context` field opts the callable into progress and cooperative cancellation.

Worker commands cannot directly implement widget, reader, writer, or sample-data contributions.
Those contribution types use napari objects or protocols in the main process.
Create a host contribution that calls the worker command instead.

## Invoke the worker from host code

Widget code imports napari's backend-neutral API and submits the declared command:

```python
from __future__ import annotations

import numpy as np
from qtpy.QtWidgets import QLabel, QProgressBar, QPushButton, QVBoxLayout, QWidget

from napari.plugins import (
    PluginTaskState,
    PluginWorkerError,
    execute_worker_command,
)


class ExampleWidget(QWidget):
    def __init__(self, napari_viewer):
        super().__init__()
        self.viewer = napari_viewer
        self.task = None

        self.run_button = QPushButton("Segment")
        self.run_button.clicked.connect(self.run)
        self.cancel_button = QPushButton("Cancel")
        self.cancel_button.clicked.connect(self.cancel)
        self.cancel_button.setEnabled(False)
        self.status = QLabel("Ready")
        self.progress = QProgressBar()

        layout = QVBoxLayout(self)
        layout.addWidget(self.run_button)
        layout.addWidget(self.cancel_button)
        layout.addWidget(self.progress)
        layout.addWidget(self.status)

    def run(self):
        layer = self.viewer.layers.selection.active
        if layer is None:
            self.status.setText("Select an image layer.")
            return

        self.run_button.setEnabled(False)
        self.cancel_button.setEnabled(True)
        self.progress.setRange(0, 0)
        self.task = execute_worker_command(
            "napari-example.segment",
            np.asarray(layer.data),
            0.5,
        )
        self.task.add_progress_callback(self.on_progress)
        self.task.add_done_callback(self.on_done)

    def cancel(self):
        if self.task is not None:
            self.task.cancel()

    def on_progress(self, update):
        self.status.setText(update.message)
        if update.current is None or update.total is None:
            self.progress.setRange(0, 0)
        else:
            self.progress.setRange(0, update.total)
            self.progress.setValue(update.current)

    def on_done(self, task):
        try:
            if task.state is PluginTaskState.COMPLETED:
                labels = task.result()
                self.viewer.add_labels(np.asarray(labels), name="Segmentation")
            elif task.state is PluginTaskState.CANCELED:
                self.status.setText("Segmentation canceled.")
            else:
                self.show_failure(task.error)
        finally:
            self.task = None
            self.run_button.setEnabled(True)
            self.cancel_button.setEnabled(False)

    def show_failure(self, error):
        if isinstance(error, PluginWorkerError) and error.failure is not None:
            failure = error.failure
            self.status.setText(
                f"{failure.remote_exception_type or 'Worker error'}: {failure.message}"
            )
        else:
            self.status.setText(str(error or "Unknown worker failure"))
```

`execute_worker_command()` returns immediately with a `PluginTask`.
Do not call `task.result()` while work is running on the GUI thread because it blocks until the operation finishes.
Use `add_progress_callback()` and `add_done_callback()` in widgets, or `await task` from async code.

The callback methods are safe for fast, cached operations.
When napari's Qt application is running, it dispatches task callbacks on the main GUI thread so the callbacks can update widgets and layers.

`PluginTask.state` moves through `PENDING`, `RUNNING`, and one terminal state: `COMPLETED`, `FAILED`, or `CANCELED`.
`PluginTask.phase` distinguishes `PREPARING`, `PROVISIONING`, `STARTING`, `EXECUTING`, and `CLEANING_UP`.
Each `PluginTaskProgress` contains the phase, a message, and optional `current` and `total` values.

`task.cancel()` requests cancellation during preparation or execution.
It returns `False` when the task has already finished.
Calling `task.result()` on a canceled task raises `PluginTaskCanceledError`.

Provisioning failures use `PluginEnvironmentProvisioningError`.
An unavailable backend uses `PluginEnvironmentUnavailableError`.
Remote execution failures use `PluginWorkerError`, whose optional `failure` value contains structured details such as the remote exception type, message, traceback, worker process, exit code, signal, timeout, or serialization context when available.
Present a concise failure in the plugin widget and retain enough detail for users to diagnose or report the problem.
Napari also presents managed-task failures through its notification system.

### Use napari's lifecycle and diagnostics interfaces

The `PluginTask` returned by `execute_worker_command()` represents the complete request, including on-demand preparation, worker startup, execution, and final cleanup.
Use one compact status label, progress bar, and cancel control beside the plugin action that created the task.
Update those controls from the task's callbacks instead of creating a separate progress item for each lifecycle phase.

Do not add a private scrolling provisioning log to each plugin widget.
The Plugin Manager's Managed Environments dialog provides one resizable, scrollable operation log shared across the selected plugin's environments.
It can filter records by environment, focus an environment through **Show log**, copy or clear displayed text, and show structured failure details.
The dialog consumes napari's bounded in-memory operation history, so it can replay recent on-demand preparation after being opened or reopened later in the same napari session.
The history is session-only and is cleared when napari exits.

The plugin widget should still show compact current status and progress for the entire request, a cancel control, and its result or concise failure.
This keeps the operation the user initiated next to its command without making every plugin implement a second diagnostics viewer or environment-management interface.

## Values that can cross the process boundary

Worker arguments and results can contain:

- `None`;
- booleans, integers, floats, strings, and bytes;
- lists and tuples containing supported values;
- dictionaries with supported scalar keys and supported values;
- NumPy arrays without object dtype or dtype metadata and within the transport's dimensional and size limits.

Pass arrays directly as `numpy.ndarray` objects.
Do not hide napari `Layer`, `Viewer`, Qt objects, generators, arbitrary class instances, or callables inside a container.
Extract the required layer data and metadata in host code, pass supported values, and reconstruct or update napari objects after the task completes.

For example, a worker can return a nested result:

```python
return {
    "labels": labels,
    "scores": scores,
    "model": {"name": model_name, "threshold": threshold},
}
```

The host can then call `viewer.add_labels(result["labels"])`.

## Choose and manage the installation lifecycle

`on_install` and `on_demand` affect preparation, not worker startup.
Workers always start lazily when a command first needs them and are reused while napari keeps the pool alive.

A managed plugin installation prepares `on_install` environments after the host package is installed or updated.
The plugin manager shows each declared environment and lets users install it ahead of first use, rebuild it, cancel provisioning, stop workers, or remove persistent files.
Napari can enforce this lifecycle only when the plugin is installed through a flow that supports managed environments.
A direct `pip install` installs the host package but does not itself trigger napari's plugin-manager lifecycle.
Executing a worker command still prepares a missing or stale environment before starting the worker.

On-demand preparation works whether or not the Plugin Manager is open.
The plugin widget that invokes a worker receives lifecycle and execution updates through the same `PluginTask`, even when the Plugin Manager is closed.
Opening Managed Environments later in the same session replays the recent bounded operation history.
If an operation is still running, opening or reopening Managed Environments reconnects its row to the napari-owned task, restores current progress and cancellation controls, and disables conflicting environment actions until the task finishes.

Prepared environments persist across napari sessions.
Napari fingerprints the main plugin version, normalized environment recipe, lockfile contents, backend version, and recipe ABI.
It reuses an environment when that identity is unchanged and marks it stale when the declaration changes.
Preparing a stale environment creates the new generation and retires the previous generation after the replacement succeeds.

Release a new main plugin version whenever embedded worker code changes.
An editable source change made without changing the main plugin version does not by itself mark a prepared environment stale, so rebuild that environment explicitly during development.

Failed, interrupted, or canceled provisioning is cleaned up.
A later preparation performs a clean build instead of resuming a partial installation.

Stopping workers releases process and transport resources without deleting the prepared environment.
Removing an environment first stops its workers and then deletes the persistent environment.
Napari stops owned workers and closes transport resources during application shutdown.

Host code normally needs only `execute_worker_command()`.
Installation and management interfaces can use the other public lifecycle functions:

```python
from napari.plugins import (
    list_plugin_environments,
    prepare_plugin_environment,
    remove_plugin_environments,
    stop_plugin_workers,
)

environments = list_plugin_environments("napari-example")
prepare_task = prepare_plugin_environment("napari-example.segmentation")
stop_task = stop_plugin_workers(
    "napari-example",
    "napari-example.segmentation",
)
remove_task = remove_plugin_environments(
    "napari-example",
    ["napari-example.segmentation"],
)
```

Preparation, stop, and removal operations return observable, cancellable `PluginTask` objects.
`list_plugin_environments()` is synchronous and returns a tuple of `PluginEnvironmentInfo` snapshots.

## Test the boundary

Test host and worker code separately before adding an end-to-end environment test.

Host-side tests should verify that:

- importing the main plugin package and constructing its widgets does not import packages declared only in worker environments;
- the widget submits the expected command identifier and supported argument values;
- progress updates change the plugin's status and progress controls;
- completion adds or updates the expected layer on the GUI thread;
- cancellation and structured failures produce useful user-facing state.

Worker-side tests can import the worker module directly in a development environment that contains its declared dependencies.
Test plain functions with NumPy arrays and a small fake context:

```python
class Context:
    cancel_requested = False

    def __init__(self):
        self.updates = []

    def update(self, message, *, current=None, maximum=None):
        self.updates.append((message, current, maximum))
```

At least one integration test should prepare the real environment and execute the qualified target.
Cover array and nested-value round trips, progress, cancellation, useful remote failures, and reuse.
An isolation test can declare incompatible versions of the same dependency in two test plugins and verify that each worker imports its own version while the napari environment remains unchanged.

## Inspect the built plugin

Always test the artifact that users will install.
Build the main plugin wheel and inspect its contents:

```sh
python -m build
python -m zipfile --list dist/napari_example-0.1.0-py3-none-any.whl
```

Confirm that the wheel contains:

```text
napari_example/napari.yaml
napari_example/worker/pyproject.toml
napari_example/worker/napari_example_worker.py
```

Inspect the main wheel metadata and confirm that every `Requires-Dist` entry follows the dependency rule above.
Inspect the inner `pyproject.toml` in the wheel and confirm that `dependencies` remains empty.
Validate the built artifact in an environment containing one supported napari version:

```sh
npe2 validate --host-dependencies dist/napari_example-0.1.0-py3-none-any.whl
```

`npe2 validate` reads the wheel's authoritative `Requires-Dist` metadata.
It rejects active requirements for packages outside napari's direct base requirements, constraints that exclude the installed version, active direct URL requirements, and dependency extras that napari does not already provide.
Validate at least the oldest and newest supported napari versions across the supported Python and platform matrix, plus any known dependency-boundary versions, because requirements and environment markers can differ between installations.
The managed installer must repeat the check against the user's exact environment.

Validating a source `napari.yaml` checks the manifest and also checks a matching static `[project].dependencies` table when it can find one.
If package metadata is unavailable, the command reports that dependency validation was skipped; the wheel check is still required before release.

Install the wheel into a clean napari environment and prepare its managed environment through the plugin manager.

An editable source installation is useful during development, but a successful source-tree test does not prove that package data is present in a release wheel.

## Understand the security boundary

Managed environments provide dependency and process isolation.
They are not sandboxes.

Worker code runs with the same operating-system user permissions as napari and can access the user's files, network, and other resources allowed to that account.
Only install plugins you trust, review qualified worker targets as ordinary executable plugin code, and do not describe managed environments as a security boundary for untrusted code.
