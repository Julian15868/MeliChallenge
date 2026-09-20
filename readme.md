# Challenge Data Scientist – Antiscraping Security (Mercado Libre)

Resolución del challenge técnico de Data Scientist: detección de sesiones de scraping en tráfico de un marketplace, y scraping + clasificación de vulnerabilidades de Debian.

## Estructura

```
.
├── data/
│   └── session_requests.csv          # dataset de requests (Problema 1, ~1350 sesiones / ~190k requests)
├── docs/
│   └── Enunciado - Challenge Data Scientist - ago26.pdf
├── notebook/
│   ├── Problema 1- Deteccion de scrapers/
│   │   ├── 01_eda_anomaly_detection.ipynb
│   │   └── reporte_sesiones_sospechosas_consolidado.pdf   # reportes generados por LLM (3 sesiones)
│   └── Problema 2- Vulnerabilidades debian/
│       ├── 02_debian_security_scraping.ipynb
│       ├── debian_security_advisories_ultimos_3_meses.csv # scrape crudo
│       └── debian_vulnerabilities_looker.csv               # dataset final clasificado (usado en Looker Studio)
├── requirements.txt
└── .gitignore
```

## Problema 1: Detección de scrapers

**Enfoque A — Reglas y etiquetado:**
- EDA por sesión (volumen de requests, paths únicos, dispositivos únicos, IPs, user agents, intervalos entre requests).
- Reglas de anomalía sobre percentiles de la población de sesiones (alto request_count, alto request_rate, muchos dispositivos únicos, intervalos muy bajos y consistentes, etc.) que producen `anomaly_score`, `strong_signal` e `is_scraping`.
- Generación de reportes en lenguaje natural vía API de Gemini para 3 sesiones (`session_1307`, `session_1042`, `session_1115`) — ver `reporte_sesiones_sospechosas_consolidado.pdf`.

**Enfoque B — Modelo de Machine Learning:**
- `RandomForestClassifier` (PySpark ML) entrenado sobre las features de sesión, usando como etiqueta el `is_scraping` del Enfoque A.
- Desbalance de clases resuelto con ponderación de clases (`weightCol`), con discusión en el notebook de las alternativas consideradas (oversampling/SMOTE, undersampling, ajuste de umbral) y por qué se descartaron.
- Métricas: F1-score, precision y recall (justificadas en el notebook frente a por qué accuracy no es confiable con clases desbalanceadas), más un ranking de importancia de variables (`featureImportances`) como chequeo cruzado contra las reglas del Enfoque A.
- ⚠️ Nota abierta: hay dos celdas que redefinen `is_scraping` (una combina `anomaly_score` y la regla de actividad rápida, la otra la pisa usando solo esta última) — señalado con una nota en el propio notebook para resolver antes de la entrega final.

## Problema 2: Vulnerabilidades Debian

- Scraping de los avisos de seguridad de Debian de los últimos 3 meses (`requests` + `BeautifulSoup`), con soporte para ventanas que cruzan un cambio de año → `debian_security_advisories_ultimos_3_meses.csv`.
- Clasificación del tipo de vulnerabilidad por similitud semántica de embeddings (`sentence-transformers`, modelo `all-MiniLM-L6-v2`) con respaldo por palabras clave, justificada en el notebook frente a otros enfoques posibles (reglas puras, clasificación vía LLM) → `debian_vulnerabilities_looker.csv`.
- Visualizaciones en Looker Studio: **[tablero](https://datastudio.google.com/reporting/3f9682cc-c45b-4c86-a58f-b82a9c01228d)** — cantidad de avisos por tipo de vulnerabilidad, evolución semanal con tendencia, y top de paquetes más afectados, con scorecards de cantidad de avisos, categorías detectadas y paquetes afectados.

## Cómo correrlo

```bash
python -m venv .venv
.venv\Scripts\activate      # Windows
pip install -r requirements.txt
```

Crear un archivo `.env` en la raíz con:

```
GEMINI_API_KEY="tu-api-key-de-gemini"
```

Abrir y correr los notebooks en `notebook/`.

## Decisiones y mejoras a futuro

- Las reglas del Enfoque A son heurísticas basadas en percentiles de la misma ventana de tráfico (no hay historial previo de las sesiones, según el enunciado); una mejora futura sería contrastarlas contra ventanas históricas para reducir falsos positivos.
- Resolver la ambigüedad en la definición final de `is_scraping` (ver nota en el Enfoque B).
- Reportar matriz de confusión y métricas específicas de la clase "scraping" (no solo promediadas) para el modelo de ML.
- Validación cruzada (`CrossValidator`) para elegir hiperparámetros del Random Forest en vez de valores fijos.
