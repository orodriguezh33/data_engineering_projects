# Lesson 1.27 — Set Operators: UNION, INTERSECT y EXCEPT

## ¿Qué son los Set Operators?

Los JOINs combinan tablas **horizontalmente** (agregan columnas). Los Set Operators combinan resultados de queries **verticalmente** (agregan filas).

Piensa en ellos como operaciones de conjuntos matemáticos:

```
Query A = {1, 1, 1, 2}
Query B = {1, 1, 3}

UNION          → valores únicos de A y B combinados
UNION ALL      → todos los valores de A y B, con duplicados
INTERSECT      → solo los valores que aparecen en ambos (únicos)
INTERSECT ALL  → los valores comunes, preservando multiplicidad
EXCEPT         → valores de A que no están en B (únicos)
EXCEPT ALL     → valores de A que no están en B, restando uno a uno
```

**Regla obligatoria:** Las queries combinadas deben tener el **mismo número de columnas** y tipos compatibles.

---

## La diferencia entre versión normal y ALL

Esta es la distinción más importante y más confundida de los set operators:

- **Versión normal** (UNION, INTERSECT, EXCEPT): elimina duplicados del resultado, como si aplicara un `DISTINCT`
- **Versión ALL** (UNION ALL, INTERSECT ALL, EXCEPT ALL): preserva todos los duplicados

Veamos con números concretos:

```sql
-- A = [1, 1, 1, 2]   B = [1, 1, 3]

SELECT UNNEST([1,1,1,2]) UNION     SELECT UNNEST([1,1,3]);
-- resultado: {1, 2, 3}             ← únicos de la unión

SELECT UNNEST([1,1,1,2]) UNION ALL SELECT UNNEST([1,1,3]);
-- resultado: {1, 1, 1, 2, 1, 1, 3} ← todos, sin eliminar nada

SELECT UNNEST([1,1,1,2]) INTERSECT     SELECT UNNEST([1,1,3]);
-- resultado: {1}                    ← el valor común, sin duplicados

SELECT UNNEST([1,1,1,2]) INTERSECT ALL SELECT UNNEST([1,1,3]);
-- resultado: {1, 1}                 ← mínimo de apariciones: min(3,2)=2

SELECT UNNEST([1,1,1,2]) EXCEPT     SELECT UNNEST([1,1,3]);
-- resultado: {2}                    ← en A pero no en B, sin duplicados

SELECT UNNEST([1,1,1,2]) EXCEPT ALL SELECT UNNEST([1,1,3]);
-- resultado: {1, 2}                 ← resta uno a uno: 3 unos en A - 2 unos en B = 1 uno
```

> `UNNEST([1,1,1,2])` convierte una lista en filas. Es la forma de DuckDB de crear datos de ejemplo rápidamente sin una tabla real.

---

## Aplicación práctica — Comparando empleos por año

La lección crea dos tablas temporales para comparar el mercado laboral entre 2023 y 2024:

```sql
CREATE OR REPLACE TEMP TABLE jobs_2023 AS
SELECT * EXCLUDE(job_id, job_posted_date)
FROM job_postings_fact
WHERE EXTRACT(YEAR FROM job_posted_date) = 2023;

CREATE OR REPLACE TEMP TABLE jobs_2024 AS
SELECT * EXCLUDE(job_id, job_posted_date)
FROM job_postings_fact
WHERE EXTRACT(YEAR FROM job_posted_date) = 2024;
```

`* EXCLUDE(columnas)` es una sintaxis de DuckDB que selecciona todas las columnas **excepto** las indicadas. Se excluyen `job_id` y `job_posted_date` para que los set operators comparen el contenido del empleo (título, empresa, salario, skills), no su identidad o cuándo fue publicado.

---

## UNION — Combinar sin duplicados

```sql
-- Resumen de cuántos registros tiene cada año
SELECT 'jobs_2023' AS table_name, COUNT(*) AS record_count FROM jobs_2023
UNION
SELECT 'jobs_2024' AS table_name, COUNT(*) AS record_count FROM jobs_2024;
```

Devuelve dos filas: el conteo de 2023 y el conteo de 2024. El UNION aquí no elimina duplicados porque los valores son distintos — pero si dos años tuvieran el mismo conteo exacto, uno desaparecería. Por eso para resúmenes de este tipo es mejor usar `UNION ALL`.

```sql
-- Todos los empleos únicos de ambos años combinados
SELECT * FROM jobs_2023
UNION
SELECT * FROM jobs_2024;
```

Si el mismo empleo (mismo título, empresa, salario) apareció en 2023 y en 2024, aparece una sola vez. Útil para obtener el catálogo único de tipos de empleos del mercado.

