# Lesson 1.3 — Subqueries y CTEs

## ¿Por qué necesitas queries dentro de queries?

SQL es poderoso, pero a veces una sola query no alcanza. Necesitas responder preguntas en etapas:

- Primero: ¿cuál es el salario mediano del mercado?
- Luego: ¿qué empleos pagan por encima de ese mediano?

No puedes hacer las dos cosas en un solo `SELECT` directo. Necesitas calcular el mediano primero y luego usarlo. Para eso existen las subqueries y los CTEs.

---

## Subqueries — Una query dentro de otra

Una subquery es simplemente un `SELECT` envuelto entre paréntesis que se usa como si fuera un valor, una tabla, o una condición.

Pueden vivir en tres lugares distintos:

### 1. Subquery en SELECT (valor escalar)

Calcula un solo valor y lo añade como columna a cada fila:

```sql
SELECT
    job_title_short,
    salary_year_avg,
    (
        SELECT MEDIAN(salary_year_avg)
        FROM job_postings_fact
    ) AS market_median_salary
FROM job_postings_fact
WHERE salary_year_avg IS NOT NULL
LIMIT 10;
```

Esto muestra el salario de cada empleo junto al mediano del mercado completo en la misma fila. La subquery se ejecuta una sola vez y el resultado se repite en cada fila.

**Cuándo usarla:** Para añadir un valor de referencia global (un total, un promedio, un máximo) a cada fila, sin hacer un JOIN.

### 2. Subquery en FROM (tabla derivada)

Filtra o transforma datos primero, y luego haces la query principal sobre ese resultado:

```sql
SELECT
    job_title_short,
    MEDIAN(salary_year_avg) AS median_salary
FROM (
    SELECT
        job_title_short,
        salary_year_avg
    FROM job_postings_fact
    WHERE job_work_from_home = TRUE   -- primero filtra solo remotos
) AS clean_jobs                       -- el resultado tiene un alias
GROUP BY job_title_short
ORDER BY median_salary DESC;
```

La subquery en el FROM actúa como una tabla temporal sin nombre. Primero filtra, y luego la query exterior agrega sobre esos datos ya filtrados.

**Cuándo usarla:** Cuando necesitas transformar o filtrar los datos antes de poder agregar o unir.

### 3. Subquery en HAVING (condición de filtro de grupos)

Filtra grupos según un valor calculado:

```sql
SELECT
    job_title_short,
    MEDIAN(salary_year_avg) AS median_salary
FROM (
    SELECT job_title_short, salary_year_avg
    FROM job_postings_fact
    WHERE job_work_from_home = TRUE
) AS clean_jobs
GROUP BY job_title_short
HAVING MEDIAN(salary_year_avg) > (
    SELECT MEDIAN(salary_year_avg)    -- mediano del mercado remoto completo
    FROM job_postings_fact
    WHERE job_work_from_home = TRUE
)
ORDER BY median_salary DESC;
```

El `HAVING` filtra después de agrupar. Aquí: "muéstrame solo los títulos cuyo mediano salarial esté por encima del mediano general del mercado remoto".

**Cuándo usarla:** Para filtrar grupos con condiciones que dependen de un valor calculado por separado.

---

## EXISTS y NOT EXISTS — Subqueries de verificación

En lugar de traer datos, estas subqueries solo verifican si existe al menos una fila que cumpla una condición:

```sql
-- Empleos QUE SÍ tienen skills asociadas
SELECT *
FROM job_postings_fact AS j
WHERE EXISTS (
    SELECT 1
    FROM skills_job_dim AS s
    WHERE j.job_id = s.job_id
);

-- Empleos QUE NO tienen skills asociadas (datos incompletos)
SELECT *
FROM job_postings_fact AS j
WHERE NOT EXISTS (
    SELECT 1
    FROM skills_job_dim AS s
    WHERE j.job_id = s.job_id
);
```

El `SELECT 1` es intencional — no necesitas traer ninguna columna real, solo confirmar si existe al menos una fila. Es más eficiente que un JOIN cuando solo necesitas verificar existencia.

**Cuándo usarlas en producción:** Para validar calidad de datos antes de cargar al warehouse. "¿Hay empleos sin empresa? ¿Hay transacciones sin cliente? ¿Hay pedidos sin producto?" Son chequeos de integridad esenciales en cualquier pipeline.

