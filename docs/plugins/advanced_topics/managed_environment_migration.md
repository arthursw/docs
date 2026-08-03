(managed-environment-migration)=

# Migrate a plugin to managed worker environments

This guide explains how to migrate an existing `npe2` plugin when it requires packages outside napari's direct base requirements for the current platform.

Managed environments solve dependency conflicts between napari and plugins, and between different plugins.
They do not make untrusted code safe to run.

## Understand the target structure

After migration, the plugin has two execution locations:

- The **main plugin package** remains installed with napari.
  Its host code creates widgets, uses napari and Qt APIs, reads layer data, starts work, and displays results.
- **Worker code** runs in a separate process created from a plugin-specific managed environment.
  It imports packages not supplied by napari and exchanges ordinary supported values and NumPy arrays with the host code.

The main plugin package may rely only on Python's standard library, its own modules, napari, and packages in napari's direct base requirements for the current platform.
Every other runtime package belongs in a declared managed environment, regardless of its size, import time, or perceived compatibility.
The main package still declares every allowed dependency that host code imports directly, and each version requirement must accept the version installed with napari.

## Decide what code moves

Keep code in the main plugin package when it:

- creates or updates widgets;
- accesses a napari viewer, layer, event, notification, or command API;
- uses Qt;
- uses only packages supplied by napari.

Move code to a managed worker environment when it:

- imports any runtime package not supplied by napari;
- can accept ordinary supported Python values and NumPy arrays;
- can return ordinary supported Python values and NumPy arrays;
- does not need napari or Qt objects.

An isolated environment is a process and dependency boundary, not a security sandbox.
Worker code runs with the user's operating-system permissions and can access the same files and network resources as other code run by that user.

## 1. Inventory the current package

Classify every import and dependency before changing the package.

| Dependency kind | Examples | Where it belongs after migration |
| --- | --- | --- |
| Host | `napari`, Python's standard library, and direct base requirements of napari such as NumPy and qtpy | Main plugin package; declare each directly imported package in `[project].dependencies` |
| Worker | Every runtime package not supplied by napari, including TensorFlow, PyTorch, Cellpose, StarDist, or a small helper library | The environment recipe in `napari.yaml` |
| Development | `pytest`, `pytest-qt`, linters, documentation tools | Optional development or test dependency groups |

NumPy may be used on both sides of the boundary.
List it in the environment recipe when worker dependencies require a particular version.
Host code can use the NumPy version installed with napari, but any declared NumPy constraint must accept that exact version.

Search for imports at module scope as well as imports inside functions.
A host widget module must not import a package declared only in a worker environment, even lazily during widget construction.

For example, a plugin might begin with this main package metadata:

```toml
[project]
name = "napari-segmenter"
dependencies = [
    "napari",
    "qtpy",
    "numpy",
    "cellpose==3.1.0",
    "tensorflow==2.16.1",
    "stardist==0.9.2",
]
```

After migration, the main package declares only the dependencies imported by host code and already required by napari:

```toml
[project]
name = "napari-segmenter"
requires-python = ">=3.11"
dependencies = [
    "napari",
    "numpy",
    "qtpy",
]
```

Do not retain a package not supplied by napari in the main package "just in case."
For managed installation to enforce this contract, the installer must validate the selected plugin wheel before changing the environment and install the accepted artifact without dependency resolution.
Direct `pip` or Conda installations remain outside this guarantee.

While the managed-environment API is under coordinated pre-release development, install compatible napari and `npe2` development checkouts explicitly.
Before publishing the migrated plugin, give `napari` a lower bound on the first released napari version that provides the API; do not guess that release number in advance.

## 2. Separate host and worker code

The host module owns napari integration and the worker module owns computation.

Before migration, a widget often mixes both:

```python
from qtpy.QtWidgets import QPushButton, QWidget
from cellpose import models


class SegmenterWidget(QWidget):
    def run(self) -> None:
        image = self.viewer.layers.selection.active.data
        model = models.Cellpose(model_type="cyto3")
        labels, *_ = model.eval(image)
        self.viewer.add_labels(labels)
```

After migration, the widget remains in the main plugin package and sends only data and parameters:

```python
from typing import Any

import numpy as np
from qtpy.QtWidgets import QPushButton, QWidget


class SegmenterWidget(QWidget):
    def run(self) -> None:
        from napari.plugins import execute_worker_command

        image = np.asarray(self.viewer.layers.selection.active.data)
        self._task = execute_worker_command(
            "napari-segmenter.segment",
            image,
            {"model_type": "cyto3"},
        )
        self._task.add_progress_callback(self._on_progress)
        self._task.add_done_callback(self._on_done)
```

