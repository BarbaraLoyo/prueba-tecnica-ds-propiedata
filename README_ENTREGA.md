# README_ENTREGA.md

## Cómo correr el código en 1 minuto

### Requisitos

- Python 3.10+
- Los tres CSV en la carpeta `data/`

### Setup

```bash
pip install -r requirements_entrega.txt
```

### Ejecutar

```bash
# Tarea 1 — Limpieza y homogeneización
python notebooks/01_limpieza.py

# Tarea 2 — Modelado
python notebooks/02_modelado.py
```

El output del Tarea 1 (`output/dataset_unificado.parquet`) es consumido automáticamente por Tarea 2. Correr en orden.

### Ver experimentos en MLflow

```bash
mlflow ui --backend-store-uri mlruns/
# Abrir http://localhost:5000
```

---

## Estructura del repositorio

```
.
├── data/
│   ├── listings_plataforma_A.csv
│   ├── listings_plataforma_B.csv
│   └── listings_plataforma_C.csv
├── notebooks/
│   ├── 01_limpieza.py        ← Tarea 1: limpieza y unificación
│   └── 02_modelado.py        ← Tarea 2: modelado y comparación
├── output/
│   └── dataset_unificado.parquet   ← generado por Tarea 1
├── mlruns/                         ← generado por Tarea 2 (MLflow)
├── PROPUESTA.md                    ← Parte B: plan técnico 4 áreas
├── README_ENTREGA.md               ← este archivo
└── requirements_entrega.txt
```

---

## Resumen de resultados

| Modelo | R² (CV 5-fold) | RMSE | MAE | MAPE |
|---|---|---|---|---|
| Random Forest (baseline) | 0.8423 ± 0.011 | 234,938 ARS | 155,517 ARS | 16.9% |
| **XGBoost + features geo** | **0.8651 ± 0.010** | **217,290 ARS** | **146,947 ARS** | **16.4%** |
| XGBoost sin features geo | 0.7636 ± 0.007 | 287,940 ARS | 199,802 ARS | 22.4% |

El modelo seleccionado es **XGBoost con features geoespaciales** (R² = 0.865, supera el objetivo de 0.8).

Las features geoespaciales (distancia al centro, precio mediano del barrio, precio por m² mediano del barrio) aportaron +0.10 puntos de R² respecto a la versión sin ellas.

---

## Decisiones clave (resumen ejecutivo)

**Limpieza:**
- Precios en USD descartados (sin tipo de cambio histórico confiable).
- Plataforma C: toda la información estructurada se parseó desde el campo `descripcion` con regex.
- Latitudes con coma decimal en lugar de punto corregidas (bug de A).
- Missings en antigüedad: imputación por grupo + flag binario `antiguedad_missing`.
- Coordenadas fuera del rango CABA/GBA → NaN (no imputadas).

**Modelado:**
- Validación cruzada 5-fold en todos los experimentos (no solo train-test split).
- Análisis ablación: experimento sin features geo para cuantificar su aporte.
- Tracking en MLflow con parámetros, métricas y artefactos de modelo.
- Análisis de errores por tipo de inmueble y barrio.

**Lo que no llegué a hacer (y por qué):**
- Deflactor de inflación por fecha: requeriría una serie IPC externa que no estaba disponible en el tiempo de la prueba.
- Target encoding con smoothing dentro del CV: prioricé tener un pipeline funcional y explicable sobre exprimir décimas de R².
- Matching de duplicados: 4 horas no alcanzan; la estrategia completa está en PROPUESTA.md
