# Lesson 1.11 — JOINs: Combinando tablas

## ¿Por qué existen los JOINs?

Los datos raramente viven en una sola tabla. En un modelo bien diseñado (como el star schema de este bootcamp), la información está distribuida:
- Los empleos en `job_postings_fact`
- Los nombres de empresas en `company_dim`
- Las habilidades requeridas en `skills_dim`

Para responder preguntas como "¿qué empresa publicó este empleo?" o "¿qué skills requiere este rol?", necesitas combinar tablas. Para eso existen los JOINs.

---

## El concepto base: el ON

Todos los JOINs tienen un `ON` que define cómo emparejar las filas de una tabla con la otra:

```sql
FROM job_postings_fact AS jpf
JOIN company_dim AS cd
ON jpf.company_id = cd.company_id
```

La lógica: "busca en `company_dim` la fila donde `company_id` sea igual al `company_id` de este empleo". Es como buscar en un diccionario: tienes una clave (`company_id`) y buscas el valor correspondiente (`name`).

---

## Los 4 tipos de JOIN

Para los ejemplos, imagina estas dos tablas pequeñas:

```
job_postings_fact          company_dim
─────────────────          ───────────
job_id | company_id        company_id | name
1001   | 10                10         | Google
1002   | 20                20         | Meta
1003   | 99  ← sin match   30         | Netflix ← sin empleos
```

### LEFT JOIN — El más usado en Data Engineering

```sql
SELECT jpf.job_id, jpf.job_title_short, cd.name AS company_name
FROM job_postings_fact AS jpf
LEFT JOIN company_dim AS cd
    ON jpf.company_id = cd.company_id;
```

**Regla:** Trae **todas** las filas de la tabla izquierda (`job_postings_fact`). Si no encuentra match en la derecha, completa con NULL.

```
Resultado:
job_id | company_name
1001   | Google
1002   | Meta
1003   | NULL         ← empleo sin empresa registrada
```

El empleo `1003` aparece aunque no tenga empresa. El dato no desaparece — te avisa con NULL que falta información.

**Cuándo usarlo:** Cuando la tabla izquierda tiene todos los registros que te importan y la derecha es información adicional que puede o no existir. En un pipeline, si usas INNER JOIN donde debería ser LEFT JOIN, puedes perder registros silenciosamente.

### INNER JOIN — Solo los que coinciden

```sql
SELECT jpf.job_id, jpf.job_title_short, cd.name AS company_name
FROM job_postings_fact AS jpf
INNER JOIN company_dim AS cd
    ON jpf.company_id = cd.company_id;
```

**Regla:** Solo devuelve filas que tienen match en **ambas** tablas. Si una fila no tiene correspondencia, desaparece.

```
Resultado:
job_id | company_name
1001   | Google
1002   | Meta
         ← 1003 desapareció (sin match)
         ← Netflix no aparece (sin empleos)
```

**Cuándo usarlo:** Cuando solo te interesan los registros que existen en ambas tablas. Por ejemplo: "muéstrame solo empleos que tengan empresa registrada". También lo usas para filtrar — en la lección 1.2.4, el INNER JOIN con `priority_roles` filtra solo los roles prioritarios.

### RIGHT JOIN — El inverso del LEFT JOIN

```sql
SELECT jpf.job_id, jpf.job_title_short, cd.name AS company_name
FROM job_postings_fact AS jpf
RIGHT JOIN company_dim AS cd
    ON jpf.company_id = cd.company_id;
```

**Regla:** Trae **todas** las filas de la tabla derecha (`company_dim`). Si no hay match en la izquierda, completa con NULL.

```
Resultado:
job_id | company_name
1001   | Google
1002   | Meta
NULL   | Netflix    ← empresa sin empleos publicados
```

**Cuándo usarlo:** Es menos común. Sirve cuando la tabla derecha es la "importante" y quieres saber cuáles no tienen correspondencia en la izquierda. En la práctica, muchos prefieren reescribirlo como LEFT JOIN invirtiendo el orden de las tablas.

### FULL OUTER JOIN — Todo de ambos lados

```sql
SELECT jpf.job_id, jpf.job_title_short, cd.name AS company_name
FROM job_postings_fact AS jpf
FULL OUTER JOIN company_dim AS cd
    ON jpf.company_id = cd.company_id;
```

**Regla:** Trae **todas** las filas de ambas tablas, con NULL donde no hay match.

