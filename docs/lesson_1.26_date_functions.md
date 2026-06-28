# Lesson 1.26 — Date Functions: Trabajando con fechas y tiempo

## ¿Por qué las fechas son especiales en Data Engineering?

Las fechas son el eje de casi todo análisis de negocio: tendencias mensuales, comparaciones año a año, análisis de temporada, métricas de retención. Pero los datos de fechas vienen en formatos inconsistentes, con zonas horarias mezcladas, y los motores SQL los tratan de formas que no siempre son intuitivas.

Dominar las funciones de fecha te permite:
- Extraer componentes específicos (año, mes, hora)
- Agrupar por períodos (semana, mes, trimestre)
- Convertir entre zonas horarias correctamente
- Estandarizar formatos inconsistentes

---

## Los tipos de fecha en DuckDB

Un mismo valor de fecha puede representarse de formas distintas según lo que necesites:

```sql
SELECT
    job_posted_date,                          -- TIMESTAMP original
    job_posted_date::DATE      AS date,       -- solo la fecha: 2024-03-15
    job_posted_date::TIME      AS time,       -- solo la hora: 14:30:00
    job_posted_date::TIMESTAMP AS timestamp,  -- fecha + hora sin zona
    job_posted_date::TIMESTAMPTZ AS timestampz -- fecha + hora con zona horaria
FROM job_postings_fact
LIMIT 10;
```

El `::` es la sintaxis corta de CAST. `job_posted_date::DATE` es equivalente a `CAST(job_posted_date AS DATE)`.

| Tipo | Guarda | Cuándo usarlo |
|---|---|---|
| `DATE` | Solo fecha | Cuando la hora no importa (cumpleaños, fechas de publicación) |
| `TIME` | Solo hora | Raro en Data Engineering |
| `TIMESTAMP` | Fecha + hora sin zona | Datos locales o cuando la zona ya es consistente |
| `TIMESTAMPTZ` | Fecha + hora con zona | Sistemas globales con usuarios en distintas zonas horarias |

---

## EXTRACT — Sacar componentes de una fecha

Extrae un componente específico (año, mes, día, hora) como número:

```sql
SELECT
    EXTRACT(YEAR  FROM job_posted_date) AS job_posted_year,
    EXTRACT(MONTH FROM job_posted_date) AS job_posted_month,
    COUNT(job_id)                       AS job_count
FROM job_postings_fact
WHERE job_title_short = 'Data Engineer'
  AND job_country = 'Colombia'
GROUP BY
    EXTRACT(YEAR  FROM job_posted_date),
    EXTRACT(MONTH FROM job_posted_date)
ORDER BY job_posted_year ASC, job_posted_month ASC;
```

Esto responde: "¿Cuántos empleos de Data Engineer se publicaron en Colombia por mes y año?"

**Partes que puedes extraer:**

| Componente | Ejemplo para `2024-03-15 14:30:00` |
|---|---|
| `YEAR` | `2024` |
| `MONTH` | `3` |
| `DAY` | `15` |
| `HOUR` | `14` |
| `MINUTE` | `30` |
| `DOW` | `5` (día de la semana, 0=domingo) |
| `QUARTER` | `1` |

> **Importante:** El resultado de EXTRACT es un número (`DOUBLE` en DuckDB). Si necesitas agrupar por él, debes repetir la expresión en el `GROUP BY`, no puedes usar el alias.

---

## DATE_TRUNC — Truncar a un período

Mientras `EXTRACT` saca un número, `DATE_TRUNC` devuelve una fecha truncada al inicio del período indicado. Es la función más usada para análisis de series de tiempo:

```sql
SELECT
    job_posted_date,
    DATE_TRUNC('year',    job_posted_date) AS job_posted_year,    -- 2024-01-01 00:00:00
    DATE_TRUNC('quarter', job_posted_date) AS job_posted_quarter,  -- 2024-01-01 00:00:00
    DATE_TRUNC('month',   job_posted_date) AS job_posted_month,    -- 2024-03-01 00:00:00
    DATE_TRUNC('week',    job_posted_date) AS job_posted_week,     -- 2024-03-11 00:00:00
    DATE_TRUNC('day',     job_posted_date) AS job_posted_day,      -- 2024-03-15 00:00:00
    DATE_TRUNC('hour',    job_posted_date) AS job_posted_hour      -- 2024-03-15 14:00:00
FROM job_postings_fact
ORDER BY RANDOM()
LIMIT 10;
```

Para marzo 15, 2024:
- `DATE_TRUNC('year', ...)` → `2024-01-01` (primer día del año)
- `DATE_TRUNC('month', ...)` → `2024-03-01` (primer día del mes)
- `DATE_TRUNC('week', ...)` → `2024-03-11` (lunes de esa semana)

