# Lesson 1.30 — Window Functions: Análisis sin perder el detalle

## ¿Qué problema resuelven las Window Functions?

Con `GROUP BY` y agregaciones normales, cuando calculas un promedio o un conteo **pierdes el detalle de las filas individuales** — el resultado es una fila por grupo:

```sql
-- Agregación normal: pierdes las filas individuales
SELECT job_title_short, AVG(salary_hour_avg)
FROM job_postings_fact
GROUP BY job_title_short;
-- resultado: 1 fila por título
```

Las Window Functions hacen el mismo cálculo pero **sin colapsar las filas**. Cada fila mantiene su identidad y además recibe el resultado del cálculo:

```sql
-- Window Function: cada fila existe + tiene el promedio de su grupo
SELECT job_id, job_title_short, salary_hour_avg,
       AVG(salary_hour_avg) OVER(PARTITION BY job_title_short) AS avg_by_title
FROM job_postings_fact;
-- resultado: todas las filas, cada una con el promedio de su título
```

Esto permite preguntas que son imposibles con GROUP BY solo: "¿Cuánto paga este empleo comparado con el promedio de su categoría?" o "¿Qué posición ocupa este empleo en el ranking de salarios?"

---

## La anatomía de una Window Function

```sql
función() OVER (
    PARTITION BY columna    -- divide las filas en grupos (opcional)
    ORDER BY columna        -- define el orden dentro de cada grupo (opcional)
)
```

- **`OVER()`** — es lo que convierte una función normal en una window function. Sin OVER, es una agregación; con OVER, es una window function.
- **`PARTITION BY`** — define las "ventanas" (grupos). El cálculo se reinicia para cada partición. Sin PARTITION BY, toda la tabla es una sola ventana.
- **`ORDER BY`** — define el orden dentro de cada ventana. Es obligatorio para rankings y cálculos acumulativos.

---

## Caso 1 — OVER() vacío: toda la tabla como ventana

```sql
-- Agregación: devuelve 1 fila
SELECT COUNT(*) FROM job_postings_fact;

-- Window Function: cada fila tiene el total global
SELECT
    job_id,
    COUNT(*) OVER() AS total_jobs
FROM job_postings_fact;
```

El `OVER()` vacío significa "calcula sobre todas las filas sin particionar". Cada fila del resultado tiene el mismo valor: el total de registros de la tabla. Útil para calcular porcentajes: `COUNT(*) * 100.0 / COUNT(*) OVER()`.

---

## Caso 2 — PARTITION BY: ventanas por grupo

```sql
SELECT
    job_id,
    job_title_short,
    company_id,
    salary_hour_avg,
    AVG(salary_hour_avg) OVER(
        PARTITION BY job_title_short, company_id
        ORDER BY salary_hour_avg DESC
    ) AS avg_salary_per_title_company
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL
ORDER BY RANDOM()
LIMIT 10;
```

`PARTITION BY job_title_short, company_id` crea una ventana separada para cada combinación de título y empresa. El promedio se calcula de forma independiente dentro de cada ventana.

**La analogía:** Es como hacer un `GROUP BY` pero sin colapsar las filas. Cada fila sabe a qué grupo pertenece y tiene el resultado del cálculo de su grupo.

---

## Caso 3 — ORDER BY en la ventana: cálculos acumulativos

Cuando agregas `ORDER BY` dentro del `OVER()`, el cálculo se vuelve acumulativo — opera sobre todas las filas desde el inicio de la partición hasta la fila actual:

```sql
-- Promedio móvil acumulativo de salario por hora para Data Engineers
SELECT
    job_posted_date,
    job_title_short,
    salary_hour_avg,
    AVG(salary_hour_avg) OVER(
        PARTITION BY job_title_short
        ORDER BY job_posted_date ASC
    ) AS running_avg_hourly_by_title
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL
  AND job_title_short = 'Data Engineer'
ORDER BY job_title_short, job_posted_date
LIMIT 10;
```

