# MLOPSII-tp6
Mini TP 6

# Mini-TP 6 · Tu modelo en el Data Lake

**Operaciones de Aprendizaje Automático II · CEIA – FIUBA** · Entrega individual · Andrea Ferenaz

El artefacto del modelo se sube a un Data Lake compatible con S3, organizado en zonas y con
versiones, y se sirve **cargándolo desde el lake**, no desde disco.

## Modelo

El de la Sesión 1 (Mini-TP 1): pronóstico de demanda diaria para **Gout** (cadena sin TACC,
9 sucursales, 96 SKUs). `RandomForestRegressor` sobre 11 features: `gout-demanda-rf`.
Los datos son sintéticos y se generan con semilla fija: el notebook no necesita archivos externos.

## Cómo correrlo

**En Colab:** abrir con el botón de arriba y ejecutar todo (*Entorno de ejecución → Ejecutar todo*).
En Colab no hay Docker y MinIO ya no distribuye binarios precompilados, así que el notebook levanta
un servidor compatible con S3 ([moto](https://github.com/getmoto/moto)) en `localhost:9000`. El
código no cambia: `boto3` usa la misma API, el mismo endpoint y las mismas credenciales. El
contenido vive en memoria y se pierde al reiniciar el entorno.

**En local, con MinIO:**

```bash
docker run -d --name minio -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minio -e MINIO_ROOT_PASSWORD=minio123 \
  cgr.dev/chainguard/minio:latest server /data --console-address ":9001"
uv add boto3 pandas pyarrow scikit-learn mlflow joblib
```

Si detecta MinIO en `localhost:9000`, el notebook lo usa directamente.

## Estructura del lake

s3://datalake-tp6/
├── raw/gout_demanda/fecha=AAAA-MM-DD/ventas.csv crudo, tal como llega (CSV)
├── curated/gout_demanda/fecha=AAAA-MM-DD/ventas.parquet curado y tipado (Parquet)
├── curated/preds/gout-demanda-rf/v1/predicciones.parquet predicciones para GraphQL
├── models/
│ ├── v1/gout-demanda-rf.pkl + metadata.json inmutable: no se sobrescribe
│ ├── v2/gout-demanda-rf.pkl + metadata.json
│ └── production.json puntero a la versión servida
└── mlflow/<run_id>/artifacts/... artefactos de MLflow (opcional)


## Qué hace el notebook

| Sección | Contenido |
|---|---|
| 1 | Cliente `boto3`, bucket y zonas `raw`, `curated`, `models` |
| 2 | Dataset crudo → curado (renombrado, tipos, validaciones) |
| 3 | Sube crudo y curado; entrena **leyendo desde `curated`**; sube el modelo a `models/v1/` con metadata y checksum SHA-256 |
| 4 | `cargar_modelo_desde_lake(bucket, key)` y `cargar_modelo_en_produccion()`, que lee el puntero y verifica el checksum |
| 4.b | Versión nueva sin romper la anterior: sube v2, la promueve y hace rollback, cada paso con una sola escritura del puntero |
| 4.c | Predicciones a `curated/preds/` y un resolver estilo GraphQL que las lee |
| 5 | MLflow con tracking en SQLite y artefactos en `s3://datalake-tp6/mlflow` |
| 6 | Reflexión |

## Resultados

| Versión | Hiperparámetros | MAE | R² |
|---|---|---|---|
| v1 | 120 árboles, profundidad 10, hoja mínima 10 | 6,26 | 0,893 |
| v2 | 250 árboles, profundidad 14, hoja mínima 5 | 5,83 | 0,907 |

## Reflexión (resumen)

- **Servir desde el lake:** desacopla el ciclo de vida del modelo del de la imagen. Publicar una
  versión es subir un objeto y mover un puntero, sin rebuild. Todos los servicios cargan el mismo
  artefacto trazable.
- **Versionado:** prefijos inmutables `models/vN/` y un puntero `production.json`. Promover y
  hacer rollback es una sola escritura.
- **Predicciones para GraphQL:** zona `curated`, versionadas por modelo, porque son dato derivado,
  tipado y listo para consumir.

La respuesta completa está en la sección 6 del notebook.
