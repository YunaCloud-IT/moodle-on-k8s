# Maintaining the Moodle Helm Chart

This document outlines the process for updating, versioning, and releasing the Moodle Helm chart.

## 1. Versioning Strategy
We follow **Semantic Versioning 2.0.0** (`MAJOR.MINOR.PATCH`). It is critical to distinguish between the **Chart Version** and the **App Version**.

| Version Type | File | Field | Description |
| :--- | :--- | :--- | :--- |
| **Chart Version** | `Chart.yaml` | `version` | The version of the *Helm Chart logic* (templates, default values). **This controls the release.** |
| **App Version** | `Chart.yaml` | `appVersion` | The version of *Moodle* inside the image (e.g., 4.3.3). |

### When to Bump Versions?

* **PATCH** (e.g., `0.1.0` -> `0.1.1`):
    * Updating `appVersion` (e.g., Moodle 4.3.2 -> 4.3.3).
    * Fixing a bug in a template.
    * Adding a default value to `values.yaml` that is backward compatible.
    * Documentation updates.
* **MINOR** (e.g., `0.1.0` -> `0.2.0`):
    * Adding a new feature (e.g., adding Redis support).
    * Adding new required values that have defaults.
* **MAJOR** (e.g., `0.1.0` -> `1.0.0`):
    * **Breaking Changes.** (e.g., Renaming a key in `values.yaml` without a fallback, changing the database structure significantly).

---

## 2. Update Workflow

### Scenario A: Updating Moodle Version (Routine)
*Goal: Update Moodle from 4.3.2 to 4.3.3*

1.  **Update Docker Image:**
    * Update the `Dockerfile` to download the new Moodle version (or verify `latest` link).
    * Build and push the image via the `docker-build.yml` workflow (usually by pushing to main).
2.  **Update Chart Metadata:**
    * Edit `my-moodle-chart/Chart.yaml`:
        ```yaml
        version: 0.1.1      # Bump the Patch version
        appVersion: "4.3.3" # Update to match new Moodle version
        ```
3.  **Update Defaults (Optional):**
    * If the new Moodle version requires new PHP settings, update `values.yaml`.

### Scenario B: Changing Chart Logic (Features/Fixes)
*Goal: Add a new ConfigMap entry or fix a bug in Deployment*

1.  **Modify Templates:** Make your changes in `templates/`.
2.  **Lint:** Run `helm lint ./my-moodle-chart` locally to ensure no syntax errors.
3.  **Bump Version:**
    * Edit `my-moodle-chart/Chart.yaml`:
        ```yaml
        version: 0.2.0 # Bump Minor if feature, Patch if fix
        ```
    * **Do not** change `appVersion` if Moodle itself hasn't changed.

---

## 3. Documentation

Before releasing, you **must** update the documentation if `values.yaml` changed.

* **README.md:** If you added a new value (e.g., `redis.timeout`), add it to the configuration table.
* **CHANGELOG.md:** (Recommended) Add a brief entry describing the change.

```markdown
## [0.2.0] - 2024-05-20
### Added
- Added Redis session timeout configuration.
### Fixed
- Fixed permission issue on cronjob.
```

---

## 4. Release Process
Our release process is fully automated via GitHub Actions.

### Commit & Push:

- Ensure Chart.yaml has a new version.

- Push changes to main.

### Trigger Release:

- Automated: The release-chart.yml workflow listens for changes to Chart.yaml. If the version has changed, it will automatically:

    1. Package the chart (.tgz).

    2. Create a GitHub Release (e.g., moodle-0.2.0).

    3. Update index.yaml on the gh-pages branch.

### Verify:

- Check the Actions tab for success.

- Check that https://itchimonji.github.io/moodle/index.yaml contains the new version.

---

## 5. Breaking Changes (Important)
If you introduce a Breaking Change (e.g., renaming externalDatabase.host to db.host):

1. Bump MAJOR version in Chart.yaml.

2. Add NOTES.txt warning: Update templates/NOTES.txt to warn users if they need to migrate data.

3. Communication: Clearly state in the Release Notes that manual intervention is required.

---

## 6. Local Testing

Before pushing, test the chart locally:

```bash
# 1. Lint
helm lint ./my-moodle-chart

# 2. Dry Run (Simulate install)
helm install --debug --dry-run test-release ./my-moodle-chart -f values.yaml

# 3. Test Template Rendering
helm template ./my-moodle-chart
```
