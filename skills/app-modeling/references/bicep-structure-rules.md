# Bicep Structure Rules

These rules apply to ALL generated app.bicep files. Read the resource type YAML schema from `radius-project/resource-types-contrib` for property names and types. This file covers structural patterns only.

## General

- `extension radius` is the only extension line and comes first (it provides every Radius type; no per-namespace or per-type extensions)
- `param environment string` always declared
- `@secure() param password string` declared if database credentials are needed
- `param image string` declared if building container images
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
        image: myImage.properties.image
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
      containerImage: {
        source: myImage.id
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
@description('The full container image reference to build and push. Must be lowercase.')
param image string

resource myImage 'Radius.Compute/containerImages@2025-08-01-preview' = {
  name: 'myapp-image'
  properties: {
    environment: environment
    application: app.id
    image: image
    build: {
      context: '.'
    }
  }
}
```

Rules:
- Image reference comes from `param image string` — NOT hardcoded
- Container must reference image as `myImage.properties.image`
- Container must have a connection to `myImage.id` for dependency ordering
- `image` must be lowercase
- `build.context` is the directory containing the Dockerfile, relative to the repository root (`'.'` if the Dockerfile is at the repo root)
- One `param image string` per built image; for multiple built images, use `param <serviceName>Image string` for each

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
2. If the repo has a Dockerfile but no published image, use `Radius.Compute/containerImages` with `param image string`
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
- Do NOT add `@description` decorators unless the user asks for them (exception: `param image string` always gets a description)