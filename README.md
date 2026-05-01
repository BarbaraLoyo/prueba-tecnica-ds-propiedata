# Prueba Técnica — Data Scientist · Propiedata (Sudata)

> Repositorio público de la prueba técnica para el rol de **Data Scientist** en
> [Sudata](https://sudata.co), aplicado al producto **Propiedata** (estimación de
> precios inmobiliarios).

---

## ¿Qué es esto?

Si llegaste acá es porque te invitamos a hacer la prueba técnica para sumarte a
Sudata como Data Scientist. Este repo contiene:

- **`Prueba_Tecnica_DS_Propiedata.docx`** — la consigna completa, con detalles de
  evaluación, entregables y rúbrica.
- **`data/`** — los tres datasets sintéticos sobre los que vas a trabajar.
- **`notebooks/`** — carpeta vacía donde vas a poner tu trabajo.
- **`output/`** — carpeta vacía para tus salidas (datasets unificados, modelos, etc.).

---

## TL;DR

- **Plazo:** 1 semana calendario desde recepción de la invitación.
- **Esfuerzo esperado:** ~4 horas de trabajo real (autoorganizadas).
- **Entrega:** fork de este repo, en una rama `prueba-tecnica/<tu-apellido>`.
- **Lenguaje:** Python. Las librerías que prefieras. MLflow opcional pero suma.

---

## Cómo arrancar

### 1. Fork

Hacé fork de este repo a tu cuenta personal de GitHub. (Botón **Fork** arriba a la
derecha en GitHub).

### 2. Cloná tu fork

```bash
git clone https://github.com/<tu-usuario>/<nombre-del-fork>.git
cd <nombre-del-fork>
git checkout -b prueba-tecnica/<tu-apellido>
```

### 3. Leé la consigna completa

Abrí el archivo `Prueba_Tecnica_DS_Propiedata.docx` para ver la consigna en detalle:
contexto del producto, las dos tareas a codear, la propuesta escrita, la rúbrica de
evaluación y los entregables esperados.

### 4. Explorá los datos

En `data/` vas a encontrar tres CSVs con listings de alquileres de tres plataformas
distintas. Cada uno tiene su propio schema. Una parte de la prueba es descubrir los
problemas que tienen los datos.

```python
import pandas as pd

a = pd.read_csv("data/listings_plataforma_A.csv")
b = pd.read_csv("data/listings_plataforma_B.csv")
c = pd.read_csv("data/listings_plataforma_C.csv")
```

### 5. Trabajá

La estructura sugerida del fork al entregar es:

```
tu-fork/
├── data/                          ← datasets recibidos
├── notebooks/
│   ├── 01_limpieza.ipynb         ← Tarea 1
│   └── 02_modelado.ipynb         ← Tarea 2
├── output/
│   └── dataset_unificado.parquet ← salida de Tarea 1
├── PROPUESTA.md                  ← Parte B (4 áreas restantes)
├── README_ENTREGA.md             ← cómo correr tu código en 1 minuto
└── requirements_entrega.txt      ← deps adicionales si las hay
```

### 6. Entregá

Cuando termines, push a tu fork y respondé al email de la convocatoria con el link
del fork (rama incluida).

---

## Lo que valoramos

- **Pensar como product DS:** criterio sobre qué vale la pena resolver, qué no.
- **Honestidad** sobre lo que no llegaste a hacer y por qué.
- **Decisiones explicables** — más importante el *por qué* que el *qué*.
- **Pragmatismo:** 4 horas alcanzan para mostrar criterio, no para resolver todo.

## Lo que NO valoramos

- Notebooks de 80 celdas con 200 gráficos exploratorios.
- Modelos perfectos sin justificación.
- Pasarse del tiempo esperado para sacar décimas de R².

---

## Sobre Sudata y Propiedata

**Sudata** es una empresa de analítica de datos que combina BI a medida con
productos propios. Trabajamos 100% remoto, en proyectos diversos: tableros de
gestión, ingestas desde APIs, datos públicos, modelos predictivos.

**Propiedata** es nuestro producto de estimación de precios inmobiliarios. Combina:

- Scraping de portales públicos para construir un dataset de propiedades.
- Limpieza, enriquecimiento geoespacial y feature engineering.
- Modelos predictivos (hoy Random Forest / XGBoost con lags espaciales).
- Una API de inferencia y una UI para usuarios finales.

El rol al que aplicás se enfoca en mejorar y escalar estos modelos, especialmente
en alquileres, donde el desempeño actual es débil.

---

## Dudas

Cualquier consulta durante la prueba, respondé al email de la convocatoria.
Te respondemos en menos de 24 hs.

¡Suerte!

---

*Este repositorio es público y los datasets son sintéticos — generados por nosotros
para simular los problemas reales del scraping. No contienen información real de
ningún portal ni de ningún cliente.*
