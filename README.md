# Power BI Weather Data Analysis Tutorial

Este repositorio es un tutorial completo sobre cómo realizar un análisis de datos meteorológicos usando Power BI Desktop. A partir de datos obtenidos de una API del clima y almacenados en Azure Cosmos DB, este tutorial te guiará en la creación de diferentes visualizaciones usando Power BI para explorar el comportamiento de distintas variables meteorológicas.

## Paso 1: Crear el entorno de desarrollo

Antes de comenzar a ejecutar el script de Python, es necesario crear un entorno virtual para gestionar las dependencias y asegurarse de que el código se ejecute correctamente sin conflictos con otras versiones de paquetes instalados.

### 1.1. Crear un entorno virtual
1. Abre la terminal (o PowerShell en Windows) y navega al directorio donde deseas crear tu proyecto.
2. Ejecuta el siguiente comando para crear un entorno virtual:
   ```bash
   python -m venv venv
   ```
   Esto creará una carpeta llamada `venv` en tu directorio actual.

3. Activa el entorno virtual:
   - En **Windows**:
     ```bash
     .\venv\Scripts\activate
     ```
   - En **Linux/macOS**:
     ```bash
     source venv/bin/activate
     ```

### 1.2. Instalar las dependencias necesarias
Una vez que el entorno virtual esté activo, instala las dependencias necesarias para el proyecto:

```bash
pip install pymongo requests
```
- **pymongo**: Para conectarse a Azure Cosmos DB utilizando la API de MongoDB.
- **requests**: Para hacer solicitudes HTTP a la API de OpenWeather.

### 1.3. Verificar la instalación
Verifica que los paquetes se hayan instalado correctamente ejecutando:

```bash
pip list
```
Deberías ver `pymongo` y `requests` en la lista de paquetes instalados.

### 1.4. Guardar dependencias (opcional)
Si deseas guardar las dependencias en un archivo `requirements.txt` para facilitar su instalación en el futuro, ejecuta:

```bash
pip freeze > requirements.txt
```
Esto generará un archivo `requirements.txt` que podrás usar para instalar todas las dependencias en otro entorno con el comando:

```bash
pip install -r requirements.txt
```

## Paso 2: Obtener los datos meteorológicos

### 2.1. Configurar la API de OpenWeather
Utilizaremos la API de OpenWeather para obtener datos meteorológicos. Asegúrte de tener una cuenta y una **API Key** de OpenWeather.

