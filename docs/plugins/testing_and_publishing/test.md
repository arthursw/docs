(plugin-test)=

# Testing guidelines

(plugin-testing-tips)=

## Tips for testing napari plugins

Testing is a big topic! If you are completely new to writing tests in Python,
consider reading this post on [Getting Started With Testing in
Python](https://realpython.com/python-testing/)

We recommend using
[pytest](https://docs.pytest.org/en/6.2.x/getting-started.html) for testing your
plugin. Aim for [100% test coverage](best-practices-test-coverage)!

## Test plugins with managed environments

A plugin with managed environments has a main package whose host code runs in napari and an embedded worker distribution whose functions run in a separate, plugin-specific environment.
Host code may use only napari and packages in napari's direct base requirements for the current platform; every other runtime dependency belongs in the environment declared for worker code.

Test the host and worker sides independently before adding a smaller number of end-to-end tests.

Your host test suite should verify that:

- Importing and discovering the plugin succeeds without installing worker-only dependencies.
- Widget and napari or Qt integration code runs in the napari process.
- The widget submits the expected qualified worker command and handles progress, completion, cancellation, and structured failure.
- NumPy arrays are converted to and from layer data at the host boundary.

Your worker test suite should import the embedded worker module without napari or Qt and call each target as an ordinary function.
Provide a small fake `napari_context` when testing progress and cooperative cancellation.

Add packaging tests that build the main plugin wheel and source distribution, then inspect them for:

- `napari.yaml`.
- `worker/pyproject.toml`.
- Every worker module and required worker resource.
- Every third-party package imported directly by host code appears in `Requires-Dist`, and every active `Requires-Dist` entry is permitted by napari's host dependency contract.
- No worker-only package in the main distribution's `Requires-Dist` metadata.
- An empty static `dependencies` list in the embedded worker project.

Build the wheel and run `npe2 validate` against that artifact in an environment containing a supported napari version:

```sh
python -m build
npe2 validate dist/napari_example-0.1.0-py3-none-any.whl
```

This checks path containment, environment references, worker command declarations, and the wheel's final `Requires-Dist` metadata.
It rejects active requirements that napari does not directly require, constraints that exclude installed host versions, direct URLs, and dependency extras that napari does not already supply.
Validate at least the oldest and newest supported napari versions across the supported Python and platform matrix, plus any known dependency-boundary versions.
The managed installer must repeat the check against the user's exact environment.

Validating a raw manifest checks its schema and any matching static `[project].dependencies` that `npe2` can find, but it cannot replace final wheel validation.

At least one integration test should use napari's real environment backend when practical.
Exercise provisioning, recipe reuse, a recipe change, array and nested-value transport, cancellation, a remote failure, and worker shutdown.
Use two small fixture plugins with incompatible versions of the same dependency to prove that neither changes napari's environment and that the plugins do not affect each other.

Do not provision large scientific frameworks in every unit-test job.
Keep unit tests fast and select a supported CI job for the real environment lifecycle.

### The `make_napari_viewer_proxy` fixture

Testing a napari `Viewer` requires some setup and teardown each time. We have
created a [pytest fixture](https://docs.pytest.org/en/6.2.x/fixture.html) called
`make_napari_viewer_proxy` that you can use (this requires that you have napari
installed in your environment).

To use a fixture in pytest, you simply include the name of the fixture in the
test parameters (oddly enough, you don't need to import it!). For example, to
create a napari viewer for testing:

```py
def test_something_with_a_viewer(make_napari_viewer_proxy):
    viewer = make_napari_viewer_proxy()
    ...  # carry on with your test
```

If you embed the viewer in your own application and need to access private attributes,
you can use the `make_napari_viewer` fixture.

(plugin-testing-prefer-unit-test)=

### Prefer smaller unit tests when possible

The most common issue people run into when designing tests for napari plugins is
that they try to test everything as a full "integration test", starting from the
napari event or action that would trigger their plugin to do something. For
example, let's say you have a dock widget that connects a mouse callback to the
viewer:

```py
class MyWidget:
    def __init__(self, viewer: 'napari.Viewer'):
        self._viewer = viewer

        @viewer.mouse_move_callbacks.append
        def _on_mouse_move(viewer, event):
            if 'Shift' in event.modifiers:
                ...
```

You might think that you need to somehow simulate a mouse movement in napari in
order to test this, but you don't! Just *trust* that napari will call this
function with a `Viewer` and an `Event` when a mouse move has been made, and
otherwise leave `napari` out of it.

Instead, focus on "unit testing" your code: just call the function directly with
objects that emulate, or "mock" the objects that your function expects to
receive from napari. You may also need to slightly reorganize your code. Let's
modify the above widget to make it easier to test:

```py
class MyWidget:
    def __init__(self, viewer: 'napari.Viewer'):
        self._viewer = viewer
        # connecting to a method rather than a local function
        # makes it easier to test
        viewer.mouse_move_callbacks.append(self._on_mouse_move)

    def _on_mouse_move(self, viewer, event):
        if 'Shift' in event.modifiers:
            ...
```

To test this, we can often just instantiate the widget with our own viewer, and
then call the methods directly. As for the `event` object, notice that all we
care about in this plugin is that it has a `modifiers` attribute that may or may
not contain the string `"Shift"`. So let's just fake it!

```py
class FakeEvent:
    modifiers = {'Shift'}

def test_mouse_callback(make_napari_viewer):
    viewer = make_napari_viewer()
    wdg = MyWidget(viewer)
    wdg._on_mouse_move(viewer, FakeEvent())
    # assert that what you expect to happen actually happened!
```
