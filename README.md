# Detector de Anomalias de Precios y Comparador de Supermercados

Sistema de extraccion web, procesamiento ETL, deteccion no supervisada de anomalias de precios mediante Machine Learning y analisis exploratorio interactivo para evaluar la competitividad de supermercados en Argentina.

## 1. Arquitectura y Flujo del Sistema

El proyecto esta dividido en una etapa de **Entrenamiento y Analisis Offline** (basada en datos historicos oficiales del sistema SEPA) y un **Pipeline Productivo Automatizado** que extrae precios en tiempo real, detecta valores atipicos mediante una API y genera reportes automaticos.

```
[Apify Web Scraper] ──(Trigger)──> [Make Workflow]
                                         │
    ┌────────────────────────────────────┴────────────────────────────────────┐
    ▼                                                                         ▼
[Get Dataset Items] ──> [Array Aggregator] ──> [HTTP POST /procesar-lote (Render)] ──> [Google Drive & Gmail]
                                                          │
                                                          ▼
                                               [Dashboard Streamlit]
```

### Orquestacion en Make (`make_workflow/`)

1. **Trigger (`Apify - Watch Actor Runs`):** Monitorea y dispara el flujo al finalizar una ejecucion exitosa del scraper.
2. **Extraccion (`Apify - Get Dataset Items`):** Obtiene los registros crudos almacenados en el dataset de Apify.
3. **Agrupamiento (`Array Aggregator`):** Consolida los items en un unico lote JSON para enviarlos en una sola peticion HTTP.
4. **Inferencia y Scoring (`HTTP - POST /procesar-lote`):** Envia el lote a la API desplegada en Render, donde se normalizan los precios por unidad (Kg/L), se filtran anomalias con `IsolationForest` y se calcula el puntaje competitivo por rubro.
5. **Respaldo (`Google Drive - Upload a File`):** Almacena el reporte CSV generado por la API en Google Drive.
6. **Notificacion (`Gmail - Send an email`):** Envia un correo con las tablas HTML de rubros validados y anomalias detectadas.

## 2. Estructura del Repositorio

```
mi-proyecto-entrega/
├── actor_apify/                                 # Configuracion y extraccion web
│   ├── input_config.json                        # Parametros de entrada y URLs objetivo
│   ├── page_function.js                         # Funcion JavaScript de extraccion en el DOM
│   └── output_sample.json                       # Muestra del formato de salida de Apify
├── api_render/                                  # Microservicio FastAPI (desplegado en Render)
│   ├── main.py                                  # Logica de inferencia, scoring y reportes
│   ├── detector_anomalias_v2.joblib             # Modelo IsolationForest entrenado
│   ├── canasta_representativa_10_rubros_v2.csv  # Precios medianos de referencia por rubro
│   └── requirements.txt                         # Dependencias de la API
├── colabs/                                      # Ciencia de datos y modelado
│   └── Entrenamiento_Modelo_de_Detección_v2.ipynb # ETL, entrenamiento, evaluacion y scoring
├── dashboard/                                   # Aplicacion web en Streamlit
│   ├── app_eda.py                               # Codigo principal del tablero EDA
│   ├── df_limpio_v2.csv                         # Dataset procesado utilizado por el dashboard
│   └── requirements.txt                         # Dependencias del frontend
├── make_workflow/                               # Automatizacion del pipeline
│   ├── blueprint.json                           # Blueprint exportado del escenario de Make
│   └── flujo_make.png                           # Captura de la arquitectura del flujo
├── data/                                        # Muestras de datos del pipeline
│   ├── raw/
│   │   └── sepa_raw_sample.csv                  # Muestra cruda del dataset historico SEPA
│   └── processed/
│       ├── df_limpio_v2.csv                     # Dataset procesado tras ETL y Feature Engineering
│       └── canasta_representativa_10_rubros_v2.csv
└── README.md                                    # Documentacion general del proyecto
```

