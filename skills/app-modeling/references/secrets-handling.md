# Secrets and Credentials

How database credentials are supplied is **type-specific** — read the type's schema. There are three models.

## Rules

- NEVER hardcode passwords, tokens, or keys in `app.bicep`. Use a `@secure() param` and pass it at deploy time (`rad deploy -p password=...`).
- The username is the **administrator** account you author for the database Radius provisions. Use a simple admin name (e.g. `myadmin`); it is not derived from the source, and an admin-style name is expected.
- `Radius.Security/secrets` is for (a) neo4j database credentials and (b) genuine app secrets (API keys, tokens) — NOT for postgres/mysql/sqlserver credentials.

## Model 1 — credentials directly on the resource (postgres, mysql, sqlserver)

The schema requires `username` and `password` on the data resource; `password` is `x-radius-sensitive`.

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

## Model 2 — secret referenced by `secretName` (neo4j)

`neo4jDatabases` takes a `secretName` pointing at a `Radius.Security/secrets` that holds `USERNAME` and `PASSWORD`.

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

## Model 3 — no credentials (redis, mongo, kafka, rabbitmq, objectStorage)

These take no credentials in the app definition; the recipe provisions the resource and generates the connection. A container `connection` to the resource injects `CONNECTION_*` env vars (host / port / connectionString). See [connection-conventions.md](connection-conventions.md).

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
- Do NOT create a `Radius.Security/secrets` for postgres/mysql/sqlserver — set `username`/`password` directly on the resource.
- Do NOT add a `secretName` property to a type whose schema doesn't have one (only neo4j does).
- Do NOT add credentials to redis/mongo/kafka/rabbitmq/objectStorage — they take none.
- Keys in a secret's `data` are UPPERCASE (`USERNAME`, `PASSWORD`, `API_KEY`).