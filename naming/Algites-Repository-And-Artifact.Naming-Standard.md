[[FINAL-APPROVED]]
# Algites Repository and Artifact Naming Standard
**Version:** 1.5
**Status:** Normative
**Scope:** All repositories and build artifacts within the Algites ecosystem

---

## 1. Purpose

This document defines a unified and normative standard for naming:

- Git repositories,
- Maven/Gradle artifacts (JARs),
- and related identifiers

across the Algites platform, tools, libraries, products, and customer solutions.

The goals are:

- **Semantic clarity** – names must express domain and purpose.
- **Consistency** – the same concepts are named the same way everywhere.
- **Uniqueness** – artifacts must be identifiable without collisions.
- **Scalability** – the standard must support long-term growth.
- **Governance visibility** – public vs private scope must be immediately visible where needed.

---

## 2. Terminology

- **Visibility** – governance scope of a repository or artifact (`pub` or `priv`).
- **Role** – technical role of a repository or artifact (`platform`, `product`, `lib`, `tool`, etc.).
- **BusinessName** – PascalCase domain or product concept (e.g., `Modustro`, `MyGreatProduct`).
- **RepoSubname** – optional lowercase technical qualifier of a repository (e.g., `common`, `core`, `backend`).
- **Module Path** – dot-separated identifier of a module or component within a repository.
  It consists of zero or more **module path folders** followed by a mandatory **module root folder**:
  `[<folders>.]<moduleroot>` (e.g., `mymodule.blfacadeintf`, `tools.profiler`, `build.parent`).
- **Module Root Folder** – the last segment of the module path that identifies the primary module name.
- **Variant** – a suffix identifying a specialized flavor of a module (e.g., `tests`).
- **Artifact** – a build output, typically a JAR published to a Maven repository.

---

## 3. Repository Naming

### 3.1 Canonical Form

Repositories MUST be named using the following structure:

```
<vis>.<role>.<BusinessName>[.<reposubname>]
```

Where:

- `<vis>` = `pub` | `priv` (lowercase)
- `<role>` = technical role (lowercase)
- `<BusinessName>` = PascalCase business or domain name
- `<reposubname>` = optional lowercase technical qualifier

### 3.2 Case Rules

- `vis` and `role` MUST be lowercase.
- `BusinessName` MUST be PascalCase.
- All `reposubname` components MUST be lowercase.

### 3.3 Examples

```
pub.platform.Modustro
priv.product.Modustro
priv.lib.Customers.common
pub.tool.Java
priv.tool.Java
```

---

## 4. Artifact Naming (artifactId)

### 4.1 Canonical Form

Each artifactId MUST be composed of:

```
<vis>.<role>.<BusinessName[.reposubname]>_<module.path>[-<variant>]
```

Where:

- the part before `_` identifies the **repository root**,
- the part after `_` identifies the **module within the repository**,
- `<module.path>` follows the form `[<folders>.]<moduleroot>`,
- `-<variant>` is an OPTIONAL suffix identifying a variant of the module root (e.g., `tests`),
- `_` is a mandatory separator between repository identity and module path.

### 4.2 Case Rules

- `<vis>`, `<role>`, `<reposubname>`, all module path folders, module root, and `<variant>` MUST be lowercase.
- `<BusinessName>` MUST preserve PascalCase from the repository name.
- The underscore `_` MUST be used exactly once in the artifactId.
- The variant suffix, if present, MUST be appended using `-`.

### 4.3 Examples

From repository:

```
pub.platform.Modustro
```

Artifacts:

```
pub.platform.Modustro_core
pub.platform.Modustro_languages
pub.platform.Modustro_mps.plugin
```

From repository:

```
priv.lib.Customers.common
```

Artifacts:

```
priv.lib.Customers.common_aaa.blfacadeintf
priv.lib.Customers.common_aaa.blfacadeintf-tests
priv.lib.Customers.common_aaa.other
```

From repositories:

```
pub.tool.Java
priv.tool.Java
```

Artifacts:

```
pub.tool.Java_build.parent
pub.tool.Java_tools.profiler

priv.tool.Java_build.parent
priv.tool.Java_tools.profiler
```

---

## 5. groupId Naming

### 5.1 Canonical Form

groupId MUST follow:

```
eu.algites.<role>.<businessname-lc>[.<reposubname>]
```

Where `<businessname-lc>` is the lowercase form of the BusinessName.

### 5.2 Case Rules

All groupId components MUST be lowercase.

### 5.3 Visibility Rule

Visibility (`pub` | `priv`) MUST NOT be part of the groupId.

The groupId expresses only the **business and technical domain**, not governance.

### 5.4 Examples

```
eu.algites.platform.modustro
eu.algites.product.modustro.studio
eu.algites.lib.customers
eu.algites.lib.customers.common
eu.algites.tool.java
```

---

## 6. Visibility and Artifact Identity