Para la primera fila (empleo más antiguo), el promedio es su propio salario. Para la segunda, es el promedio de las dos primeras. Y así sucesivamente. Esto crea un **promedio móvil** que muestra cómo evoluciona el mercado salarial con el tiempo.

```sql
-- Suma acumulativa (mismo patrón)
SELECT
    job_posted_date,
    salary_hour_avg,
    SUM(salary_hour_avg) OVER(
        PARTITION BY job_title_short
        ORDER BY job_posted_date ASC
    ) AS running_total_salary
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL
  AND job_title_short = 'Data Engineer';
```

---

## Caso 4 — Funciones de ranking

### RANK() — Ranking con saltos

```sql
SELECT
    job_posted_date,
    job_title_short,
    salary_hour_avg,
    RANK() OVER(ORDER BY salary_hour_avg DESC) AS rank_salary
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL
ORDER BY salary_hour_avg DESC
LIMIT 140;
```

`RANK()` asigna el mismo número a empates, pero **salta** los siguientes números. Si dos empleos empatan en el puesto 1, el siguiente es el puesto 3 (no el 2).

```
salary | RANK
------+------
100    |  1
100    |  1
 90    |  3   ← salta el 2
 80    |  4
```

### DENSE_RANK() — Ranking sin saltos

```sql
SELECT
    salary_hour_avg,
    DENSE_RANK() OVER(ORDER BY salary_hour_avg DESC) AS rank_salary
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL
ORDER BY salary_hour_avg DESC
LIMIT 140;
```

Igual que `RANK()` pero **sin saltar** números. Los empates tienen el mismo número y el siguiente es consecutivo.

```
salary | RANK | DENSE_RANK
------+------+----------
100    |  1   |  1
100    |  1   |  1
 90    |  3   |  2   ← no salta
 80    |  4   |  3
```

**¿Cuándo usar cada uno?**
- `RANK()`: cuando el salto importa (ej. "quedaste en el puesto 3 de 100" tiene significado de que hay 2 mejores que tú)
- `DENSE_RANK()`: cuando quieres "top N categorías" sin importar empates (ej. los 5 niveles salariales más altos)

### RANK() con PARTITION BY — Ranking por grupo

```sql
SELECT
    job_id,
    job_title_short,
    salary_hour_avg,
    RANK() OVER(
        PARTITION BY job_title_short
        ORDER BY salary_hour_avg DESC
    ) AS rank_within_title
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL
ORDER BY salary_hour_avg DESC, job_title_short
LIMIT 10;
```

El ranking se reinicia para cada `job_title_short`. El empleo mejor pagado de cada título recibe el rango 1. Esto responde: "¿Qué posición ocupa este empleo dentro de su categoría?"

### ROW_NUMBER() — Número de fila único

```sql
SELECT *,
    ROW_NUMBER() OVER(ORDER BY job_posted_date) AS row_num
FROM job_postings_fact
ORDER BY job_posted_date
LIMIT 20;
```

`ROW_NUMBER()` siempre asigna números únicos, incluso en empates. Si dos filas tienen la misma fecha, una recibe el 1 y la otra el 2 (de forma arbitraria pero consistente).

**Diferencia clave con RANK():**

```
salary | RANK | DENSE_RANK | ROW_NUMBER
------+------+-----------+----------
100    |  1   |     1     |    1
100    |  1   |     1     |    2     ← siempre único
 90    |  3   |     2     |    3
```

**Cuándo usar ROW_NUMBER():** Para deduplicar registros o crear IDs únicos artificiales. El patrón clásico es `WHERE row_num = 1` para quedarte con una sola fila por grupo.

---

## Caso 5 — LAG(): comparar con la fila anterior

```sql
SELECT
    job_id,
    company_id,
    job_title_short,
    job_posted_date,
    salary_year_avg,
    LAG(salary_year_avg) OVER(
        PARTITION BY company_id
        ORDER BY job_posted_date ASC
    ) AS previous_posting_salary,
    salary_year_avg - LAG(salary_year_avg) OVER(
        PARTITION BY company_id
        ORDER BY job_posted_date ASC
    ) AS salary_change
FROM job_postings_fact
WHERE salary_year_avg IS NOT NULL
ORDER BY company_id, job_posted_date
LIMIT 20;
```

