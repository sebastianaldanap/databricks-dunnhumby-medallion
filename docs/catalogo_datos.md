# Catálogo de datos

Proyecto *Del ticket al cliente: un lakehouse para la fidelización en retail* (Despensa Club).
Catálogo de Unity Catalog: `dunnhumby`. Esquemas: `bronze`, `silver`, `gold`. Volume de archivos crudos: `dunnhumby.bronze.raw_files`.

Estas descripciones se aplican en Unity Catalog con `notebooks/04_catalogo_datos.ipynb`, que también lista los tipos reales guardados en Delta. En Gold, "decimal" indica un importe; la precisión exacta la fija Spark al agregar y se ve en el Catalog Explorer.

## Bronze

Todas las columnas del CSV se guardan como `string`. Además, cada tabla tiene:

| Columna | Tipo | Descripción |
|---|---|---|
| `_archivo_origen` | string | Auditoría: ruta del CSV del Volume del que vino la fila |
| `_fecha_carga` | timestamp | Auditoría: fecha y hora en que se cargó la fila en Bronze |

| Tabla | Descripción |
|---|---|
| `bronze.campaign_desc` | Crudo de campaign_desc.csv (Kaggle). Todas las columnas como texto. Una fila por campaña de marketing. |
| `bronze.campaign_table` | Crudo de campaign_table.csv (Kaggle). Todas las columnas como texto. Qué hogares recibieron cada campaña. |
| `bronze.coupon` | Crudo de coupon.csv (Kaggle). Todas las columnas como texto. Productos a los que aplica cada cupón por campaña. |
| `bronze.coupon_redempt` | Crudo de coupon_redempt.csv (Kaggle). Todas las columnas como texto. Cupones canjeados por los hogares. |
| `bronze.hh_demographic` | Crudo de hh_demographic.csv (Kaggle). Todas las columnas como texto. Datos demográficos de 801 hogares. |
| `bronze.product` | Crudo de product.csv (Kaggle). Todas las columnas como texto. Catálogo de productos. |
| `bronze.transaction_data` | Crudo de transaction_data.csv (Kaggle). Todas las columnas como texto. Una fila por producto comprado en una canasta. |

## Silver

### `silver.campaign_desc`

Campañas de marketing con su tipo y vigencia. Limpieza: tipos y texto sin espacios.

| Columna | Tipo | Descripción |
|---|---|---|
| `description` | string | Tipo de campaña: TypeA, TypeB o TypeC |
| `campaign` | int | Identificador de la campaña de marketing (1 a 30) |
| `start_day` | int | Día de inicio de la campaña (misma escala que day) |
| `end_day` | int | Día de fin de la campaña (misma escala que day) |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |

### `silver.campaign_table`

Hogares que recibieron cada campaña. Una fila por hogar y campaña.

| Columna | Tipo | Descripción |
|---|---|---|
| `description` | string | Tipo de campaña: TypeA, TypeB o TypeC |
| `household_key` | int | Identificador único del hogar (cliente con tarjeta de fidelización) |
| `campaign` | int | Identificador de la campaña de marketing (1 a 30) |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |

### `silver.coupon`

Productos a los que aplica cada cupón en cada campaña. Sin filas duplicadas (regla 12).

| Columna | Tipo | Descripción |
|---|---|---|
| `coupon_upc` | bigint | Código único del cupón |
| `product_id` | bigint | Identificador único del producto |
| `campaign` | int | Identificador de la campaña de marketing (1 a 30) |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |

### `silver.coupon_redempt`

Canjes de cupones: qué hogar canjeó qué cupón, qué día y en qué campaña. Carga incremental por día con MERGE.

| Columna | Tipo | Descripción |
|---|---|---|
| `household_key` | int | Identificador único del hogar (cliente con tarjeta de fidelización) |
| `day` | int | Día del canje (1 a 711) |
| `coupon_upc` | bigint | Código único del cupón |
| `campaign` | int | Identificador de la campaña de marketing (1 a 30) |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |

### `silver.hh_demographic`

Perfil demográfico de 801 hogares (32% del panel). Los valores de relleno pasan a NULL (regla 10).

