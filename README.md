# Modelo descriptivo-predictivo-prescriptivo de siniestros viales y localización de ambulancias en Aguascalientes

Este repositorio contiene los notebooks de la tesis llamada "Modelo predictivo espaciotemporal y optimización de cobertura para la gestión de recursos de emergerían vial en Aguascalientes", desarrollados para que los resultados sean replicables y verificables. El trabajo aborda la atención oportuna ante siniestros viales en Aguascalientes mediante una arquitectura secuencial de cuatro fases:

```
Preparación  ──►  Descriptiva  ──►  Predictiva  ──►  Prescriptiva
(Muestras)        (EDA)             (RF calibrado)   (MCLP / CPLEX)
```

|#|Notebook|Descripción|Entradas principales|Salidas principales|
|-|-|-|-|-|
|1|`01_Fase_de_preparación`|Integra accidentes, red vial, clima y calendario. Se construye la muestra de entrenamiento (casos y controles 1:4, diseño caso-cruzado), las variables de memoria temporal y las 60 instancias de prueba de 2024.|`Generados/accidentes_franjas.parquet`, `Generados/calendario.parquet`, `Generados/consultas_meteo_franjas.parquet`, `Generados/rnc_ags.parquet`|`df_entrenamiento.parquet`, `Instancias_Prueba/`|
|2|`02_Fase_descriptiva`|Análisis exploratorio de casos vs. controles: dinámica temporal, infraestructura vial, clima y memoria histórica local.|`df_entrenamiento.parquet`|Tablas y gráficos|
|3|`03_Fase_predictiva`|Se estima el riesgo dinámico de accidente en cada tramo de la red vial con un ensamble de Random Forest calibrado, con validación temporal (Walk Forward), búsqueda de hiperparámetros, calibración isotónica, umbral óptimo por F1, explicabilidad con SHAP, evaluación geoespacial y mapas de riesgo.|`df_entrenamiento.parquet`, `Instancias_Prueba/`|`config_modelo.json`, `umbral_config.json`, `predicciones.parquet`, `modelo_evaluacion.pkl`, `modelo_predictivo.pkl`, `shap_datos_holdout.joblib`, `evaluaciones_predictivo.csv`, `Outputs/Distribuciones de riesgo/`, `Outputs/Mapas de riesgo`|
|4|`04_Fase_prescriptiva`|Se utilizan las probabilidades predichas por el modelo anterior como demanda ponderada en un MCLP para ubicar ambulancias. Se definen candidatos de ubicación, se reducen candidatos por cobertura, se resuelve el MCLP con IBM CPLEX y se generan mapas de solución.|`predicciones.parquet`, `rnc_ags.parquet`, `umbral_config.json`|`resultados_optimizacion.parquet`, `Outputs/Soluciones MCLP/`|

> Los notebooks deben ejecutarse en orden, ya que cada fase consume los archivos generados por la anterior.

## Estructura del repositorio

```
.
├── 01_Fase_de_preparación.ipynb
├── 02_Fase_descriptiva.ipynb
├── 03_Fase_predictiva.ipynb
├── 04_Fase_prescriptiva.ipynb
├── Generados/                  		# Bases de datos procesadas e insumos intermedios
│   └── Instancias_Prueba/      		# 60 instancias de prueba
│   └── accidentes_franjas.parquet		# Base de datos de accidentes viales en franjas de seis horas en Ags.
│   └── calendario.parquet			# Calendario con los periodos no ordinarios
│   └── consultas_meteo_franjas.parquet	        # Base de datos meteorológica por franjas de seis horas por punto de consulta en Ags.
│   └── rnc_ags.parquet			        # Base de datos de la red vial de Ags.
│   └── nodos_rnc_ags.parquet			# Archivo complementario de la red vial de Ags. para conectividad
│   └── df_entrenamiento.parquet		# Registros para entrenamiento del modelo predictivo
│   └── predicciones.parquet			# Predicciones del modelo predictivo por instancia para todos la red vial
│   └── evaluaciones.csv			# Evaluaciones del modelo predictivo por instancia
│   └── resultados_optimizacion.parquet	        # Resultados del MCLP sobre las instancias de prueba
├── Outputs/                    		# Mapas y gráficos generados
│   └── Distribuciones de riesgo/      		# Gráficos de distribucion de riesgo sobre las instancias de prueba
│   └── Mapas de riesgo/      		        # Mapas de riesgo sobre las instancias de prueba
│   └── Soluciones MCLP/      		        # Soluciones del MCLP sobre las predicciones de las instancias de prueba
├── config_modelo.json          		# Mejores hiperparámetros del RF
├── umbral_config.json          		# Umbrales de clasificación (base y calibrado)
├── modelo_evaluacion.pkl          		# Modelo entrenado para evaluaciones
├── modelo_predictivo.pkl       		# Modelo final entrenado para predicciones
└── shap_datos_holdout.joblib       		# Resultados del proceso SHAP
```

