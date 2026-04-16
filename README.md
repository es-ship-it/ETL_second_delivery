# Proyecto ETL — Segunda Entrega

Análisis de cobertura de vacunación COVID-19 y decesos causados por este en Ecuador alineado con el ODS 3 (Salud y Bienestar). Esta entrega automatiza la ingesta, transformación y validación de datos mediante Apache Airflow, Great Expectations y SQLite, y genera visualizaciones a partir del Data Warehouse validado.

---

## 📖 Descripción del proyecto

El objetivo es construir un pipeline ETL reproducible y automatizado que integre tres fuentes de datos:

1. **Primera entrega** — registros históricos de vacunación por cantón (SQLite)
2. **BigQuery (JHU CSSE)** — muertes acumuladas y diarias por COVID-19 en Ecuador
3. **API World Bank (UHC Index)** — índice de cobertura universal de salud (UHC Score) para Ecuador

Con esto se busca responder preguntas como:
- ¿Cómo evolucionó la campaña de vacunación mes a mes?
- ¿Qué regiones, provincias y cantones lideraron en cobertura?
- ¿Existe relación entre el ritmo de vacunación y las muertes COVID en 2021?
- ¿Cómo ha mejorado el índice UHC de Ecuador en los últimos años?
- Entre otros posibles analisis derivados.

---

## 🎯 Alineación con ODS 3

| Meta | Indicador cubierto |
|------|--------------------|
| 3.8 – Cobertura universal de salud | UHC Score |
| 3.d – Gestión de riesgos de salud | Muertes COVID |

---

## 🏗️ Arquitectura del pipeline

```
[SQLite 1ª entrega]  [BigQuery JHU]  [API World Bank]
        │                  │                │
        └──────────────────┴────────────────┘
                           │
                    [Apache Airflow]
                         (DAG)
                           │
              ┌────────────┴────────────┐
         [Extracción]            [Transformación]
           3 tareas             Limpieza + Merge
              │                        │
              └────────────┬───────────┘
                           │
                [Great Expectations]
                Validación de calidad
                           │
                     (si pasa)
                           │
                    [Data Warehouse]
                  SQLite — Star Schema
                           |
                           |
                          EDA
                           │
                    [Visualizaciones]

```

### Flujo del DAG

```
extract_db ──┐
             ├──► transform ──► quality_checks ──► load
extract_bq ──┤
             │
extract_api ─┘
```

---

## ⭐ Modelo de datos — Galaxy Schema

### Tablas de hechos

| Tabla | Descripción |
|-------|-------------|
| `fact_vacunacion` | Dosis aplicadas por cantón y fecha |
| `fact_decesos` | Muertes COVID diarias y acumuladas por fecha |

### Dimensiones

| Tabla | Descripción |
|-------|-------------|
| `dim_fecha` | Calendario (año, mes, semana, día, trimestre) |
| `dim_canton` | Cantones con población |
| `dim_provincia` | Provincias con población |
| `dim_region` | Regiones y zonas geográficas |
| `dim_indice_uhc` | UHC Score anual de Ecuador |

---

## 🛠️ Stack tecnológico

| Capa | Herramienta | Propósito |
|------|-------------|-----------|
| Orquestación | Apache Airflow 2.11 | Automatización y scheduling del ETL |
| Fuente externa | BigQuery (JHU CSSE) | Datos de muertes COVID-19 |
| Fuente API | World Bank Data360 | Índice UHC Ecuador |
| Almacenamiento | SQLite | Data Warehouse (star schema) |
| Validación | Great Expectations ≥1.0 | Calidad de datos antes de cargar |
| Transformación | Python / Pandas | Limpieza y enriquecimiento |
| Visualización | Matplotlib / Sweetviz | Dashboard y reportes automáticos |
| Documentación | GitHub + Markdown | Versionado y trazabilidad |

> **Nota sobre SQLite:** Se utiliza SQLite en lugar de PostgreSQL/MySQL por las restricciones del entorno Google Colab (sin servidor dedicado). El esquema relacional, las claves foráneas y las restricciones de integridad se mantienen igual que en un motor de producción.

---

