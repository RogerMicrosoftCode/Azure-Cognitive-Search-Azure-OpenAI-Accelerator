# 01 - Análisis del repositorio

[Volver al índice](README.md)

## 1. Qué problema intenta resolver

El objetivo es ofrecer una experiencia de preguntas y respuestas sobre fuentes distintas sin que el usuario tenga que consultar manualmente cada sistema:

| Pregunta de ejemplo | Especialista previsto | Fuente |
|---|---|---|
| ¿Qué dicen nuestros documentos sobre este tema? | `DocSearchAgent` | Azure AI Search y documentos en Blob Storage. |
| ¿Cuántos registros cumplen esta condición? | `CSVSearchAgent` | CSV local analizado con Python. |
| ¿Cuál es el total por estado y fecha? | `SQLSearchAgent` | Base de datos SQL consultada mediante herramientas. |
| ¿Qué información hay en internet? | `WebSearchAgent` | Buscador externo y lectura de páginas. |
| ¿Cuál es el dato actual que ofrece esta API? | `APISearchAgent` | API descrita con OpenAPI; ejemplo Kraken. |

El LLM ayuda a interpretar la pregunta, seleccionar herramientas y redactar la respuesta. Los datos empresariales no se incorporan mediante un entrenamiento del modelo: se consultan durante la interacción.

## 2. Las tres capas del acelerador

### A. Aprendizaje y preparación de datos

Los [notebooks de la raíz](../) enseñan a cargar datos, crear índices, consultar modelos, añadir memoria y construir agentes. Son también parte del procedimiento de preparación: aprovisionar servicios no equivale a tener los índices y tablas ya cargados.

### B. Lógica reutilizable

La carpeta [common](../common/) contiene las herramientas y la orquestación. Es la parte más importante si quieres adaptar el acelerador a otros datos.

### C. Aplicaciones e infraestructura

La carpeta [apps](../apps/) expone la experiencia mediante Streamlit, Bot Framework y FastAPI. La carpeta [infra](../infra/) y la [plantilla principal de recursos](../azuredeploy.bicep) intentan automatizar el aprovisionamiento.

## 3. Mapa de componentes

| Archivo o carpeta | Responsabilidad real |
|---|---|
| [README original](../README.md) | Presentación del workshop/POC, arquitectura general y preparación manual. |
| [Notas de versión](../Latest_Release_Notes.md) | Declaran versión 4.0.0, Python 3.12, LangGraph, multimodalidad y FastAPI nativo. |
| [Notebooks](03-GUIA-NOTEBOOKS.md) | Secuencia educativa y ejemplos de ejecución. |
| [Datos](../data/) | Archivos de ejemplo para documentos, CSV y API. |
| [graph.py](../common/graph.py) | Construye el supervisor y el grafo usado por ambos backends. |
| [utils.py](../common/utils.py) | Búsqueda híbrida, retriever, herramientas, creación de agentes y utilidades de documentos. |
| [prompts.py](../common/prompts.py) | Instrucciones de agentes, supervisor, bienvenida y resumen para audio. |
| [cosmosdb_checkpointer.py](../common/cosmosdb_checkpointer.py) | Persistencia síncrona y asíncrona de checkpoints de LangGraph. |
| [audio_utils.py](../common/audio_utils.py) | Transcripción, síntesis de voz y preparación del texto para audio. |
| [requirements.txt](../common/requirements.txt) | Dependencias compartidas, sin versiones fijadas. |
| [Frontend](../apps/frontend/) | Aplicación Streamlit con inicio y tres páginas funcionales. |
| [Backend Bot Service](../apps/backend/botservice/) | Recepción de actividades de Bot Framework y envío de respuestas. |
| [Backend FastAPI](../apps/backend/fastapi/) | API HTTP de invocación, lote y streaming SSE. |
| [azuredeploy.bicep](../azuredeploy.bicep) | Declara servicios de datos/IA de base. No incluye las aplicaciones ni crea los índices. |
| [infra/main.bicep](../infra/main.bicep) | Orquesta grupos de recursos, OpenAI y módulos de aplicaciones/datos. |
| [azure.yaml](../azure.yaml) | Servicios y hooks de Azure Developer CLI. |
| [Workflows](../.github/workflows/) | Compilación de Bicep y empaquetado/despliegue de aplicaciones. |
| [Dev Container](../.devcontainer/devcontainer.json) | Entorno de desarrollo basado en Python 3.10. |
| [Imágenes](../images/) | Material gráfico ya incluido en el repositorio. |

## 4. La interfaz no sigue una única ruta

### Página Search

[1_Search.py](../apps/frontend/app/pages/1_Search.py) consulta **directamente** Azure AI Search y Azure OpenAI desde el proceso Streamlit.

