# Algites GitHub CI Policy

This document defines the **standard CI behavior** for Algites repositories using the unified
GitHub Actions pipeline.

The goal is to provide a predictable, low-friction workflow:

- Keep **Gradle** as the primary, fast path.
- Keep **Maven** compatible and up-to-date (typically when Maven config changes).
- Enable **safe test-only runs** on dedicated branches without publishing.
- Enable **fast compile-only runs** on feature branches.
- Support Algites-style **variant branches** (e.g. `jvm17/*`, `jvm21/*`, `and21/*`, `mps2025.1/*`) naturally.

> In Algites, a *variant* prefix encodes the **target platform baseline**, e.g. `jvm17` (JVM runtime ≥ 17), `and21` (Android minSdk 21), or `mps2025.1` (MPS toolchain version).

---

## 1. Terminology

- **Stub workflow**: a small workflow committed inside each repository (e.g. `.github/workflows/algites-ci.yml` or `.github/workflows/publish-gradle.yml`),
  which triggers on push / manual dispatch and calls the central reusable workflow.
- **Central workflow**: the reusable workflow hosted centrally (e.g. in `priv.tool.Java`) that contains the logic for:
  - deciding what to do based on branch name,
  - detecting config changes,
  - resolving build tool (Gradle vs Maven),
  - resolving Java version,
  - generating tokens,
  - running compile/tests/publishing,
  - detecting issue references for build traceability.

---

## 2. Modes (High-Level Behavior)

The CI pipeline chooses a **mode** based on the branch name:

- **publish**: build and publish artifacts.
- **test**: run tests only; **no** publish/deploy/install.
- **compile**: compile only; **no** tests and **no** publish/deploy/install.
- **skip**: exit early; no compile/test/publish.

---

## 3. Branch Naming Rules

### 3.1 Publish branches

Publishing is enabled on these branches (hotfix is treated like develop):

- `main`
- `master`
- `develop`
- `hotfix`
- `*/main`
- `*/master`
- `*/develop`
- `*/hotfix`

Examples (variant branching):

- `jvm17/main`, `jvm17/master`, `jvm17/develop`, `jvm17/hotfix`
- `jvm21/main`, `jvm21/master`, `jvm21/develop`, `jvm21/hotfix`
- `and21/main`, `and21/develop`, `and21/hotfix` (Android variant, if used)
- `mps2025.1/main`, `mps2025.1/develop` (MPS variant, if used)

**Mode = publish**

### 3.2 Test-only branches

Branches containing the segment `feature-testrun` are treated as test-only:

- `feature-testrun/*`
- `*/feature-testrun/*`
Examples:

- `feature-testrun/ci-experiment`
- `jvm17/feature-testrun/maven-compat`
- `jvm21/feature-testrun/gradle-plugin-upgrade`
- `and21/feature-testrun/android-ci-experiment`

**Mode = test**

---

### 3.3 Compile-only feature branches

Feature branches are compiled without tests:

- `feature/*`
- `*/feature/*`

Examples:

- `feature/ci-experiment`
- `jvm17/feature/ID.123_maven-compat`

**Mode = compile**

---

### 3.4 All other branches

All branches not matching the above patterns are treated as:

**Mode = skip**

---

## 4. Config-change detection (Maven vs Gradle)

The central workflow detects changes between the pushed commits and classifies them as:

- **POM change** (Maven config changed): e.g. `**/pom.xml`, `**/*.pom`
- **Gradle config change**: e.g.
  - `**/build.gradle`, `**/build.gradle.kts`
  - `**/settings.gradle`, `**/settings.gradle.kts`
  - `**/gradle.properties`
  - `gradle/**` (wrapper/config)
  - `buildSrc/**`
  - `gradlew`, `gradlew.bat`

This is used to decide which build tool(s) to run in **auto** mode.

---

## 5. Build Tool Selection (Optimized Auto)

The pipeline can run **Gradle**, **Maven**, or **both** in a single CI run.

### 5.1 Repository configuration file (optional)

