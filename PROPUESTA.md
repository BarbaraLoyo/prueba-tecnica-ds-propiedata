# PROPUESTA.md — Plan Técnico para las Cuatro Áreas Restantes

**Propiedata · Prueba Técnica DS**

---

## 4.1 Matching de Inmuebles Únicos

### Diagnóstico

El dataset unificado contiene publicaciones del mismo inmueble físico repetidas entre plataformas y dentro de una misma plataforma. Esto distorsiona el entrenamiento (el modelo ve el mismo inmueble con variaciones mínimas de precio como si fueran observaciones independientes) y sesga las métricas de validación. En la prueba detecté que Nuñez aparece como "Nunez" y "Nuñez" en el mismo dataset, es un problema menor, pero indicativo de cuánto mayor es el problema a nivel de listings completos.

### Propuesta

El matching se aborda como un problema de **duplicación en dos etapas**:

**Etapa 1 — Bloqueo (Blocking):** No comparar todos los pares posibles (O(n²)). Usar reglas para agrupar candidatos, mismo barrio + mismo tipo_inmueble + superficie_cubierta dentro de ±5 m². Esto reduce el espacio de comparación en órdenes de magnitud.

**Etapa 2 — Scoring de similitud:** Para cada par candidato, calcular un score compuesto:
- Distancia entre coordenadas (si existen): umbral < 100m sugiere mismo inmueble.
- Precio: diferencia porcentual < 10% (considerando desfasaje temporal).
- Superficie, ambientes, dormitorios, baños: coincidencia exacta o con tolerancia de ±1.
- Título/descripción: similitud de texto con TF-IDF o Jaccard sobre tokens.

Agregar los scores en un clasificador binario (mismo inmueble / distinto). Si hay ejemplos etiquetados disponibles, un clasificador logístico sobre estas features es suficiente. Si no, un umbral de score > 0.85 funciona como regla heurística inicial.

**Features clave para el modelo de matching:** distancia_coords_m, diff_precio_pct, diff_superficie_m2, misma_antiguedad, jaccard_descripcion, mismo_barrio.

**Métricas de evaluación:** Precision y Recall sobre pares etiquetados manualmente (muestra de ~200 pares). F1 como métrica principal. El trade-off depende del objetivo: si deduplicamos para entrenamiento, preferimos alta Precision (no queremos descartar inmuebles distintos). Si deduplicamos para UI, preferimos alto Recall (no queremos mostrar duplicados al usuario).

**Riesgos:** Falsos positivos colapsan inmuebles distintos (el más costoso). Un PH y un departamento con misma superficie y barrio pueden tener precio muy diferente. Mitigación: el tipo_inmueble debe ser feature de bloqueo obligatorio, no solo de scoring.

**Esfuerzo estimado:** 2-3 semanas para una versión funcional con reglas + heurísticas. Un mes adicional para un clasificador supervisado si se generan etiquetas.

## 4.2 Arquitectura de Tablas del Pipeline

### Diagnóstico

Hoy el output del scraping llega directamente al dataset de entrenamiento, mezclando datos crudos con datos procesados. Esto hace frágil cualquier cambio en el scraper (un campo que cambia de nombre rompe el pipeline completo) y dificulta la trazabilidad de errores.

### Propuesta

Arquitectura en tres capas, inspirada en Medallion Architecture (Bronze / Silver / Gold):

```
SCRAPING
   │
   ▼
[Bronze] — raw_listings_{plataforma}_{fecha}
   Datos exactamente como vienen del scraper.
   Sin transformaciones. Inmutable. Versionado por fecha de ingesta.
   Schema mínimo: id_fuente, json_raw, timestamp_ingesta, fuente.
   Propósito: auditoría y re-procesamiento sin necesidad de re-scrapear.
   │
   ▼
[Silver] — listings_clean_{fecha}
   Schema unificado (el definido en notebooks/01_limpieza.py).
   Limpieza, normalización, parseo de tipos, flags de calidad.
   Una fila por publicación (sin deduplicar todavía).
   Versionado por fecha de procesamiento.
   │
   ▼
[Silver+] — listings_dedup_{fecha}  (requiere §4.1)
   Matching aplicado. Una fila por inmueble único estimado.
   Columna cluster_id que agrupa publicaciones del mismo inmueble.
   │
   ▼
[Gold] — dataset_entrenamiento_{version}
   Features de modelado calculadas (medianas de barrio, distancias, etc.).
   Ligado a una versión de modelo en MLflow.
   Input directo al pipeline de entrenamiento.
   │
   ▼
[Gold UI] — predictions_{fecha}
   Output de inferencia con precio estimado, intervalo de confianza,
   percentil relativo al barrio. Consumido por la API/UI.
```

