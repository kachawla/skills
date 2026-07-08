# Secrets and Credentials

How database credentials are supplied is **type-specific** — read the type's schema. There are three models.

## Rules

- NEVER hardcode passwords, tokens, or keys in `app.bicep`. Use a `@secure() param` and pass it at deploy time (`rad deploy -p password=...`).
- **Determine the credential shape from the type's schema — do not assume by engine.** The three shapes below key off which properties the schema defines.
- The username is the **administrator** account you author for the database Radius provisions. Use a simple admin name (e.g. `myadmin`); it is not derived from the source.
- `Radius.Security/secrets` is for (a) a data type whose schema requires `secretName` and (b) genuine app secrets (API keys, tokens).

## Shape 1 — schema defines `username` + `password`

Set them directly on the data resource; `password` is `x-radius-sensitive`. (Today: postgres, mysql, sqlserver.)

```bicep
@secure()
param password string

resource postgresql 'Radius.Data/postgreSqlDatabases@2025-08-01-preview' = {
  name: 'postgresql'
  properties: {
    environment: environment
    application: app.id
    database: 'appdb'      // derived from source
    username: 'myadmin'
    password: password
  }
}
```

## Shape 2 — schema defines `secretName`

Create a `Radius.Security/secrets` holding `USERNAME` and `PASSWORD`, and reference it via `secretName`. (Today: neo4j — but it is migrating to Shape 1, so always follow the current schema rather than assuming.)

```bicep
@secure()
param password string

resource neo4jSecret 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'neo4j-secret'
  properties: {
    environment: environment
    application: app.id
    data: {
      USERNAME: { value: 'neo4j' }
      PASSWORD: { value: password }
    }
  }
}

resource neo4jDb 'Radius.Data/neo4jDatabases@2025-08-01-preview' = {
  name: 'neo4j'
  properties: {
    environment: environment
    application: app.id
    secretName: neo4jSecret.name
  }
}
```

## Shape 3 — schema defines no credential properties

The type takes no credentials in the app definition; the recipe provisions the resource and generates the connection. A container `connection` to the resource injects `CONNECTION_*` env vars (host / port / connectionString). See [connection-conventions.md](connection-conventions.md). (Today: redis, mongo, kafka, rabbitmq, objectStorage.)

## App-specific secrets

Use `Radius.Security/secrets` for API keys and tokens the app needs, referenced by the container via a connection or env.

```bicep
@secure()
param apiKey string

resource appSecrets 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'app-secrets'
  properties: {
    environment: environment
    application: app.id
    data: {
      API_KEY: { value: apiKey }
    }
  }
}
```

## Common mistakes to avoid

- Do NOT write `password: 'mysecretpassword'` — always use a `@secure() param`.
- Do NOT assume a credential model by engine — read the schema and follow the properties it defines.
- Do NOT create a `Radius.Security/secrets` when the schema defines `username`/`password` — set them directly on the resource.
- Do NOT add a `secretName` property unless the schema defines one.
- Do NOT add credentials when the schema defines none.
- Keys in a secret's `data` are UPPERCASE (`USERNAME`, `PASSWORD`, `API_KEY`).