## 3. Descripcion de los Modulos

### A. Extraccion Web (`actor_apify/`)

* **Actor utilizado:** `apify/web-scraper` (Actor oficial de Apify Store).
* **Configuracion:** El archivo `input_config.json` define las categorias objetivo (Leche, Yerba Mate, Aceite, Fideos, Arroz, Azucar, Queso Cremoso, Gaseosa, Detergente y Papel Higienico) sobre tiendas web de supermercados (ej. Carrefour Argentina). El script `page_function.js` extrae descripcion, precio de lista, precio promocional y precio de referencia por Kg/Litro.

### B. ETL, Modelo ML y Reglas de Negocio (`colabs/`)

* **Dataset de Entrenamiento:** Registros oficiales del sistema **SEPA** (Precios Claros), procesando mas de **26.8 millones de filas** diarias de cadenas en Buenos Aires (`AR-B`): DIA, Carrefour (INC S.A.), Coto, Cooperativa Obrera y Cencosud.
* **ETL y Feature Engineering:**
  * Descarte de registros sin precio o con valores negativos, y validacion de falsos descuentos.
  * Normalizacion por unidad de medida (`Kg` o `L`) a partir de `productos_precio_referencia`, filtrando errores de unitizacion extremos (`0.1 <= ratio_ref <= 20`).
  * Agrupamiento diario por cadena y **10 rubros testigo**, construyendo tres ratios relativos frente a la mediana del mercado: `ratio_vs_mercado`, `descuento_anunciado` y `ratio_promo_vs_mercado`.
* **Modelo Seleccionado:** `IsolationForest` (`n_estimators=100`, `contamination=0.02`, `random_state=42`) para detectar precios atipicos (errores de carga o scraping) sin requerir datos etiquetados previamente.
* **Formula de Scoring Competitivo (0 a 100 puntos):**
  * **Precio de Lista vs. Mercado (50%):** `clip((precio_mercado_medio / precio_lista_ref) * 35, 0, 50)`.
  * **Precio Final con Promo vs. Mercado (35%):** `clip((precio_mercado_medio / precio_promo_ref) * 25, 0, 35)`.
  * **Agresividad de Ofertas (15%):** Otorga 15 puntos adicionales si el precio promocional mediano mejora el precio de lista de referencia.
* **Evaluacion Heuristica y Definicion de Rangos:**
  Al tratarse de un modelo no supervisado (`IsolationForest`) combinado con un indice continuo de competitividad (`0 a 100`), se establecieron reglas heuristicas por umbrales para interpretar los resultados:
  * **Rango 0 (`IsolationForest.predict() == -1` - Anomalia IA):** Dato atipico extremo por error de unitizacion (`Kg/L`) o fallo de scraping; se filtra y se envia al reporte de anomalias.
  * **Rango 1 (`Score >= 70` - Ofertas agresivas / Alta competitividad):** Precios significativamente por debajo de la mediana del mercado o con descuentos promocionales fuertes.
  * **Rango 2 (`55 <= Score < 70` - Precios equilibrados / En linea con el mercado):** Precios alineados con la mediana del mercado (`ratio_vs_mercado` cercano a `1.0`).
  * **Rango 3 (`Score < 55` - Sobreprecios / Fuera de competencia):** Precios regularmente por encima de la mediana del mercado sin promociones compensatorias.

### C. API de Inferencia (`api_render/`)

