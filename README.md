![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | Workflow Medallion y Modelado Silver en Snowflake

<!-- (Módulo 1) -->

## 🎯 Objetivo

En este lab implementarás un pipeline completo de ingestión, limpieza y modelado usando Snowflake y la arquitectura Medallion. Practicarás el uso de stages internos, `COPY INTO`, nomenclatura SEAT y modelado normalizado en Silver.


## 📦 Dataset: `orders.csv`

Contiene:

```
order\_id,customer\_id,product\_id,quantity,price\_total,order\_date,order\_type
1001,123,P001,3,59.90,2024-12-01,online
1002,124,P002,1,19.90,2024-12-01,store
1003,125,P003,2,45.00,2024-12-02,online
1004,123,P001,1,19.97,2024-12-02,store
1005,126,P004,abc,39.90,2024-12-03,promo
1006,127,P005,2,invalid,invalid,online
````

🧠 **Preguntas:**
- ¿Qué errores detectas a simple vista en el dataset?
- ¿Qué tipo de validaciones deberías aplicar antes de pasar a Silver?


## 🥉 1: Ingesta en Bronze con COPY INTO

### 1.1 Crear tabla RAW

```sql
CREATE OR REPLACE TABLE DEV_BRONZE_DB.RAW.ORDERS_RAW (
  ORDER_ID STRING,
  CUSTOMER_ID STRING,
  PRODUCT_ID STRING,
  QUANTITY STRING,
  PRICE_TOTAL STRING,
  ORDER_DATE STRING,
  ORDER_TYPE STRING,
  AUD_TST_INGESTED TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  AUD_SRC_FILE STRING
);
````

### 1.2 Crear un internal stage

```sql
CREATE OR REPLACE STAGE DEV_BRONZE_DB.RAW.STAGE_ORDERS;
```

### 1.3 Subir el archivo `orders.csv`

Sube el archivo a Snowflake desde Snowsight o usando SnowSQL:

```
snowsql -q "PUT file://orders.csv @DEV_BRONZE_DB.RAW.STAGE_ORDERS AUTO_COMPRESS=TRUE"
```

### 1.4 Ejecutar `COPY INTO`

```sql
COPY INTO DEV_BRONZE_DB.RAW.ORDERS_RAW (
  ORDER_ID, CUSTOMER_ID, PRODUCT_ID, QUANTITY, PRICE_TOTAL, ORDER_DATE, ORDER_TYPE, AUD_SRC_FILE
)
FROM (
  SELECT $1, $2, $3, $4, $5, $6, $7, METADATA$FILENAME
  FROM @DEV_BRONZE_DB.RAW.STAGE_ORDERS (FILE_FORMAT => (
    TYPE = 'CSV', FIELD_OPTIONALLY_ENCLOSED_BY = '"', SKIP_HEADER = 1
  ))
);
```

## 🧪 2: Exploración y validación del Bronze

```sql
SELECT * FROM DEV_BRONZE_DB.RAW.ORDERS_RAW;
```

🧠 **Preguntas:**

* ¿Qué registros parecen corruptos o inválidos?
* ¿Qué campos requieren `TRY_TO_*` al pasar a Silver?
* ¿Debemos conservar los errores o excluirlos?

## 🥈 Parte 3: Limpieza y transformación en Silver

### 3.1 Crear tabla Silver

```sql
CREATE OR REPLACE TABLE DEV_SILVER_DB.S_SUPPLY_CHAIN.ORDERS_CLEAN AS
SELECT
  TRY_TO_NUMBER(ORDER_ID) AS ID_ORDER,
  TRY_TO_NUMBER(CUSTOMER_ID) AS ID_CUSTOMER,
  PRODUCT_ID AS ID_PRODUCT,
  CASE
    WHEN IS_NUMBER(QUANTITY) AND TRY_TO_NUMBER(QUANTITY) > 0 THEN TRY_TO_NUMBER(QUANTITY)
    ELSE NULL
  END AS QTY_ORDERED,
  TRY_TO_NUMBER(PRICE_TOTAL) AS AMT_TOTAL,
  TRY_TO_DATE(ORDER_DATE, 'YYYY-MM-DD') AS DTE_ORDER,
  NULLIF(UPPER(ORDER_TYPE), '') AS CAT_ORDER_TYPE,
  CURRENT_TIMESTAMP() AS AUD_TST_INSERTED
FROM DEV_BRONZE_DB.RAW.ORDERS_RAW
WHERE TRY_TO_NUMBER(ORDER_ID) IS NOT NULL;
```

### 3.2 Verificar la carga limpia

```sql
SELECT * FROM DEV_SILVER_DB.S_SUPPLY_CHAIN.ORDERS_CLEAN;
```

🧠 **Preguntas:**

* ¿Qué registros fueron descartados o modificados?
* ¿Qué decisiones tomaste al limpiar los datos?
* ¿Cómo documentarías estas reglas en tu `lab-notes.md`?

## 🧩 Parte 4: Modelado lógico en SqlDBM

1. Crear proyecto `S_SUPPLY_CHAIN`
2. Añadir tabla `ORDERS_CLEAN`
3. Añadir campos y prefijos:

   * `ID_`, `QTY_`, `AMT_`, `DTE_`, `CAT_`, `AUD_`
4. Marcar clave primaria `ID_ORDER`
5. Añadir documentación por campo (descripción, tipo)

📤 Exportar el script `CREATE TABLE` generado por SqlDBM para entregarlo

## 💡 Parte 5: Reflexión final

En tu archivo `lab-notes.md`, responde:

1. ¿Qué aprendiste sobre separar el dato crudo del limpio?
2. ¿Por qué es importante normalizar y limpiar antes de crear dashboards?
3. ¿Qué tips propones para mantener un pipeline ETL flexible?
4. ¿Cómo aplicarías esto en una empresa real como SEAT?

## ✅ Entregables

* `orders.csv`: archivo cargado
* `bronze_copyinto.sql`: tabla + ingestión COPY
* `silver_transformation.sql`: transformación a Silver
* `sqlDBM_model.sql`: script exportado del modelo
* `lab-notes.md`: documentación, decisiones de limpieza, reflexiones y tips

## 🏁 Conclusión

Este lab te ha permitido:

* Usar `COPY INTO` con internal stage
* Limpiar datos con funciones `TRY_TO_*`
* Diseñar una tabla Silver con buenas prácticas
* Documentar un modelo en SqlDBM
* Reflexionar sobre la eficiencia y escalabilidad del pipeline

➡️ Estos pasos son esenciales para garantizar calidad, trazabilidad y adaptabilidad en cualquier sistema moderno de datos.
