<h2 align="center">
Selenoid via Aerokube CM — GitHub Composite Action
</h2>

<div align="center">

[![license](https://img.shields.io/github/license/Vikindor/selenoid-github-action.svg)](https://github.com/Vikindor/selenoid-github-action/blob/main/LICENSE)
[![release](https://img.shields.io/github/release/Vikindor/selenoid-github-action.svg)](https://github.com/Vikindor/selenoid-github-action/releases/latest)
[![CodeFactor](https://www.codefactor.io/repository/github/Vikindor/selenoid-github-action/badge)](https://www.codefactor.io/repository/github/Vikindor/selenoid-github-action)

</div>

Start a local [Selenoid](https://aerokube.com/selenoid/) on GitHub-hosted runners using **Aerokube CM** easily — with sane defaults and optional parameters when needed.

✨ **Defaults (no inputs):**
  - Installs CM via the official installer (script method).
  - Starts Selenoid with Chrome, pulling the latest available version and applying --force.
  - Exposes the WebDriver hub URL as the remote output: http://localhost:4444/wd/hub.

🧩 This is a **composite action** (pure YAML). No Node.js runtime or `node_modules` are required.

---

## Usage

### Minimal
```yaml
- uses: vikindor/selenoid-github-action@v1

- name: Run tests
  run: ./gradlew clean test -Dselenide.remote=http://localhost:4444/wd/hub
```

### With output (optional)
```yaml
- id: selenoid
  uses: vikindor/selenoid-github-action@v1

- name: Run tests
  run: ./gradlew test -Dselenide.remote=${{ steps.selenoid.outputs.remote }}
```

### Advanced (override defaults)
```yaml
- uses: vikindor/selenoid-github-action@v1
  with:
    browsers: "chrome,firefox"     # default: chrome
    last-versions: "2"             # default: 1
    force: "true"                  # default: true
    install-method: "direct"       # default: script
    cm-version: "1.8.2"            # default: (empty) -> latest
    extra-args: "--vnc --args '-limit 2'"
```

---

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `browsers` | no | `chrome` | Comma-separated list of browsers to pull (e.g. `chrome,firefox`). |
| `last-versions` | no | `1` | How many latest versions to fetch per browser. |
| `force` | no | `true` | If `true`, adds `--force` to CM command. |
| `install-method` | no | `script` | `script` to use official installer (`curl … | bash`) or `direct` to download CM binary from GitHub Releases. |
| `cm-version` | no | `` | Pin CM version when `install-method=direct` (e.g. `1.8.2`). Empty means latest. |
| `extra-args` | no | `` | Extra args appended to `cm selenoid start` as-is. |

## Outputs

| Name | Value |
|------|-------|
| `remote` | `http://localhost:4444/wd/hub` |

---

## How it works

1. **Install CM**
   - `install-method: script` → download from https://aerokube.com/ (default)
   - `install-method: direct` → download CM binary for the current CPU arch from GitHub Releases (optionally pinned with `cm-version`)
2. **Start Selenoid** via CM with provided inputs
3. **Expose remote URL** as an action output `remote`

---

## Example: full Java workflow (JDK + Selenide)
```yaml
name: CI
on: [push]

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: '21'

      - uses: vikindor/selenoid-github-action@v1

      - name: Run tests
        run: ./gradlew clean test -Dselenide.remote=http://localhost:4444/wd/hub
```

---

## Notes
- This action does not manage Selenoid lifecycle after the job ends; containers will be stopped by the runner cleanup.
- If you need video/VNC, pass `extra-args` (e.g. `--vnc`).
- Works with any test runner that can point WebDriver to a remote hub URL.

---

## Gradle note: pass `-D` properties to the test JVM
If you pass settings via `-D…` in the workflow (e.g. `-Dselenide.remote=http://localhost:4444/wd/hub`), make sure the Gradle **Test** task forwards them to the test JVM. Otherwise JUnit tests won’t see those values and Selenide will use defaults.

**Kotlin DSL (`build.gradle.kts`)**
```kotlin
tasks.test {
    useJUnitPlatform()
    // Forward ALL system properties (including -Dselenide.*) to the test JVM
    systemProperties(
        System.getProperties()
            .entries
            .associate { (k, v) -> k.toString() to v }
    )
}
```

> Tip: to forward only Selenide-related keys, use:
```kotlin
systemProperties(
    System.getProperties().stringPropertyNames()
        .filter { it.startsWith("selenide.") }
        .associateWith { System.getProperty(it) }
)
```

**Groovy DSL (`build.gradle`)**
```groovy
test {
    useJUnitPlatform()
	systemProperties(System.getProperties())
}
```

This step is unnecessary if you set Selenide configuration directly in code instead of via `-D` flags.