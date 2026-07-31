(how-to-build-a-plugin)=

# Building a plugin

Plugins allow developers to customize and extend napari.
They can add:

- File format support with [readers] and [writers].
- Custom [widgets] and user interface elements.
- [Sample data][sample_data].
- Color [themes][theme].
- Dependency-heavy computation in a napari-managed environment.

## Choose where your code runs

Every plugin has host code that runs inside the napari process.
The manifest, widgets, menus, readers, writers, sample-data providers, and any code that uses napari or Qt APIs are host code.
Keep this part lightweight because its dependencies share napari's Python environment.

A plugin whose functionality needs packages outside napari's default installation can also declare isolated worker environments.
Napari installs those dependencies separately for each plugin and executes declared worker commands outside the napari process.
Use ordinary supported Python values and NumPy arrays to communicate between the host widget and worker.

Use a host-only plugin when its runtime dependencies are lightweight and compatible with napari.
Use a managed worker when a dependency is large, slow to import, has platform-specific binaries, or could constrain packages used by napari or other plugins.

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

Split a dependency-heavy plugin into lightweight host integration and isolated worker commands.
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

Move heavy dependencies and computation out of an existing plugin without moving its GUI code.
```
````

[readers]: contributions-readers
[sample_data]: contributions-sample-data
[theme]: contributions-themes
[widgets]: contributions-widgets
[writers]: contributions-writers