---

## El problema con las Subqueries anidadas

Las subqueries se vuelven difíciles de leer cuando se anidan:

```sql
SELECT *
FROM (
    SELECT *
    FROM (
        SELECT *
        FROM tabla
        WHERE condicion1
    ) AS nivel1
    WHERE condicion2
) AS nivel2
WHERE condicion3;
```

A partir del segundo o tercer nivel, nadie entiende qué está pasando — incluyendo tú mismo seis meses después. Para eso existen los CTEs.

---

## CTEs — Common Table Expressions

Un CTE es como darle un nombre a una subquery para poder referenciarla más fácilmente. La sintaxis usa `WITH`:

```sql
WITH valid_salaries AS (
    SELECT *
    FROM job_postings_fact
    WHERE salary_year_avg IS NOT NULL
       OR salary_hour_avg IS NOT NULL
)
SELECT * FROM valid_salaries LIMIT 10;
```

Esto es **equivalente** a:

```sql
SELECT *
FROM (
    SELECT *
    FROM job_postings_fact
    WHERE salary_year_avg IS NOT NULL
       OR salary_hour_avg IS NOT NULL
) LIMIT 10;
```

Pero el CTE es mucho más legible. Le das un nombre descriptivo a la lógica intermedia.

### CTEs múltiples y autorreferencia

El poder real de los CTEs aparece cuando encadenas varios:

```sql
WITH title_median AS (
    SELECT
        job_title_short,
        job_work_from_home,
        MEDIAN(salary_year_avg)::INT AS market_median_salary
    FROM job_postings_fact
    WHERE job_country = 'Colombia'
    GROUP BY job_title_short, job_work_from_home
)
SELECT
    r.job_title_short,
    r.market_median_salary AS remote_median_salary,
    o.market_median_salary AS onsite_median_salary,
    (r.market_median_salary - o.market_median_salary) AS remote_premium
FROM title_median AS r          -- el mismo CTE usado dos veces
INNER JOIN title_median AS o    -- una vez como remoto, otra como presencial
    ON r.job_title_short = o.job_title_short
WHERE r.job_work_from_home = TRUE
  AND o.job_work_from_home = FALSE
ORDER BY remote_premium DESC;
```

Esto responde: "¿Cuánto más pagan los trabajos remotos vs presenciales por título, en Colombia?"

El truco elegante: el CTE `title_median` se usa dos veces en el mismo query — una vez para los roles remotos (`r`) y otra para los presenciales (`o`). Esto sería imposible de escribir limpiamente con subqueries anidadas.

El `::INT` es un CAST abreviado: convierte el DOUBLE del MEDIAN a INTEGER para mostrar sin decimales.

---

## Subquery vs CTE — ¿Cuándo usar cada uno?

| | Subquery | CTE |
|---|---|---|
| Legibilidad | Difícil cuando se anida | Mucho más clara |
| Reutilización | No (tienes que repetirla) | Sí (la defines una vez, la usas varias) |
| Debugging | Difícil de aislar | Puedes consultar el CTE solo |
| Performance | Similar | Similar (en la mayoría de motores) |
| Cuándo usar | Casos simples y rápidos | Lógica compleja o que se reutiliza |

**Regla práctica:** Si la subquery tiene más de 5 líneas o la necesitas más de una vez en el mismo query, conviértela en CTE.

---

## ¿Por qué importa en un proyecto real?

- **CTEs son la base de dbt:** Cada modelo de dbt se construye encadenando CTEs. El estándar en la industria es tener un CTE por cada transformación lógica, con nombres descriptivos como `stg_orders`, `filtered_active_users`, `aggregated_revenue`.
- **NOT EXISTS para validación de datos:** Antes de cargar al data warehouse, los pipelines profesionales tienen una capa de validación que usa `NOT EXISTS` para detectar registros huérfanos o incompletos.
- **Subqueries en SELECT para métricas de referencia:** Dashboards que muestran "tu número vs el promedio del mercado" usan exactamente el patrón de subquery en SELECT.
- **Performance:** En motores como Snowflake o BigQuery, los CTEs bien escritos permiten al optimizador de queries generar planes de ejecución más eficientes. Un CTE mal escrito puede materializar demasiados datos en memoria.
