# Lesson 1.2.1 y 1.2.2 — DDL y DML: Construyendo y manipulando datos

## ¿Qué son DDL y DML?

Son dos categorías de comandos SQL con propósitos completamente distintos:

**DDL — Data Definition Language**
Define la *estructura* de tu base de datos. Son los comandos que crean, modifican o eliminan objetos (bases de datos, schemas, tablas, columnas).
```
CREATE, ALTER, DROP, TRUNCATE, RENAME
```

**DML — Data Manipulation Language**
Manipula los *datos* dentro de esas estructuras. Son los comandos que insertan, actualizan, eliminan o consultan filas.
```
INSERT, UPDATE, DELETE, SELECT
```

La distinción importa en producción: los DDL son cambios estructurales (como reformar un edificio), los DML son cambios en el contenido (como mover muebles dentro del edificio). Los DDL tienen consecuencias más graves si se ejecutan por error.

---

## Parte 1 — DDL: Construyendo la estructura

### La jerarquía en DuckDB

```
Base de datos (DATABASE)
  └── Schema
        └── Tabla / Vista / Temp Table
```

Antes de crear cualquier tabla, necesitas saber en qué base de datos y en qué schema vas a trabajar.

### Crear y usar bases de datos

```sql
DROP DATABASE IF EXISTS job_mart;      -- elimina si existe (evita error)
CREATE DATABASE IF NOT EXISTS job_mart; -- crea solo si no existe
USE job_mart;                           -- cambia el contexto activo
SHOW DATABASES;                         -- lista todas las bases de datos
```

`IF EXISTS` e `IF NOT EXISTS` son buenas prácticas: hacen que tus scripts sean **idempotentes** — puedes ejecutarlos varias veces sin errores. Esto es crítico en pipelines automáticos.

### Crear schemas

```sql
CREATE SCHEMA IF NOT EXISTS staging;
```

El schema es una carpeta lógica dentro de la base de datos. Permite organizar tablas por propósito:

| Schema | Propósito |
|---|---|
| `staging` | Datos en proceso, antes de limpiar |
| `main` | Datos de producción, limpios y listos |
| `analytics` | Tablas para consumo de analistas |

### Crear una tabla

```sql
CREATE TABLE IF NOT EXISTS staging.preferred_roles (
    role_id   INTEGER PRIMARY KEY,
    role_name VARCHAR
);
```

`PRIMARY KEY` significa que `role_id` es el identificador único de cada fila. La base de datos garantiza que no habrá dos filas con el mismo `role_id`.

### Modificar la estructura — ALTER TABLE

Una vez creada la tabla, puedes modificar su estructura sin borrarla ni recrearla:

```sql
-- Agregar una columna
ALTER TABLE staging.preferred_roles
ADD COLUMN preferred_role BOOLEAN;

-- Renombrar la tabla
ALTER TABLE staging.preferred_roles
RENAME TO priority_roles;

-- Renombrar una columna
ALTER TABLE staging.priority_roles
RENAME COLUMN preferred_role TO priority_lvl;

-- Cambiar el tipo de una columna
ALTER TABLE staging.priority_roles
ALTER COLUMN priority_lvl TYPE INTEGER;
```

> **Error común:** Cambiar el tipo de una columna puede fallar si los datos existentes no son compatibles. Por ejemplo, cambiar `VARCHAR` a `INTEGER` falla si hay texto que no puede convertirse. Prueba siempre en staging antes de aplicar en producción.

---

## Parte 2 — DML: Manipulando los datos

### INSERT — Agregar filas

```sql
INSERT INTO staging.preferred_roles (role_id, role_name)
VALUES
    (1, 'Data Engineer'),
    (2, 'Senior Data Engineer'),
    (3, 'Software Engineer');
```

Siempre especifica explícitamente los nombres de las columnas. Si la tabla cambia en el futuro y el INSERT no los especifica, puede insertar valores en las columnas equivocadas sin dar error.

### UPDATE — Modificar filas existentes

```sql
UPDATE staging.priority_roles
SET priority_lvl = 3
WHERE role_id = 3;
```

> **Regla de oro del UPDATE:** Siempre usa `WHERE`. Un `UPDATE` sin `WHERE` modifica **todas** las filas de la tabla. En producción, esto puede ser un desastre difícil de revertir.

### DELETE — Eliminar filas específicas

```sql
DELETE FROM staging.job_postings_flat
WHERE job_posted_date < '2024-01-01';
```

Elimina solo las filas que cumplen la condición. Las demás quedan intactas. Igual que el UPDATE: **siempre usa WHERE**.

### TRUNCATE — Vaciar una tabla completamente

```sql
TRUNCATE TABLE staging.job_postings_flat;
```

