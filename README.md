# Análisis Geoespacial de Accesibilidad a Bienes de Consumo Básicos
### Lima Metropolitana y Callao — 2019–2024

---

## Tabla de Contenidos

1. [Problemática](#problemática)
2. [Objetivo](#objetivo)
3. [Fuentes de Datos](#fuentes-de-datos)
4. [Tecnologías y Librerías](#tecnologías-y-librerías)
5. [Instalación de Dependencias](#instalación-de-dependencias)
6. [Estructura del Proyecto](#estructura-del-proyecto)
7. [Flujo del Análisis](#flujo-del-análisis)
8. [Documentación de Funciones](#documentación-de-funciones)
   - [cargar_datos](#cargar_datos)
   - [guardar_data](#guardar_data)
   - [obtener_osm](#obtener_osm)
   - [por_fecha](#por_fecha)
   - [revisar_dataset](#revisar_dataset)
   - [calcular_cobertura](#calcular_cobertura)
   - [crear_mapa](#crear_mapa)
   - [crear_mapa_buffer](#crear_mapa_buffer)
   - [plot_graphic](#plot_graphic)
   - [plot_metricas](#plot_metricas)
   - [plot_metricas_normalizadas](#plot_metricas_normalizadas)
   - [generar_mapa_interactivo](#generar_mapa_interactivo)
9. [Métricas Calculadas](#métricas-calculadas)
10. [Visualizaciones Generadas](#visualizaciones-generadas)
11. [Resultados y Salidas](#resultados-y-salidas)
12. [Notas de Ejecución](#notas-de-ejecución)

---

## Problemática

El acceso a bienes de consumo básicos en entornos urbanos depende en gran medida de la **distribución espacial de establecimientos** como bodegas, minimarkets y supermercados. En zonas densamente pobladas como Lima Metropolitana y Callao, esta distribución no siempre es proporcional a la población, lo que puede generar **brechas en el acceso**.

El presente análisis utiliza técnicas de **analítica geoespacial** y datos de fuentes abiertas como **OpenStreetMap** para evaluar la cobertura, déficit y densidad de estos establecimientos, con el objetivo de identificar áreas con sobreoferta, cobertura adecuada y **zonas críticas con potencial de mejora**.

---

## Objetivo

Evaluar la evolución temporal (2019–2024) de la accesibilidad a establecimientos de alimentos en Lima Metropolitana y Callao, midiendo para cada distrito:

- La **cobertura territorial** basada en un radio de influencia de 500 m por tienda.
- El **déficit poblacional** sin acceso caminable a un establecimiento.
- La **densidad de tiendas** por cada 1 000 habitantes.

---

## Fuentes de Datos

| Fuente | Descripción | Formato |
|---|---|---|
| **OpenStreetMap (Overpass API)** | Ubicación de establecimientos de alimentos (bodegas, supermercados, kioscos, verdulerías, carnicerías), con snapshots históricos por año | GeoDataFrame de puntos |
| **Población Estimada (INEI)** | Estimaciones de población 2019–2024 a nivel distrital para Lima y Callao, con geometría de polígonos distritales | GeoJSON |

### Tipos de Establecimientos Consultados en OSM

| Clave OSM (`shop=`) | Descripción |
|---|---|
| `convenience` | Bodega / Minimarket |
| `supermarket` | Supermercado |
| `kiosk` | Kiosko |
| `greengrocer` | Verdulería / Frutería |
| `butcher` | Carnicería |

### Área de Estudio (Bounding Box)

```
bbox_lima = [-77.25, -12.30, -76.80, -11.80]
            [lon_min, lat_min, lon_max, lat_max]
```

---

## Tecnologías y Librerías

```
pandas          — Manipulación y análisis de datos tabulares
geopandas       — Manejo de datos geoespaciales (GeoDataFrames)
shapely         — Geometrías y operaciones espaciales (buffers, intersecciones)
overpy          — Cliente Python para la API de Overpass (OSM)
folium          — Mapas interactivos basados en Leaflet.js
branca          — Colormaps para mapas Folium
matplotlib      — Gráficos estáticos y mapas temáticos (coropléticos)
seaborn         — Estilos y paletas de color adicionales
plotly          — Gráficos interactivos (líneas normalizadas)
contextily      — Capas de tiles de fondo
adjustText      — Evita superposición de etiquetas en matplotlib
os / time       — Gestión de archivos y pausas entre consultas
```

---

## Instalación de Dependencias

El proyecto fue desarrollado en **Google Colab**. Las instalaciones adicionales necesarias son:

```bash
pip install contextily
pip install adjustText
pip install overpy
pip install geopandas
```

Las demás librerías (`pandas`, `matplotlib`, `folium`, `plotly`, `shapely`) ya están disponibles por defecto en Colab.

---

## Estructura del Proyecto

```
ProyectoFinalC2DIplomado/
│
├── proyectofinalc2diplomado.py       # Código principal (exportado desde Colab)
├── README.md                         # Este archivo
│
└── (Google Drive - datos de entrada)
    ├── /ProyectoFinalDiplomado/Entregable1/
    │   ├── poblacion_estimada.geojson    # Polígonos distritales + población INEI
    │   └── alimentos_lima.geojson        # Tiendas OSM (caché local)
    │
    └── mapa_alimentos.html               # Mapa interactivo generado (salida)
```

---

## Flujo del Análisis

```
┌─────────────────────────────────────────────────────────────────┐
│  1. CARGA DE DATOS                                              │
│     · Polígonos de distritos (Lima y Callao) + población INEI   │
│     · Consulta Overpass API → tiendas OSM por año (2019–2024)   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  2. VALIDACIÓN Y LIMPIEZA                                       │
│     · Detección de nulos, vacíos y duplicados                   │
│     · Filtrado a departamentos LIMA y CALLAO                    │
│     · Deduplicación por UBIGEO                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  3. CÁLCULO DE COBERTURA (por año, por distrito)                │
│     · Buffer de 500 m alrededor de cada tienda                  │
│     · Intersección buffer ∩ polígono distrital → área cubierta  │
│     · Cobertura = área_cubierta / área_total                    │
│     · Población cubierta y déficit                              │
│     · Densidad = tiendas / 1000 hab                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  4. VISUALIZACIÓN                                               │
│     · Mapas coropléticos (cobertura, déficit, densidad)         │
│     · Gráficas temporales 2019–2024                             │
│     · Mapa interactivo Folium con capas y leyenda               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Documentación de Funciones

### `cargar_datos`

```python
def cargar_datos(ruta, crs_destino='EPSG:4326') -> GeoDataFrame
```

Carga un archivo geoespacial (GeoJSON, Shapefile, etc.) y lo reproyecta al CRS indicado.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `ruta` | `str` | Ruta al archivo en disco |
| `crs_destino` | `str` | Sistema de referencia de coordenadas destino (default: WGS84) |

**Retorna:** `GeoDataFrame` reproyectado, o `None` si ocurre un error.

---

### `guardar_data`

```python
def guardar_data(gdf, ruta) -> None
```

Serializa un `GeoDataFrame` a un archivo GeoJSON en la ruta indicada.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `gdf` | `GeoDataFrame` | Datos a guardar |
| `ruta` | `str` | Ruta de destino del archivo GeoJSON |

---

### `obtener_osm`

```python
def obtener_osm(bbox, shops, fecha, crs="EPSG:4326") -> GeoDataFrame
```

Consulta la **Overpass API de OpenStreetMap** para obtener nodos con la etiqueta `shop` correspondiente a los tipos indicados, dentro del bounding box especificado y usando un **snapshot histórico** de la fecha dada.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `bbox` | `list[float]` | `[lon_min, lat_min, lon_max, lat_max]` del área de consulta |
| `shops` | `list[str]` | Lista de valores del tag `shop=` en OSM (ej. `["convenience", "supermarket"]`) |
| `fecha` | `str` | Año de snapshot (ej. `"2023"`); construye la fecha `{año}-12-31T00:00:00Z` |
| `crs` | `str` | CRS del GeoDataFrame resultante (default: `EPSG:4326`) |

**Mecanismo de consulta:** Construye una query Overpass con la cabecera `[date:...]` para obtener el estado del mapa en esa fecha histórica. Cada nodo resultante se convierte en un `Point` con sus atributos `name`, `shop`, `lat`, `lon` y `fecha`.

**Retorna:** `GeoDataFrame` con una fila por establecimiento encontrado.

> **Nota:** La función incluye lógica de reintentos (hasta 3 intentos por año) y pausas entre consultas (30 s en fallo, 5 s entre años exitosos) para respetar los límites de la API.

---

### `por_fecha`

```python
def por_fecha(gdf, fecha) -> GeoDataFrame
```

Filtra un `GeoDataFrame` de tiendas para devolver únicamente los registros correspondientes a un año específico.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `gdf` | `GeoDataFrame` | Dataset completo (todos los años) |
| `fecha` | `str` | Año a filtrar (ej. `"2024"`) |

---

### `revisar_dataset`

```python
def revisar_dataset(gdf) -> dict
```

Realiza una auditoría de calidad sobre un `GeoDataFrame` o `DataFrame`, detectando:

- **Nulos** por columna (`isnull().sum()`)
- **Valores vacíos** (strings en blanco) por columna
- **Filas duplicadas** totales

**Retorna:** Diccionario con las claves `nulos_por_columna`, `vacios_por_columna` y `duplicados`.

---

### `calcular_cobertura`

```python
def calcular_cobertura(distritos_gdf, tiendas_gdf, ano, buffer_m=500) -> GeoDataFrame
```

**Función central del análisis.** Calcula seis métricas de accesibilidad para cada distrito a partir de la geometría de las tiendas y la población estimada.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `distritos_gdf` | `GeoDataFrame` | Polígonos distritales con columna de población `A_{ano}` |
| `tiendas_gdf` | `GeoDataFrame` | Puntos de establecimientos del año a analizar |
| `ano` | `str` | Año (ej. `"2024"`); construye el nombre de columna `A_2024` |
| `buffer_m` | `int` | Radio de influencia en metros (default: 500 m) |

**Proceso interno:**

1. Reproyección a EPSG:3857 (métrico) para cálculos precisos de distancia.
2. Se genera un buffer de `buffer_m` metros alrededor de cada tienda.
3. Se calcula la **unión** de todos los buffers (`unary_union`).
4. Para cada distrito se calcula la **intersección** entre su polígono y la zona de cobertura.
5. Se realiza un `spatial join` para contar tiendas dentro de cada distrito.

**Columnas resultantes:**

| Columna | Descripción |
|---|---|
| `area_total` | Área total del distrito en m² |
| `area_cubierta` | Área del distrito cubierta por algún buffer de tienda |
| `cobertura` | Fracción cubierta: `area_cubierta / area_total` (0 a 1) |
| `pob_cubierta` | Estimación de población con acceso: `cobertura × población` |
| `deficit` | Población sin acceso estimado: `población - pob_cubierta` |
| `n_tiendas` | Número de tiendas del año dentro del distrito |
| `densidad` | Tiendas por cada 1 000 habitantes |
| `poblacion` | Población total del distrito para el año |

---

### `crear_mapa`

```python
def crear_mapa(gdf_poligonos, gdf_puntos, campo_tooltip="nombre_completo", zoom_start=9) -> folium.Map
```

Genera un **mapa interactivo Folium** con polígonos distritales y puntos de tiendas.

- Los polígonos se muestran en verde con tooltip al pasar el cursor.
- Las tiendas se representan como `CircleMarker` rojos con popup al hacer click.
- Incluye control de capas.

---

### `crear_mapa_buffer`

```python
def crear_mapa_buffer(gdf_poligonos, gdf_puntos, campo_tooltip="nombre_completo", zoom_start=9, buffer_m=500) -> folium.Map
```

Extiende `crear_mapa` añadiendo una capa adicional con los **buffers de 500 m** alrededor de cada tienda, visualizados en color naranja semitransparente. Permite identificar visualmente las zonas de cobertura.

---

### `plot_graphic`

```python
def plot_graphic(distritos_gdf, columna, titulo, cmap) -> None
```

Genera un **mapa coroplético estático** con `matplotlib` y `geopandas`, coloreando cada distrito según el valor de la columna indicada. Añade el nombre del distrito como etiqueta sobre el centroide de cada polígono.

| Parámetro | Descripción |
|---|---|
| `columna` | Métrica a representar: `"cobertura"`, `"deficit"` o `"densidad"` |
| `titulo` | Título del mapa |
| `cmap` | Paleta de colores matplotlib (ej. `"YlGn"`, `"Reds"`, `"Blues"`) |

**Uso en el código:**

```python
plot_graphic(coberturas["2024"], "cobertura", "Cobertura de tiendas (500m)", "YlGn")
plot_graphic(coberturas["2024"], "deficit",   "Déficit de acceso (500m)",     "Reds")
plot_graphic(coberturas["2024"], "densidad",  "Tiendas por 1000 hab (500m)",  "Blues")
```

---

### `plot_metricas`

```python
def plot_metricas(df_metricas, columna, titulo, ylabel, color="blue") -> None
```

Genera una **gráfica de línea temporal** (2019–2024) con `matplotlib` para visualizar la evolución de una métrica agregada a nivel ciudad.

El `DataFrame` de métricas se construye calculando para cada año:

| Columna | Descripción |
|---|---|
| `cobertura_promedio` | Media de la cobertura distrital |
| `pob_cubierta_total` | Suma de población cubierta en todos los distritos |
| `deficit_total` | Suma del déficit en todos los distritos |
| `densidad_promedio` | Media de la densidad distrital |

---

### `plot_metricas_normalizadas`

```python
def plot_metricas_normalizadas(df_metricas) -> None
```

Genera un **gráfico interactivo Plotly** con las tres métricas principales (cobertura, déficit, densidad) **normalizadas entre 0 y 1** mediante Min-Max Scaling, lo que permite comparar tendencias con escalas distintas en un mismo eje.

$$
x_{norm} = \frac{x - x_{min}}{x_{max} - x_{min}}
$$

---

### `generar_mapa_interactivo`

```python
def generar_mapa_interactivo(coberturas, alimentos_gdf, año, colores, nombres, buffer_m=500) -> folium.Map
```

Genera el **mapa final de resultados**, el más completo del proyecto. Integra en un único mapa Folium con control de capas:

**Capas de distritos (métricas):**

| Capa | Descripción | Colores |
|---|---|---|
| `Cobertura` | Fracción del área distrital cubierta | Naranja claro / naranja / naranja oscuro |
| `Densidad` | Tiendas por 1 000 habitantes | Azul claro / azul / azul oscuro |
| `Déficit` | Población sin acceso estimado | Rosa / rojo / rojo oscuro |

**Capas de establecimientos (por tipo):**

Cada tipo de tienda tiene su propia capa activable, con un **CircleMarker** para el punto y un **Circle** de 500 m para el radio de cobertura.

| Tipo OSM | Color | Nombre mostrado |
|---|---|---|
| `convenience` | Rojo `#e41a1c` | Bodega / Minimarket |
| `supermarket` | Azul `#377eb8` | Supermercado |
| `kiosk` | Verde `#4daf4a` | Kiosko |
| `greengrocer` | Naranja `#ff7f00` | Verdulería / Frutería |
| `butcher` | Morado `#984ea3` | Carnicería |

El mapa final incluye también un **título HTML** y una **leyenda** embebidos como elementos HTML dentro del mapa Folium.

---

## Métricas Calculadas

El análisis produce las siguientes métricas para cada distrito y año del período 2019–2024:

```
cobertura  = area_cubierta / area_total         → [0, 1]
pob_cubierta = cobertura × población            → habitantes
deficit    = población − pob_cubierta           → habitantes
n_tiendas  = conteo espacial (sjoin)            → unidades
densidad   = (n_tiendas / población) × 1000     → tiendas/1000 hab
```

---

## Visualizaciones Generadas

### 1. Mapa base con puntos
Vista inicial de los 43 distritos de Lima y Callao con la ubicación de todos los establecimientos del año 2024.

### 2. Mapa con buffers de 500 m
Mismo mapa base con la adición de círculos de 500 m alrededor de cada tienda, permitiendo visualizar las zonas de cobertura caminable.

### 3. Mapas coropléticos (matplotlib)

| Mapa | Paleta | Interpretación |
|---|---|---|
| Cobertura distrital 2024 | `YlGn` | Verde más intenso = mayor cobertura territorial |
| Déficit poblacional 2024 | `Reds` | Rojo más intenso = más habitantes sin acceso |
| Densidad de tiendas 2024 | `Blues` | Azul más intenso = más tiendas por 1 000 hab |

### 4. Gráficas temporales (matplotlib)

- **Cobertura promedio** 2019–2024 (línea verde)
- **Déficit total** 2019–2024 (línea roja)
- **Densidad promedio** 2019–2024 (línea morada)

### 5. Gráfica normalizada (Plotly)
Las tres métricas anteriores normalizadas (0–1) en un solo gráfico interactivo con tema oscuro, para comparar tendencias relativas.

### 6. Mapa interactivo final (`mapa_alimentos.html`)
Mapa Folium completo con capas activables por métrica y por tipo de tienda, buffers individuales, tooltips, leyenda y título. Se exporta como archivo HTML autocontenido.

---

## Resultados y Salidas

| Salida | Formato | Descripción |
|---|---|---|
| `alimentos_lima.geojson` | GeoJSON | Caché de datos OSM (2019–2024 concatenados) |
| `mapa_alimentos.html` | HTML | Mapa interactivo Folium con todas las capas |
| Gráficas inline | PNG (Colab) | Mapas coropléticos y gráficas temporales |

---

## Notas de Ejecución

- El proyecto está diseñado para ejecutarse en **Google Colab** con acceso a **Google Drive** para la carga y persistencia de archivos.
- La consulta a Overpass API puede tomar varios minutos. El código guarda un caché local (`alimentos_lima.geojson`) para evitar repetir la descarga.
- Si la API devuelve error 429 (demasiadas solicitudes), el código espera 30 segundos antes de reintentar automáticamente (máximo 3 intentos por año).
- Todos los cálculos espaciales (buffers, intersecciones, áreas) se realizan en **EPSG:3857** (métrico) para garantizar precisión en metros; las visualizaciones se convierten a **EPSG:4326** (WGS84) para Folium y Matplotlib.
- El buffer de 500 m representa una distancia caminable aproximada de 5–7 minutos a pie.

---

*Proyecto desarrollado como entrega final del Diplomado — Curso 2, módulo de Analítica Geoespacial.*