## Datos

Los notebooks parten de la etapa de integración, por lo que se omite la obtención y filtrado de los datos crudos. Los archivos esperados en `Generados/` son:

|Archivo|Contenido|Fuentes|
|-|-|-|
|`accidentes_franjas.parquet`|Accidentes 2019–2024 proyectados a franjas de 6 horas y asociados a un tramo vial|Accidentes de Tránsito Terrestre en Zonas Urbanas y Suburbanas (ATUS), INEGI|
|`rnc_ags.parquet`|Red vial de Aguascalientes (segmentos de ~200 m) con sus atributos y geometría|Red Nacional de Caminos (RNC), INEGI, SICT e IMT|
|`consultas_meteo_franjas.parquet`|Variables meteorológicas agregadas por centro de consulta y franja|API de Open-Meteo (reanálisis ERA5)|
|`calendario.parquet`|Calendario de festivos, vacaciones, FNSM, promociones y pandemia|Elaboración propia bajo calendarios oficiales nacionales y locales|

## Requisitos

Los notebooks 1 a 3 se ejecutaron con **Python 3.13** y el notebook 4 con **Python 3.10.11**, por compatibilidad con `docplex`. Cada notebook verifica e instala sus dependencias en la primera celda.

|Notebooks|Librerías|
|-|-|
|1|`pandas`, `geopandas`, `numpy`, `swifter`|
|2|`pandas`, `numpy`, `matplotlib`, `seaborn`|
|3|`pandas`, `geopandas`, `numpy`, `scikit-learn`, `shap`, `joblib`, `matplotlib`, `seaborn`|
|4|`pandas`, `geopandas`, `matplotlib`, `docplex`|

El notebook 4 requiere además una instalación de **IBM ILOG CPLEX** accesible desde `docplex` con licencia activa para evitar restricciones de uso.

## Resumen de la metodología

* **Muestras:** 2019–2023 para entrenamiento y 2024 para prueba. Por cada accidente se generan dos controles espaciales y dos temporales (1 y 2 semanas atrás), con filtrado anti-solapamiento. Los primeros 56 días se usan solo para construir las variables de memoria.
* **Variables:** 55 covariables en cuatro bloques: infraestructura vial, meteorología de la franja previa, temporalidad (codificación cíclica) y memoria local de accidentes.
* **Modelo:** Random Forest con validación temporal, hiperparámetros seleccionados por PR-AUC y calibración *post hoc* con Regresión Isotónica. El último bloque temporal (~16.7 %) se reserva como *holdout*.
* **Instancias de prueba:** 60 franjas de 2024 en tres esquemas: contraste de clima, contraste de rutina y evolución de franjas consecutivas en un mismo día.
* **Optimización:** MCLP con radios de cobertura de 2000 y 3000 m y 5 o 6 ambulancias, es decir, 240 problemas independientes (60 instancias × 2 radios × 2 capacidades).

## Autores

Bryan Mauricio Castañeda Ramos · Centro de Investigación en Matemáticas (CIMAT) · bryan.castaneda@cimat.mx

Joel Antonio Trejo Sánchez (Director) · Centro de Investigación en Matemáticas (CIMAT) · jvelazco@cimat.mx

Jonás Velasco Álvarez (Codirector) · Centro de Investigación en Matemáticas (CIMAT) · joel.trejo@cimat.mx