* **Repositorio original:** [https://github.com/Fxzerh/API_Modelo_Detector](https://github.com/Fxzerh/API_Modelo_Detector)
* **Endpoint en Produccion:** `POST https://api-modelo-detector.onrender.com/procesar-lote`
* **Contrato de Entrada (Request JSON):**

```json
{
  "fecha": "2026-11-23",
  "productos": [
    {
      "supermercado": "Carrefour",
      "id_producto": "7790742000102",
      "descripcion": "LECHE ENTERA LA SERENISIMA 1L",
      "precio_lista": 1650.0,
      "precio_referencia": 1650.0,
      "precio_promo": 1450.0
    }
  ]
}
```

* **Contrato de Salida (Response JSON):**
  Devuelve el conteo de rubros evaluados, las listas estructuradas `validados` (con su `score_competitividad`) y `anomalias` (con el motivo de rechazo), ademas de tablas formateadas (`html_validados`, `html_anomalias`) y el reporte descargable (`csv`) listo para ser consumido por Make.

### D. Dashboard Exploratorio (`dashboard/`)

* **Repositorio original:** [https://github.com/Fxzerh/datasets_CastomProject](https://github.com/Fxzerh/datasets_CastomProject)
* **Aplicacion en Vivo:** [https://datasetscastomproject-ueslf5xjxitlxkzv6wheji.streamlit.app/](https://datasetscastomproject-ueslf5xjxitlxkzv6wheji.streamlit.app/)
* **Secciones de `app_eda.py`:**
  1. **Comparativa de Precio por Supermercado segun Rubro:** Grafico de barras interactivo con el precio mediano de referencia (Kg/L) por cadena para cada uno de los 10 rubros.
  2. **Distribuciones de Precios y Ratios:** Histogramas para analizar la dispersion de precios por categoria y la competitividad relativa de cada cadena frente a la mediana del mercado.
  3. **Matriz de Correlacion por Rubro:** Mapa de calor (coeficiente de Pearson) entre las 7 variables numericas del dataset por categoria.
  4. **Seleccion Final de Features:** Documentacion de las 3 variables lineales y 4 ratios no lineales utilizados en el modelado.

## 4. Metricas de Evaluacion

* **Silhouette Score:** `0.8110`
* **Puntaje de Desempeno Compuesto:** `92.35 / 100`
  * *Sub-score Separacion (70%):* `90.55 / 100`
  * *Sub-score Calibracion (30%):* `96.55 / 100` (tasa real de anomalias detectadas de `2.07%` frente al `2.00%` configurado).
* **Impacto Operativo:** Filtrado automatico de precios corruptos o mal unitizados durante el scraping antes de generar los reportes de competitividad.

## 5. Instrucciones de Ejecucion Local

### 1. Levantar la API (`api_render/`)

```bash
cd api_render
pip install -r requirements.txt
uvicorn main:app --reload
```

### 2. Ejecutar el Dashboard (`dashboard/`)

```bash
cd dashboard
pip install -r requirements.txt
streamlit run app_eda.py
```

### 3. Reproducir el Scraper (`actor_apify/`)

1. Ingresar a `apify/web-scraper` en Apify Console.
2. Cambiar la vista de **Input** de `Form` a `JSON`.
3. Pegar el contenido de `actor_apify/input_config.json` y ejecutar **Start**.

### 4. Importar el Flujo en Make (`make_workflow/`)

1. Crear un nuevo escenario en Make.
2. Ir a **More (`...`) -> Import Blueprint** y cargar `make_workflow/blueprint.json`.
3. Configurar las credenciales correspondientes para Apify, Google Drive y Gmail.

## Presentación / Video

Podés ver nuestra presentación narrada sobre inteligencia competitiva acá:  
[Inteligencia competitiva para decidir a tiempo (Prezi)](https://prezi.com/page/kiNoHXPs53ij23MuZkSb67/inteligencia-competitiva-para-decidir-a-tiempo?mode=narrated&referral_token=gEcU9XlnB3FN)

> **Cuak-Data** — Comisión 1, Grupo 5  
> Somos el equipo **Cuak-Data** (Comisión 1, Grupo 5), integrado por:
> - **Alfonso Rodriguez Carlos German** — Data Science
> - **Milagro Agustina Ramos Abdala Bello** — Solution Engineer
> - **Melani Alexia Sol Morales Riveros** — Business Analyst
