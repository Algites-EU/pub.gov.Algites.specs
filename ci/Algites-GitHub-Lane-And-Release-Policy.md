# Algites GitHub Release & Upmerge Policy (B1 Lanes)

This document describes how Algites repositories do **versioning, releases, and upmerges** using:
- **variant-prefixed branches** (e.g. `jvm17/...`, `and21/...`)
- **lane branches** (e.g. `*/lane/1.1.x`)
- **version computed from tags** (B1 policy)
- GitHub Actions workflows for **release** and **lane creation**

---

## 1. Branch layout

### 1.1 Variant
A **variant** is the first path segment in the branch name.

Examples:
- `jvm17/lane/1.1.x`
- `and21/lane/1.1.x`

Variant encodes the **target platform baseline** (JVM / Android / MPS, …).

### 1.2 Lane
A **lane** is a stable minor line `<A>.<B>.x` and lives in the branch name:

- `*/lane/<A>.<B>.x` (example: `jvm17/lane/1.1.x`)

Lane meaning:
- `A` = major line
- `B` = minor line
- `C` = patch/fix (computed from tags, not stored in sources)

---

## 2. Repository metadata

Each repo MUST contain `algites-repository.properties` in repository root:

```properties
repo.lane=1.1.x
```

CI enforces:
- If the branch name contains `/lane/<X>/`, then `repo.lane` must equal `<X>`.

> To enforce this as a *hard rule*, protect branches (e.g. `*/lane/*`) and require the CI status check.

---

## 3. Versioning: Computed patch

### 3.1 Tag format (release)
Release tags MUST be:

- `v<A>.<B>.<C>-<variant>`

Example:
- `v1.1.15-jvm17`

### 3.2 Snapshot version (CI)
For lane `<A>.<B>.x` and variant `<variant>`:
- Find max released `C` from tags `vA.B.*-variant`
- Compute `nextC = maxC + 1` (or `0` if no tags exist)
- CI uses snapshot version:

- `<A>.<B>.<nextC>-<variant>-SNAPSHOT`

Example:
- last tag: `v1.1.15-jvm17`
- CI snapshot: `1.1.16-jvm17-SNAPSHOT`

This eliminates constant conflicts during upmerge (because `C` is not stored in repo files).

---

## 4. CI: when/what runs

The unified CI workflow computes the snapshot version and then runs either Gradle or Maven:

### 4.1 Build tool selection
`algites-build.properties` (optional):

```properties
build-tool=auto     # default
build-tool=gradle
build-tool=maven
```

In `auto`, CI prefers Gradle if a Gradle wrapper / build files exist; otherwise Maven if a root `pom.xml` exists.

---

## 5. Release process (manual action)

Releases are **explicit** and start from the GitHub UI:

- **Actions → Algites Universal Release → Run workflow**
- Choose the branch, typically:
  - `jvm17/lane/1.1.x` (or any other lane branch)

The release workflow will:
1) validate lane consistency (`repo.lane` vs branch lane)
2) compute next `C` from tags
3) create a new tag `vA.B.C-variant`
4) publish artifacts (Gradle or Maven, according to selection / override)

### 5.1 Automated tag naming
Tag is computed automatically. You do **not** type it manually.

---

## 6. Upmerge support

After a successful release, you can upmerge the resulting changes to higher lanes (same major).

The release workflow supports:
- `upmerge_mode=none` → do nothing
- `upmerge_mode=next` → upmerge to next minor lane if it exists (e.g. `1.1.x` → `1.2.x`)
- `upmerge_mode=all-higher` → upmerge to all higher lanes that exist (within the same major)
- `upmerge_mode=explicit` → upmerge only to branches you list

### 6.1 “Preview only” (copy/paste workflow)
Set `upmerge_preview_only=true` to only print a **single CSV line** with compatible branches into the Action Summary, e.g.:

```
jvm17/lane/1.2.x,jvm17/lane/1.3.x
```

You can copy that line and re-run the release with `upmerge_mode=explicit` and paste it into `upmerge_targets`.

### 6.2 What the upmerge does technically
Upmerge is attempted via:
- creating a temporary branch from the target lane
- cherry-picking commits from the previous release tag (or only HEAD if no previous tag exists)
- opening a PR to the target lane

If cherry-pick conflicts, the workflow skips that target (manual resolution is required).

---

## 7. Creating a new lane (automation)

To start a new minor lane across variants, use:

- **Actions → Algites Create Lane (All Variants) → Run workflow**

Inputs:
- `source_lane`: e.g. `1.1.x`
- `new_lane`: e.g. `1.2.x`
- `variants`: `auto` (discover existing variants with the source lane) or `explicit`

The workflow:
- creates `*/lane/<new_lane>` from `*/lane/<source_lane>`
- updates `repo.lane=<new_lane>` in the new branch
- pushes the new branch

---

## 8. Practical notes

### 8.1 “How do I release the next lane after upmerge?”
Upmerge only transfers commits. A release of that next lane is still:
- a deliberate manual action: run the Release workflow on `*/lane/1.2.x`.

### 8.2 “What if I need different versions for different artifacts?”
This model assumes **one version per repository**.
If different artifacts must have independent version lifecycles, split them into separate repositories.

---

## 9. Counterpoints (why this might bite you)

- **Tag-driven versioning** is great for avoiding merge conflicts, but you lose the “look at sources and see exact patch version” feeling. You’ll rely more on tags/releases and CI summaries.
- **Cherry-pick upmerge** is pragmatic, but it can silently skip targets on conflicts unless you watch the summary carefully.
- **One version per repo** simplifies governance, but it can force repo splits earlier than you’d like (more repos, more wiring).

If any of these are deal-breakers for a particular repo, you can still fall back to “version in a single file” (with more merge conflicts) or maintain separate version tracks per module (with more complexity).