The worker accepts the array and parameters without importing napari or Qt:

```python
from typing import Any

import numpy as np


def segment(
    image: np.ndarray,
    parameters: dict[str, Any],
) -> np.ndarray:
    from cellpose import models

    model = models.Cellpose(model_type=parameters["model_type"])
    labels, *_ = model.eval(image)
    return np.asarray(labels)
```

Moving an additional-package import inside the host function is not sufficient.
The function itself must run as a worker command in the managed environment.

Do not pass a viewer, layer, widget, Qt object, generator, open file, or arbitrary callable to a worker.
Pass the layer's array data and ordinary parameters, then apply the returned result to the viewer in host code.

## 3. Add an embedded worker project

The preferred layout adds a minimal worker project inside the main plugin package:

```text
napari-segmenter/
├── pyproject.toml
└── src/
    └── napari_segmenter/
        ├── __init__.py
        ├── _widget.py
        ├── napari.yaml
        └── worker/
            ├── pyproject.toml
            └── napari_segmenter_worker.py
```

The inner project needs only two files for a single-module worker.
It does not need an `__init__.py`, README, or separate source tree.
It is shipped inside the main plugin wheel and is not published as a second distribution.

Create `src/napari_segmenter/worker/pyproject.toml`:

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "napari-segmenter-worker"
version = "1.0.0"
requires-python = ">=3.10"
dependencies = []

[tool.setuptools]
py-modules = ["napari_segmenter_worker"]
```

The empty `dependencies` list is intentional.
The `napari.yaml` environment recipe is the single authoritative declaration of worker dependencies.
Duplicating them in the inner project can produce a second, inconsistent resolution path.

The inner project metadata tells the managed-environment backend how to make `napari_segmenter_worker` importable in the worker environment.
Napari still discovers the plugin from the main package's `napari.manifest` entry point and never treats the inner project as another plugin.

## 4. Include worker source in the main wheel

Add the manifest and embedded worker files to the main package's package data:

```toml
[tool.setuptools]
include-package-data = true

[tool.setuptools.package-data]
napari_segmenter = [
    "napari.yaml",
    "worker/pyproject.toml",
    "worker/*.py",
]
```

This step is required even though the main package does not import the worker module.
The qualified worker target identifies which callable to import, but it does not transfer the callable's source code.

When napari prepares the environment, it resolves `worker` relative to the installed `napari.yaml` and asks the backend to install that embedded project into the isolated environment.
The backend installs the worker project without using it as the dependency authority because the environment recipe already declares those dependencies.

This design has two useful properties:

- the plugin author publishes one main plugin wheel;
- the worker code prepared later is the code shipped with that installed plugin version.

## 5. Declare the environment and worker command

Move worker dependencies into `contributions.environments` and associate the command with that environment:

```yaml
schema_version: 0.3.0
name: napari-segmenter
display_name: Segmenter
contributions:
  environments:
    - id: napari-segmenter.cellpose
      display_name: Cellpose
      provision: on_install
      python: "3.10.*"
      conda:
        - numpy
        - cellpose==3.1.0
      channels:
        - conda-forge
      local_packages:
        - path: worker

  commands:
    - id: napari-segmenter.segment
      title: Segment with Cellpose
      python_name: napari_segmenter_worker:segment
      environment: napari-segmenter.cellpose
```

Environment and command identifiers must begin with the plugin name.
The command's `python_name` is a qualified Python target in the form `module:callable`.
The `local_packages.path` is a safe path relative to the directory containing `napari.yaml`.

Declare Conda requirements in `conda` and Python package requirements in `pypi`.
Declare channels explicitly when their order matters.
Use an optional Pixi lockfile when the plugin needs a reviewed, reproducible resolution.

Do not reference a managed worker command directly from a widget, reader, writer, or sample-data contribution.
Those contribution types execute in the napari process and may use napari-specific values.
Contribute a host command for the widget, then let its code invoke the worker command.

### Choose when to provision

Use:

```yaml
provision: on_install
```

when most users need the environment and the managed plugin installer should prepare it during plugin installation or update.
Provisioning does not start worker processes.
Workers start lazily on first execution and stop during napari shutdown.

Use:

```yaml
provision: on_demand
```

when the environment is optional, unusually large, or used by only some plugin features.
The plugin UI must make first-use preparation visible through its progress and cancellation controls.

The Plugins window shows declared environments and lets users prepare, rebuild, cancel, stop, or remove them.
An installation performed outside napari's managed plugin flow can install the main plugin package, but napari cannot promise installation-time provisioning for that external flow.
The environment remains available for explicit or on-demand preparation.

## 6. Add progress and cooperative cancellation

Set `accepts_worker_context` on commands that report progress or respond cooperatively to cancellation:

```yaml
commands:
  - id: napari-segmenter.segment
    title: Segment with Cellpose
    python_name: napari_segmenter_worker:segment
    environment: napari-segmenter.cellpose
    accepts_worker_context: true
