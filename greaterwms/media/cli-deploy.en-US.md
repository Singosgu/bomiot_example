# bomiot deploy

Generate the GitHub Actions workflow configuration for the project, enabling automated build & packaging for Windows / macOS / Linux on GitHub.

```bash
bomiot deploy
```

## Overview

`bomiot deploy` creates (or updates) `.github/workflows/greaterwms.yaml` in the current project. Once the project is pushed to GitHub, the workflow automatically builds and packages the app in CI environments of all three platforms, producing ready-to-ship portable artifacts — configure once, get three-platform builds on every push.

The command must be run from the **project root** (where `launcher.py` lives).

## Prerequisites

1. The project was initialized with `bomiot init`; `launcher.py` defines `app_name` and `version`
2. The project is (or will be) hosted on GitHub — the workflow only runs on GitHub Actions
3. **`requirements.txt` must exist in the project root** — GitHub Actions installs dependencies via `pip install -r requirements.txt`; a missing file fails the cloud build immediately. Generate it in your local dev environment with `pip freeze > requirements.txt`

## Execution Flow

```text
bomiot deploy
  │
  ├─ 1. Read launcher.py for app_name and version
  ├─ 2. Generate a pair of auth keys for this project (COMMUNITY_KEY / SPONSOR_KEY)
  ├─ 3. Create .github/workflows/ (reused if it already exists)
  ├─ 4. Write greaterwms.yaml and inject project info as workflow env vars
  └─ 5. Done — push the repository to trigger three-platform builds
```

## Injected Environment Variables

The workflow's top-level `env` block carries 4 project-level variables used throughout the build:

| Variable | Source | Purpose |
|---|---|---|
| `APP_NAME` | `app_name` in launcher.py | Artifact naming |
| `BASE_VERSION` | `version` in launcher.py | Version number |
| `COMMUNITY_KEY` | Generated at run time | Project auth identity (community) |
| `SPONSOR_KEY` | Generated at run time | Project auth identity (sponsor) |

> Re-running `bomiot deploy` updates these 4 values in place (e.g. after a version bump) without touching any other customizations in the workflow.
>
> A fresh key pair is generated on every run; together they form the project's unique identity — the origin of `build.json` in the build artifacts.

## How It Enables Cross-Platform Builds

`bomiot deploy` compiles nothing itself — it is the **switch for cloud three-platform builds**:

| Scenario | Command | Description |
|---|---|---|
| Local build | `bomiot build` | Builds only the platform of the current machine; good for local verification |
| Cloud build | `bomiot deploy` + git push | Generates the workflow; GitHub Actions builds all three platforms in parallel |

The full pipeline:

1. **Configure**: `bomiot deploy` writes the app name, version and project keys into the workflow
2. **Trigger**: pushing to the `main` branch starts GitHub Actions automatically
3. **Parallel builds**: the workflow runs on Windows, macOS and Linux runners in parallel; each installs dependencies (from `requirements.txt`) and packages the app, producing a platform artifact carrying the project identity (build.json)
4. **Delivery**: the three artifacts are attached to the workflow run as Artifacts — download, unzip and run, no Python environment required
5. **Update**: artifacts can be published with `bomiot publish`; clients receive new versions via incremental updates

For per-platform build details and artifact descriptions, see the **Build & Distribute** document under the Build tab.

## Notes

- `bomiot deploy` only creates/updates the workflow file — it never uploads code; a manual `git push` is required to trigger builds
- The workflow filename is fixed to `.github/workflows/greaterwms.yaml`; do not rename it or GitHub will not recognize it
- Project keys are written into workflow env variables — consider repository visibility (env values are visible to everyone in Public repositories)
- After changing the version in `launcher.py`, re-run `bomiot deploy` to sync `BASE_VERSION` in the workflow
