# Lesson 1.28 — Text y NULL Functions: Limpiando datos en SQL

## ¿Por qué necesitas estas funciones?

Los datos del mundo real llegan sucios: títulos de empleo con mayúsculas mezcladas, espacios en blanco extras, salarios nulos en una columna pero presentes en otra, valores de cero donde debería haber NULL. Antes de analizar cualquier dato, tienes que limpiarlo.

Estas funciones son tu kit de limpieza directamente en SQL — sin necesidad de Python ni herramientas externas.

---

## Funciones de texto

### Longitud e inspección

```sql
SELECT CHAR_LENGTH('SQL');  -- → 3 (número de caracteres)
```

Útil para detectar valores anómalos: si una columna de código postal siempre debería tener 5 caracteres, `CHAR_LENGTH != 5` te muestra los registros con problemas.

### Cambio de capitalización

```sql
SELECT LOWER('SQL');  -- → 'sql'
SELECT UPPER('SQL');  -- → 'SQL'
```

**El uso más importante:** Normalizar texto antes de comparar. `'Data Engineer' = 'data engineer'` es `FALSE` en SQL. `LOWER('Data Engineer') = LOWER('data engineer')` es `TRUE`.

### Extracción de partes del texto

```sql
SELECT LEFT('SQL', 2);         -- → 'SQ'   (los primeros N caracteres)
SELECT RIGHT('SQL', 2);        -- → 'QL'   (los últimos N caracteres)
SELECT SUBSTRING('SQL', 2, 1); -- → 'Q'    (desde posición 2, toma 1 carácter)
```

`SUBSTRING(texto, inicio, longitud)`:
- La posición empieza en `1`, no en `0` (a diferencia de Python)
- `SUBSTRING('SQL', 2, 1)` → empieza en posición 2 (`Q`), toma 1 carácter → `'Q'`

### Concatenar texto

```sql
SELECT CONCAT('SQL', '-', 'Functions');  -- → 'SQL-Functions'
SELECT 'SQL' || '-' || 'Functions';      -- → 'SQL-Functions'  (sintaxis alternativa)
```

El operador `||` es la forma estándar SQL de concatenar. `CONCAT` es más legible cuando combinas muchas partes.

### Limpiar espacios

```sql
SELECT TRIM('SQL  ');   -- → 'SQL'  (elimina espacios al final e inicio)
```

Los espacios en blanco son invisibles pero rompen comparaciones y agrupaciones. `'Data Engineer'` y `'Data Engineer '` son distintos para SQL — el TRIM los iguala.

DuckDB también tiene `LTRIM` (solo izquierda) y `RTRIM` (solo derecha) si necesitas más precisión.

### Reemplazar texto

```sql
SELECT REPLACE('SQL', 'Q', 'R');  -- → 'SRL'  (reemplaza todas las ocurrencias)
```

Para reemplazos más complejos con patrones, existe `REGEXP_REPLACE`:

```sql
-- Elimina todos los caracteres no numéricos de un teléfono
SELECT REGEXP_REPLACE('(555) 123-4567', '[^0-9]', '', 'g');  -- → '5551234567'
```

El cuarto argumento `'g'` significa "global" — reemplaza todas las ocurrencias, no solo la primera.

---

## Combinar funciones de texto — Limpieza real

La lección muestra el patrón más importante: limpiar antes de comparar.

```sql
WITH title_lower AS (
    SELECT
        job_title,
        LOWER(TRIM(job_title)) AS job_title_clean  -- normaliza: sin espacios, todo minúsculas
    FROM job_postings_fact
)
SELECT
    job_title,
    CASE
        WHEN job_title_clean LIKE '%data%' AND job_title_clean LIKE '%analyst%'   THEN 'Data Analyst'
        WHEN job_title_clean LIKE '%data%' AND job_title_clean LIKE '%engineer%'  THEN 'Data Engineer'
        WHEN job_title_clean LIKE '%data%' AND job_title_clean LIKE '%scientist%' THEN 'Data Scientist'
    END AS job_title_category
FROM title_lower
ORDER BY RANDOM()
LIMIT 20;
```

Sin `LOWER(TRIM(...))`, el LIKE fallaría en casos como `'DATA ENGINEER '` o `'  Data Engineer'`. Con la limpieza en el CTE, el CASE funciona de forma consistente sin importar cómo venga el texto original.

**Patrón clave:** Normaliza en un CTE, luego compara en la query principal. No mezcles la limpieza con la lógica de negocio — se vuelve ilegible.

---

## Funciones NULL

Los NULLs son uno de los temas más importantes y menos intuitivos de SQL. Estas funciones te dan control preciso sobre cómo manejarlos.

### NULLIF — Convertir un valor a NULL condicionalmente

```sql
SELECT NULLIF(10, 20);    -- → 10   (son distintos, devuelve el primero)
SELECT NULLIF(5+5, 10);   -- → NULL (son iguales, devuelve NULL)
```

`NULLIF(a, b)` devuelve NULL si `a = b`, de lo contrario devuelve `a`.