- **API Key**: Puedes obtenerla creando una cuenta en [OpenWeather](https://openweathermap.org/api).
- **Ciudad**: Define la ciudad de la cual deseas obtener los datos, en este caso, `Toronto`.

### 2.2. Crear el script para extraer datos
Utiliza el siguiente script de Python para extraer los datos meteorológicos y guardarlos en **Azure Cosmos DB**:

```python
from pymongo import MongoClient
import requests
import time

# Configuración de OpenWeather API
API_KEY = "TU_API_KEY"  # Reemplaza con tu API Key
CITY = "Toronto"  # Cambia por la ciudad deseada
WEATHER_URL = f"http://api.openweathermap.org/data/2.5/weather?q={CITY}&appid={API_KEY}&units=metric"

# Configuración de Cosmos DB
DB_NAME = "testingcosmosdb01"  # Nombre de tu base de datos
COLLECTION_NAME = "pruebadatabrickcosmosdb"  # Nombre de tu colección
CONNECTION = "mongodb://<TU_CONEXION_A_COSMOS_DB>"  # Conexión a Cosmos DB

# Funciones para obtener e insertar datos
client = MongoClient(CONNECTION)

def insert_weather_data(data):
    db = client[DB_NAME]
    collection = db[COLLECTION_NAME]
    document_id = collection.insert_one(data).inserted_id
    print(f"Documento insertado con _id: {document_id}")

def get_weather_data():
    response = requests.get(WEATHER_URL)
    if response.status_code == 200:
        weather_data = response.json()
        formatted_data = {
            "city": weather_data["name"],
            "temperature": weather_data["main"]["temp"],
            "weather": weather_data["weather"][0]["description"],
            "humidity": weather_data["main"]["humidity"],
            "pressure": weather_data["main"]["pressure"],
            "wind_speed": weather_data["wind"]["speed"],
            "timestamp": time.strftime('%Y-%m-%d %H:%M:%S')
        }
        return formatted_data
    else:
        print(f"Error al obtener los datos: {response.status_code}")
        return None

# Obtener e insertar datos
weather_data = get_weather_data()
if weather_data:
    insert_weather_data(weather_data)
```

Este script se conecta a la API de OpenWeather para obtener datos de la ciudad de Toronto y luego los guarda en una base de datos de Azure Cosmos DB.

## Paso 3: Conectar Azure Cosmos DB con Databricks

### 3.1. Configurar Spark en Azure Databricks

En **Azure Databricks**, configura tu clúster de Spark con las siguientes configuraciones:

```properties
spark.master=local[*]
spark.databricks.cluster.profile=singleNode
spark.mongodb.output.uri=mongodb://<TU_CONEXION_A_COSMOS_DB>
spark.mongodb.input.uri=mongodb://<TU_CONEXION_A_COSMOS_DB>
```

Asegúrate de reemplazar `<TU_CONEXION_A_COSMOS_DB>` con la conexión correspondiente a tu Cosmos DB.

### 3.2. Leer los datos de Cosmos DB en Databricks

Utiliza el siguiente código en un notebook de Databricks para leer los datos meteorológicos desde Cosmos DB:

```python
from pyspark.sql import SparkSession

databasename = 'testingcosmosdb01'
collection = 'pruebadatabrickcosmosdb'
uri = "mongodb://<TU_CONEXION_A_COSMOS_DB>"

spark = SparkSession.builder.appName("TestingApp").config('spark.jars.packages', 'org.mongodb.spark:mongo-spark-connector_2.12:3.0.1').getOrCreate()

df = spark.read.format("com.mongodb.spark.sql.DefaultSource") \
    .option("uri", uri) \
    .option("database", databasename) \
    .option("collection", collection) \
    .load()

df.show()
```

Este código crea una sesión de Spark, conecta con la base de datos de Cosmos DB y muestra los datos extraídos.

## Paso 4: Guardar la tabla en Databricks

Puedes guardar el DataFrame como una tabla en la metastore de Databricks para facilitar su uso posterior en Power BI:

```python
df.write.mode("overwrite").saveAsTable("PowerBITable")
```

Esto guardará los datos en una tabla llamada `PowerBITable`, accesible desde Spark SQL.

## Paso 5: Conectar Power BI a Azure Databricks

### 5.1. Configurar la conexión

1. Abre **Power BI Desktop**.
2. Selecciona **Obtener Datos** > **Azure** > **Azure Databricks**.
3. Introduce la **URL del clúster** y las credenciales de acceso.

### 5.2. Cargar la tabla

1. Navega hasta la tabla `PowerBITable` que creaste en Databricks.
2. Carga los datos en Power BI para comenzar a crear las visualizaciones.

## Paso 6: Crear visualizaciones en Power BI

En este paso, aprenderás a crear diferentes visualizaciones para explorar los datos meteorológicos y obtener insights significativos. A continuación, se detallan algunas de las visualizaciones recomendadas y cómo crearlas en Power BI.

### 6.1. Gráfico de línea: Cambios en la temperatura a lo largo del tiempo
- **Propósito**: Este gráfico te permitirá visualizar cómo cambia la temperatura a lo largo del tiempo y detectar patrones o tendencias.
- **Configuración**:
  - **Eje X**: `timestamp` (tiempo).
  - **Eje Y**: `temperature` (temperatura).
- **Instrucciones**:
  1. Inserta un **gráfico de línea** desde la barra de visualizaciones.
  2. Arrastra el campo `timestamp` al eje X y el campo `temperature` al eje Y.
  3. Personaliza el formato del eje para mostrar las fechas en un formato que sea fácil de leer.

### 6.2. Gráfico de barras: Frecuencia de condiciones meteorológicas
- **Propósito**: Este gráfico muestra la cantidad de veces que se ha registrado cada condición meteorológica (como "clear sky", "rain", etc.).
- **Configuración**:
  - **Eje X**: `weather` (condición climática).
  - **Eje Y**: Conteo de registros.
- **Instrucciones**:
  1. Inserta un **gráfico de barras agrupadas**.
  2. Arrastra el campo `weather` al eje X y selecciona `Conteo` como valor en el eje Y.
  3. Ajusta los colores para diferenciar mejor cada condición meteorológica.

### 6.3. Gráfico combinado: Humedad y velocidad del viento
- **Propósito**: Permite visualizar simultáneamente los niveles de humedad y la velocidad del viento para analizar la relación entre ambas variables.
- **Configuración**:
  - **Eje X**: `timestamp` (tiempo).
  - **Eje Y1**: `humidity` (barras).
  - **Eje Y2**: `wind_speed` (línea).
- **Instrucciones**:
  1. Inserta un **gráfico combinado**.
  2. Arrastra `timestamp` al eje X, `humidity` al eje Y1 y `wind_speed` al eje Y2.

### 6.4. Gráfico de dispersión: Relación entre presión y temperatura
- **Propósito**: Este gráfico ayuda a identificar posibles correlaciones entre la presión atmosférica y la temperatura.
- **Configuración**:
  - **Eje X**: `pressure` (presión).
  - **Eje Y**: `temperature` (temperatura).
- **Instrucciones**:
  1. Inserta un **gráfico de dispersión**.
  2. Arrastra `pressure` al eje X y `temperature` al eje Y.

### 6.5. Mapa: Ubicaciones y datos climáticos
- **Propósito**: Visualiza los datos climáticos de diferentes ubicaciones en un mapa, lo cual es útil para ver las condiciones meteorológicas por ciudad.
- **Configuración**:
  - **Campo de ubicación**: `city`.
  - **Tamaño del marcador**: `temperature` o `humidity`.
  - **Color del marcador**: Puede ser categórico basado en `weather`.
- **Instrucciones**:
  1. Inserta un **gráfico de mapa**.
  2. Arrastra `city` al campo de ubicación y `temperature` o `humidity` al tamaño del marcador.

### 6.6. KPI o Indicadores de tarjetas: Valores actuales
- **Propósito**: Destaca los valores clave como la temperatura, humedad y velocidad del viento en una forma simple y visual.
- **Instrucciones**:
  1. Inserta una **tarjeta de KPI**.
  2. Arrastra `temperature`, `humidity` y `wind_speed` a tarjetas separadas para mostrar los valores actuales.

### 6.7. Histograma: Distribución de temperaturas
- **Propósito**: Analiza la distribución de las temperaturas registradas para ver cuáles son los rangos más comunes.
- **Instrucciones**:
  1. Inserta un **histograma**.
  2. Arrastra `temperature` al eje X para visualizar la distribución de los datos.

### 6.8. Gráfico de columnas apiladas: Condiciones climáticas por ciudad
- **Propósito**: Muestra la proporción de diferentes condiciones meteorológicas registradas por ciudad.
- **Configuración**:
  - **Eje X**: `city`.
  - **Eje Y**: Conteo de registros.
  - **Leyenda**: `weather`.
- **Instrucciones**:
  1. Inserta un **gráfico de columnas apiladas**.
  2. Arrastra `city` al eje X, y `weather` a la leyenda.

### 6.9. Gráfico de líneas con múltiples series: Evolución de métricas
- **Propósito**: Permite comparar la evolución de varias métricas meteorológicas a lo largo del tiempo.
- **Configuración**:
  - **Eje X**: `timestamp`.
  - **Eje Y**: Valores de `temperature`, `humidity` y `pressure`.
- **Instrucciones**:
  1. Inserta un **gráfico de línea**.
  2. Arrastra `timestamp` al eje X, y `temperature`, `humidity`, y `pressure` al eje Y.

### 6.10. Gráfico de indicadores: Cambios entre lecturas
- **Propósito**: Analiza los cambios porcentuales entre lecturas consecutivas para ver tendencias en los datos meteorológicos.
- **Instrucciones**:
  1. Inserta un **gráfico de indicadores**.
  2. Calcula la diferencia porcentual entre lecturas para `temperature` y `humidity`, y agrégala al gráfico.

## Recomendaciones para Power BI
- **Usa filtros**: Permiten seleccionar rangos de tiempo específicos o ciudades para un análisis más detallado.
- **Personaliza formatos de ejes**: Por ejemplo, muestra las fechas en formato corto para los gráficos de tiempo.
- **Agrega segmentadores**: Para filtrar datos por ciudad o tipo de clima (`weather`).

## Recursos adicionales
- [Documentación de Power BI](https://docs.microsoft.com/en-us/power-bi/)
- [API de OpenWeather](https://openweathermap.org/api)
- [Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/)

## Contribuciones
Si tienes sugerencias para mejorar este tutorial o deseas agregar nuevas visualizaciones, no dudes en hacer un pull request o abrir un issue.

¡Espero que disfrutes de este proyecto y aprendas cómo analizar datos meteorológicos con Power BI! 😊

