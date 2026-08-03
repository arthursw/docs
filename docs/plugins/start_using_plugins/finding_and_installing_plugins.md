(find-and-install-plugins)=

# Finding and installing plugins

Plugins are Python packages which extend napari's functionality.
They can be used to add new features, such as file format support, new visualizations, or new tools.
This page will show you how to find and install plugins for napari.

## Finding plugins

Plugins can be discovered in the following places:

- **napari hub:** The [napari hub](https://napari-hub.org) offers a user-friendly
  way to find napari plugins.
- **PyPI:** The Python Package Index (PyPI) stores and distributes plugin packages.
  Search for plugins annotated with the classifier [`Framework :: napari`](https://pypi.org/search/?q=&o=&c=Framework+%3A%3A+napari).
- **conda-forge:** Many scientific packages are available on conda-forge.
  Use the conda-forge [package search page](https://conda-forge.org/packages/) to find napari plugins.

Users may also find plugins by searching on GitHub, napari's Zulip chat, and
the image.sc forum.

## Installing plugins with napari

The [napari plugin manager](https://napari.org/napari-plugin-manager/) is a tool
that allows users to install plugins directly from within napari.
This napari plugin manager offers users a convenient "Plugins" menu integrated
with the napari viewer.

From the "Plugins menu", select "Install/Uninstall Plugins..." to open the
a dialog that allows you to search for and install plugins.

![napari viewer's Plugins menu with Install/Uninstall Plugins as the first item.](../../_static/images/plugin-menu.png)

From the dialog, you can install plugins in the following ways:

- **Using search:** Start typing in the text box at the top of the dialog to dynamically search
  and filter plugins. Install a desired plugin by clicking the `Install` button found in the chosen plugin's tile.

- **Via manual input** Manual input offer additional flexibility when installing plugins.
  Depending on how you installed the napari application, a text box at the bottom
  of the napari plugin manager window will display either:

  - "install with 'pip' by name/url, or drop file..."
  - "install with 'conda' by name/url, or drop file..."

  In this text box, enter:

  - the plugin name to install
  - *any* valid pip or conda [requirement specifier](https://pip.pypa.io/en/stable/reference/requirement-specifiers/)
  - a valid [VCS scheme](https://pip.pypa.io/en/stable/topics/vcs-support).

  Then, click the "Install" button next to the input bar.

  ![napari viewer's Plugin dialog. At the bottom of the dialog, there is a place to install by name, URL, or dropping in a file.](../../_static/images/plugin-install-dialog.png)

  ```{admonition} Installing the Current Release
  To install `napari-svg`, enter `napari-svg` in the text field and press {kbd}`Enter` or click "Install". This is equivalent to running `pip install napari-svg`.
  ```

  ```{admonition} Installing from a Github Branch
  If you want to install `napari-svg` directly from the development branch on the [github repository](https://github.com/napari/napari-svg), enter `git+https://github.com/napari/napari-svg.git` in the text field.
  ```

  ```{admonition} Installing a Specific Release
  If you want to install `napari-svg` from a specific release, enter `napari-svg==0.1.0` in the text field.
  ```

  ```{admonition} Installing with Optional Dependency Groups
  To install a plugin with a group of option dependencies, use the optional group in brackets. If you want to install `napari-svg` with the optional testing group, enter `napari-svg[testing]` in the text field. 
  ```

- **Advanced installation:** After searching for the plugin you wish to install, click on the
  "Installation Info" button to open choices for installation source (conda or pip) and version selection. After choosing the desired options, you can click the "Install" button to install the plugin.

  ![napari plugin manager with the Installation Info button expanded to show conda or pip as installation source.](../../_static/images/plugin-manager.png)

The [napari-plugin-manager's documentation](https://napari.org/napari-plugin-manager/) provides more
detail on installing plugins.

### Know which installations protect the napari environment

Napari can guarantee dependency isolation only when it controls the complete main-package installation.
For a managed PyPI installation, napari selects one exact plugin wheel, checks its runtime requirements against the running napari environment, and rejects it before installation if it requires an additional package or an incompatible version of a package napari already uses.
Napari then installs that checked wheel without dependency resolution, so accepting the plugin cannot add, upgrade, or downgrade packages alongside napari.

The following routes are not managed main-package installations:

- choosing a Conda package;
- entering a package name, URL, local path, or VCS reference in a direct-install field;
- running `pip` or Conda outside napari.

Those routes use their package manager's normal dependency resolution and can change napari's environment.
Napari still discovers any managed-environment declarations after the plugin is installed, but that does not retroactively make the main-package installation isolated or run an `on_install` provisioning transaction.

If a managed PyPI installation is blocked by the host dependency check, the error identifies the rejected requirement.
The plugin author must [move that dependency and the code that imports it to a managed worker environment](managed-environment-migration); bypassing the check with a direct install gives up napari's conflict-prevention guarantee.

## Managing isolated plugin environments

Plugin interface code runs inside napari, but a plugin may need packages that napari does not supply.
To prevent those packages from changing napari or conflicting with another plugin, the plugin declares a separate managed environment and runs the corresponding functions in a worker process.
An installed plugin can declare one or more such environments.
Select **Environments** on that plugin's entry to see each environment's installation policy, persistent state, and worker state.

Available actions include:

- **Install** prepares a missing or changed environment ahead of first use.
- **Rebuild** recreates the current environment from a clean state.
- **Cancel** interrupts provisioning and removes the incomplete build.
- **Stop workers** cancels active work and closes worker processes without deleting the provisioned environment.
- **Remove** stops workers and deletes the persistent environment.

An `on_install` environment is prepared after the plugin package is installed or updated through napari's plugin manager.
An `on_demand` environment remains uninstalled until the user prepares it or the plugin first invokes its worker command.
Provisioning does not start a worker; napari starts workers lazily when a command needs them and reuses the environment while its recipe is unchanged.

Environment preparation does not depend on the Plugins window remaining open.
During preparation, napari's Activity panel shows lifecycle progress for preparing, provisioning, starting, and cleanup, with a cancel control.
When a worker command begins executing, its progress and cancel control remain in the plugin widget that started it instead of occupying the Activity panel.

The Managed Environments dialog contains one resizable, scrollable operation log shared by the selected plugin's environments.
Use its environment filter or an environment row's **Show log** action to find relevant records, including structured failure details.
**Copy** copies the displayed diagnostics and **Clear** clears the displayed text.
Napari keeps a bounded operation history for the current application session, so opening the dialog after an on-demand first use replays recent preparation records even when the manager was closed during the operation.
This history is for the current session only and is not a persistent log across napari restarts.

```{important}
Napari can coordinate environment provisioning only for plugin installation flows it manages.
Installing or updating a plugin through Conda, a direct-entry field, `pip`, or another external tool still discovers its declarations, but does not run an `on_install` provisioning transaction.
Open the plugin manager and install the environment, or let a worker invocation prepare it on demand.
```

Managed environments prevent a plugin's worker dependencies from changing napari or another plugin's environment.
They are not security sandboxes, and worker code has the user's operating-system permissions.

## Uninstalling and updating plugins

Like installation, the plugin dialog can also be used to uninstall or update plugins in a similar way.
For plugins with managed environments, uninstall first stops owned workers and removes the environments.
If cleanup fails, the plugin package remains installed so the operation can be retried without leaving hidden owned resources.
