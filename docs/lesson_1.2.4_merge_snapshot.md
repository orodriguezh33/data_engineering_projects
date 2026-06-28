# Lesson 1.2.4 — Snapshot Tables y el patrón MERGE

## ¿Por qué existe esto en el mundo real?

Imagina que trabajas como Data Engineer en una empresa de reclutamiento. Tienes una tabla con miles de empleos publicados y necesitas saber qué roles son prioritarios para tus clientes.

El problema: **los datos cambian constantemente**.
- Hoy un rol puede ser prioridad 1, mañana prioridad 3
- Se publican empleos nuevos cada hora
- Empleos viejos desaparecen

Tu tabla snapshot es como una **foto actualizada del estado actual** de esos datos. Cada vez que la actualizas, necesitas:
1. Modificar los registros que cambiaron
2. Agregar los que son nuevos
3. Eliminar los que ya no existen

Hacer eso manualmente en producción, tres veces al día, es imposible. Para eso existe el patrón que vas a aprender en esta lección.

---

## Los archivos de esta lección

```
lessons/1.2.4/
├── priority_roles.sql               ← Define qué roles son prioritarios
├── priority_jobs_snapshot_INITIAL.sql ← Carga inicial de la tabla snapshot
└── priority_jobs_snapshot.sql       ← Actualiza el snapshot (el MERGE)
```

**Orden de ejecución obligatorio:**
```
1. priority_roles.sql
2. priority_jobs_snapshot_INITIAL.sql   (solo la primera vez)
3. priority_jobs_snapshot.sql           (cada vez que quieras actualizar)
```

---

## Paso 1 — La tabla de roles prioritarios

**Archivo:** `priority_roles.sql`

```sql
CREATE OR REPLACE TABLE staging.priority_roles (
    role_id      INTEGER PRIMARY KEY,
    role_name    VARCHAR,
    priority_lvl INTEGER
);

INSERT INTO staging.priority_roles (role_id, role_name, priority_lvl)
VALUES
    (1, 'Data Engineer',        2),
    (2, 'Senior Data Engineer', 1),
    (3, 'Software Engineer',    3),
    (4, 'Data Scientist',       3);
```

Esta tabla vive en el schema `staging` — un area intermedia donde guardas datos de referencia antes de procesarlos.

**¿Qué significa `priority_lvl`?**

| Nivel | Significado |
|---|---|
| 1 | Prioridad más alta — monitorear siempre |
| 2 | Prioridad media |
| 3 | Prioridad baja |

> **Error común #1:** Si escribes `'Data Scients'` en lugar de `'Data Scientist'`, ese rol nunca aparecerá en el snapshot. El JOIN entre tablas es una comparación exacta de texto — un solo carácter diferente hace que no haya match.

---

## Paso 2 — La carga inicial del snapshot

**Archivo:** `priority_jobs_snapshot_INITIAL.sql`

Este archivo se ejecuta **una sola vez** para crear la tabla y cargarla con el estado actual.

```sql
CREATE OR REPLACE TABLE main.priority_jobs_snapshot (
    job_id           INTEGER PRIMARY KEY,
    job_title_short  VARCHAR,
    company_name     VARCHAR,
    job_posted_date  TIMESTAMP,
    salary_year_avg  DOUBLE,
    priority_lvl     INTEGER,
    updated_at       TIMESTAMP        -- registra cuándo cambió el dato
);
```

Luego hace un INSERT combinando tres tablas:

```sql
INSERT INTO main.priority_jobs_snapshot (...)
SELECT
    jpf.job_id,
    jpf.job_title_short,
    cd.name          AS company_name,
    jpf.job_posted_date,
    jpf.salary_year_avg,
    r.priority_lvl,
    CURRENT_TIMESTAMP                 -- momento en que se cargó
FROM data_jobs.job_postings_fact AS jpf
LEFT JOIN data_jobs.company_dim AS cd
    ON jpf.company_id = cd.company_id
INNER JOIN staging.priority_roles AS r
    ON jpf.job_title_short = r.role_name;
```

