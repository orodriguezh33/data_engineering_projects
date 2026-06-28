# Dominando el MERGE — De cero a experto

## El problema que MERGE resuelve

Antes de ver una sola línea de código, necesitas entender el problema.

Tienes dos tablas:

**Target** — tu tabla en producción, la que ya existe con datos históricos:

```
job_id | job_title_short      | priority_lvl | updated_at
-------|----------------------|--------------|------------
1001   | Data Engineer        | 2            | 2026-01-01
1002   | Software Engineer    | 3            | 2026-01-01
1003   | Senior Data Engineer | 1            | 2026-01-01
```

**Source** — el estado actual que viene de la fuente de datos (hoy):

```
job_id | job_title_short      | priority_lvl | updated_at
-------|----------------------|--------------|------------
1001   | Data Engineer        | 1            | 2026-06-25  ← cambió de 2 a 1
1003   | Senior Data Engineer | 1            | 2026-06-25  ← sin cambio
1004   | Data Scientist       | 3            | 2026-06-25  ← nuevo
```

¿Qué necesitas hacer para que el Target quede sincronizado con el Source?

- `1001` cambió de prioridad → **actualizar**
- `1002` ya no existe en el source → **eliminar**
- `1003` sigue igual → **no tocar**
- `1004` es nuevo → **insertar**

Ese es exactamente el trabajo del MERGE: **sincronizar dos tablas en un solo statement**.

---

## La anatomía del MERGE

```sql
MERGE INTO [tabla_destino] AS tgt      -- 1. ¿A qué tabla voy a escribir?
USING [tabla_fuente] AS src            -- 2. ¿De dónde vienen los datos nuevos?
ON [condición de join]                 -- 3. ¿Cómo reconozco si una fila ya existe?

WHEN MATCHED [AND condición] THEN      -- 4. Existe en ambas → UPDATE o DELETE
    UPDATE SET ...

WHEN NOT MATCHED [BY TARGET] THEN      -- 5. Existe en source, no en target → INSERT
    INSERT (...) VALUES (...)

WHEN NOT MATCHED BY SOURCE THEN        -- 6. Existe en target, no en source → DELETE
    DELETE;
```

Cada sección tiene un propósito específico. Las vamos a ver una por una.

---

## Sección 1 — MERGE INTO (el destino)

```sql
MERGE INTO main.priority_jobs_snapshot AS tgt
```

`tgt` es el alias de la tabla destino. En el resto del statement, cuando escribas `tgt.columna` estás haciendo referencia a un valor que ya existe en la tabla, el valor "viejo".

**Importante:** El MERGE **modifica** esta tabla. No crea una nueva. Es una operación de escritura directa sobre los datos existentes.

---

## Sección 2 — USING (la fuente)

```sql
USING src_priority_jobs AS src
```

La fuente puede ser:

- Una tabla permanente
- Una tabla temporal (`TEMP TABLE`)
- Una subquery directa

Por ejemplo, esto también es válido:

```sql
MERGE INTO main.priority_jobs_snapshot AS tgt
USING (
    SELECT jpf.job_id, r.priority_lvl, CURRENT_TIMESTAMP AS updated_at
    FROM data_jobs.job_postings_fact AS jpf
    INNER JOIN staging.priority_roles AS r
        ON jpf.job_title_short = r.role_name
) AS src
ON tgt.job_id = src.job_id
...
```

Usar una tabla temporal (como en la lección) es mejor práctica porque puedes inspeccionarla antes de ejecutar el MERGE y verificar que el source tiene los datos correctos.

---

## Sección 3 — ON (la llave de comparación)

```sql
ON tgt.job_id = src.job_id
```

Esta es la pregunta que el MERGE hace por cada fila: **¿existe esta fila en ambas tablas?**

El `ON` funciona como el `ON` de un JOIN — define cómo emparejar filas del source con filas del target.

**Regla crítica:** La columna que uses en el `ON` debe identificar cada fila de forma única. Si usas una columna que puede repetirse, el MERGE puede actualizar filas incorrectas o lanzar un error.