```

Define the small context protocol in the worker module.
The worker must not import napari or the execution backend:

```python
from typing import Protocol


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
```

Accept the reserved keyword-only argument and check for cancellation between meaningful units of work:

```python
def segment(
    image: np.ndarray,
    parameters: dict[str, object],
    *,
    napari_context: WorkerContext | None = None,
) -> np.ndarray | None:
    if napari_context is not None:
        napari_context.update("Loading model", current=0, maximum=2)
        if napari_context.cancel_requested:
            return None

    model = load_model(parameters)

    if napari_context is not None:
        napari_context.update("Segmenting image", current=1, maximum=2)
        if napari_context.cancel_requested:
            return None

    labels = model.predict(np.asarray(image))

    if napari_context is not None:
        napari_context.update("Returning labels", current=2, maximum=2)
    return np.asarray(labels)
```

Calling `PluginTask.cancel()` requests cancellation of provisioning or execution.
Worker context checks allow long-running Python code to stop at safe boundaries.
Code blocked inside a third-party call may not respond until that call returns or the backend terminates its worker.

## 7. Present task state in the plugin UI

`execute_worker_command()` returns a `PluginTask`.
Use callbacks in Qt widget code so the GUI remains responsive:

```python
from napari.plugins import PluginTaskState, execute_worker_command
from napari.utils.notifications import show_error


def run(self) -> None:
    self.run_button.setEnabled(False)
    self.cancel_button.setEnabled(True)
    self.progress_bar.setRange(0, 0)

    self._task = execute_worker_command(
        "napari-segmenter.segment",
        np.asarray(self.viewer.layers.selection.active.data),
        {"model_type": self.model_type.currentText()},
    )
    self._task.add_progress_callback(self._on_progress)
    self._task.add_done_callback(self._on_done)


def cancel(self) -> None:
    if self._task is not None:
        self._task.cancel()


def _on_progress(self, progress) -> None:
    self.status_label.setText(progress.message)
    if progress.current is None or progress.total is None:
        self.progress_bar.setRange(0, 0)
    else:
        self.progress_bar.setRange(0, progress.total)
        self.progress_bar.setValue(progress.current)


def _on_done(self, task) -> None:
    self.run_button.setEnabled(True)
    self.cancel_button.setEnabled(False)

    if task.state is PluginTaskState.COMPLETED:
        self.viewer.add_labels(task.result(), name="Segmentation")
    elif task.state is PluginTaskState.CANCELED:
        self.status_label.setText("Canceled")
    else:
        self.status_label.setText("Segmentation failed")
        show_error(str(task.error))