**Cuándo usar UNION:** Cuando quieres combinar datos de múltiples fuentes o períodos en una sola tabla sin repetir filas idénticas. Por ejemplo, combinar empleos de distintas regiones en un solo dataset.

---

## UNION ALL — Combinar con duplicados

```sql
SELECT 'jobs_2023' AS table_name, COUNT(*) AS record_count FROM jobs_2023
UNION ALL
SELECT 'jobs_2024' AS table_name, COUNT(*) AS record_count FROM jobs_2024;
```

Devuelve todas las filas de ambas queries, sin eliminar nada. Es significativamente más rápido que `UNION` porque no necesita hacer el paso de deduplicación.

**Cuándo usar UNION ALL:** Casi siempre. En pipelines de datos, cuando combinas datos de múltiples particiones o fuentes, usas `UNION ALL` porque:
1. Es más rápido (no hay deduplicación)
2. Si hay duplicados, quieres saberlo (no esconderlos)
3. La deduplicación la manejas explícitamente después si la necesitas

---

## EXCEPT — Lo que está en A pero no en B

```sql
-- ¿Cuántos registros únicos hay en 2023 que no aparecen en 2024?
SELECT 'jobs_2023' AS table_name, COUNT(*) AS record_count FROM jobs_2023
EXCEPT
SELECT 'jobs_2024' AS table_name, COUNT(*) AS record_count FROM jobs_2024;
```

```sql
-- ¿Cuántos registros hay en 2023 que no aparecen en 2024, contando uno a uno?
SELECT 'jobs_2023' AS table_name, COUNT(*) AS record_count FROM jobs_2023
EXCEPT ALL
SELECT 'jobs_2024' AS table_name, COUNT(*) AS record_count FROM jobs_2024;
```

**EXCEPT** es útil para encontrar qué dejó de existir: empleos que había en 2023 pero ya no en 2024, clientes que cancelaron, productos descontinuados.

**La diferencia entre EXCEPT y EXCEPT ALL:**
- `EXCEPT`: si un valor aparece en ambas tablas (aunque sea una vez), desaparece del resultado
- `EXCEPT ALL`: resta ocurrencias una a una. Si el valor aparece 3 veces en A y 1 vez en B, queda 2 veces en el resultado

---

## INTERSECT — Lo que está en ambos

```sql
-- ¿Cuántos registros aparecieron en ambos años?
SELECT 'jobs_2023' AS table_name, COUNT(*) AS record_count FROM jobs_2023
INTERSECT
SELECT 'jobs_2024' AS table_name, COUNT(*) AS record_count FROM jobs_2024;

-- Preservando conteos duplicados
SELECT 'jobs_2023' AS table_name, COUNT(*) AS record_count FROM jobs_2023
INTERSECT ALL
SELECT 'jobs_2024' AS table_name, COUNT(*) AS record_count FROM jobs_2024;
```

**INTERSECT** es útil para encontrar qué persiste: empleos que existían en ambos años, clientes que compraron dos veces, productos que siguen en el catálogo.

---

## Tabla de referencia rápida

| Operador | Duplicados | Qué devuelve | Velocidad |
|---|---|---|---|
| `UNION` | Elimina | Todo de A + todo de B, únicos | Media |
| `UNION ALL` | Preserva | Todo de A + todo de B, con repeticiones | Rápida |
| `INTERSECT` | Elimina | Solo lo que está en A y en B | Media |
| `INTERSECT ALL` | Preserva | Lo común, respetando multiplicidad | Media |
| `EXCEPT` | Elimina | Lo de A que no está en B | Media |
| `EXCEPT ALL` | Preserva | Lo de A que no está en B, uno a uno | Media |

---

## ¿Por qué importa en un proyecto real?

- **UNION ALL en pipelines:** El patrón más común en ETL es cargar datos de múltiples fuentes (archivos por mes, por región, por sistema) con `UNION ALL` antes de transformarlos. En dbt esto se ve como múltiples `ref()` combinados con `UNION ALL`.

- **EXCEPT para detección de cambios:** `EXCEPT` es una forma simple de comparar dos snapshots y detectar qué filas cambiaron o desaparecieron — similar a lo que hace el MERGE, pero solo para leer, no para escribir.

- **INTERSECT para validación de datos:** "¿Qué registros de mi sistema A también existen en el sistema B?" es una pregunta de reconciliación de datos que se responde con `INTERSECT`.

- **UNION para consolidar catálogos:** Cuando tienes múltiples sistemas con listas de productos, clientes o cuentas, `UNION` consolida el catálogo único sin duplicados.