**Criterios de versionado:** Cada tabla se versiona por fecha de procesamiento y se guarda en Parquet particionado por fecha. Las tablas Gold se ligan a un model_version de MLflow. El dataset de entrenamiento nunca se sobreescribe: se añade una versión nueva. Esto permite reproducir cualquier experimento pasado.

**Tecnología sugerida:** En la escala actual (decenas de miles de rows), DuckDB + Parquet local es suficiente y evita la complejidad de un data warehouse. Con crecimiento, migrar a BigQuery o Redshift es directo porque el schema está definido.

**Esfuerzo estimado:** 2 semanas para definir el schema y migrar el pipeline actual. 1 semana adicional para automatizar la orquestación con Airflow (ya existente en el stack).


## 4.3 Modelado Avanzado para R² = 0.8

### Diagnóstico

En esta prueba el modelo XGBoost con features geoespaciales alcanzó R² = 0.865 en validación cruzada 5-fold. Sin embargo, hay dos fuentes de sobreestimación: (1) el dataset puede tener duplicados del mismo inmueble entre train y validation, y (2) las medianas de barrio se calcularon sobre todo el dataset. Corregir esto puede bajar el R² real. El objetivo de 0.8 es alcanzable con el set de mejoras que detallo.

### Propuesta Priorizada por Impacto/Costo

**1. Target encoding de barrio con smoothing (alto impacto, bajo costo):** Reemplazar el simple label encoding de barrio por target encoding con factor de suavizado. Esto captura la señal de precio del barrio sin sobreajustar zonas con pocas observaciones. Implementación: la librería `category_encoders` tiene `TargetEncoder` con smoothing configurable. Debe aplicarse dentro del pipeline de CV para evitar leakage.

**2. Deflactor de inflación por fecha (alto impacto, costo medio):** El dataset cubre publicaciones de 2024-2025, período de alta inflación en Argentina. Un precio de enero 2024 no es comparable a uno de diciembre 2024. Incorporar un índice de inflación mensual (IPC del INDEC) como feature de deflación. Alternativamente, entrenar con precios deflactados y predecir en moneda constante. Esto debería reducir el RMSE significativamente.

**3. Segmentación por tipo de inmueble (impacto medio, bajo costo):** El análisis de errores mostró que Casas tiene error absoluto promedio 2x mayor que Departamentos, con solo 681 observaciones. Entrenar un modelo específico para Casas (posiblemente con features distintas) o usar un ensemble donde el tipo sea la variable de ramificación.

**4. Features de vecindad con radio (impacto medio, costo medio):** Para cada inmueble con coordenadas, calcular: precio mediano de los k vecinos más cercanos (k=20, 50), densidad de publicaciones en radio de 500m. Esto captura microzonas que el barrio administrativo no distingue. Implementación: KDTree sobre coordenadas.

**5. LightGBM + Optuna para tuning (impacto bajo, costo bajo):** LightGBM suele igualar o superar a XGBoost con menos tiempo de entrenamiento. Combinar con Optuna para búsqueda bayesiana de hiperparámetros con CV anidada (10 folds externos, 5 internos). Esto agrega disciplina experimental sin cambiar la arquitectura.

**6. Stacking (impacto bajo, costo alto):** Un meta-modelo (Ridge) sobre las predicciones de RF + XGBoost + LightGBM. Justifica el costo solo si los modelos base tienen errores poco correlacionados, lo cual hay que verificar antes de implementar.

**Esfuerzo estimado:** Items 1-3: 1 semana. Items 4-5: 1 semana adicional. Item 6: solo si los anteriores no alcanzan.