```

Keep progress near the action that initiated it.
For an on-demand environment, the same task reports preparation, provisioning, worker startup, and execution phases, so a widget can present the whole first-use operation in one place.

Do not call `task.result()` from the Qt main thread before the task is done.
Use `add_done_callback`, or await the task from an async integration.

`PluginEnvironmentProvisioningError` represents environment preparation failures.
`PluginWorkerError` represents remote worker failures and may contain structured details such as the remote exception type, traceback, process exit information, and serialization context.
Show a concise message in the widget or a napari notification, and preserve detailed diagnostics in a copyable log or error view.

## 8. Validate the migration

### Validate the source metadata

Install a version of `npe2` that supports managed environments, then validate the plugin:

```sh
npe2 validate --host-dependencies src/napari_segmenter/napari.yaml
```

Validation should reject:

- environment IDs outside the plugin namespace;
- commands that reference a missing or foreign environment;
- a worker command without a qualified `python_name`;
- `accepts_worker_context` on an in-process command;
- unsafe embedded-package paths;
- worker commands used directly as widgets, readers, writers, or sample data.
- active requirements for packages other than napari or its direct base requirements in the validation environment;
- host version constraints that exclude the installed version;
- active direct URL requirements and dependency extras not already provided by napari.

When `npe2` finds a matching `pyproject.toml`, it checks its static `[project].dependencies` as an early authoring check.
Missing package metadata produces a warning; dynamic dependencies require validation of the built wheel.

### Inspect the built wheel

Build the distribution and inspect its contents:

```sh
python -m build
npe2 validate --host-dependencies dist/napari_segmenter-1.0.0-py3-none-any.whl
python -m zipfile -l dist/napari_segmenter-1.0.0-py3-none-any.whl
```

The wheel check uses final `Requires-Dist` metadata and is required even when source validation passed.
Validate at least the oldest and newest supported napari versions across the supported Python and platform matrix, plus any known dependency-boundary versions.
The managed installer must repeat the check against the user's exact environment.

Confirm that the wheel contains:

```text
napari_segmenter/napari.yaml
napari_segmenter/worker/pyproject.toml
napari_segmenter/worker/napari_segmenter_worker.py
```

Install the built wheel into a clean test environment.
Testing only an editable checkout can hide missing package-data declarations.

### Test the host/worker boundary

Add tests that prove:

- importing the main plugin package and constructing its widget does not import packages declared only in worker environments;
- widget callbacks run in the napari process;
- the qualified target runs in the declared worker environment;
- arrays, scalars, strings, bytes, and nested supported containers round-trip correctly;
- object-dtype arrays and unsupported values fail with useful errors;
- progress reaches the widget;
- cancellation works during provisioning and execution;
- remote exceptions appear as `PluginWorkerError` with useful diagnostics;
- the environment is reused when its recipe is unchanged;
- changing a dependency, Python constraint, channel, lockfile, or main plugin version produces a new recipe and a clean rebuild;
- changing embedded worker source is accompanied by a new main plugin version for releases, or an explicit environment rebuild during editable development;
- two test plugins can use incompatible versions of the same dependency without changing the napari environment or each other;
- stopping workers does not remove the persistent environment;
- napari shutdown closes workers and transport resources;
- ordinary plugin discovery and host contributions still work.

Use small test packages for isolation and lifecycle tests.
Reserve real frameworks and model downloads for explicit integration tests.

## 9. Plan installation and release compatibility

Managed worker declarations require compatible versions of both `npe2` and napari.
Publish the migrated plugin only after the schema and napari runtime versions it targets are available.

Choose one of these compatibility strategies:

1. Release a new plugin version that requires the first compatible napari version and remove every main-package dependency not supplied by napari.
2. Keep the previous plugin release available for older napari versions while documenting the version boundary.
3. Use a maintenance branch for the older in-process implementation if the supported user base requires it.

Do not silently expose the same command as both an in-process command with additional requirements and a managed worker based on runtime version.
That makes dependency installation and execution location difficult for users to predict.

When updating an existing environment recipe, describe the rebuild and expected download size in the release notes.
Napari reuses a persistent environment while its declared recipe and main plugin version are unchanged.
Release a new main plugin version when embedded worker source changes; during editable development, rebuild the environment explicitly.
Interrupted, failed, or canceled provisioning is cleaned up so the next attempt performs a clean rebuild rather than resuming a partial installation.

If a release must be rolled back:

1. stop the plugin's active workers;
2. remove the affected managed environments from the Plugins window;
3. install the previous compatible plugin release;
4. prepare that release's environment again.

Do not instruct users to delete napari's application-data directories manually.
Use napari's environment lifecycle controls so ownership and cleanup remain consistent.

## Migration checklist

- [ ] Host, worker, and development dependencies are inventoried.
- [ ] Napari and Qt code remains in the main plugin package.
- [ ] Every import not supplied by napari occurs only in worker functions.
- [ ] Host and worker exchange only supported ordinary values and NumPy arrays.
- [ ] The embedded worker project has a minimal `pyproject.toml` with no dependencies.
- [ ] Worker files are included as main-package data.
- [ ] Worker dependencies are declared once in `napari.yaml`.
- [ ] Every worker command has a qualified target and an environment.
- [ ] Provisioning policy is chosen deliberately.
- [ ] The widget presents progress, cancellation, and useful failures.
- [ ] The built wheel, not only an editable checkout, passes integration tests.
- [ ] Tests cover reuse, rebuild, isolation, cancellation, failures, and shutdown.
- [ ] Release notes state the napari and `npe2` compatibility boundary.
- [ ] Documentation states that managed environments isolate dependencies but are not sandboxes.
