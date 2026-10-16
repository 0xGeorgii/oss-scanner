# Threat model



## Scope and trust boundary

pipx installs Python applications into per-package virtual environments and exposes their commands, manual pages and
shell completions into user-facing directories. It can run a package ephemerally, inject dependencies, upgrade or
reinstall environments, execute pip or uv inside them, install a standalone Python, and import or export manifests.

Packages the user asks pipx to install are trusted to execute arbitrary installation and application code. pipx is not a
package sandbox. Its security responsibilities are to interpret the requested package and options without argument
injection, keep one package's metadata and environment from steering operations onto another path, and expose or remove
only the files it owns.

Hostile or malformed inputs include:

- package specifications, index URLs, requirement files, VCS URLs, editable paths, suffixes, interpreter paths and
  passthrough installer arguments;
- package metadata and entry points returned by pip, uv or inspection of an installed environment;
- manifest and pipx metadata JSON, including old schema versions and values written by another process;
- application, manual-page and completion filenames chosen by an installed distribution;
- existing files, directories and symlinks in `PIPX_HOME`, `PIPX_BIN_DIR`, `PIPX_MAN_DIR`, shared-library locations,
  caches and temporary directories;
- archive names, redirects and checksums used when downloading standalone Python builds;
- subprocess output used to decide whether an install, upgrade or inspection succeeded.

Environment variables and explicit CLI options that relocate pipx directories are trusted user configuration. A finding
should show an effect outside the configured locations or a mismatch between the package the user requested and the code
pipx selected.



## Components to examine

Prioritize `package_specifier.py`, the pip and uv backends, and `commands/install.py`, `run.py`, `inject.py`,
`upgrade.py`, `reinstall.py` and `execute.py`. Verify that package strings remain one installer argument unless the user
used an explicit passthrough option, and that interpreter selection cannot be changed by package metadata.

Review `venv.py`, `venv_inspect.py` and `pipx_metadata_file.py` for trust in installed metadata. Path-sensitive code in
`commands/expose.py`, `commands/uninstall.py`, `commands/transaction.py`, `paths.py` and `shared_libs.py` must handle
pre-existing symlinks, partial transactions and concurrent pipx processes without overwriting or deleting unrelated
files. Application names such as `../tool`, absolute paths, reserved names and collisions need explicit tests.

`standalone_python.py` and cache handling deserve supply-chain review: redirects, archive selection, integrity checks,
extraction paths, permissions and replacement of an existing interpreter. `ensure_path` edits shell configuration, so it
must select and modify only the intended file.



## Exercising the image

The repository is installed from `/src` into `/opt/venv` with the uv extra and test group. Run
`python -m pytest tests -n auto`. Tests use temporary pipx homes, local package fixtures and a local package server. For
offline reproductions, build a minimal wheel with crafted entry points, then set every `PIPX_*` directory to a temporary
tree containing attacker-controlled symlinks.

Avoid public package indexes in a proof of concept. A local wheel or fixture makes clear whether pipx or the installer
caused the behavior.



## Severity guidance

- Critical: pipx accepts one package specification but fetches and executes a different attacker-selected package or
  standalone interpreter under default index and verification settings, with no prior attacker code execution.
- High: crafted package metadata or a filesystem entry makes pipx overwrite, expose or delete a file outside its
  configured directories; installer argument injection changes the requested package or index; or a standalone Python
  archive escapes its destination.
- Medium: one unprivileged local user can replace another user's exposed application through a pipx race or unsafe
  permissions, or a malformed manifest redirects an operation to the wrong managed environment.
- Low: denial of service, stale links, incorrect listings, or corruption confined to the disposable environment of the
  package the user asked to install.

Reports should include the command, environment variables, package artifact and metadata, initial filesystem tree and
the files changed by pipx.



## Out of scope

- Code execution by a package, dependency, build backend or application the user asked pipx to install or run.
- Behavior explicitly requested through `--pip-args`, `--uv-args`, a custom index, interpreter or relocated pipx path.
- A user who can already write all configured pipx directories or the shell configuration being edited.
- Vulnerabilities wholly inside pip, uv, virtualenv, packaging tools or an installed application.
- Typosquatting and dependency confusion where pipx passes the requested name and index configuration unchanged.
