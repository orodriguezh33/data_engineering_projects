# Lesson 1.25 — CASE Expressions: Lógica condicional en SQL

## ¿Por qué necesitas lógica condicional en SQL?

SQL es un lenguaje de consultas, pero los datos del mundo real rara vez vienen listos para analizar. Los salarios vienen como números crudos — los analistas de negocio quieren categorías como "Alto", "Medio", "Bajo". Los títulos de empleo vienen en texto libre — los reportes quieren categorías limpias.

El `CASE` es la forma de introducir lógica `if / else if / else` directamente en una query. Sin salir de SQL, sin scripts adicionales.

---

## La estructura del CASE

```sql
CASE
    WHEN condición1 THEN resultado1
    WHEN condición2 THEN resultado2
    ELSE resultado_por_defecto
END AS nombre_columna
```

**Reglas importantes:**
- Las condiciones se evalúan **en orden**, de arriba hacia abajo
- En cuanto una condición es verdadera, se devuelve ese resultado y las demás se ignoran
- `ELSE` es opcional, pero si lo omites y ninguna condición aplica, el resultado es `NULL`
- Siempre cierra con `END` y dale un alias descriptivo

---

## Uso 1 — Categorizar valores numéricos (bucketing)

Convierte rangos numéricos en etiquetas legibles:

```sql
SELECT
    job_title_short,
    salary_hour_avg,
    CASE
        WHEN salary_hour_avg < 25  THEN 'Low'
        WHEN salary_hour_avg < 50  THEN 'Medium'
        ELSE 'High'
    END AS salary_category
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL
LIMIT 10;
```

**¿Por qué funciona con `< 50` y no `BETWEEN 25 AND 50`?**

Por el orden de evaluación. Si llegaste al segundo `WHEN`, ya sabes que `salary_hour_avg >= 25` (porque si fuera menor, el primer WHEN lo habría capturado). Entonces `salary_hour_avg < 50` equivale a `25 <= salary_hour_avg < 50`. Es más conciso y fácil de mantener.

---

## Uso 2 — Manejar NULLs explícitamente

Si no tratas los NULLs, la categoría `ELSE` los absorbe y los clasifica como `'High'` — un error silencioso:

```sql
SELECT
    job_title_short,
    salary_hour_avg,
    CASE
        WHEN salary_hour_avg IS NULL THEN 'Missing'  -- ← primero atrapa los NULLs
        WHEN salary_hour_avg < 25   THEN 'Low'
        WHEN salary_hour_avg < 50   THEN 'Medium'
        ELSE 'High'
    END AS salary_category
FROM job_postings_fact
LIMIT 10;
```

La regla: **si esperas NULLs, ponlos en el primer WHEN**. Así los distingues de los valores reales y puedes monitorear cuántos datos están incompletos.

---

## Uso 3 — Categorizar texto con LIKE

Clasifica valores de texto libre usando patrones:

```sql
SELECT
    job_title,
    CASE
        WHEN job_title LIKE '%Data%' AND job_title LIKE '%Analyst%'   THEN 'Data Analyst'
        WHEN job_title LIKE '%Data%' AND job_title LIKE '%Engineer%'  THEN 'Data Engineer'
        WHEN job_title LIKE '%Data%' AND job_title LIKE '%Scientist%' THEN 'Data Scientist'
    END AS job_title_category,
    job_title_short
FROM job_postings_fact
ORDER BY RANDOM()
LIMIT 20;
```

`LIKE '%texto%'` busca si el patrón aparece en cualquier posición del string. El `%` es un comodín que representa cualquier cantidad de caracteres.

**Nota:** Este CASE no tiene `ELSE`, así que títulos que no coincidan con ningún patrón devuelven `NULL`. Esto es intencional — si un título no encaja en las categorías definidas, es mejor saber que queda sin clasificar que asignarlo a una categoría incorrecta.

---

## Uso 4 — Agregación condicional

El CASE también funciona **dentro de funciones de agregación**. Esto permite calcular métricas para subgrupos sin hacer múltiples queries:

```sql
SELECT
    job_title_short,
    COUNT(*) AS total_postings,
    MEDIAN(
        CASE WHEN salary_year_avg < 100000 THEN salary_year_avg END
    ) AS median_low_salary,
    MEDIAN(
        CASE WHEN salary_year_avg >= 100000 THEN salary_year_avg END
    ) AS median_high_salary
FROM job_postings_fact
WHERE salary_year_avg IS NOT NULL
GROUP BY job_title_short;
```

Cuando el CASE no devuelve nada (porque la condición es falsa y no hay `ELSE`), devuelve `NULL`. Las funciones de agregación como `MEDIAN`, `AVG`, `SUM` ignoran los NULLs automáticamente. Esto es lo que hace el truco funcionar: el `MEDIAN` solo agrega los valores del bucket que te interesa.

En una sola query obtienes el mediano de salarios bajos Y el mediano de salarios altos por cada título de empleo, sin joins ni subqueries.

---

## Uso 5 — CASE dentro de un CTE (ejemplo final)

Combinando CTEs y CASE para estandarizar salarios de fuentes distintas:

```sql
WITH salaries AS (
    SELECT
        job_title_short,
        salary_hour_avg,
        salary_year_avg,
        CASE
            WHEN salary_year_avg IS NOT NULL THEN salary_year_avg
            WHEN salary_hour_avg IS NOT NULL THEN salary_hour_avg * 2080
        END AS standardized_salary      -- si tiene anual, lo usa; si no, convierte el por hora
    FROM job_postings_fact
)
SELECT *,
    CASE
        WHEN standardized_salary IS NULL  THEN 'Missing'
        WHEN standardized_salary < 75000  THEN 'Low'
        WHEN standardized_salary < 150000 THEN 'Medium'
        ELSE 'High'
    END AS salary_bucket
FROM salaries
ORDER BY standardized_salary ASC
LIMIT 10;
```

El `2080` es el número estándar de horas laborales al año (40 horas/semana × 52 semanas). Así conviertes un salario por hora a su equivalente anual para poder comparar ambas métricas en la misma escala.

> **Bug en el código original de la lección:** hay un typo en el umbral de Medium: `150_00` debería ser `150_000`. Con `150_00` (que es `15000`), casi todo quedaría clasificado como `'High'` — un error silencioso que produce resultados completamente incorrectos. En producción, este tipo de errores son difíciles de detectar si no validas los resultados.

---

## Resumen — Cuándo usar cada variante

| Situación | Patrón |
|---|---|
| Convertir números a categorías | `CASE WHEN num < X THEN '...'` |
| Texto libre a categorías | `CASE WHEN col LIKE '%..%'` |
| Manejar NULLs | Primer `WHEN` con `IS NULL` |
| Métricas de subgrupos | `MEDIAN(CASE WHEN ... END)` |
| Múltiples transformaciones | CTE + CASE en capas |

---

## ¿Por qué importa en un proyecto real?

- **En pipelines ETL**, el CASE es la herramienta principal para limpiar y estandarizar datos antes de cargarlos al warehouse. Texto libre → categorías limpias. Múltiples formatos de salario → una sola métrica comparable.
- **En dbt**, cada modelo de transformación usa CASE extensively para crear columnas derivadas y categorías de negocio.
- **En reportes y dashboards**, los analistas consumen categorías (Low/Medium/High), no números crudos. El CASE hace esa traducción directamente en SQL.
- **Agregación condicional** (MEDIAN/SUM de un CASE) es el patrón que evita hacer múltiples queries o subqueries cuando necesitas métricas de varios segmentos en una sola tabla.
