# Bicep Structure Rules

These rules apply to ALL generated app.bicep files. Read the resource type YAML schema from `radius-project/resource-types-contrib` for property names and types. This file covers structural patterns only.

## General

- `extension radius` is the only extension line and comes first (it provides every Radius type; no per-namespace or per-type extensions)
- `param environment string` always declared
- `@secure() param password string` declared if database credentials are needed
- Exactly ONE `Radius.Core/applications@2025-08-01-preview` resource
- The `@<apiVersion>` shown in the examples below (e.g. `2025-08-01-preview`) is illustrative — use the API version from each type's schema
- All output files go in `.radius/` directory

## Radius.Compute/containers structure

```bicep
resource myContainer 'Radius.Compute/containers@2025-08-01-preview' = {
  name: 'my-container'
  properties: {
    environment: environment
    application: app.id
    containers: {                     // object map, NOT array
      myapp: {                        // key = container name (camelCase)
        image: myImage.properties.imageReference
        ports: {                      // object map, NOT array
          web: {
            containerPort: 3000       // NOT "port"
          }
        }
        env: {                        // only for vars NOT auto-injected
          MY_VAR: {
            value: 'some-value'       // must use { value: '...' } syntax
          }
        }
      }
    }
    connections: {                    // TOP-LEVEL — sibling of "containers"
      mysqldb: {                     // object map, NOT array
        source: mysqlDb.id
      }
    }
  }
}
```

Rules:
- `containers` is an object map, NOT an array
- `ports` is an object map, NOT an array
- `connections` is an object map, NOT an array
- `connections` is a TOP-LEVEL property under `properties` — NOT inside `containers`
- `disableDefaultEnvVars` goes on the connection entry, NOT on the container
- Port property is `containerPort`, NOT `port`
- `env` values use `{ value: 'string' }` syntax, NOT bare strings
- Do NOT reference readOnly properties of other resources (e.g. `mysqlDb.properties.host`)

## Radius.Compute/containerImages structure

```bicep
resource myImage 'Radius.Compute/containerImages@2025-08-01-preview' = {
  name: 'myapp-image'
  properties: {
    environment: environment
    application: app.id
    tag: 'v1.2.3'   // pin to a commit SHA or immutable tag; omit for a content-addressable digest
    build: {
      source: 'git::https://github.com/<org>/<repo>.git//<subdir>?ref=<sha-or-tag>'
    }
  }
}
```

Rules:
- The image is BUILT from `build.source` — there is NO `image` property and NO `param image string`
- `build.source` is the repo git URL: `git::https://github.com/<org>/<repo>.git//<subdir>?ref=<sha-or-tag>`. Omit `//<subdir>` when the build context is the repo root; pin `?ref=` to a commit SHA or release tag for reproducible builds
- Optional `build.dockerfile` (path to the Dockerfile relative to the source; defaults to `Dockerfile`) and optional `build.platforms`
- `tag` is optional — pin it to a SHA/immutable tag, otherwise the recipe computes a content-addressable digest
- The container references the built image via `<serviceName>Image.properties.imageReference`; this reference creates the dependency edge, so NO separate connection to the image is needed

## Radius.Data/* structure

```bicep
resource mysqlDb 'Radius.Data/mySqlDatabases@2025-08-01-preview' = {
  name: 'mysql'
  properties: {
    environment: environment
    application: app.id
    database: 'todos'      // derived from source (e.g. MYSQL_DATABASE)
    version: '8.0'         // derived from source (e.g. image tag mysql:8.0)
    username: 'myadmin'    // administrator you author for the provisioned DB
    password: password     // from a @secure() param
  }
}
```

Rules:
- Credentials follow whatever the type's schema defines (do not assume by engine):
  - schema has `username` + `password`: set them on the resource (`password` from a `@secure() param`, marked `x-radius-sensitive`)
  - schema has `secretName`: create a `Radius.Security/secrets` and reference it (see below)
  - schema has neither: no credentials — the recipe generates the connection
- Symbolic name is engine/instance-derived (`mysqlDb`), NOT fixed — so multiple data stores never collide
- Developer-facing props (`database`, `version`, `size`, `topic`, `queue`, `container`) are derived from source — do NOT hardcode; only set properties the schema defines
- Do NOT set readOnly properties (`host`, `port`, `connectionString`) — these are recipe outputs

## Radius.Security/secrets structure

```bicep
@secure()
param password string

resource dbSecret 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'db-secret'
  properties: {
    environment: environment
    application: app.id
    data: {
      USERNAME: {
        value: 'myadmin'
      }
      PASSWORD: {
        value: password
      }
    }
  }
}
```

Rules:
- Used when a data type's schema defines `secretName` (referenced from the data resource) and for app secrets (API keys, tokens) — not when the schema takes `username`/`password` directly
- NEVER hardcode passwords — use `@secure() param`
- `data` is an object map, NOT an array
- Keys in `data` are UPPERCASE (`USERNAME`, `PASSWORD`, `API_KEY`)
- `USERNAME` is the database administrator you author (e.g. `myadmin`) — it is not derived from the source

## Radius.Compute/routes structure

```bicep
resource myRoute 'Radius.Compute/routes@2025-08-01-preview' = {
  name: 'my-route'
  properties: {
    environment: environment
    application: app.id
    rules: [
      {
        matches: [
          { httpPath: '/' }
        ]
        destinationContainer: {
          resourceId: myContainer.id
          containerName: 'myapp'
          containerPort: 3000
        }
      }
    ]
  }
}
```

Rules:
- Do NOT use `target`, `source`, `destination`, or `backend` — these do NOT exist
- `rules` is the ONLY valid structure — array of objects with `matches` and `destinationContainer`
- `destinationContainer` requires ALL THREE: `resourceId`, `containerName`, `containerPort`
- Only add routes if external ingress is needed

## Image resolution

1. If the repo publishes a pre-built container image, use it directly
2. If the repo has a Dockerfile but no published image, use `Radius.Compute/containerImages` with `build.source` set to the repo git URL
3. Do NOT use a bare runtime base image (e.g. `node:22-alpine`) — it runs without app code

## Properties that do NOT exist

These are commonly hallucinated. They will cause deployment errors:

| Resource Type | Invalid property |
|---|---|
| `Radius.Compute/containers` | `port` (use `containerPort`), `image` at top level |
| `Radius.Compute/routes` | `target`, `source`, `destination`, `backend` |

## Output rules

- Do NOT include comments explaining skill rules in generated Bicep
- Do NOT set readOnly properties
- Do NOT add `@description` decorators unless the user asks for them