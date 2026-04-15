# Debspawn Action
[![Test](https://github.com/ximion/debspawn-action/actions/workflows/tests.yml/badge.svg)](https://github.com/ximion/debspawn-action/actions/workflows/tests.yml)

Build Debian packages and run custom commands in clean Debian/Ubuntu containers on GitHub Actions!

This repository contains **three actions**:

- `setup` - creates (or updates) a debspawn image and must run first
- `build` - builds a Debian package using the previously created container image
- `run`   - runs arbitrary commands/scripts in the container

## Why do I need this?

- Reproducible, containerized builds for Debian and Ubuntu
- Very small CI surface for Debian packaging
- Easy custom command execution in a clean distro environment, using modern Debian or Ubuntu releases
- Built-in caching support for faster repeated runs

## Usage

All actions require the `setup` action to have run first, which will create and cache the container image to use.

### Setup action inputs

All inputs except for `suite` are optional.

| Input          | Examples                | Default  | Description                                                                                                         |
|----------------|-------------------------|----------|---------------------------------------------------------------------------------------------------------------------|
| `name`         | `build`, `my-ci-tests`  | `suite`  | Name of the image to create/use. If not set, the suite name will be used.                                           |
| `suite`        | `trixie`, `resolute`    | `stable` | Debian/Ubuntu suite to create the environment for. This input is required.                                          |
| `components`   | `main`, `main universe` | `main`   | Archive components to include.                                                                                      |
| `packages`     | `curl`, `git`, ...      | _(none)_ | Extra packages to install in the environment, as part of the setup process..                                        |
| `script`       | -                       | _(none)_ | Post-setup script (inline commands or a script file path). Changes made by the script are persisted into the image. |
| `cache`        | `true`, `false`         | `true`   | Cache image files between workflow runs if set to `true`.                                                           |
| `cache-suffix` | -                       | `gha`    | Additional cache key suffix to separate cache sets.                                                                 |

### Build action inputs

All inputs except for `suite` are optional.

| Input          | Examples                   | Default         | Description                                                                                                        |
|----------------|----------------------------|-----------------|--------------------------------------------------------------------------------------------------------------------|
| `name`         | `build`, `my-ci-tests`     | `suite`         | Image name to build in, or left empty to use the suite name.                                                       |
| `suite`        | `trixie`, `resolute`       | `stable`        | Ubuntu or Debian suite to build for.                                                                               |
| `source-dir`   | `.`, `src`                 | `.`             | Source directory root used for the package build.                                                                  |
| `debian-dir`   | e.g. `packaging/debian`    | `debian`        | Debian packaging directory. Will be copied into `source-dir` during the build.                                     |
| `version`      | `1.0.0`, `2.1.4`           | _(none)_        | Explicit package version. If omitted, a version is automatically determined from the git commit and any `v*` tags. |
| `version-slug` | `deb14`, `ubu26`, ...      | _(none)_        | Suffix appended to the version to distinguish build for different distributions.                                   |
| `results-dir`  | `build-results`, `out/deb` | `build-results` | Directory receiving build artifacts and logs.                                                                      |
| `network`      | `false`, `true`            | `false`         | Whether network access should be allowed during build.                                                             |

### Run action inputs

The input `suite` or `name` must be present, and `commands` must be present. All other inputs are optional.

| Input      | Examples                            | Default        | Description                                       |
|------------|-------------------------------------|----------------|---------------------------------------------------|
| `name`     | `testing-image`, ...                | `suite`        | Image name to run commands in.                    |
| `suite`    | `trixie`, `resolute`                | `stable`       | Suite of the image to run in.                     |
| `summary`  | `Run unit tests`, `Lint in testing` | auto-generated | Header text shown in debspawn logs.               |
| `commands` | `.github/scripts/ci.sh`             | _(none)_       | Inline commands or a script file path to execute. |
| `boot`     | `false`, `true`                     | `false`        | Boots the container before running commands.      |
| `allow`    | `all`, `CAP_SYS_ADMIN`              | _(none)_       | Extra allow-list permissions (newline-separated). |

## Examples

### Build a simple Debian package and upload it as an artifact

This example assumes minimal Debian packaging instructions are in the `packaging/debian` directory.
You do not need much data to build a simple Debian package - just a `control` file with the package metadata and a `rules` file with
the build instructions and a `changelog` template. Have a look at the [dummy package example](https://github.com/ximion/debspawn-action/tree/main/tests/dummy/debian)
used for testing this package for the most minimal example.

For your project, you will have to adjust the metadata in `d/control` and add the package build-dependencies, as well as adding
build instructions to `d/rules` if they are not automatically detected for your project.

Then you can build the package automatically with debspawn:

```yaml
name: Build Package

on:
  push:
  pull_request:

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v6

      - name: Setup debspawn
        uses: ximion/debspawn-action/setup@v1
        with:
          suite: trixie

      - name: Build package
        uses: ximion/debspawn-action/build@v1
        with:
          suite: trixie
          debian-dir: packaging/debian
          version-slug: deb13
          results-dir: build-results

      - name: Upload build artifacts
        uses: actions/upload-artifact@v7
        with:
          name: debian13-packages
          path: build-results/*
```

### Run arbitrary CI commands on Debian Testing

This is extremely nice if the GitHub-hosted Ubuntu runners do not have the right environment for your tests and you need newer packages
from a newer release of Ubuntu, or want to build for Debian instead of Ubuntu.

You can run any commands or scripts in the debspawn container, and it will have the clean environment of the specified suite with only
the packages you explicitly installed in the `setup` step.

Example to perpetually test against the newer packages in Debian Testing:

```yaml
name: Debian Testing CI

on:
  push:
  pull_request:

jobs:
  test-on-testing:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v6

      - name: Setup debspawn (Debian Testing)
        uses: ximion/debspawn-action/setup@v1
        with:
          suite: testing
          script: |
            # Install extra packages needed for tests
            apt-get install -y --no-install-recommends \
              build-essential \
              pkg-config

      - name: Run CI commands in debspawn
        uses: ximion/debspawn-action/run@v1
        with:
          suite: testing
          summary: "Run project tests on Debian Testing"
          commands: |
            set -e
            ./ci/run-tests.sh
```

## Troubleshooting

- Environment does not exist:
  - Cause: `build` or `run` executed before `setup`
  - Fix: run `setup` first in the same job and ensure your `name` and `suite` inputs match between actions.
- Expected package not found:
  - Check `results-dir` and package name/version pattern
- Build needs network access:
  - Set `network: true` in `build` when needed

## License

This action is MPL-2.0 licensed, see `LICENSE` for details.