## 📁 Estructura del repositorio

```
etl-segunda-entrega/
│
├── dags/
│   └── etl_vacunacion_ecuador.py      # DAG principal de Airflow
│
├── notebooks/
│   ├── ETL_second_delivery_def.ipynb  # Pipeline completo (ETL + Airflow)
│   └── EDA_second_delivery_ETL_def.ipynb  # Análisis exploratorio + Dashboard
│
├── docs/
│   ├── arquitectura_pipeline.png      # Diagrama de arquitectura
│   └── star_schema.png                # Diagrama del modelo de datos
│
├── visualizations/
│   ├── dashboard_vacunacion_ecuador.png
│   ├── eda_covid_timeseries.png
│   ├── sweetviz_fact.html
│   ├── sweetviz_covid.html
│   └── sweetviz_uhc.html
│
├── README.md
└── .gitignore
```

---

## ▶️ Instrucciones de replicación (Google Colab)

### Requisitos previos

- Cuenta de Google con acceso a Google Drive y BigQuery
- Acceso a este repositorio

### Pasos

1. **Clonar el repositorio** o descargar los notebooks

2. **Abrir en Google Colab** el archivo `ETL_second_delivery_def.ipynb`

3. **Montar Google Drive** y ajustar las rutas:
   ```python
   ruta_directorio_drive = "/content/drive/Shared drives/TU_CARPETA/"
   ```

4. **Autenticación con BigQuery:**
   ```python
   from google.colab import auth, userdata
   auth.authenticate_user()
   os.environ["GCP_PROJECT_ID"] = userdata.get("project_gcp")
   ```

5. **Ejecutar las celdas en orden:**
   - Instalación de Airflow y Great Expectations
   - Definición del DAG
   - Creación del usuario administrador
   - Inicio del webserver + scheduler + túnel Cloudflare

6. **Acceder a la UI de Airflow** con la URL generada por Cloudflare  
   `Usuario: admin | Contraseña: admin`

7. **Para el EDA**, abrir `EDA_second_delivery_ETL_def.ipynb` y ejecutar las celdas en orden (requiere que el ETL haya corrido primero y los archivos `tf_*.csv` estén en Drive)

---

## 📊 Análisis y visualizaciones

El dashboard final (8 paneles) incluye:

1. Dosis totales por mes
2. Composición de dosis por mes (apilado)
3. Top 10 cantones por dosis totales
4. Top 10 provincias por dosis totales
5. Dosis totales por región
6. Muertes COVID vs vacunación en 2021
7. Cobertura vacunal por cantón (% sobre población)
8. UHC Score histórico de Ecuador

---

## ✅ Great Expectations — Resumen de validaciones

| Dataset | Expectativas aplicadas |
|---------|----------------------|
| `fact_vacunacion` | No nulos en IDs, valores ≥ 0 en dosis, conteo mínimo de filas |
| `dim_canton` | ID único, nombre no nulo, población ≥ 0 |
| `dim_fecha` | ID único, fecha no nula, mes entre 1–12, día entre 1–31 |
| `covid_deaths` | Fecha no nula, muertes ≥ 0, mes entre 1–12 |
| `uhc_index` | Año único, score entre 0–100 |
| `fact_merged` | Año único, dosis ≥ 0 |

Solo se carga al DWH si **todas las validaciones pasan**. En caso contrario, el DAG lanza excepción y detiene la ejecución.

---

## 📌 Decisiones de diseño

- **Datos de dosis son acumulativos por cantón:** se toma el `MAX` por cantón/mes antes de sumar a nivel nacional, para evitar doble conteo.
- **Muertes diarias COVID:** se calculan como diferencia entre valores acumulados consecutivos (`diff().clip(lower=0)`).
- **UHC Index:** se filtra solo `INDEX_COMP_LABEL = "Full index"` para evitar sub-indicadores.
- **Merge por año:** las tres fuentes se unen a granularidad anual dado que el UHC Score solo tiene datos anuales.

---

## 👥 Equipo

Proyecto desarrollado para el curso **ETL (G51)** — Ingeniería de Datos e Inteligencia Artificial, Universidad Autónoma de Occidente.