```sql
-- Mal: job_title_short puede repetirse, un mismo título tiene miles de filas
ON tgt.job_title_short = src.job_title_short

-- Bien: job_id es la primary key, identifica exactamente una fila
ON tgt.job_id = src.job_id
```

---

## Sección 4 — WHEN MATCHED (fila existe en ambos lados)

```sql
WHEN MATCHED AND tgt.priority_lvl IS DISTINCT FROM src.priority_lvl THEN
    UPDATE SET
        priority_lvl = src.priority_lvl,
        updated_at   = src.updated_at
```

**¿Cuándo aplica?** Cuando el `ON` encontró un match — el `job_id` existe en el target Y en el source.

La condición extra `AND tgt.priority_lvl IS DISTINCT FROM src.priority_lvl` es opcional. Sin ella, el UPDATE se ejecuta para **todas** las filas que hacen match, aunque no hayan cambiado. Con ella, el UPDATE solo se ejecuta si el valor realmente cambió.

### ¿Por qué `IS DISTINCT FROM` y no `!=`?

Esto es sutil pero importante. Considera estos casos:

| tgt.priority_lvl | src.priority_lvl | `!=`   | `IS DISTINCT FROM` |
| ---------------- | ---------------- | -------- | -------------------- |
| 2                | 3                | TRUE ✅  | TRUE ✅              |
| 2                | 2                | FALSE ✅ | FALSE ✅             |
| NULL             | 3                | NULL ❌  | TRUE ✅              |
| NULL             | NULL             | NULL ❌  | FALSE ✅             |

Con `!=`, cualquier comparación que involucre NULL devuelve NULL (que SQL trata como falso), y la fila se ignora silenciosamente. Puedes tener datos desactualizados sin que el MERGE lo detecte.

`IS DISTINCT FROM` es la forma correcta de comparar cuando los valores pueden ser NULL.

### Puedes tener múltiples WHEN MATCHED

```sql
WHEN MATCHED AND tgt.priority_lvl IS DISTINCT FROM src.priority_lvl THEN
    UPDATE SET
        priority_lvl = src.priority_lvl,
        updated_at   = src.updated_at

WHEN MATCHED THEN
    -- fila que coincide pero no cambió → aquí no hacemos nada,
    -- pero si quisieras refrescar updated_at siempre, irías aquí
    UPDATE SET updated_at = src.updated_at
```

**Regla de evaluación:** Las cláusulas se evalúan en orden. La primera que coincide se aplica y las demás se ignoran para esa fila. Es como un `if / else if`.

---

## Sección 5 — WHEN NOT MATCHED (nueva fila en el source)

```sql
WHEN NOT MATCHED THEN
    INSERT (
        job_id, job_title_short, company_name,
        job_posted_date, salary_year_avg, priority_lvl, updated_at
    )
    VALUES (
        src.job_id, src.job_title_short, src.company_name,
        src.job_posted_date, src.salary_year_avg, src.priority_lvl, src.updated_at
    )
```

**¿Cuándo aplica?** Cuando el `ON` NO encontró un match — el `job_id` existe en el source pero **no** en el target. Es una fila nueva que el target desconoce.

`WHEN NOT MATCHED` es equivalente a `WHEN NOT MATCHED BY TARGET`. Son la misma cosa, pero la versión corta es más común.

**Importante:** Aquí solo puedes acceder a columnas del `src`, porque no existe una fila en `tgt` con la que comparar.

---

## Sección 6 — WHEN NOT MATCHED BY SOURCE (fila huérfana en el target)

```sql
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

**¿Cuándo aplica?** Cuando el `ON` NO encontró un match desde el otro lado — el `job_id` existe en el target pero **no** en el source. Es una fila que "desapareció" de la fuente.

Esta cláusula es la más poderosa y también la más peligrosa. **Si tu source está incompleto** (por un error en el pipeline, por ejemplo), esta cláusula puede borrar datos válidos del target.

En producción, muchos equipos evitan `WHEN NOT MATCHED BY SOURCE THEN DELETE` y en su lugar hacen un soft delete:

```sql
WHEN NOT MATCHED BY SOURCE THEN
    UPDATE SET is_active = false, updated_at = CURRENT_TIMESTAMP