No llama al supervisor, no pasa por FastAPI y no utiliza el checkpointer de Cosmos DB. Recupera documentos, construye un contexto, genera una respuesta y muestra resultados y enlaces.

### Página Bot Service Chat

[2_BotService_Chat.py](../apps/frontend/app/pages/2_BotService_Chat.py) incrusta Bot Framework Web Chat. El navegador se comunica por Direct Line; Bot Service entrega las actividades al backend.

El [bot](../apps/backend/botservice/app/bot.py) usa el identificador de conversación como `thread_id` e invoca el grafo compartido.

### Página FastAPI Chat

[3_FastAPI_Chat.py](../apps/frontend/app/pages/3_FastAPI_Chat.py) envía la pregunta al endpoint `/stream`, recibe eventos SSE y mantiene historial de pantalla en la sesión de Streamlit.

Permite entrada y salida por voz. El estado persistente del backend no es ese historial visual: lo administra Cosmos DB.

## 5. El núcleo multiagente

En [graph.py](../common/graph.py) hay un `StateGraph` con:

- `messages`: historial acumulado de mensajes.
- `next`: nombre del siguiente nodo.
- Un supervisor con salida estructurada.
- Cinco nodos de agentes.
- Un retorno de cada especialista al supervisor.
- Una salida `FINISH` que lleva al final del grafo.

No son cinco microservicios separados. Son agentes/herramientas dentro del proceso del backend, coordinados por un grafo.

La selección está influida por el [prompt del supervisor](../common/prompts.py): `@docsearch`, `@csvsearch`, `@sqlsearch`, `@websearch` y `@apisearch`. Sin etiqueta, el prompt exige usar el agente web. Es una instrucción al LLM, **no un parser determinista** que garantice el enrutamiento.

El [notebook integrador](../12-Smart_Agent.ipynb) usa `create_supervisor` de `langgraph-supervisor`, mientras que las aplicaciones usan el grafo manual. Es importante no asumir que probar uno valida automáticamente el otro.

## 6. Qué datos vienen incluidos

Inventario de archivos locales, sin inspeccionar el contenido de los ZIP:

| Archivo | Tamaño en bytes | Uso previsto |
|---|---:|---|
| [all-states-history.csv](../data/all-states-history.csv) | 2.759.381 | Estadísticas de ejemplo para CSV y carga SQL. |
| [all-states-history.csv.1](../data/all-states-history.csv.1) | 2.759.381 | Segundo archivo del mismo tamaño; no se verificó igualdad de contenido. |
| [books.zip](../data/books.zip) | 56.432.764 | Documentos para el escenario de libros/PDF. |
| [cord19mini.zip](../data/cord19mini.zip) | 44.545.346 | Datos de ejemplo de publicaciones. |
| [friends_transcripts.zip](../data/friends_transcripts.zip) | 2.299.277 | Texto de ejemplo para la ingesta documental. |
| [openapi_kraken.json](../data/openapi_kraken.json) | 381.019 | Especificación usada por el agente API. |

El número de documentos, páginas o registros que aparecen en la demo original no se ha comprobado contra estos archivos. Su presencia tampoco demuestra que estén cargados en Azure.

## 7. Tecnologías y dependencias

- **Python** para notebooks, lógica y aplicaciones.
- **LangChain** para LLMs, prompts, retrievers y herramientas.
- **LangGraph** para agentes y estado de conversación.
- **Azure OpenAI** para generación y embeddings; voz opcional según configuración.
- **Azure AI Search** para recuperación documental híbrida y ranking semántico.
- **Azure Blob Storage** para documentos fuente.
- **Azure Cosmos DB** para checkpoints.
- **Azure SQL + SQLAlchemy + pyodbc** para preguntas sobre tablas.
- **SerpAPI** en la herramienta web compartida actual, aunque hay referencias históricas a Bing.
- **Streamlit**, **FastAPI**, **aiohttp/Bot Framework** para las interfaces.
- **Bicep/ARM, Azure CLI y azd** para despliegue.

## 8. Qué puedes reutilizar y qué no debes dar por terminado

**Reutilizable como base:** la secuencia didáctica, el patrón RAG, el diseño de especialistas, el contrato HTTP y los ejemplos de interfaz.

**Requiere validación o adaptación:** el ensamblado del grafo, la compatibilidad de dependencias, la persistencia, la configuración de modelos, la automatización de despliegue y la operación empresarial.

No se identificó una suite de pruebas automatizadas en los archivos versionados consultados. Las demostraciones y ejercicios de evaluación en notebooks no sustituyen pruebas repetibles de los backends.

Continúa con los [dibujos de arquitectura](02-ARQUITECTURA-Y-FLUJOS.md) y los [hallazgos](05-HALLAZGOS-Y-SIGUIENTES-PASOS.md).
