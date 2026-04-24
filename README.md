# dew-lingo-openapi

Single source of truth for the dew-lingo HTTP contract. This module
produces two versioned artifacts that the backend and frontend consume
as normal dependencies.

## Artifacts produced

| Target   | Coordinates                                | Output of `mvn install`                                  |
|----------|--------------------------------------------|----------------------------------------------------------|
| Backend  | `com.mrDew:dew-lingo-api:<version>` (JAR)  | Installed to `~/.m2/repository/com/mrDew/dew-lingo-api/` |
| Frontend | `@dew-lingo/api-client@<version>` (npm)    | `target/dew-lingo-api-client-<version>.tgz`              |

The npm tgz is consumed locally via a `file:` reference for now. When
a third consumer appears or this leaves single-machine dev, publish to
Verdaccio (planned for the top-level docker compose) or a hosted registry —
the workflow stays identical, only the registry URL changes.

## Build

```bash
mvn install
```

Runs the full pipeline:

1. Install pinned Node (v22) into `target/node/` — no host npm needed.
2. `npm install` Redocly CLI + openapi-generator-cli into `node_modules/`.
3. **Lint** the spec with Redocly (rules in `redocly.yaml`).
4. **Bundle** the split spec into `target/openapi.bundled.yaml`.
5. **Generate** Java Spring interfaces and DTOs into `target/generated-sources/`.
6. **Compile** and **package** the Java JAR.
7. **Install** the JAR to the local Maven repository.
8. **Generate** the TypeScript fetch client into `target/typescript-client/`.
9. `npm install && npm run build && npm pack` the TS client →
   `target/dew-lingo-api-client-<version>.tgz`.

## Spec layout

```
src/openapi/
  openapi.yaml                  # entry — only $refs, no inline definitions
  paths/
    <resource>.yaml             # one file per resource
  components/
    schemas/
      common.yaml               # shared (Problem, Level, Pagination, ...)
      <resource>.yaml           # resource-specific schemas
    responses/
      errors.yaml               # reusable error responses
    parameters/
      common.yaml               # shared path/query params
```

Inline definitions in `openapi.yaml` are a smell — put them in the
right components file.

## Versioning (SemVer)

The version in `src/openapi/openapi.yaml` (`info.version`) and `pom.xml`
(`<version>`) move together on every contract change.

- **MAJOR** (`x.0.0`) — removing/renaming an endpoint, field, or enum
  value; changing a type; tightening a constraint; flipping a field
  from optional to required.
- **MINOR** (`x.y.0`) — adding an endpoint; adding an optional field;
  adding an enum value used only in responses.
- **PATCH** (`x.y.z`) — doc, description, or example changes only.

Pre-1.0 explicitly signals the contract is unstable.

## Consumer integration

### Backend (`../dew-lingo-be`)

Add to `pom.xml`:

```xml
<dependency>
    <groupId>com.mrDew</groupId>
    <artifactId>dew-lingo-api</artifactId>
    <version>0.1.0</version>
</dependency>
```

Implement controllers against the generated interfaces in
`com.mrdew.dewlingo.api`. Don't hand-write request/response DTOs.

### Frontend (`../dew-lingo-fe`)

Until Verdaccio is running, depend on the local tgz:

```bash
npm install ../dew-lingo-openapi/target/dew-lingo-api-client-0.1.0.tgz
```

Once Verdaccio is in place, switch to a registry-resolved version:

```json
"@dew-lingo/api-client": "^0.1.0"
```

## Workflow for contract changes

1. Edit files under `src/openapi/`.
2. Bump `info.version` in `openapi.yaml` **and** `<version>` in `pom.xml`
   (same value, same commit).
3. `mvn install` — lints, generates, packages, installs both artifacts.
4. In each consumer, bump the dep version and rebuild.

Never patch generated code in either consumer to "work around" the spec —
the next regeneration erases the patch and the contract drifts silently.

## Working here

This directory is its own git repository. `git` commands run from the
parent working directory will fail — stay inside `dew-lingo-openapi/`.
