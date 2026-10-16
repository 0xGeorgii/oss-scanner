# Threat model



## Scope and trust boundary

tox reads `tox.ini` or `pyproject.toml`, creates and reuses isolated environments, builds the project under test,
installs dependencies and packages, and executes configured commands. The main paths are configuration loading under
`src/tox/config/`, environment selection in `session/`, package construction in `tox_env/package/`, installation in
`tox_env/installer.py`, command execution under `execute/`, and automatic provisioning in `provision.py`.

A repository's tox configuration and source code are trusted to execute arbitrary code. Running tox on an untrusted
checkout is equivalent to running its test commands and build backend; command execution requested by that checkout is
not a sandbox escape. Security boundaries remain around data or filesystem effects that exceed what the configuration
asked tox to do, and around secrets tox promises to redact.

Inputs worth treating as hostile are:

- configuration syntax, substitutions, factors, labels, command argument arrays, environment variable names and values,
  dependency specifications and package configuration from a checkout;
- package metadata, wheel and sdist filenames, build-backend output, installer output and subprocess byte streams;
- paths inside the checkout and work directory, including symlinks planted before environment recreation or cleanup;
- plugin entry points and inline plugins. Installed plugins are trusted code, but malformed metadata must not make tox
  load a different distribution than the one selected;
- CI environment variables and credentials that must not appear in command rendering, journals or error reports.



## Components to examine

Prioritize subprocess construction and streaming in `execute/`, especially shell selection, argument boundaries,
interrupt handling, terminal behavior and decoding of attacker-controlled output. Review configuration replacement and
type conversion in `config/` for cases where a value crosses from data into an executable argument or selects an
unexpected environment.

Filesystem-sensitive code includes environment recreation, package staging, temporary directories, result journals,
work-directory cleanup and the virtualenv redirect helper. Check symlink and race behavior before recursive deletion or
replacement. Provisioning must install and re-exec the requested tox requirement without losing the original argument
boundaries or selecting an unintended interpreter.

Secret handling in `util/redact.py`, reporting and journals matters when a failing command includes credentials in an
URL or environment value. Package managers, build backends, virtualenv and user plugins are dependencies; defects in
their internals are out of scope unless tox supplies unsafe arguments or mishandles their output.



## Exercising the image

The repository is installed from `/src` into `/opt/venv`, with the completion extra and test dependency group. Run
`python -m pytest tests`. Focused fixtures under `tests/config`, `tests/execute`, `tests/session` and `tests/tox_env`
create temporary projects without requiring network access. A useful reproducer is a minimal temporary project plus a
single tox invocation showing an effect outside its configured work and environment directories.



## Severity guidance

- Critical: a repository that is only permitted to run a constrained tox command gains code execution in a separate
  privileged service that treats tox configuration as data. This rating needs a documented caller that establishes that
  boundary; tox itself does not sandbox project commands.
- High: tox deletes or overwrites a path outside its work/environment directories through a crafted checkout or symlink,
  or leaks a CI credential that its redaction rules promise to protect.
- Medium: argument injection invokes an additional command beyond the configured argv, provisioning selects an
  unintended package or interpreter, or crafted output causes persistent corruption outside the project workspace.
- Low: denial of service, terminal escape output with no demonstrated impact, or a failure confined to the disposable
  tox environment.

Include the tox command, configuration, platform, relevant environment variables and a minimal checkout in reports.



## Out of scope

- Commands, build backends, package installers or plugins deliberately selected by the project configuration.
- A dependency package executing code during installation; tox is explicitly an installer frontend.
- Access by a configured command to the user's files, network or environment. tox environments isolate dependencies, not
  operating-system privileges.
- Configuration mistakes, dependency confusion caused solely by an untrusted package index, and unsupported plugins.
- Release automation, documentation and test-only helpers.