```
Resultado:
job_id | company_name
1001   | Google
1002   | Meta
1003   | NULL         ← empleo sin empresa
NULL   | Netflix      ← empresa sin empleos
```

**Cuándo usarlo:** Para auditar relaciones y encontrar inconsistencias en ambas direcciones. "¿Hay empleos sin empresa? ¿Hay empresas sin empleos?" Un solo FULL OUTER JOIN responde las dos preguntas.

---

## JOINs en cadena — Múltiples tablas

El mundo real requiere unir más de dos tablas. Simplemente encadenas los JOINs:

```sql
-- Empleos con sus skills (relación muchos a muchos)
SELECT
    jpf.job_id,
    jpf.job_title_short,
    sd.skills
FROM job_postings_fact AS jpf
LEFT JOIN skills_job_dim AS sjd
    ON jpf.job_id = sjd.job_id        -- paso 1: unir con la tabla puente
LEFT JOIN skills_dim AS sd
    ON sjd.skill_id = sd.skill_id;    -- paso 2: unir con el catálogo de skills
```

Por qué necesitas la tabla puente `skills_job_dim`: un empleo puede requerir Python, SQL y dbt al mismo tiempo. Una sola columna `skill` en `job_postings_fact` no puede almacenar múltiples valores. La tabla puente resuelve la relación muchos-a-muchos almacenando una fila por cada par (empleo, skill).

---

## El diagrama visual de los JOINs

```
     A         B
  ┌─────┐   ┌─────┐
  │▓▓▓▓▓│   │     │   LEFT JOIN  → todo A + intersección
  │▓▓▓▓▓│───│     │
  │▓▓▓▓▓│   │     │
  └─────┘   └─────┘

  ┌─────┐   ┌─────┐
  │     │   │     │   INNER JOIN → solo la intersección
  │     │─▓─│     │
  │     │   │     │
  └─────┘   └─────┘

  ┌─────┐   ┌─────┐
  │     │   │▓▓▓▓▓│   RIGHT JOIN → intersección + todo B
  │     │───│▓▓▓▓▓│
  │     │   │▓▓▓▓▓│
  └─────┘   └─────┘

  ┌─────┐   ┌─────┐
  │▓▓▓▓▓│   │▓▓▓▓▓│   FULL OUTER → todo A + todo B
  │▓▓▓▓▓│───│▓▓▓▓▓│
  │▓▓▓▓▓│   │▓▓▓▓▓│
  └─────┘   └─────┘
```

---

## El error más común con JOINs

**Multiplicar filas sin querer.** Si la tabla derecha tiene múltiples filas con el mismo valor en la columna del `ON`, el resultado tendrá más filas que la tabla izquierda:

```sql
-- Si un job_id tiene 5 skills, aparecerá 5 veces en el resultado
SELECT jpf.job_id, sd.skills
FROM job_postings_fact AS jpf
LEFT JOIN skills_job_dim AS sjd ON jpf.job_id = sjd.job_id
LEFT JOIN skills_dim AS sd ON sjd.skill_id = sd.skill_id;
```

Esto es correcto cuando quieres una fila por skill. Pero si luego haces `COUNT(*)` sin darte cuenta, contarás 5 veces ese empleo. Siempre verifica el `COUNT(*)` antes y después de un JOIN para confirmar que el número de filas es el esperado.

---

## ¿Por qué importa en un proyecto real?

- **LEFT JOIN es el default en pipelines:** En un ETL, siempre prefieres LEFT JOIN para no perder registros. Si usas INNER JOIN y la tabla de referencia tiene un gap, tus datos desaparecen silenciosamente y el pipeline "funciona" pero los números están mal.
- **FULL OUTER JOIN para auditoría:** Cuando tienes dos sistemas que deberían estar sincronizados (ej. CRM vs base de datos de ventas), un FULL OUTER JOIN te dice exactamente qué está en uno pero no en el otro.
- **JOINs en cadena son la norma:** Cualquier query analítica en producción une 3, 4, o 5 tablas. El star schema está diseñado para que esos JOINs sean simples y eficientes.
- **Performance:** En Snowflake o BigQuery, la estrategia de JOIN (hash join, nested loop, merge join) es decidida por el optimizador de queries. Entender qué tipo de JOIN usas y con qué tablas ayuda a escribir queries que el motor puede optimizar mejor.