**¿Para qué sirve?** Para tratar valores específicos como si fueran NULL. El caso más común: ceros que deberían ser NULL en cálculos de mediana o promedio.

```sql
SELECT
    MEDIAN(NULLIF(salary_year_avg, 0)),   -- ignora los ceros al calcular la mediana
    MEDIAN(NULLIF(salary_hour_avg, 0))
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL OR salary_year_avg IS NOT NULL;
```

Sin `NULLIF`, un salario de `0` contaminaría la mediana hacia abajo. Con `NULLIF(salary_year_avg, 0)`, los ceros se convierten en NULL y `MEDIAN` los ignora automáticamente.

### COALESCE — El primer valor no NULL de una lista

```sql
SELECT COALESCE(NULL, NULL, 2);  -- → 2  (devuelve el primer valor no NULL)
```

`COALESCE(val1, val2, val3, ...)` evalúa los valores en orden y devuelve el primero que no sea NULL.

**El uso más importante en Data Engineering — estandarizar salarios:**

```sql
SELECT
    salary_year_avg,
    salary_hour_avg,
    COALESCE(salary_year_avg, salary_hour_avg * 2080) AS standardized_salary
FROM job_postings_fact
WHERE salary_hour_avg IS NOT NULL OR salary_year_avg IS NOT NULL;
```

Lógica: "si tiene salario anual, úsalo; si no, convierte el por hora a anual (40h/semana × 52 semanas = 2080h/año)".

Esto es exactamente lo que hacías con CASE en la lección 1.25, pero más conciso:

```sql
-- Equivalentes:
COALESCE(salary_year_avg, salary_hour_avg * 2080)

CASE
    WHEN salary_year_avg IS NOT NULL THEN salary_year_avg
    WHEN salary_hour_avg IS NOT NULL THEN salary_hour_avg * 2080
END
```

### Combinando COALESCE y CASE

```sql
SELECT
    job_title_short,
    salary_hour_avg,
    salary_year_avg,
    COALESCE(salary_year_avg, salary_hour_avg * 2080) AS standardized_salary,
    CASE
        WHEN COALESCE(salary_year_avg, salary_hour_avg * 2080) IS NULL    THEN 'Missing'
        WHEN COALESCE(salary_year_avg, salary_hour_avg * 2080) < 75000    THEN 'Low'
        WHEN COALESCE(salary_year_avg, salary_hour_avg * 2080) < 150000   THEN 'Medium'
        ELSE 'High'
    END AS salary_bucket
FROM job_postings_fact
ORDER BY standardized_salary DESC
LIMIT 10;
```

**Diferencia con la lección 1.25:** En la 1.25 usabas un CTE para calcular `standardized_salary` y luego lo referenciabas en el CASE. Aquí repites `COALESCE(...)` en cada condición. El CTE es más limpio — en producción siempre preferirás el enfoque con CTE para no repetir lógica.

---

## Tabla de referencia rápida

| Función | Qué hace | Ejemplo |
|---|---|---|
| `CHAR_LENGTH(s)` | Longitud del string | `CHAR_LENGTH('SQL')` → `3` |
| `LOWER(s)` | Convierte a minúsculas | `LOWER('SQL')` → `'sql'` |
| `UPPER(s)` | Convierte a mayúsculas | `UPPER('sql')` → `'SQL'` |
| `TRIM(s)` | Elimina espacios al inicio y final | `TRIM(' sql ')` → `'sql'` |
| `LEFT(s, n)` | Primeros n caracteres | `LEFT('SQL', 2)` → `'SQ'` |
| `RIGHT(s, n)` | Últimos n caracteres | `RIGHT('SQL', 2)` → `'QL'` |
| `SUBSTRING(s, i, n)` | Extrae n caracteres desde posición i | `SUBSTRING('SQL', 2, 1)` → `'Q'` |
| `CONCAT(s1, s2)` | Une strings | `CONCAT('A', 'B')` → `'AB'` |
| `REPLACE(s, a, b)` | Reemplaza a por b | `REPLACE('SQL', 'Q', 'R')` → `'SRL'` |
| `NULLIF(a, b)` | NULL si a=b, si no devuelve a | `NULLIF(0, 0)` → `NULL` |
| `COALESCE(a, b, ...)` | Primer valor no NULL | `COALESCE(NULL, 5)` → `5` |

---

## ¿Por qué importa en un proyecto real?

- **LOWER + TRIM** son las primeras transformaciones que aplicas a cualquier columna de texto antes de hacer JOINs o comparaciones. Sin ellas, perderás matches por diferencias de capitalización o espacios.
- **COALESCE** está en casi todo pipeline que combina múltiples fuentes de datos. Cuando distintos sistemas tienen el mismo dato en columnas diferentes, COALESCE te da el primero que exista.
- **NULLIF** es crítico en cálculos financieros — un promedio o mediana contaminado por ceros produce KPIs incorrectos que llegan a los dashboards del negocio sin que nadie lo detecte.
- **REGEXP_REPLACE** es esencial para limpiar datos de entrada de usuarios: números de teléfono, códigos postales, identificaciones que vienen en formatos inconsistentes.
