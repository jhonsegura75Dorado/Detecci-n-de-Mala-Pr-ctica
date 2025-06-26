# Análisis y Detección de Mala Práctica Transaccional (Fraccionamiento)

![Python](https://img.shields.io/badge/python-3.9%2B-blue.svg)
![Pandas](https://img.shields.io/badge/pandas-1.5%2B-yellow.svg)
![License](https://img.shields.io/badge/license-MIT-green.svg)

## Resumen del Proyecto

Este proyecto tiene como objetivo desarrollar un sistema para la detección de un patrón de comportamiento conocido como **"Mala Práctica Transaccional"**. Este comportamiento consiste en el fraccionamiento intencional de una transacción de alto valor en múltiples transacciones de menor monto para, posiblemente, evadir controles o límites de seguridad.

La solución propuesta utiliza un enfoque basado en heurísticas y análisis de datos para identificar grupos de transacciones sospechosas que, en conjunto, evidencian este patrón. Se analizan transacciones agrupadas por cliente, comercio y ventana de tiempo, aplicando un sistema de puntuación para cuantificar el nivel de sospecha.

## 1. Definición del Problema

La "Mala Práctica Transaccional" o fraccionamiento (también conocido en inglés como *structuring* o *smurfing*) se caracteriza por los siguientes atributos clave:

-   **Multiplicidad:** Una transacción grande se divide en varias transacciones más pequeñas.
-   **Ventana de Tiempo Corta:** Las transacciones fraccionadas ocurren en un período de tiempo acotado, típicamente dentro de 24 horas.
-   **Entidad Común:** Todas las transacciones están vinculadas a una misma entidad, ya sea:
    -   La misma cuenta de origen/destino (`account_number`).
    -   El mismo cliente (`user_id`).
    -   El mismo comercio y sucursal (`merchant_id`, `subsidiary`).
-   **Naturaleza Idéntica:** Generalmente, todas las transacciones son del mismo tipo (`transaction_type`, ej. todas débito).

El desafío es identificar estos patrones ocultos dentro de un gran volumen de datos transaccionales legítimos.

## 2. Metodología Propuesta

La solución se estructura como un pipeline de datos que consta de las siguientes etapas:



#### a. Ingesta y Preprocesamiento de Datos

-   **Carga de datos:** Lectura del dataset desde su fuente original (ej. CSV, base de datos).
-   **Limpieza:** Manejo de valores nulos o inconsistentes.
-   **Conversión de Tipos:** Asegurar que las columnas tengan el formato adecuado, especialmente `transaction_date` (a formato `datetime`) y `transaction_amount` (a numérico).
-   **Extracción de Features:** Creación de una columna `fecha` (solo la parte de la fecha) a partir de `transaction_date` para facilitar la agrupación en ventanas de 24 horas.

#### b. Agrupación de Transacciones

El núcleo del algoritmo consiste en agrupar las transacciones para formar "candidatos" a ser un evento de fraccionamiento. La clave de agrupación se define por las entidades comunes que deben compartir las transacciones:

`group_key = ['user_id', 'merchant_id', 'subsidiary', 'transaction_type', 'fecha']`

Al agrupar por esta clave, aislamos conjuntos de transacciones realizadas por el mismo usuario, en el mismo comercio/sucursal, del mismo tipo y en el mismo día.

#### c. Definición de Heurísticas de Detección

Una vez agrupadas las transacciones, aplicamos un conjunto de reglas (heurísticas) para evaluar si el grupo es sospechoso. A cada grupo se le calculan métricas agregadas:

-   `n_transacciones`: Número de transacciones en el grupo.
-   `monto_total`: Suma de los montos de las transacciones.
-   `monto_promedio`: Monto promedio por transacción.
-   `monto_std_dev`: Desviación estándar de los montos.
-   `rango_horario_min`: Diferencia de tiempo entre la primera y última transacción del grupo (si la data lo permite).

#### d. Sistema de Puntuación de Riesgo (Risk Scoring)

En lugar de una clasificación binaria (sospechoso/no sospechoso), implementamos un sistema de puntuación que ofrece mayor granularidad. Se asignan puntos a un grupo si cumple con ciertas condiciones. Un puntaje más alto indica una mayor probabilidad de ser una mala práctica.

| Heurística                                        | Condición de Ejemplo                  | Puntos Asignados | Racional                                                                   |
| ------------------------------------------------- | ------------------------------------- | ---------------- | -------------------------------------------------------------------------- |
| **H1: Frecuencia Anómala**                        | `n_transacciones` > 3                 | +3               | El fraccionamiento requiere múltiples transacciones.                         |
| **H2: Monto Agregado Relevante**                  | `monto_total` > 1,000,000             | +2               | Filtra grupos de bajo impacto económico.                                   |
| **H3: Montos Sospechosamente Similares**          | `monto_std_dev` / `monto_promedio` < 0.1 | +2               | Transacciones divididas en partes casi iguales son un fuerte indicador.    |
| **H4: Patrón de "Casi Límite" (Avanzado)**        | `monto_promedio` cercano a un límite conocido  | +1               | Podría indicar un intento de mantenerse por debajo de un umbral de reporte. |
| **H5: Concentración Temporal**                    | `rango_horario_min` < 60              | +1               | Eventos que ocurren en menos de una hora son más sospechosos.              |

*Nota: Los umbrales y puntos son configurables y deben ser calibrados con datos históricos y conocimiento del negocio.*

#### e. Generación de Reportes

El resultado final es un archivo (ej. `reporte_sospechosos.csv`) que contiene los grupos que superaron un umbral de riesgo (ej. `score >= 5`). Este reporte incluye las métricas agregadas y los identificadores del grupo para una posterior investigación manual o automática.

## 3. Estructura del Repositorio

El proyecto está organizado de la siguiente manera para facilitar la reproducibilidad y la colaboración:

```
.
├── data/
│   ├── raw/
│   │   └── transactions.csv      # Datos brutos de entrada (no subir a Git si son sensibles)
│   └── processed/
│       └── suspicious_groups.csv # Output del análisis
│
├── notebooks/
│   ├── 01_EDA_y_Preprocesamiento.ipynb  # Análisis Exploratorio y limpieza de datos
│   └── 02_Desarrollo_Algoritmo_Deteccion.ipynb # Desarrollo y prueba de la lógica de detección
│
├── src/
│   ├── __init__.py
│   ├── data_processing.py      # Funciones para cargar y preprocesar los datos
│   ├── detection_logic.py      # Lógica de agrupación, heurísticas y scoring
│   └── main.py                 # Script principal para ejecutar el pipeline completo
│
├── .gitignore                    # Archivos y carpetas a ignorar por Git
├── LICENSE                       # Licencia del proyecto (e.g., MIT)
├── README.md                     # Este documento
└── requirements.txt              # Dependencias de Python para el proyecto
```

## 4. Cómo Ejecutar el Proyecto

Siga estos pasos para replicar el análisis:

1.  **Clonar el repositorio:**
    ```bash
    git clone https://github.com/tu-usuario/deteccion-mala-practica.git
    cd deteccion-mala-practica
    ```

## 5. Resultados y Siguientes Pasos

### Resultados Preliminares
El script genera un listado de grupos de transacciones que, según el sistema de puntuación, son altamente sospechosos de constituir un fraccionamiento. Este listado es el insumo principal para los equipos de prevención de fraude, auditoría o cumplimiento.

### Siguientes Pasos y Mejoras Potenciales

-   **Calibración de Umbrales:** Realizar un análisis de sensibilidad para ajustar los umbrales de las heurísticas y los puntos del sistema de scoring, utilizando datos etiquetados si estuvieran disponibles.
-   **Modelo de Machine Learning:** Utilizar el output de este sistema basado en reglas como "etiquetas débiles" (`weak labels`) para entrenar un modelo de clasificación supervisado (ej. `Logistic Regression`, `Gradient Boosting`) que pueda aprender patrones más complejos y sutiles.
-   **Visualización Interactiva:** Desarrollar un dashboard (usando Streamlit o Dash) para que los analistas puedan explorar los grupos sospechosos, ver las transacciones individuales y filtrar por diferentes criterios.
-   **Automatización y Despliegue:** Empaquetar la solución en un pipeline automatizado (ej. usando Airflow) que se ejecute periódicamente (ej. diariamente) sobre los nuevos datos transaccionales.
-   **Análisis de Grafos:** Modelar las relaciones entre usuarios, cuentas y comercios como un grafo para detectar comunidades o patrones de colusión más complejos.