**¿Por qué LEFT JOIN con company_dim pero INNER JOIN con priority_roles?**

- `LEFT JOIN company_dim` → queremos todos los empleos aunque no tengan empresa registrada. Si no hay empresa, el campo queda NULL pero el empleo entra igual.
- `INNER JOIN priority_roles` → solo queremos empleos de roles que están en nuestra lista. Si el rol no está en la lista, el empleo no nos interesa y lo descartamos.

---

## Paso 3 — Actualizar el snapshot

**Archivo:** `priority_jobs_snapshot.sql`

Aquí viene el concepto central de la lección. Cada vez que quieres actualizar el snapshot, primero necesitas saber cuál es el estado actual de la fuente. Para eso creas una tabla temporal:

### 3a. La tabla temporal (source)

```sql
CREATE OR REPLACE TEMP TABLE src_priority_jobs AS
SELECT
    jpf.job_id,
    jpf.job_title_short,
    cd.name          AS company_name,
    jpf.job_posted_date,
    jpf.salary_year_avg,
    r.priority_lvl,
    CURRENT_TIMESTAMP AS updated_at
FROM data_jobs.job_postings_fact AS jpf
LEFT JOIN data_jobs.company_dim AS cd
    ON jpf.company_id = cd.company_id
INNER JOIN staging.priority_roles AS r
    ON jpf.job_title_short = r.role_name;
```

Piensa en esta tabla temporal como **una foto del estado actual**. La vas a comparar contra tu snapshot para ver qué cambió.

> **Error común #2:** El MERGE necesita que `src_priority_jobs` exista antes de ejecutarse. Si está comentada, el MERGE falla con "tabla no encontrada".

---

## Estrategia A — Tres statements separados

Antes de que existiera el MERGE en SQL, esto se hacía con tres operaciones independientes. Es importante entenderlas porque son la base del MERGE.

### UPDATE — Actualizar lo que cambió

```sql
UPDATE main.priority_jobs_snapshot AS tgt
SET
    priority_lvl = src.priority_lvl,
    updated_at   = src.updated_at
FROM src_priority_jobs AS src
WHERE tgt.job_id = src.job_id
  AND tgt.priority_lvl IS DISTINCT FROM src.priority_lvl;
```

**¿Por qué `IS DISTINCT FROM` en lugar de `!=`?**

Con `!=`, si alguno de los valores es NULL, la comparación devuelve NULL (no true ni false), y la fila se ignora silenciosamente. `IS DISTINCT FROM` maneja los NULLs correctamente: considera que dos NULLs son iguales y que NULL vs un valor son distintos.

### INSERT — Agregar lo que es nuevo

```sql
INSERT INTO main.priority_jobs_snapshot (...)
SELECT src.*
FROM src_priority_jobs AS src
WHERE NOT EXISTS (
    SELECT 1
    FROM main.priority_jobs_snapshot AS tgt
    WHERE tgt.job_id = src.job_id
);
```

Solo inserta filas del source que no existen en el snapshot. El `SELECT 1` es un truco de performance: no necesitas traer ninguna columna, solo verificar si existe al menos una fila.

### DELETE — Eliminar lo que ya no existe

```sql
DELETE FROM main.priority_jobs_snapshot AS tgt
WHERE NOT EXISTS (
    SELECT 1
    FROM src_priority_jobs AS src
    WHERE src.job_id = tgt.job_id
);
```

Elimina del snapshot los empleos que ya no están en el source. Esto mantiene la tabla sincronizada con la realidad.

**El problema con esta estrategia:** Son tres statements que deben ejecutarse en orden, en la misma transacción, sin errores. Si uno falla a la mitad, los datos quedan en un estado inconsistente. Además, el código es largo y repetitivo.

---

## Estrategia B — MERGE INTO (la forma moderna)

El MERGE reemplaza los tres statements anteriores en uno solo, atómico y más legible:

