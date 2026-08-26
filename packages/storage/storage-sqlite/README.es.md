# @deepseek-ai/dsh-storage-sqlite

[English](README.md) | Español

Backend SQLite para el [hub de almacenamiento](../storage/README.md.es.md): se registra como backend `sqlite` y sirve la faceta `kv` sobre un único archivo de base de datos de `node:sqlite` (o `:memory:`). Diseño y compensaciones: [Agent Note de almacenamiento KV de dominio](../../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md).

## Modelo de almacenamiento

Un documento por fila: cada tabla de unidad se convierte en una tabla física `"u_<unit>_<table>" (key TEXT PRIMARY KEY, value TEXT)` STRICT cuyo `value` es el texto JSON del registro, de modo que una clave actualiza una fila (la razón para enrutar aquí un dominio de alta rotación en lugar de al backend JSON). La identidad de la unidad vive en dos tablas de metadatos — `units` sella la versión de formato de cada unidad en la primera apertura y rechaza un descriptor divergente con `version-mismatch`; `unit_globals` mantiene la fila singleton global de cada unidad. La versión del diseño físico vive en `PRAGMA user_version`; cualquier otro valor sellado rechaza (formato no publicado, sin migraciones). Los nombres de unidad y tabla se validan contra el `UNIT_NAME_RE` del hub antes de llegar al DDL, de modo que ninguna entrada externa se interpola jamás en identificadores SQL.

Cada primitiva de escritura es una única sentencia preparada — la atomicidad por sentencia de SQLite satisface el contrato KV sin transacciones explícitas, y el orden de escritura sigue siendo responsabilidad del llamador (la cadena de escritura de la capa de dominio). Los directorios y archivos de base de datos ausentes se crean solo para el propietario (`0o700`/`0o600`), en línea con el backend SQLite de persistencia de sesión.

## Configuración (schemastery)

```ts
interface Config {
  path: string   // SQLite database file path, or ':memory:' for an in-process DB
  journalMode?: 'wal' | 'delete' | 'truncate' | 'persist'   // journal_mode pragma; default 'wal'
}
```

## Experiencia del modelo

### Registros de dominio almacenados

#### Qué ve el modelo

Nada. Este backend no aporta prompt, herramienta ni schema; persiste datos de dominio no-sesión (registros de workspace, metadatos de futuros sidecars de sesión) detrás de `ctx.storage` solo para consumidores del lado del host.

#### Efecto de tokens

Cero tokens en solicitudes en vivo.

#### Efecto de KV Cache

Ninguno — el backend nunca toca prefijos de solicitud en vivo.

## Limitaciones conocidas y trabajo diferido

- **`DatabaseSync` es síncrono** — cada escritura bloquea el event loop durante su duración (de una sola sentencia); aceptable a la escala de datos de dominio.
- **Sin política de busy-wait ni de reintento** — otra conexión que mantenga una transacción de escritura rechaza la operación de inmediato; no hay protección de escritura multiproceso.
- **Solo abre la `STORAGE_SQLITE_SCHEMA_VERSION` actual** — cualquier otra versión sellada se rechaza en lugar de migrarse (postura pre-release).
- **`openDatabase` duplica la secuencia de apertura SQLite de la persistencia de sesión** — la extracción a una capa de medio compartida queda diferida a la migración planificada del backend de sesión (consulta la auditoría de reutilización de la Agent Note).
