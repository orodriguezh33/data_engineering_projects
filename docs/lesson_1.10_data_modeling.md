# Lesson 1.10 — Data Modeling: Conociendo tu modelo de datos

## ¿Qué es el Data Modeling?

Antes de escribir una sola query de análisis, un Data Engineer necesita entender cómo están organizados los datos. Esa organización se llama **modelo de datos**.

El modelo define:
- Qué tablas existen
- Qué columnas tiene cada una
- Cómo se relacionan entre sí
- Qué restricciones garantizan la integridad de los datos

Sin entender el modelo, escribirás queries incorrectas o ineficientes sin darte cuenta.

---

## El modelo de datos de este bootcamp

La base de datos `data_jobs` usa un **esquema estrella (star schema)**, el modelo más común en Data Warehousing.

```
                    skills_dim
                        │
                        │ skill_id
                        │
job_postings_fact ──────────────── skills_job_dim
       │
       │ company_id
       │
  company_dim
```

### Tabla de hechos — `job_postings_fact`

La tabla central. Contiene los eventos medibles: cada fila es un empleo publicado.

```sql
SELECT job_id, job_title, salary_year_avg, company_id
FROM job_postings_fact
LIMIT 10;
```

Las tablas de hechos tienen:
- Una **llave primaria** (`job_id`)
- **Llaves foráneas** que apuntan a tablas de dimensiones (`company_id`)
- **Métricas** que se pueden agregar (`salary_year_avg`, `salary_hour_avg`)

### Tablas de dimensiones

Contienen el contexto descriptivo de los hechos. No tienen métricas, tienen atributos.

**`company_dim`** — información de las empresas:
```sql
SELECT * FROM company_dim LIMIT 10;
```

**`skills_dim`** — catálogo de habilidades técnicas:
```sql
SELECT * FROM skills_dim LIMIT 5;
```

**`skills_job_dim`** — tabla puente entre empleos y skills (relación muchos a muchos):
```sql
SELECT * FROM skills_job_dim LIMIT 5;
```

Un empleo puede requerir muchas skills. Una skill puede aparecer en muchos empleos. La tabla `skills_job_dim` resuelve esa relación muchos a muchos almacenando pares `(job_id, skill_id)`.

---

## Cómo explorar un modelo desconocido

Cuando llegas a un proyecto nuevo y no conoces la base de datos, estos son los comandos que usas para entenderla:

### Ver todas las tablas

```sql
SELECT * FROM information_schema.tables
WHERE table_catalog = 'data_jobs';
```

### Ver columnas y tipos de todas las tablas

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_catalog = 'data_jobs';
```

### Ver las restricciones (primary keys, foreign keys)

```sql
SELECT *
FROM information_schema.table_constraints
WHERE table_catalog = 'data_jobs';
```

Las restricciones son el "contrato" del modelo: te dicen qué columnas son llaves primarias y cómo se relacionan las tablas.

### Comandos rápidos de DuckDB

```sql
PRAGMA show_tables_expanded;   -- resumen visual de todas las tablas
DESCRIBE job_postings_fact;    -- columnas y tipos de una tabla específica
```

---

## Star Schema — La arquitectura más importante en Data Engineering

El esquema estrella es el estándar de facto en data warehouses. Su nombre viene de que una tabla de hechos central está rodeada de tablas de dimensiones, como una estrella.

**¿Por qué se usa en producción?**

1. **Queries simples:** Para analizar datos solo necesitas hacer JOIN entre la tabla de hechos y las dimensiones que te interesan
2. **Performance:** Las tablas de dimensiones son pequeñas; la tabla de hechos es grande pero solo tiene números. Los motores columnaresestán optimizados para esto
3. **Entendible para el negocio:** Un analista puede entender "empleos + empresas + skills" sin necesidad de ser DBA

**Ejemplo de query típica en un star schema:**

```sql
SELECT
    cd.name AS company_name,
    sd.skills,
    COUNT(*) AS job_count,
    AVG(jpf.salary_year_avg) AS avg_salary
FROM job_postings_fact AS jpf
LEFT JOIN company_dim AS cd
    ON jpf.company_id = cd.company_id
LEFT JOIN skills_job_dim AS sjd
    ON jpf.job_id = sjd.job_id
LEFT JOIN skills_dim AS sd
    ON sjd.skill_id = sd.skill_id
GROUP BY cd.name, sd.skills
ORDER BY job_count DESC;
```

---

## ¿Por qué importa en un proyecto real?

- **Antes de cualquier pipeline**, el Data Engineer diseña o estudia el modelo de datos. Es el plano de lo que vas a construir.
- **En dbt**, cada modelo SQL transforma datos hacia una tabla del star schema. Entender el modelo destino es obligatorio para escribir las transformaciones correctamente.
- **En entrevistas de Data Engineering**, te piden diseñar un modelo de datos para un caso de negocio. El star schema es la respuesta más común para sistemas analíticos.
- **`information_schema`** es la herramienta que usan los ingenieros para auditar bases de datos desconocidas, documentar modelos automáticamente, y validar que las restricciones están correctamente definidas.