```sql
MERGE INTO main.priority_jobs_snapshot AS tgt
USING src_priority_jobs AS src
ON tgt.job_id = src.job_id

WHEN MATCHED AND tgt.priority_lvl IS DISTINCT FROM src.priority_lvl THEN
    UPDATE SET
        priority_lvl = src.priority_lvl,
        updated_at   = src.updated_at

WHEN NOT MATCHED THEN
    INSERT (job_id, job_title_short, company_name, job_posted_date,
            salary_year_avg, priority_lvl, updated_at)
    VALUES (src.job_id, src.job_title_short, src.company_name,
            src.job_posted_date, src.salary_year_avg,
            src.priority_lvl, src.updated_at)

WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

### Cómo DuckDB evalúa cada fila

El MERGE recorre cada combinación posible de filas entre source y target:

| ¿Existe en source? | ¿Existe en target? | ¿priority_lvl cambió? | Acción |
|---|---|---|---|
| Sí | Sí | Sí | UPDATE |
| Sí | Sí | No | Sin cambio |
| Sí | No | — | INSERT |
| No | Sí | — | DELETE |

### ¿Qué significa cada cláusula?

**`WHEN MATCHED`** → el `job_id` existe en ambas tablas (fila conocida).
La condición extra `AND tgt.priority_lvl IS DISTINCT FROM src.priority_lvl` restringe el UPDATE solo a filas donde algo cambió. Si el nivel de prioridad es el mismo, no se toca la fila.

**`WHEN NOT MATCHED`** → equivale a "NOT MATCHED BY TARGET": el `job_id` está en el source pero no en el snapshot. Es un empleo nuevo que hay que insertar.

**`WHEN NOT MATCHED BY SOURCE`** → el `job_id` está en el snapshot pero ya no está en el source. Es un empleo que desapareció y hay que eliminar.

---

## El comportamiento de `updated_at`

Esta columna registra **cuándo cambió el dato**, no cuándo se ejecutó el pipeline.

```
Primera carga (ayer):   Senior Data Engineer → priority_lvl=1, updated_at=ayer
Cambias priority_lvl a 3, ejecutas MERGE hoy:
  → El MATCHED detecta el cambio → UPDATE → updated_at=hoy ✅

Si ejecutas el MERGE otra vez sin cambiar nada:
  → El MATCHED no detecta cambio → sin acción → updated_at sigue siendo hoy ✅
```

Esto es comportamiento correcto. Si quisieras que `updated_at` siempre refleje la última ejecución del pipeline (no el último cambio del dato), necesitarías un segundo `WHEN MATCHED` sin condición — pero eso es un patrón diferente con otro significado semántico.

---

## Resumen — ¿Qué aprendiste?

| Concepto | Qué es |
|---|---|
| Snapshot table | Foto del estado actual de los datos, actualizable |
| Tabla temporal | Estado actual del source, para comparar contra el snapshot |
| INNER vs LEFT JOIN | INNER filtra, LEFT preserva todos los registros del lado izquierdo |
| IS DISTINCT FROM | Comparación segura que maneja NULLs correctamente |
| MERGE INTO | Sincroniza dos tablas en un solo statement atómico |
| updated_at | Registra cuándo cambió el dato, no cuándo corrió el pipeline |

---

## ¿Por qué importa esto en un proyecto real de Data Engineering?

Este patrón se usa en producción todos los días:

- **Pipelines de datos:** Los ETL que corren cada hora o cada día usan MERGE para mantener sincronizadas las tablas de hechos y dimensiones.
- **SCD Tipo 1 (Slowly Changing Dimension):** Es el nombre formal de lo que hiciste: una dimensión que se actualiza in-place cuando los datos cambian.
- **Idempotencia:** Si corres el MERGE dos veces con los mismos datos, el resultado es el mismo. Esto es crítico en producción porque los pipelines fallan y se re-ejecutan.
- **Herramientas como dbt:** Cuando usas `dbt` (la herramienta estándar de transformación de datos), el modo `incremental` con `unique_key` genera exactamente este patrón MERGE por debajo.

En empresas como Spotify, Uber o cualquier startup con un data warehouse, hay cientos de tablas que se mantienen con este mismo patrón corriendo en Snowflake, BigQuery o DuckDB.
