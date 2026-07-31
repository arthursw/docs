(plugin-deploy)=

# Publish your plugin

## Preparing for release

To help users find your plugin, make sure to use the `Framework :: napari`
[classifier] in your package's core metadata. (If you used the napari plugin
template, this has already been done for you.)

Once your package is listed on [PyPI] (and includes the `Framework :: napari`
[classifier]), it will also be visible on the [napari
hub](https://napari-hub.org/).

To ensure you are providing the best metadata and description for your plugin,
see our comprehensive guide: [](hub-customization). This guide covers:

- Setting package metadata in `pyproject.toml`
- Using npe2 manifest metadata for hub display
- Controlling plugin visibility

## Deployment

When you are ready to share your plugin, [upload the Python package to
PyPI][pypi-upload] after which it will be installable using `python -m pip install <yourpackage>`, or (assuming you added the `Framework :: napari` classifier)
in the builtin plugin installer dialog.

### Publishing a plugin with embedded worker code

Publish one outer plugin distribution.
The embedded worker project is package data inside that wheel and source distribution; it is not a second package that users install or a second project that you must publish.

Before upload, build both artifacts and inspect their contents:

```sh
python -m build
python -m zipfile --list dist/napari_example-1.0.0-py3-none-any.whl
tar --list --file dist/napari_example-1.0.0.tar.gz
```

Verify that `napari.yaml`, the worker's `pyproject.toml`, worker modules, lockfiles, and any declared local resources are present.
Install the built wheel into a clean napari environment and prepare each `on_install` environment through the plugin manager.
Invoke every `on_demand` worker at least once or prepare it manually.

Only host requirements belong in the outer `[project].dependencies`.
Worker requirements belong in `contributions.environments`, and the embedded worker project's dependency list remains empty.
This allows napari's managed installer to provision the manifest recipe without changing the environment running napari.

See [Isolated worker environments](managed-worker-environments) for the required layout and [Migrating an existing plugin](managed-environment-migration) for a release checklist.

If you used the {ref}`napari-plugin-template`, you can also
[setup automated deployments][autodeploy] on GitHub for every tagged commit.

```{admonition} conda-forge
---
class: attention
---
You can also deploy your plugin to conda-forge. Check out [deploying to conda-forge](deploying-to-conda-forge) for more
details on how to do that.
```

The [napari-plugin-manager](https://napari.org/napari-plugin-manager/) can be used to install plugins deployed to both
PyPI and conda-forge.

When you are ready for users, announce your plugin on the [Image.sc
forum](https://forum.image.sc/tag/napari).

[autodeploy]: https://github.com/napari/napari-plugin-template#set-up-automatic-deployments
[classifier]: https://pypi.org/classifiers/
[pypi]: https://pypi.org/
[pypi-upload]: https://packaging.python.org/en/latest/tutorials/packaging-projects/#uploading-the-distribution-archives
