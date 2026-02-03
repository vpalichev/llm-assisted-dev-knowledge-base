The transactor is a **shared service**, not a project-specific component.

**Why global:**

- The transactor runs as an independent process (like PostgreSQL itself)
- It serves all peers across all projects
- Multiple Datomic databases (`my-app`, `testing`, `analytics`) use the same transactor
- It's infrastructure, not application code

**Typical locations:**

|Location|Use case|
|---|---|
|`D:\bin\datomic\datomic-pro-1.0.7469\config\`|Next to the Datomic installation|
|`%USERPROFILE%\.datomic\`|User-level config|
|`C:\ProgramData\datomic\`|System-wide on Windows|

**What belongs where:**

```
Global (transactor config):
  dev-transactor.properties  ← PostgreSQL connection, memory settings

Project-specific:
  db-config map / make-uri   ← Which logical Datomic DB to use
  deps.edn                   ← Datomic peer dependency
  schema.edn                 ← Your app's schema
```

The project only needs to know the Datomic URI to connect - it doesn't need to know how the transactor is configured. That's an ops/infrastructure concern.