### EXTRACT vs DATE_TRUNC — ¿Cuándo usar cada uno?

```sql
-- EXTRACT: para agrupar y filtrar por componente numérico
GROUP BY EXTRACT(MONTH FROM job_posted_date)  -- agrupa todos los meses de marzo juntos

-- DATE_TRUNC: para series de tiempo donde el orden cronológico importa
GROUP BY DATE_TRUNC('month', job_posted_date)  -- agrupa marzo-2023 separado de marzo-2024
```

Esta diferencia es crítica: si usas `EXTRACT(MONTH)`, agrupas todos los meses de marzo de todos los años juntos. Si usas `DATE_TRUNC('month')`, cada mes de cada año es un grupo separado — que es lo que quieres en una gráfica de tendencias.

```sql
-- Tendencia mensual de empleos en Colombia 2024
SELECT
    DATE_TRUNC('month', job_posted_date) AS job_posted_month,
    COUNT(job_id) AS job_count
FROM job_postings_fact
WHERE job_title_short = 'Data Engineer'
  AND EXTRACT(YEAR FROM job_posted_date) = 2024
  AND job_country = 'Colombia'
GROUP BY DATE_TRUNC('month', job_posted_date)
ORDER BY job_posted_month ASC;
```

---

## Zonas horarias — El tema más confuso de las fechas

Las zonas horarias son la fuente de bugs más silenciosa en Data Engineering. Un evento que ocurrió a las `23:00 UTC` puede aparecer como el día siguiente en `CST` (UTC-6).

### Cómo convertir entre zonas horarias

```sql
-- Convertir de UTC a CST (Central Standard Time, UTC-6)
SELECT
    job_title_short,
    job_location,
    job_posted_date AT TIME ZONE 'UTC' AT TIME ZONE 'CST'
FROM job_postings_fact
WHERE job_location = 'New York, NY'
LIMIT 10;
```

La doble conversión `AT TIME ZONE 'UTC' AT TIME ZONE 'CST'` funciona así:
1. `AT TIME ZONE 'UTC'` → le dice a DuckDB que el valor original está en UTC
2. `AT TIME ZONE 'CST'` → convierte ese valor UTC a la hora local de CST

### Ejemplo práctico — ¿A qué hora del día se publican más empleos en NY?

```sql
SELECT
    EXTRACT(HOUR FROM job_posted_date AT TIME ZONE 'UTC' AT TIME ZONE 'CST') AS job_posted_hour,
    COUNT(job_id) AS job_count
FROM job_postings_fact
WHERE job_location = 'New York, NY'
GROUP BY
    EXTRACT(HOUR FROM job_posted_date AT TIME ZONE 'UTC' AT TIME ZONE 'CST')
ORDER BY job_posted_hour DESC;
```

Sin la conversión, analizarías las horas en UTC y los resultados no reflejarían el comportamiento real de los usuarios en su zona horaria local.

---

## Reglas prácticas para trabajar con fechas

**1. Guarda siempre en UTC en producción**
Los sistemas globales guardan todo en UTC. La conversión a zona local se hace al momento de mostrar, no al guardar.

**2. Usa DATE_TRUNC para series de tiempo, EXTRACT para filtros**
```sql
WHERE EXTRACT(YEAR FROM fecha) = 2024          -- filtrar por año
GROUP BY DATE_TRUNC('month', fecha)            -- agrupar cronológicamente
```

**3. Verifica el tipo antes de operar**
Si tu columna de fecha es `VARCHAR`, las comparaciones y truncaciones no funcionarán. Convierte primero:
```sql
CAST(fecha_texto AS DATE) >= '2024-01-01'
```

---

## ¿Por qué importa en un proyecto real?

- **Series de tiempo:** El 80% de los análisis de negocio son tendencias a lo largo del tiempo. `DATE_TRUNC` es la función que construye esas series.
- **Pipelines incrementales:** Los ETL que corren diariamente usan fechas para saber qué datos son nuevos: `WHERE fecha >= CURRENT_DATE - INTERVAL '1 day'`.
- **Zonas horarias en sistemas globales:** Un pipeline que no maneja zonas horarias correctamente puede duplicar o perder eventos que ocurren en la transición de medianoche UTC.
- **Particionamiento:** En Snowflake y BigQuery, las tablas grandes se particionan por fecha. Las queries que filtran por `DATE_TRUNC` o `EXTRACT` aprovechan ese particionamiento y son órdenes de magnitud más rápidas y baratas.
