# Workflow: orquestación del pipeline

El pipeline corre como un Job de Databricks con cómputo serverless, disponible en Free Edition. El Job está definido como código en un bundle (Databricks Asset Bundles): `databricks.yml` y `resources/job_medallion.yml`. Así queda versionado en Git junto con los notebooks y se puede recrear igual en cualquier workspace.

## Desplegar el bundle

**Opción A, desde el workspace (sin instalar nada):**
1. En Databricks, ve a Workspace y crea una carpeta Git (Create > Git folder) con la URL de este repositorio.
2. Abre `databricks.yml` dentro de la carpeta y usa el panel de despliegue del bundle (ícono de bundles o botón **Deploy**). Elige el target `dev`.
3. El Job aparece en Jobs & Pipelines como `[dev <tu usuario>] dunnhumby_pipeline_medallion`.

**Opción B, con la CLI de Databricks en tu computadora:**

```bash
databricks auth login --host <url-de-tu-workspace>
databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run -t dev pipeline_medallion --params dia_hasta=650
```

La tarea `ingesta_kaggle` descarga los CSV desde Kaggle, así que no hay que subir nada a mano.

## Tareas

Cada tarea es un notebook del repositorio y depende de la anterior. `setup_unity_catalog` va primero porque solo crea lo que no existe, así que repetirla no cambia nada:

| # | Tarea | Notebook | Qué hace |
|---|---|---|---|
| 0 | `setup_unity_catalog` | `notebooks/00_setup_unity_catalog` | Crea, si no existen, el catálogo, los esquemas y el Volume |
| 1 | `ingesta_kaggle` | `notebooks/00b_ingesta_kaggle` | Descarga los 7 CSV del dataset público de Kaggle al Volume `raw_files` (sin `causal_data`) |
| 2 | `bronze` | `notebooks/01_bronze` | Carga completa de los 7 CSV del Volume a `dunnhumby.bronze` |
| 3 | `silver` | `notebooks/02_silver` | Limpieza y `MERGE` incremental por día hacia `dunnhumby.silver` |
| 4 | `gold` | `notebooks/03_gold` | Tablas de las 4 preguntas de negocio con `MERGE` en `dunnhumby.gold` |
| 5 | `catalogo_datos` | `notebooks/04_catalogo_datos` | Comentarios de tablas y columnas en Unity Catalog, más una verificación |

Los notebooks `01b_diagnostico_calidad` y `02b_exploracion_gold` son exploratorios y no forman parte del Job.

## Parámetros del Job

Están declarados en el bundle como *Job parameters*, y Databricks los pasa a los widgets de `02_silver`. El Job también tiene una programación diaria (6:00 a. m., hora Perú) que queda en pausa, para ejecutarlo a mano en la demo:

| Parámetro | Valor por defecto | Significado |
|---|---|---|
| `dia_hasta` | `711` | Último día que se procesa. El dataset es estático (días 1 a 711), así que un valor menor simula que los días siguientes aún no llegan |
| `dias_reproceso` | `0` | Días ya cargados que se vuelven a procesar, por si llegan datos tardíos |
| `dataset_kaggle` | `frtgnn/dunnhumby-the-complete-journey` | Dataset de Kaggle del que se descargan los CSV |

## Cómo funciona la carga incremental

- **Ingesta:** cada corrida vuelve a descargar los 7 CSV de [Kaggle](https://www.kaggle.com/datasets/frtgnn/dunnhumby-the-complete-journey). Si Kaggle no responde pero el archivo ya está en el Volume, se usa esa copia (plan B: también se pueden subir a mano desde Catalog Explorer > Volume > Upload).
- **Bronze** se recarga completa en cada corrida, porque es la copia fiel del archivo de origen, que llega entero.
- **Silver** calcula una ventana de días: desde el día siguiente al último que ya tiene `transaction_data` (menos `dias_reproceso`) hasta `dia_hasta`. Solo esas filas de `transaction_data`, su cuarentena y `coupon_redempt` pasan por el `MERGE`, con la clave `basket_id` + `product_id` en las transacciones. Las tablas de referencia no tienen día, así que se comparan completas por su clave natural.
- **Gold:** `ventas_departamento_periodo` recalcula solo los periodos de 4 semanas que recibieron líneas nuevas. Las otras tres tablas se recalculan completas, porque el decil de gasto y las semanas sin comprar dependen de todos los hogares, y luego se hace el `MERGE`.
- **Idempotencia:** un `MERGE` solo actualiza una fila si cambió algún dato. Si se vuelve a correr el Job con el mismo `dia_hasta`, no se inserta, actualiza ni borra nada.

## Demostración sugerida

1. Ejecutar el Job con `dia_hasta = 650` (Run now with different parameters). Silver procesa los días 1 a 650.
2. Ejecutar el Job con `dia_hasta = 711`. Silver procesa solo los días 651 a 711 y Gold recalcula solo los periodos 24 a 26.
3. Ejecutar el Job otra vez con `711`. Todos los `MERGE` informan 0 insertadas, 0 actualizadas y 0 borradas.

Para volver a empezar desde cero, hay que borrar las tablas de `silver` y `gold` (`DROP TABLE`) antes de la primera corrida.