## 4.4 MLOps / CI-CD

### Diagnóstico

Hoy no existe tracking sistemático de experimentos ni despliegue automatizado. El riesgo es que el modelo en producción no sea el mejor entrenado, que no haya manera de detectar degradación de performance, y que un rollback sea manual y costoso.

### Propuesta

**Stack mínimo viable (implementable en 2-3 semanas):**

- **Tracking:** MLflow ya incluido en la prueba. Cada run loguea parámetros, métricas y el artefacto del modelo. El MLflow Tracking Server se despliega en una VM o en Railway (costo ~$5/mes). Esto ya está implementado en el notebook 02.
- **Registro de modelos:** MLflow Model Registry con tres etapas: `Staging` (entrenado, validado en CV), `Production` (promovido manualmente tras revisión), `Archived` (reemplazado). La promoción requiere que R² ≥ 0.80 y RMSE no regrese más de 5% respecto al modelo en Production.
- **Reentrenamiento:** Script Python orquestado por Airflow (ya en el stack) que corre semanalmente, entrena, logguea en MLflow y abre un PR en GitHub con el nuevo run_id. La revisión sigue siendo manual.
- **Monitoreo básico:** Log de predicciones con timestamp, precio_predicho, features de entrada. Dashboard en Metabase (o similar, compatible con el stack actual) que muestra distribución de predicciones vs. histórico. Alerta si la distribución de precios predichos se desplaza más de 2 desviaciones estándar.

**Stack ideal (a 3-6 meses):**

- **CI/CD con GitHub Actions:** Pipeline que corre los tests de limpieza y el entrenamiento en cada push a `main`. Si las métricas superan el threshold, promueve automáticamente a Staging.
- **Monitoreo de drift:** Evidently AI para detectar data drift (cambio en distribución de features) y concept drift (cambio en la relación features→precio). Corre semanalmente contra una ventana de referencia de 30 días.
- **Serving:** FastAPI como wrapper del modelo cargado desde MLflow. Dockerizado y desplegado en Railway o Fly.io. El endpoint `/predict` recibe las features del inmueble y retorna precio estimado + intervalo de confianza.
- **Feature Store ligero:** DuckDB con las medianas de barrio precalculadas, actualizadas semanalmente. Evita recalcular features en tiempo de inferencia.

**Riesgos:** El mayor riesgo en MLOps para un equipo pequeño es la sobreingeniería. Recomiendo implementar el stack mínimo viable primero y migrar al ideal cuando el volumen de predicciones o el equipo lo justifiquen. Un MLflow bien usado vale más que un Kubeflow mal configurado.

**Esfuerzo estimado:** Stack mínimo: 2-3 semanas. Stack ideal: 2-3 meses adicionales, iterando.


## Priorización General: Si tuviese 1 mes

Si tuviese un mes para atacar estos cuatro frentes, el orden sería:

**Semana 1 — Arquitectura de tablas (§4.2):** Es la base de todo lo demás. Sin un pipeline limpio y versionado, cualquier mejora de modelo es frágil. El impacto es multiplicador: ordena el scraping, facilita el reentrenamiento, habilita el MLOps. Es la deuda técnica más urgente.

**Semana 2 — MLOps mínimo viable (§4.4):** Con MLflow ya integrado en el código de la prueba, el esfuerzo marginal es bajo. Levantar el Tracking Server y el Model Registry en una semana da visibilidad inmediata sobre qué modelos están en producción y cuál fue el mejor experimento histórico.

**Semana 3 — Modelado avanzado (§4.3):** Con el pipeline ordenado y el tracking funcionando, los experimentos de mejora de modelo son reproducibles y comparables. Priorizar target encoding + deflactor de inflación, que son los cambios de mayor impacto esperado.

**Semana 4 — Matching (§4.1):** Es el trabajo más costoso y el que más datos etiquetados requiere. Lo dejo para el final porque la mejora en el modelo no lo requiere de manera urgente (los duplicados suman ruido pero no invalidan el R²), pero sí es crítico para la UI y para la validez de las métricas a largo plazo.

Esta priorización privilegia impacto inmediato y fundamentos sólidos sobre la feature más llamativa.
