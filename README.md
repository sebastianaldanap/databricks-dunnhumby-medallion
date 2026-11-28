# Del ticket al cliente: un lakehouse para la fidelización en retail

Proyecto final del curso Fundamentos de Databricks. Es un pipeline con arquitectura Medallion (Bronze, Silver y Gold) gobernado con Unity Catalog y orquestado con un Workflow. Trabaja sobre el dataset público [Dunnhumby · The Complete Journey](https://www.kaggle.com/datasets/frtgnn/dunnhumby-the-complete-journey).

**Integrantes:** Sebastian Aldana Pachas y Brichedl Rojas Gonzalez.

## Caso de negocio

Despensa Club es una cadena ficticia de supermercados con 582 tiendas y una tarjeta de fidelización. El dataset registra las compras de 2,500 hogares durante 711 días (102 semanas).

Hoy el área comercial reparte el presupuesto de marketing por igual entre todos los clientes. No sabe qué hogares sostienen las ventas ni cuáles se están yendo.

El pipeline responde 4 preguntas de negocio:

| # | Pregunta | Tabla Gold |
|---|---|---|
| P1 | ¿Qué departamentos generan más ventas y cómo evolucionan en el tiempo? | `ventas_departamento_periodo` |
| P2 | ¿Gastan más los hogares que canjean cupones que los que no? | `gasto_hogar_cupones` |
| P3 | ¿Cómo cambia el gasto según el ingreso y la composición del hogar? | `gasto_por_perfil_hogar` |
| P4 | ¿Qué hogares concentran el gasto y cuáles dejaron de comprar? | `segmentos_hogar` |

## Arquitectura

```
Kaggle (7 CSV) ─► Volume raw_files ─► bronze (7 tablas) ─► silver (8 tablas) ─► gold (4 tablas)
                    └──────────── catálogo dunnhumby en Unity Catalog ────────────┘
```

- **Unity Catalog:** catálogo `dunnhumby`, esquemas `bronze`, `silver` y `gold`, y el Volume `/Volumes/dunnhumby/bronze/raw_files`.
- **Bronze:** copia fiel de cada CSV, con las columnas de auditoría `_archivo_origen` y `_fecha_carga`. Se recarga completa en cada corrida.
- **Silver:** limpieza, tipado y `MERGE` incremental por día. Las líneas con cantidad 0 van a `transaction_data_cuarentena` junto con su motivo.
- **Gold:** una tabla por pregunta de negocio, guardada con `MERGE`. `ventas_departamento_periodo` es incremental por periodo.
- **Catálogo de datos:** todas las tablas tienen comentario, y también todas las columnas de Silver y Gold. Detalle en [docs/catalogo_datos.md](docs/catalogo_datos.md).
- **Workflow:** un Job serverless de 6 tareas, definido como Databricks Asset Bundle. Detalle en [docs/workflow.md](docs/workflow.md).

## Reglas de limpieza (Silver)

Los comentarios de `02_silver` citan estas reglas por su número.

| Regla | Qué hace |
|---|---|
| 1 | Nombres de columna en minúsculas (Bronze mezcla mayúsculas y minúsculas) |
| 2 | Texto sin espacios sobrantes |
| 3 | Importes a `decimal(10,2)`: se redondean y desaparece el ruido de punto flotante (2.22E-16) |
| 4 | Una línea con cantidad 0 no es una compra: va a `transaction_data_cuarentena` con su motivo |
| 5 | Las líneas con unidades y venta 0 se quedan en Silver |
| 6 | `trans_time` viene como HHMM sin ceros a la izquierda; se deriva la columna `hora` |
| 7 | Un tamaño vacío (`curr_size_of_product`) pasa a NULL: es ausencia de dato |
| 8 | Los 15 productos sin departamento se conservan con la etiqueta `SIN CLASIFICAR` |
| 9 | Combustible y misceláneos (KIOSK-GAS, MISC SALES TRAN, MISC. TRANS.) se marcan con `es_no_mercaderia` |
| 10 | Los valores de relleno "Unknown" pasan a NULL; `kid_category_desc` no se toca porque "None/Unknown" mezcla dos significados |
| 11 | `income_orden` da el orden real de los rangos de ingreso (como texto, "15-24K" queda después de "125-149K") |
| 12 | Se eliminan las filas idénticas (los 5,164 duplicados de `coupon`) |

## Decisiones de diseño (Gold)

Los comentarios de `03_gold` citan estas decisiones como D1 a D6.

| # | Decisión |
|---|---|
| D1 | La métrica de dinero es `sales_value` |
| D2 | La pregunta 1 y las métricas de unidades excluyen `es_no_mercaderia` |
| D2b | El gasto total del hogar sí usa todas sus ventas, incluida la no mercadería |
| D3 | El tiempo va en semanas y en periodos de 4 semanas |
| D4 | La demografía se une con LEFT JOIN y queda un grupo "Sin datos demográficos" |
| D5 | La pregunta 2 compara grupos, no afirma causalidad |
| D6 | Activo: 0 a 3 semanas sin comprar; en riesgo: 4 a 12; inactivo: 13 o más |

## Estructura del repositorio

| Ruta | Contenido |
|---|---|
| `notebooks/00_setup_unity_catalog` | Crea el catálogo, los esquemas y el Volume |
| `notebooks/00b_ingesta_kaggle` | Descarga los 7 CSV desde Kaggle al Volume. `causal_data` queda fuera de alcance |
| `notebooks/01_bronze` | Carga los CSV a `dunnhumby.bronze` |
| `notebooks/01b_diagnostico_calidad` | Diagnóstico exploratorio de calidad sobre Bronze (no es parte del Job) |
| `notebooks/02_silver` | Limpieza y `MERGE` incremental hacia `dunnhumby.silver` |
| `notebooks/02b_exploracion_gold` | Exploración previa al diseño de Gold (no es parte del Job) |
| `notebooks/03_gold` | Tablas de las 4 preguntas de negocio |
| `notebooks/04_catalogo_datos` | Comentarios de tablas y columnas, más una verificación |
| `databricks.yml`, `resources/job_medallion.yml` | Bundle del Workflow |
| `docs/` | Catálogo de datos y guía del Workflow |
| `screenshots/` | Evidencias de cada etapa, numeradas en el orden de la presentación |

## Cómo ejecutarlo

1. En Databricks (funciona en Free Edition), crea una carpeta Git con la URL de este repositorio.
2. Abre `databricks.yml`, despliega el bundle con el target `dev` y ejecuta el Job `dunnhumby_pipeline_medallion`.
3. Para demostrar la carga incremental, ejecuta el Job con `dia_hasta = 650`. Luego repite esa corrida y después ejecútalo con `dia_hasta = 711`. Los pasos están en [docs/workflow.md](docs/workflow.md).

## Resultados principales

- El 20 % de los hogares que más gasta genera el 53.05 % de las ventas. El 10 % genera el 34.29 %.
- GROCERY concentra el 55.45 % de las ventas de mercadería y DRUG GM el 14.30 %.
- Los 434 hogares que canjean cupones gastan en promedio 6,615, frente a 2,510 de los que no canjean. Es una asociación, no prueba que el cupón cause el gasto.
- Hay 335 hogares en riesgo (de 4 a 12 semanas sin comprar) y 174 inactivos (13 semanas o más).
- Limitación: solo 801 de los 2,500 hogares (32 %) tienen datos demográficos, así que la P3 se lee como una tendencia.
