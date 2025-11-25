
                    ┌─────────────────────────┐
                    │        LANDING          │
                    │ CSV (maestros) / JSONL  │
                    └─────────────┬───────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │         BRONZE          │
                    │  Datos raw tipificados  │
                    │  Partición: ingest_date │
                    └─────────────┬───────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │         SILVER          │
                    │ Limpieza + Features     │
                    │ Enriquecimiento (join)  │
                    │ DQ + Quarantine         │
                    └─────────────┬───────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │          GOLD           │
                    │   Mart FinOps diario    │
                    │ Partición: event_date   │
                    └─────────────┬───────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │      CASSANDRA          │
                    │   Serving Layer Query   │
                    │ PK: (org_id, date, svc) │
                    └─────────────────────────┘
                    
//1. Patrón de arquitectura: Lambda

Decisión: Se implementó un pipeline Lambda con rutas batch (maestros) y streaming (eventos).
Justificación:

La consigna exige batch + streaming simultáneo.

Los maestros no requieren latencia baja → batch.

Los eventos de uso sí → streaming.

La capa Silver unifica ambos, tal como prescribe Lambda.

//2. Por qué NO se usó Kappa

Kappa requiere que todo ingrese por streaming.

Va en contra de la consigna (“al menos 3 maestros batch”).

Incrementaría complejidad para datos estáticos.

//3. Particionado del datalake
Bronze

partitionBy("ingest_date")

Razón: facilita reprocesos (idempotencia), evita mezclar días distintos.

Silver

Sin particiones (para evitar errores de Google Drive y particiones corruptas).
Razón técnica: en Colab el FS no maneja bien metadatabases tipo Hive.

Gold

partitionBy("event_date")
Justificación:

Optimiza queries diarias/semanales.

Reduce tamaño de cada archivo.

Permite purgado histórico.

//4. Clave primaria en Cassandra

PRIMARY KEY ((org_id, date), service_name)

Justificación:

org_id es la dimensión principal de análisis.

date permite rangos de fechas sin ALLOW FILTERING.

service_name como clustering ordena por servicio dentro del día.

//5. Umbrales y reglas de calidad

Regla	Motivo
event_id no nulo	evita duplicación y problemas de integridad
cost_usd_increment ≥ -0.01	evita costos negativos no reales (billing)
unit requerido quando value existe	garantiza consistencia semántica
event_date nulo → cuarentena	evita partición __HIVE_DEFAULT_PARTITION__

//6. Idempotencia

Estrategia aplicada:

Bronze: partición por fecha de ingesta + dedupe stream.

Silver/Gold: overwrite + lógica determinística.

Verificación del TP implementada comparando conteos antes/después.

Resultado final:
Silver y Gold producen los mismos resultados en cada re-ejecución.
