# Threat model



## Scope and untrusted input

platformdirs maps an application name, version, author and roaming choice to the operating system's conventional data,
configuration, cache, state, log, runtime and site directories. Callers use both the functional API exported from
`platformdirs/__init__.py` and the `PlatformDirs` classes in `api.py`, `unix.py`, `windows.py`, `macos.py`, `android.py`
and `ios.py`. With `ensure_exists=True`, property access also creates the returned directory.

The library has no network or file-format parser. Inputs that may cross a trust boundary are:

- application-controlled `appname`, `appauthor` and `version` values, including path separators, absolute paths,
  traversal components, drive prefixes, reserved names, control characters and very long strings;
- environment variables used as platform directory sources, including `HOME`, `XDG_*`, `TMPDIR`, `APPDATA`,
  `LOCALAPPDATA`, `PROGRAMDATA`, `ANDROID_*` and variables supplied by the macOS and iOS runtimes;
- platform APIs and files used to discover a home or runtime directory, such as `pwd`, Windows known-folder APIs,
  Android environment state and `/etc/xdg`;
- the existing filesystem when `ensure_exists=True`, including symlinks and files at a directory the library tries to
  create.

An application's own environment is normally trusted. Findings are most useful when a caller passes metadata obtained
from a package, plugin, archive or remote configuration, or when platformdirs violates an operating-system convention
that callers rely on to keep user data separate.



## Components to examine

Prioritize the path join and normalization logic in `PlatformDirsABC` and every platform implementation. Check whether
absolute child components or traversal escape the selected base, whether multi-path properties preserve boundaries, and
whether `ensure_exists` can create a directory outside the path returned to the caller. Windows drive, UNC and roaming
behavior; Unix `XDG_RUNTIME_DIR` and site-directory lists; macOS library paths; and Android/iOS detection need separate
review because their trust assumptions differ.

Also examine the pytest plugin in `pytest_plugin.py`. It redirects platform directories for tests and must not leave a
test writing into the real user's directories through an omitted property or stale cached value.

Documentation, release automation and compatibility-only comparisons with `appdirs` are lower priority. Wrong spelling
or ordering with no effect on path isolation is a correctness issue.



## Exercising the image

The repository is at `/src`, installed in editable mode in `/opt/venv`, which is on `PATH`. Test dependencies and a
built wheel are available offline. Run `python -m pytest tests`; platform-specific tests use monkeypatching and can run
on Linux. Small reproductions can instantiate a platform class directly with crafted constructor arguments and patched
environment variables. Use a temporary directory containing symlinks and regular files when testing `ensure_exists`.



## Severity guidance

- High: normal use with attacker-controlled application metadata makes an application create or select a directory
  outside the expected user or application boundary, enabling overwrite or disclosure of credentials or executable
  configuration without requiring control of the process environment.
- Medium: a platform API or filesystem race selects another user's directory, or `ensure_exists` follows an attacker
  controlled final component and performs an unintended filesystem operation.
- Low: denial of service, malformed output, or an incorrect platform convention without a demonstrated confidentiality
  or integrity impact.

Report the platform class, constructor values, relevant environment, returned path and filesystem setup. A regression
test using only the public API is preferred.



## Out of scope

- A user who already controls the process environment, home directory or every parent of the selected base directory.
- An application deliberately passing an absolute path or traversal component and then using the result without its own
  validation, unless platformdirs documents that component as a confined name.
- OS-specific behavior that follows the documented operating-system convention but differs from another platform.
- The caller's later handling of a correctly returned path.