Add an optional file at repository root:

- `algites-build.properties`

Example:

```properties
build-tool=maven
```

Supported values:

- `build-tool=gradle`
- `build-tool=maven`
- `build-tool=auto`

If the file is missing (or set to `auto`), optimized auto-selection is used.

### 5.2 Optimized auto-selection rules

If `build-tool` is not forced (file missing or set to `auto`, and workflow input is `auto`):

- If **POM changed** → run **Maven only**
- Else if **Gradle config changed** → run **Gradle**
- Else (no POM/Gradle config change) → run **Gradle**
- If **both POM and Gradle config changed** → run **both Maven and Gradle**

If `build-tool` is forced to `maven` or `gradle`, only that tool runs (even if both configs changed).

### 5.3 Practical note (double execution)

If both Maven and Gradle configurations are changed, CI will execute **both toolchains**.
To keep changes easier to review and to avoid multiple CI runs across separate pushes, it is usually best to
change Maven + Gradle config together **in one commit** (or one push).

---

## 6. What Gets Executed (by Mode)

### 6.1 Mode = publish

- Gradle (default): `./gradlew publish`
- Maven (default): `mvn deploy`

### 6.2 Mode = test

- Gradle (default): `./gradlew test`
- Maven (default): `mvn test`

### 6.3 Mode = compile

- Gradle (default): `./gradlew classes`
- Maven (default): `mvn -DskipTests compile`

### 6.4 Mode = skip

Exit early after logging the decision.

---

## 7. Java Version Resolution

If Java version is not provided explicitly, the workflow auto-detects it in this order:

1. `.java-version` (single integer per line, e.g. `17`)
2. `gradle.properties` containing `javaVersion=<n>`
3. root `pom.xml` containing either:
   - `<maven.compiler.release>n</maven.compiler.release>`
   - `<maven.compiler.source>n</maven.compiler.source>`
4. fallback default (currently `17`)

---

## 8. Issue references (optional but recommended)

Algites CI can automatically detect **GitHub Issue references** related to a build and display them in the
Actions **Job Summary** as clickable links.

### 8.1 Branch naming convention (strict)

For feature branches, prefix the feature “slug” with an explicit **ID marker** and terminate it with an underscore:

- `feature/ID.123_some-description`
- `jvm17/feature/ID.123_some-description`
- `feature-testrun/ID.123_ci-experiment`

Cross-repository shorthand (Algites-EU only) is also supported:

- `feature/ID.pub.tool.Java-123_some-description`
- `j25/feature-testrun/ID.pub.tool.Java-123_ci-experiment`

> The `ID.` prefix is **required** in branch names to avoid accidental matches (e.g. variant branches like `jvm17/*` or `and21/*`).
> The underscore `_` acts as the delimiter that ends the issue reference.

### 8.2 Commit message convention (flexible)

In commit subjects, CI detects references introduced by `#`.

Supported forms:

- `#123` (issue in the same repository)
- `#ID.123` (same as `#123`)
- `#pub.tool.Java-123` (cross-repo shorthand, Algites-EU only)
- `#ID.pub.tool.Java-123` (same as above)

### 8.3 What CI does with it

If issue references are detected (from branch name and/or commit subjects), CI will:

- list them as clickable links in the Actions **Job Summary**
- emit log notices for quick scanning

> Note: this is informational only. Posting comments to issues would require additional GitHub App permissions
> and is intentionally **not enabled by default**.

---

## 9. Recommendations

- Prefer variant branching (`jvm17/*`, `jvm21/*`, `and21/*`, `mps2025.1/*`) to keep cross-variant feature migration explicit.
- Use `feature-testrun/*` for safe CI experiments without publishing.
- Use `feature/*` for normal development with fast compile feedback.
- Keep publishing limited to publish branches (`main`, `master`, `develop`, `hotfix`, and versioned variants).
- Use the identification of the issues everywhere it is possible and reasonable to keep the transparency of the changes ión highes possible level

---

**© Algites**