```

Así el registro no desaparece, solo queda marcado como inactivo. Puedes recuperarlo si fue un error.

---

## El diagrama mental del MERGE

Imagina que el MERGE hace un FULL OUTER JOIN entre source y target:

```
SOURCE                    TARGET
------                    ------
job_id=1001  ←──match──→  job_id=1001   → WHEN MATCHED
job_id=1004  ←──sin match              → WHEN NOT MATCHED (BY TARGET)
             ←──sin match──  job_id=1002 → WHEN NOT MATCHED BY SOURCE
```

Cada fila del resultado de ese join virtual cae en exactamente una de las tres categorías.

---

## Cómo verificar que el MERGE funcionó

Después de ejecutar el MERGE, siempre deberías verificar el resultado. En DuckDB, puedes usar el `updated_at` como indicador:

```sql
-- Ver qué cambió hoy
SELECT *
FROM main.priority_jobs_snapshot
WHERE DATE(updated_at) = CURRENT_DATE
ORDER BY updated_at DESC;
```

```sql
-- Contar cuántas filas están en cada estado
SELECT
    DATE(updated_at) AS fecha,
    COUNT(*)         AS filas
FROM main.priority_jobs_snapshot
GROUP BY DATE(updated_at)
ORDER BY fecha DESC;
```

Si el source tenía N filas nuevas, el target debería tener N filas más. Si cambiaron M valores, M filas deberían tener `updated_at` de hoy.

---

## Las tres reglas del MERGE que nunca debes olvidar

**Regla 1 — El ON debe usar la llave primaria (o algo equivalentemente único)**
Si no es única, el MERGE puede dar resultados incorrectos o un error. Siempre usa el campo que identifica unívocamente cada registro.

**Regla 2 — El source debe estar completo antes del MERGE**
Si por un bug tu source tiene solo la mitad de las filas que debería, el `WHEN NOT MATCHED BY SOURCE THEN DELETE` va a borrar la otra mitad del target. Valida el source antes de correr el MERGE.

**Regla 3 — El orden de las cláusulas importa**
Las cláusulas `WHEN MATCHED` se evalúan en orden. Pon las más específicas primero. Si inviertes el orden, la condición genérica consume todas las filas y la específica nunca se ejecuta.

---

## MERGE en el mundo real — ¿Dónde lo verás?

| Herramienta             | Cómo usa MERGE                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------- |
| **dbt**           | El modo`incremental` con `unique_key` genera un MERGE por debajo automáticamente |
| **Snowflake**     | `MERGE INTO` es el comando estándar para pipelines de carga incremental            |
| **BigQuery**      | Usa`MERGE` para mantener tablas de dimensiones (SCD)                                |
| **DuckDB**        | Como en esta lección — ideal para pipelines locales o analítica embebida           |
| **Azure Synapse** | `MERGE` para sincronizar staging con producción                                    |

Cuando uses dbt en tu carrera y configures una tabla como `incremental`, dbt genera exactamente este SQL por debajo. Entender el MERGE manual te da la base para depurar cualquier pipeline cuando algo falle.

---

## Resumen visual

```
MERGE INTO target AS tgt
USING source AS src
ON tgt.id = src.id
│
├── id en target Y en source → WHEN MATCHED
│   ├── AND condición true   → UPDATE (solo si algo cambió)
│   └── AND condición false  → sin acción
│
├── id en source, no en target → WHEN NOT MATCHED → INSERT
│
└── id en target, no en source → WHEN NOT MATCHED BY SOURCE → DELETE
```

El MERGE es atómico: o todo funciona, o nada cambia. Eso lo hace seguro para producción.
