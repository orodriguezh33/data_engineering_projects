# Lesson 1.9 — Setup del entorno: VS Code y DuckDB

## ¿Para qué sirve esta lección?

Esta lección no enseña SQL — enseña a configurar el entorno de trabajo. En Data Engineering, saber manejar tus herramientas es tan importante como saber escribir queries.

El único código del archivo es:

```sql
SELECT 42 AS answer;
```

Si esto devuelve `42`, tu entorno está funcionando correctamente. Es el equivalente de "Hello World" en programación — una verificación mínima de que todo está conectado.

---

## ¿Por qué VS Code para SQL?

VS Code es el editor más usado en Data Engineering por varias razones:

- **Extensiones para DuckDB:** puedes ejecutar queries directamente desde el editor sin abrir otra herramienta
- **Git integrado:** control de versiones de tus scripts SQL sin salir del editor
- **Terminal integrada:** puedes ejecutar el CLI de DuckDB (`duckdb`) o herramientas como dbt sin cambiar de ventana
- **IntelliSense:** autocompletado de palabras clave SQL, nombres de tablas y columnas

En empresas reales, los data engineers trabajan en VS Code o en entornos similares (DataGrip, DBeaver) — no en interfaces web simples. Aprender a trabajar en un editor profesional desde el principio es una ventaja.

---

## DuckDB — ¿Por qué este motor?

DuckDB es una base de datos analítica embebida que corre localmente en tu máquina. Para un bootcamp de Data Engineering es ideal porque:

| Ventaja | Descripción |
|---|---|
| Sin servidor | No necesitas instalar ni configurar un servidor de base de datos |
| Rápido en analítica | Optimizado para queries de lectura sobre grandes volúmenes |
| Compatible con Parquet, CSV, JSON | Lee archivos directamente sin importarlos |
| SQL estándar | Lo que aprendes aquí aplica en Snowflake, BigQuery, Redshift |

La sintaxis de DuckDB es casi idéntica a PostgreSQL. Los patrones que practicas aquí los usarás en motores de producción.

---

## El CLI de DuckDB

Desde la terminal, puedes abrir DuckDB así:

```bash
duckdb job_mart.duckdb    # abre o crea el archivo de base de datos
```

Comandos útiles dentro del CLI:

```bash
.databases          # lista las bases de datos cargadas
.tables             # lista las tablas del schema actual
.read archivo.sql   # ejecuta un archivo SQL completo
.quit               # salir
```

El `.read` es el comando que usas en este bootcamp para ejecutar los ejercicios:

```bash
.read lessons/1.9/1.9_VScode_intro.sql
```

---

## ¿Por qué importa en un proyecto real?

El entorno de desarrollo profesional en Data Engineering generalmente incluye:

1. **VS Code** (o similar) — para escribir y versionar código SQL, Python, YAML
2. **Git** — para controlar versiones de los scripts
3. **Un motor SQL local** (como DuckDB) — para desarrollo y pruebas antes de deploy
4. **Un motor SQL en la nube** (Snowflake, BigQuery) — para producción

Aprender a manejar el entorno local correctamente desde el inicio te ahorra horas de frustración cuando empieces a trabajar con pipelines reales.
