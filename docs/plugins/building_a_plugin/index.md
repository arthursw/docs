(how-to-build-a-plugin)=

# Building a plugin

Plugins allow developers to customize and extend napari.
They can add:

- File format support with [readers] and [writers].
- Custom [widgets] and user interface elements.
- [Sample data][sample_data].
- Color [themes][theme].
- Computation that needs packages not supplied by napari, using a napari-managed environment.

## Choose where your code runs

Every plugin has a main package that napari installs into its own Python environment.
Code from this package runs inside the napari process; the documentation calls it **host code**.
The manifest, widgets, menus, readers, writers, sample-data providers, and all code that uses napari or Qt APIs belong there.

Host code may rely only on Python's standard library, the plugin's own modules, napari, and packages in napari's direct base requirements for the current platform.
Any functionality that needs another runtime package must run as a declared **worker command** in a plugin-specific managed environment, regardless of how small or common that package is.
Napari installs those additional dependencies outside its own environment and exchanges ordinary supported Python values and NumPy arrays with the worker process.

Keeping every additional requirement in worker environments prevents those packages from being installed alongside napari.
It is the basis for allowing independently developed plugins to require incompatible versions of the same library without conflicting with one another.

```{important}
Managed environments isolate Python dependencies; they are not security sandboxes.
Worker code runs with the user's operating-system permissions.
```

````{grid} 2
```{grid-item-card} Your first plugin
:link: your-first-plugin
:link-type: ref

Build a small host-only plugin and learn the manifest and contribution model.
```

```{grid-item-card} Isolated worker environments
:link: managed-worker-environments
:link-type: ref

Keep napari integration in the main process and make every additional runtime dependency available through isolated worker commands.
```

```{grid-item-card} Plugin functionality
:link: plugin-contribution-guides
:link-type: ref

Learn how readers, writers, widgets, commands, environments, and other contributions fit together.
```

```{grid-item-card} Best practices
:link: best-practices
:link-type: ref

Keep host imports fast, dependencies compatible, resources managed, and releases testable.
```

```{grid-item-card} Testing and publishing
:link: plugin-test-deploy
:link-type: ref

Test both sides of the process boundary and publish one complete plugin distribution.
```

```{grid-item-card} Migrate an existing plugin
:link: managed-environment-migration
:link-type: ref

Move dependencies not supplied by napari out of an existing plugin without moving its GUI code.
```
````

[readers]: contributions-readers
[sample_data]: contributions-sample-data
[theme]: contributions-themes
[widgets]: contributions-widgets
[writers]: contributions-writers