Elimina **todas** las filas pero mantiene la estructura de la tabla. Es mucho más rápido que `DELETE` sin `WHERE` porque no evalúa condiciones ni registra cada fila eliminada — simplemente vacía la tabla.

| | DELETE sin WHERE | TRUNCATE |
|---|---|---|
| Velocidad | Lenta | Muy rápida |
| Se puede hacer rollback | Sí | No en todos los motores |
| Mantiene la estructura | Sí | Sí |
| Cuándo usar | Cuando necesitas condiciones | Cuando quieres vaciar todo |

---

## CTAS — CREATE TABLE AS SELECT

Una de las operaciones más útiles en Data Engineering:

```sql
CREATE OR REPLACE TABLE staging.job_postings_flat AS
SELECT
    jpf.job_id,
    jpf.job_title_short,
    jpf.job_location,
    jpf.job_posted_date,
    jpf.salary_year_avg,
    cd.name AS company_name
FROM data_jobs.job_postings_fact AS jpf
LEFT JOIN data_jobs.company_dim AS cd
    ON jpf.company_id = cd.company_id;
```

CTAS hace dos cosas en un solo paso: **crea la tabla y la llena con datos**. Los tipos de columna se infieren automáticamente del SELECT.

En pipelines de ETL, CTAS es el comando más común para crear tablas intermedias: tomas datos de la fuente, los transformas en el SELECT, y el resultado queda guardado en una tabla nueva lista para el siguiente paso.

---

## Vistas vs Tablas vs Tablas Temporales

Tres formas de guardar una query, con usos distintos:

### Vista (VIEW)

```sql
CREATE OR REPLACE VIEW staging.priority_jobs_flat_view AS
SELECT jpf.*
FROM staging.job_postings_flat AS jpf
JOIN staging.priority_roles AS r
    ON jpf.job_title_short = r.role_name
WHERE priority_lvl = 1;
```

Una vista **no guarda datos** — guarda la definición de la query. Cada vez que consultas la vista, ejecuta la query por detrás. Útil para:
- Simplificar queries complejas dándoles un nombre
- Dar acceso a analistas sin exponer la tabla base completa
- Siempre muestra datos actualizados (no es una foto estática)

### Tabla temporal (TEMP TABLE)

```sql
CREATE OR REPLACE TEMPORARY TABLE senior_jobs_flat_temp AS
SELECT *
FROM staging.priority_jobs_flat_view
WHERE job_title_short = 'Senior Data Engineer';
```

Existe solo durante la sesión actual. Cuando cierras la conexión, desaparece automáticamente. Útil para:
- Guardar resultados intermedios de queries costosas
- Trabajar con subsets de datos sin afectar tablas permanentes
- El `MERGE` de la lección 1.2.4 usa exactamente este patrón

### Tabla permanente

Persiste hasta que la elimines explícitamente. Es el destino final de los datos procesados.

| | Vista | Temp Table | Tabla permanente |
|---|---|---|---|
| Guarda datos | No | Sí | Sí |
| Duración | Permanente | Solo la sesión | Permanente |
| Cuándo usar | Abstraer queries | Resultados intermedios | Datos de producción |

---

## El impacto de las Vistas en DELETE

Un detalle importante que viste en la lección: cuando haces DELETE en la tabla base de una vista, la vista refleja el cambio automáticamente:

```sql
-- Antes del DELETE
SELECT COUNT(*) FROM staging.job_postings_flat;         -- 787.686 filas
SELECT COUNT(*) FROM staging.priority_jobs_flat_view;   -- refleja las mismas

-- Después del DELETE
DELETE FROM staging.job_postings_flat WHERE job_posted_date < '2024-01-01';

SELECT COUNT(*) FROM staging.job_postings_flat;         -- menos filas
SELECT COUNT(*) FROM staging.priority_jobs_flat_view;   -- también menos (es la misma data)
SELECT COUNT(*) FROM senior_jobs_flat_temp;             -- NO cambia (es una foto estática)
```

La temp table es una **foto** tomada en el momento de su creación. La vista es una **ventana** a los datos actuales.

---

## ¿Por qué importa en un proyecto real?

- **DDL en migraciones:** Cada cambio de esquema en producción se maneja como una "migración" (archivo `.sql` versionado). Herramientas como dbt, Flyway o Liquibase automatizan esto.
- **DML en pipelines:** INSERT, UPDATE y DELETE son las operaciones de carga de datos en un ETL. La lección 1.2.4 (MERGE) unifica los tres en un solo statement.
- **CTAS en transformaciones:** En dbt, cada modelo es esencialmente un CTAS — escribes un SELECT y dbt crea la tabla por ti.
- **Vistas en Data Marts:** Los analistas de negocio consumen vistas, no tablas crudas. Las vistas actúan como una capa de abstracción que puedes cambiar sin que los dashboards de Tableau o Looker se rompan.