| Columna | Tipo | Descripción |
|---|---|---|
| `age_desc` | string | Rango de edad estimado del jefe de hogar |
| `marital_status_code` | string | Estado civil: A o B según la guía del dataset; NULL si era U (desconocido) |
| `income_desc` | string | Rango de ingreso anual del hogar (por ejemplo 35-49K) |
| `homeowner_desc` | string | Tenencia de la vivienda; NULL si era Unknown |
| `hh_comp_desc` | string | Composición del hogar (adultos y niños); NULL si era Unknown |
| `household_size_desc` | string | Número de personas en el hogar (1 a 5+) |
| `kid_category_desc` | string | Número de niños; None/Unknown se conserva porque mezcla dos significados |
| `household_key` | int | Identificador único del hogar (cliente con tarjeta de fidelización) |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |
| `income_orden` | int | Orden numérico del rango de ingreso (1 = Under 15K, 12 = 250K+), para ordenar correctamente (regla 11) |

### `silver.product`

Catálogo de productos con su jerarquía. Productos sin clasificar etiquetados y marca de no mercadería.

| Columna | Tipo | Descripción |
|---|---|---|
| `product_id` | bigint | Identificador único del producto |
| `manufacturer` | int | Código del fabricante |
| `department` | string | Departamento del producto; SIN CLASIFICAR si venía vacío (regla 8) |
| `brand` | string | Tipo de marca: National (marca de fabricante) o Private (marca propia) |
| `commodity_desc` | string | Categoría del producto; SIN CLASIFICAR si venía vacío |
| `sub_commodity_desc` | string | Subcategoría del producto; SIN CLASIFICAR si venía vacío |
| `curr_size_of_product` | string | Tamaño o presentación del producto; NULL si venía vacío (regla 7) |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |
| `es_no_mercaderia` | boolean | True para KIOSK-GAS, MISC SALES TRAN y MISC. TRANS., cuya cantidad no es comparable (regla 9) |

### `silver.transaction_data`

Líneas de compra limpias: una fila por producto en una canasta, solo con cantidad mayor que 0. Carga incremental por día con MERGE por basket_id + product_id. Tabla central del modelo.

| Columna | Tipo | Descripción |
|---|---|---|
| `household_key` | int | Identificador único del hogar (cliente con tarjeta de fidelización) |
| `basket_id` | bigint | Identificador único de la canasta (una visita de compra) |
| `day` | int | Día de la transacción, relativo al inicio del estudio (1 a 711) |
| `product_id` | bigint | Identificador único del producto |
| `quantity` | int | Unidades compradas del producto en la canasta |
| `sales_value` | decimal(10,2) | Monto cobrado por la línea, en dólares; redondeado a 2 decimales en Silver |
| `store_id` | int | Identificador de la tienda |
| `retail_disc` | decimal(10,2) | Descuento por tarjeta de fidelización; negativo cuando hay descuento |
| `trans_time` | int | Hora de la transacción en formato HHMM (45 = 00:45) |
| `week_no` | int | Semana de la transacción, relativa al inicio del estudio (1 a 102) |
| `coupon_disc` | decimal(10,2) | Descuento por cupón; negativo cuando se aplicó |
| `coupon_match_disc` | decimal(10,2) | Descuento de la tienda que iguala el cupón del fabricante; negativo cuando se aplicó |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |
| `hora` | int | Hora del día de la transacción (0 a 23), derivada de trans_time |

### `silver.transaction_data_cuarentena`

Líneas de transacción rechazadas por tener cantidad 0 (regla 4), con su motivo. Se conservan para auditoría. Carga incremental por día con MERGE.

| Columna | Tipo | Descripción |
|---|---|---|
| `household_key` | int | Identificador único del hogar (cliente con tarjeta de fidelización) |
| `basket_id` | bigint | Identificador único de la canasta (una visita de compra) |
| `day` | int | Día de la transacción, relativo al inicio del estudio (1 a 711) |
| `product_id` | bigint | Identificador único del producto |
| `quantity` | int | Unidades compradas del producto en la canasta |
| `sales_value` | decimal(10,2) | Monto cobrado por la línea, en dólares; redondeado a 2 decimales en Silver |
| `store_id` | int | Identificador de la tienda |
| `retail_disc` | decimal(10,2) | Descuento por tarjeta de fidelización; negativo cuando hay descuento |
| `trans_time` | int | Hora de la transacción en formato HHMM (45 = 00:45) |
| `week_no` | int | Semana de la transacción, relativa al inicio del estudio (1 a 102) |
| `coupon_disc` | decimal(10,2) | Descuento por cupón; negativo cuando se aplicó |
| `coupon_match_disc` | decimal(10,2) | Descuento de la tienda que iguala el cupón del fabricante; negativo cuando se aplicó |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |
| `hora` | int | Hora del día de la transacción (0 a 23), derivada de trans_time |
| `motivo` | string | Motivo del rechazo: cantidad y venta en cero, o cantidad cero con venta positiva |

