
# TP 02 - MD II: Pipeline de Analytics para Cloud Provider

## 📋 Descripción

Pipeline completo de ingesta, procesamiento y serving de datos para analytics de un proveedor cloud, implementado en **Google Colab** con **PySpark** y **AstraDB (Cassandra)**, siguiendo las especificaciones del TP 02.

La solución implementa una arquitectura tipo **Lambda** con zonas:
**Landing → Bronze → Silver → Gold → Serving (Cassandra)**.

---

### 🧱 1. BATCH A BRONZE – **COMPLETADO**

**Objetivo:** Ingestar al menos 3 maestros desde archivos CSV a Parquet particionado.

**✅ Acciones implementadas:**

- ✅ Lectura de:
  - `customers_orgs.csv`
  - `users.csv`
  - `billing_monthly.csv`
- ✅ Aplicación de esquemas tipados:
  - `created_at` como `TimestampType`
  - montos como `DecimalType`
  - fechas de billing como `DateType`
- ✅ Columnas técnicas agregadas:
  - `ingest_ts`: timestamp de ingesta
  - `source_file`: nombre del archivo origen
  - `ingest_date`: fecha de ingesta para particionado
- ✅ Deduplicación por clave natural:
  - `customers`: por `org_id`
  - `users`: por `user_id`
  - `billing`: por `billing_id`
- ✅ Almacenamiento en formato **Parquet particionado por `ingest_date`** en:
  - `bronze/customers/ingest_date=YYYY-MM-DD/`
  - `bronze/users/ingest_date=YYYY-MM-DD/`
  - `bronze/billing/ingest_date=YYYY-MM-DD/`

---

### 🔁 2. STREAMING A BRONZE – **COMPLETADO**

**Objetivo:** Leer eventos de uso en tiempo real desde `usage_events_stream/*.jsonl` y estandarizarlos en Bronze.

**✅ Acciones implementadas:**

- ✅ Definición de `usage_events_schema` con tipos explícitos:
  - `event_timestamp` como `TimestampType`
  - `cost_usd_increment` como `DecimalType(10,4)`
  - `genai_tokens` como `LongType`
  - `carbon_kg` como `DecimalType(10,6)`
- ✅ Implementación de **Structured Streaming** (versión de demo):
  - `readStream` sobre `usage_events_stream/*.jsonl`
  - `withWatermark("event_timestamp", "10 minutes")` para manejo de datos tardíos
  - `dropDuplicates(["event_id"])` para evitar duplicados por reenvíos
  - columnas técnicas: `ingest_ts`, `source_file`, `ingest_date`, `event_date`
- ✅ Escritura del stream en Bronze:
  - Formato **Parquet**
  - `outputMode("append")`
  - `checkpointLocation` configurado en `bronze/checkpoints/usage_events`
  - particionado por `ingest_date`
- ✅ Adicionalmente, se incluyó una versión batch para simular streaming en Colab, manteniendo la misma lógica de transformación.

---

### 🧼 3. CAPA SILVER (Limpieza y Enriquecimiento) – **COMPLETADO**

**Objetivo:** Unir eventos de uso con datos maestros, crear features y aplicar reglas de calidad.

**✅ Enriquecimiento:**

- ✅ Join de `usage_events` (Bronze) con `customers` (Bronze) por `org_id`.
- ✅ Campos enriquecidos:
  - `org_name`
  - `industry`
  - `customer_tier`

**✅ Features calculadas:**

1. `daily_cost_usd`  
   - Suma diaria de `cost_usd_increment` por (`org_id`, `event_date`).

2. `requests`  
   - Cantidad de requests procesados cuando `unit = 'count'`, 0 en otros casos.

3. `genai_tokens_clean`  
   - Limpieza de `genai_tokens` con `coalesce(..., 0)` para evitar NULLs.

4. `carbon_kg_clean`  
   - Limpieza de `carbon_kg` con `coalesce(..., 0.0)` para evitar NULLs.

5. `cost_anomaly_flag`  
   - Flag booleano que marca `True` cuando `cost_usd_increment < 0`.

**✅ Reglas de calidad (Data Quality):**

1. `event_id` no nulo ni vacío:
   - `event_id IS NOT NULL AND event_id != ''`

2. `cost_usd_increment ≥ -0.01`:
   - evita outliers muy negativos.

3. `unit` no nulo cuando existe `value`:
   - si hay `value` y `unit` es NULL → va a cuarentena.

**✅ Cuarentena:**

- Registros que no cumplen alguna regla se envían a **Quarantine**.
- Se manejan **dos canales**:
  - Registros con reglas fallidas (campos inválidos).
  - Registros que tenían `event_date` nulo (evitan partición `__HIVE_DEFAULT_PARTITION__`).
- Se generan **muestras de cuarentena**:
  - `quarantine/usage_events/` con todos los rechazados.
  - `quarantine/samples/` con un subconjunto (ej. primeros 20) para análisis.

**✅ Escritura de Silver:**

- Silver se escribe en:
  - `silver/usage_events/`
- Formato **Parquet**
- Modo `overwrite` (para facilitar idempotencia)
- Sin particionar adicionalmente (para evitar errores por particiones corruptas).

---

### 📊 4. CAPA GOLD (Mart FinOps) – **COMPLETADO**

**Objetivo:** Construir un mart analítico de FinOps para uso diario.

**✅ Mart generado: `org_daily_usage_by_service`**

**Clave de agrupación:**

- `org_id`
- `org_name`
- `service_name`
- `event_date`

**✅ Aspectos técnicos:**

- Se agrega columna técnica `load_ts` al mart.
- Se escribe en:
  - `gold/org_daily_usage_by_service/`
- Formato **Parquet**
- Modo `overwrite`
- **Particionado por `event_date`**, permitiendo:
  - queries eficientes por rango de fechas
  - administración segmentada por día.

---

### DATOS DE PRUEBA 

Los datos de prueba se encuentran en:
https://drive.google.com/drive/folders/1BRdZ05vFzLtfBP-nSTKS4Ewm24ILpZmw?usp=drive_link

---

### 🗃️ 5. SERVING EN CASSANDRA (AstraDB) – **COMPLETADO**

**Objetivo:** Modelar una tabla orientada a consulta (query-first) y cargar el Mart Gold.

**✅ Modelo de datos en Cassandra (AstraDB):**

Keyspace utilizado: `default_keyspace`.

Tabla:

```sql
CREATE TABLE IF NOT EXISTS default_keyspace.org_daily_usage (
    org_id text,
    date date,
    service_name text,
    org_name text,
    total_cost_usd decimal,
    total_requests bigint,
    total_genai_tokens bigint,
    total_carbon_kg decimal,
    total_events bigint,
    avg_cost_per_event decimal,
    load_ts timestamp,
    PRIMARY KEY ((org_id, date), service_name)
) WITH CLUSTERING ORDER BY (service_name ASC);

** METRICAS CALCULADAS ** 

-- Total de costo por organización en un rango de fecha especifico

USE default_keyspace;

SELECT org_id,
    sum(total_cost_usd) as cost_periodo 
FROM org_daily_usage
WHERE org_id = 'ORG001'
  AND date >= '2022-01-01'
  AND date <= '2022-01-31'
ALLOW FILTERING;

-- Consumo de un período especifico 

USE default_keyspace;

SELECT org_id, date, service_name, total_cost_usd
FROM org_daily_usage
WHERE org_id = 'ORG001'
  AND date >= '2022-01-01'
  AND date <= '2022-01-31'
ALLOW FILTERING;
