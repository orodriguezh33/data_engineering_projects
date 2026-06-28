# Lesson 1.2 — Data Types en SQL

## ¿Por qué importan los tipos de datos?

Imagina que guardas salarios como texto (`VARCHAR`) en lugar de números (`DOUBLE`). Cuando intentes calcular el promedio, SQL no puede sumar texto. O imagina que guardas fechas como texto — ordenarlas cronológicamente daría resultados incorrectos porque ordenaría alfabéticamente.

Los tipos de datos son el contrato que defines con la base de datos: "esta columna siempre tendrá este tipo de valor". Ese contrato te da:
- **Validación automática:** si intentas guardar texto en una columna INTEGER, la base de datos lo rechaza
- **Operaciones correctas:** puedes sumar números, calcular diferencias entre fechas, hacer búsquedas eficientes en booleanos
- **Espacio en disco optimizado:** un INTEGER ocupa 4 bytes; un VARCHAR con el mismo número puede ocupar mucho más

En un pipeline de datos en producción, definir mal los tipos de datos es una de las causas más comunes de errores silenciosos — datos que "entran" pero producen resultados incorrectos.

---

## Los tipos más comunes en DuckDB

| Tipo | Qué guarda | Ejemplo |
|---|---|---|
| `INTEGER` | Números enteros | `1`, `42`, `-7` |
| `DOUBLE` | Números decimales | `95000.50`, `3.14` |
| `VARCHAR` | Texto de longitud variable | `'Data Engineer'` |
| `BOOLEAN` | Verdadero o falso | `TRUE`, `FALSE` |
| `TIMESTAMP` | Fecha y hora | `2026-06-25 10:30:00` |
| `DATE` | Solo fecha | `2026-06-25` |

---

## Cómo inspeccionar los tipos de tu tabla

### Opción 1 — DESCRIBE

```sql
DESCRIBE job_postings_fact;
```

Devuelve un resumen rápido: nombre de columna, tipo, si acepta NULL, si es primary key. Es el comando que usarás más seguido para entender una tabla que no conoces.

### Opción 2 — information_schema

```sql
SELECT
    table_name,
    column_name,
    data_type
FROM information_schema.columns
WHERE table_name = 'job_postings_fact';
```

`information_schema` es una base de datos especial que todos los motores SQL tienen. No guarda tus datos — guarda **metadatos**: información sobre tus tablas, columnas, restricciones, etc. Es como el índice de un libro.

La ventaja sobre `DESCRIBE`: puedes filtrar, combinar con otras tablas de metadatos, o hacer queries más complejas sobre la estructura de tu base de datos.

---

## Conversión de tipos — CAST

A veces los datos llegan en un tipo incorrecto y necesitas convertirlos:

```sql
SELECT CAST(123 AS VARCHAR)
-- resultado: '123' (número convertido a texto)
```

### ¿Cuándo necesitas CAST en la práctica?

- **Columnas numéricas guardadas como texto:** si un CSV tiene salarios como `'95000'` (con comillas), necesitas convertirlos a `DOUBLE` para operar con ellos
- **Fechas como texto:** `CAST('2026-06-25' AS DATE)` para poder filtrar rangos de fechas
- **Cálculos de porcentajes:** `CAST(count AS DOUBLE) / total` para evitar división entera

### Sintaxis alternativa en DuckDB

```sql
-- Estas tres líneas hacen lo mismo:
SELECT CAST(123 AS VARCHAR);
SELECT 123::VARCHAR;          -- sintaxis corta de PostgreSQL, funciona en DuckDB
SELECT TRY_CAST(123 AS VARCHAR);  -- no lanza error si falla, devuelve NULL
```

`TRY_CAST` es especialmente útil en pipelines donde los datos pueden venir sucios — en lugar de que todo el proceso falle por un valor incorrecto, ese registro devuelve NULL y puedes manejar el error después.

---

## Error común: NULL y los tipos

NULL en SQL significa "valor desconocido" y **no es lo mismo que cero o texto vacío**:

```sql
SELECT NULL = NULL    -- devuelve NULL (no TRUE)
SELECT NULL + 5       -- devuelve NULL
SELECT '' = NULL      -- devuelve NULL
```

Cuando definas una columna como `NOT NULL`, estás diciendo que ese campo siempre debe tener un valor. Si intentas insertar un NULL, la base de datos lo rechaza. Esto es importante para columnas como `job_id` o `company_id` — si no tienen valor, el registro no tiene sentido.

---

## ¿Por qué importa en un proyecto real?

En un pipeline de Data Engineering:

1. **Al ingerir datos de CSVs o APIs**, los datos llegan como texto. Tu primera tarea es convertir cada columna al tipo correcto antes de cargarlo al warehouse.

2. **En dbt**, defines los tipos en tus modelos. Si defines mal un tipo, downstream todos los modelos que dependan de esa columna heredan el error.

3. **En Snowflake o BigQuery**, los tipos determinan el costo: una tabla con tipos mal definidos (ej. números como VARCHAR) ocupa más espacio y las queries son más lentas y costosas.

4. **El `information_schema`** es lo que usan herramientas como dbt, Great Expectations y Airflow para descubrir automáticamente la estructura de tus tablas y validar que los datos son correctos.