`LAG(columna)` accede al valor de la fila **anterior** dentro de la ventana. Con `PARTITION BY company_id ORDER BY job_posted_date`, compara cada publicación de empleo de una empresa con la publicación anterior de esa misma empresa.

El resultado `salary_change` muestra si el salario subió, bajó o se mantuvo respecto al empleo anterior de la misma empresa.

**`LEAD(columna)`** hace lo opuesto — accede a la fila siguiente. Útil para calcular tiempo hasta el próximo evento.

---

## Patrón avanzado — Window Function dentro de CTE (código comentado de la lección)

```sql
WITH ranked_jobs AS (
    SELECT
        job_id,
        job_title_short,
        company_id,
        job_country,
        COALESCE(salary_hour_avg, salary_year_avg / 2080) AS standardized_salary,
        RANK() OVER(
            PARTITION BY job_title_short, job_country
            ORDER BY COALESCE(salary_hour_avg, salary_year_avg / 2080) DESC
        ) AS rank_salary
    FROM job_postings_fact
)
SELECT *
FROM ranked_jobs
WHERE job_country = 'Colombia'
  AND job_title_short = 'Data Analyst'
ORDER BY standardized_salary ASC;
```

Este es el patrón más importante de las window functions en producción: **calcular el ranking en un CTE y filtrar por él en la query exterior**.

No puedes usar `WHERE rank_salary = 1` en la misma query donde defines la window function — el WHERE se evalúa antes de que el OVER() calcule el ranking. El CTE resuelve esto: primero calcula el ranking completo, luego filtras.

---

## Resumen visual

```
                    OVER()
                   ┌──────────────────────────────────────┐
función()  OVER(   │ PARTITION BY col1  →  define ventanas │
                   │ ORDER BY col2      →  define el orden  │
                   └──────────────────────────────────────┘)

Sin PARTITION BY:  toda la tabla es una ventana
Con PARTITION BY:  una ventana por grupo (como GROUP BY sin colapsar)
Sin ORDER BY:      cálculo sobre toda la ventana de una vez
Con ORDER BY:      cálculo acumulativo hasta la fila actual
```

| Función | Qué calcula | Necesita ORDER BY |
|---|---|---|
| `COUNT() OVER()` | Total de filas | No |
| `AVG() OVER()` | Promedio (global o por grupo) | No (o sí para acumulativo) |
| `SUM() OVER()` | Suma acumulativa | Sí |
| `RANK()` | Ranking con saltos en empates | Sí |
| `DENSE_RANK()` | Ranking sin saltos | Sí |
| `ROW_NUMBER()` | Número único por fila | Sí |
| `LAG()` | Valor de la fila anterior | Sí |
| `LEAD()` | Valor de la fila siguiente | Sí |

---

## ¿Por qué importa en un proyecto real?

- **Ranking y top-N por grupo:** "El empleo mejor pagado por empresa" o "el top 3 de productos por región" son queries imposibles sin window functions + CTE. Es uno de los patrones más pedidos en entrevistas de Data Engineering.
- **Cálculos acumulativos:** Revenue acumulado por mes, usuarios activos en los últimos 30 días, tickets resueltos en el sprint — todos usan `SUM OVER` o `COUNT OVER` con `ORDER BY`.
- **Detección de cambios:** `LAG()` es la forma estándar de detectar cambios entre eventos consecutivos: "¿el precio subió desde la última semana?", "¿el usuario estuvo inactivo más de 30 días?".
- **Deduplicación:** `ROW_NUMBER() OVER(PARTITION BY id ORDER BY fecha DESC) = 1` es el patrón estándar para quedarse con el registro más reciente de cada entidad cuando hay duplicados — uno de los problemas más comunes en pipelines de datos.
- **En dbt:** Los modelos de mart usan window functions extensamente para calcular métricas de negocio complejas que no se pueden obtener con GROUP BY simple.