## Gold

### `gold.ventas_departamento_periodo`

Pregunta 1: ventas por departamento y periodo de 4 semanas, sin no mercadería. Incremental: solo se recalculan los periodos con datos nuevos.

| Columna | Tipo | Descripción |
|---|---|---|
| `periodo` | int | Periodo de 4 semanas (1 a 26); el 26 solo tiene 2 semanas |
| `department` | string | Departamento del producto |
| `ventas` | decimal | Suma de sales_value del departamento en el periodo |
| `lineas` | bigint | Número de líneas de transacción |
| `canastas` | bigint | Número de canastas distintas con productos del departamento |
| `semanas_en_periodo` | bigint | Semanas con datos en el periodo; menos de 4 indica un periodo incompleto |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |

### `gold.gasto_hogar_cupones`

Pregunta 2: una fila por hogar con su gasto, cupones canjeados y campañas recibidas.

| Columna | Tipo | Descripción |
|---|---|---|
| `household_key` | int | Identificador único del hogar (cliente con tarjeta de fidelización) |
| `ventas_total` | decimal | Gasto total del hogar en todo el periodo (suma de sales_value) |
| `descuento_cupon_total` | decimal | Descuento total por cupones (coupon_disc + coupon_match_disc), en positivo |
| `n_canastas` | bigint | Número de canastas (visitas) del hogar |
| `n_cupones_canjeados` | bigint | Número de cupones canjeados; 0 si no canjeó |
| `n_campanas_recibidas` | bigint | Número de campañas distintas que recibió; 0 si ninguna |
| `canjeo_cupon` | boolean | True si el hogar canjeó al menos un cupón |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |

### `gold.gasto_por_perfil_hogar`

Pregunta 3: gasto por hogar según ingreso y composición, incluido el grupo sin datos demográficos.

| Columna | Tipo | Descripción |
|---|---|---|
| `dimension` | string | Dimensión de análisis: ingreso o composicion_hogar |
| `categoria` | string | Valor de la dimensión; incluye Sin datos demográficos y Composición desconocida |
| `orden` | int | Orden de presentación de la categoría (99 = sin datos demográficos) |
| `n_hogares` | bigint | Número de hogares en la categoría |
| `ventas_total` | decimal | Gasto total de los hogares de la categoría |
| `gasto_promedio_hogar` | decimal | Gasto promedio por hogar |
| `gasto_mediano_hogar` | decimal | Gasto mediano por hogar (aproximado), menos sensible a hogares muy grandes |
| `canastas_promedio` | double | Canastas promedio por hogar |
| `gasto_por_canasta` | decimal | Gasto total dividido entre canastas (ticket promedio) |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |

### `gold.segmentos_hogar`

Pregunta 4: una fila por hogar con su decil de gasto y su estado de actividad.

| Columna | Tipo | Descripción |
|---|---|---|
| `household_key` | int | Identificador único del hogar (cliente con tarjeta de fidelización) |
| `ventas_total` | decimal | Gasto total del hogar en todo el periodo |
| `n_canastas` | bigint | Número de canastas del hogar |
| `ultima_semana` | int | Última semana con compra |
| `semanas_sin_comprar` | int | Semanas desde la última compra hasta la semana 102 |
| `estado_actividad` | string | Activo (0 a 3 semanas sin comprar), En riesgo (4 a 12) o Inactivo (13 o más) |
| `decil_gasto` | int | Decil de gasto: 1 es el 10% de hogares que más gasta |
| `_fecha_proceso` | timestamp | Auditoría: fecha y hora en que la fila se insertó o cambió por última vez en esta capa |