Visibility represents **governance and distribution scope**.

### 6.1 Rules

Visibility MUST:

- be part of the **repository name**,
- be part of the **artifactId**,
- influence CI/CD and publishing targets.

Visibility MUST NOT:

- be part of the **groupId**.

### 6.2 Rationale

Including visibility in the artifactId provides:

- immediate visual distinction in dependency trees and logs,
- early detection of accidental use of private artifacts in public builds,
- clear auditability of JAR files by name alone.

---

## 7. Variant Suffix Rules

### 7.1 Purpose

Variants represent specialized flavors of a module root that are published as independent artifacts,
most notably shared test artifacts. It SHOULD be used whenever products that would otherwise be
distinguished only by a build classifier require their transitive dependencies to be tracked and
resolved by Maven/Gradle.

### 7.2 Reserved Variants

The following variant suffixes are RESERVED:

- `tests` – shared tests for the base module root.

Additional variants (e.g., `it`, `bench`) MAY be introduced if documented.

### 7.3 Terminal Rule

Artifacts with variant suffix `-tests` MUST be terminal:

- They MUST NOT define or publish further variants.
- In particular, `*-tests-tests` MUST NOT exist.

This prevents recursive test variants and aligns with Maven idioms where `-tests` denotes a test flavor.

### 7.4 Rationale

The `-tests` suffix intentionally mirrors the common Maven convention
`<artifactId>-tests-<version>.jar` used for test classifiers, while elevating it to a
first-class artifact to enable transitive sharing of test dependencies.

---

## 8. Mapping Rules

### 8.1 From Repository to Artifact Root

Given repository:

```
<vis>.<role>.<BusinessName>[.<reposubname>]
```

Artifact root becomes:

```
<vis>.<role>.<BusinessName>[.<reposubname>]
```

and the artifactId is completed by appending:

```
_<module.path>[-<variant>]
```

### 8.2 From BusinessName to groupId

BusinessName is normalized to lowercase:

```
Modustro -> modustro
Customers -> customers
Java -> java
```

---

## 9. JAR Naming

The resulting JAR file name MUST follow Maven convention:

```
<artifactId>-<version>.jar
```

Examples:

```
pub.platform.Modustro_core-1.2.0.jar
priv.lib.Customers.common_bai.blfacadeintf-1.0.0.jar
priv.lib.Customers.common_bai.blfacadeintf-tests-1.0.0.jar
pub.tool.Java_build.parent-1.4.0.jar
```

---

## 10. Roles (Recommended Set)

Common roles include:

- `platform` – Algites platform components
- `product` – concrete applications
- `lib` – reusable libraries
- `tool` – build and development tools
- `framework` – generic frameworks
- `lab` – experimental or laboratory work

New roles MAY be introduced but MUST be lowercase and documented.

---

## 11. CI/CD Governance

CI pipelines MUST:

- infer visibility from repository name prefix (`pub.` vs `priv.`),
- enforce that artifactId starts with the same visibility prefix and follows the `_` separator rule,
- enforce variant rules, including the terminal nature of `-tests`,
- publish artifacts to the corresponding Maven repository:
  - public repo for `pub.*`,
  - private repo for `priv.*`.

groupId MUST remain identical for public and private variants within the same domain.

---

## 12. Migration Rule

When migrating legacy projects:

- Preserve BusinessName and structure where possible.
- Align repository names to this standard.
- ArtifactId MUST be adapted to include visibility, the `_` separator,
  and, where applicable, the `-tests` variant suffix.

---

## 13. Rationale

This standard balances:

- **Domain identity** (PascalCase BusinessName),
- **Technical structure** (lowercase qualifiers),
- **Practical build compatibility** (lowercase groupId),
- **Governance clarity** (visibility in repository and artifactId),
- **Human auditability** (pub/priv visible in JAR names),
- **Structural readability** (explicit `_` separator between repo and module identity),
- **Maven idioms** (use of `-tests` for test variants),
- **Module semantics** (explicit distinction between module path folders and module root).

It follows proven historical patterns used in legacy conventions such as:

```
lib.Customers.common.aaa.blfacadeintf
```

while extending them with explicit visibility and repository/module separation semantics.

---

## 14. Summary

- Repositories express **visibility + role + business identity**.
- Artifacts express **visibility + role + business identity + module specialization**, separated by `_`.
- `<module.path>` is composed of optional path folders and a mandatory module root.
- Optional variant suffixes (e.g., `-tests`) express specialized flavors of the module root.
- `-tests` is terminal and MUST NOT be nested.
- groupId expresses **organizational and domain namespace only**.
- Visibility is part of artifact identity, but not of groupId.
- BusinessName remains PascalCase across repos and artifacts.
- The `_` separator cleanly delineates repository identity from module identity.

This standard is normative for all Algites projects.

---

**End of Document**
[[/FINAL-APPROVED]]
