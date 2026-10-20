# 04 - Configuración, ejecución y despliegue

[Volver al índice](README.md)

> Esta es una guía para orientarte en el repositorio, no una certificación de despliegue exitoso. No se ejecutó `azd up`, ni se crearon recursos, identidades o modelos.

## 1. Servicios y responsabilidades

| Servicio | Uso en la solución | Qué no demuestra su plantilla |
|---|---|---|
| Azure OpenAI | Chat, embeddings y audio opcional. | Que los modelos estén disponibles en tu región, tengan cuota o coincidan con los nombres configurados. |
| Azure AI Search | Índices híbridos, ranking semántico y pipelines de ingesta. | Que existan los tres índices o tengan contenido. |
| Blob Storage | Documentos originales; contenedores `books`, `cord19`, `friends`. | Que los datasets ya estén cargados. |
| Cosmos DB | Estado persistente del grafo. | Que la base, partición y retención coincidan con el checkpointer. |
| Azure SQL | Fuente tabular para el especialista SQL. | Que el notebook de carga haya creado/poblado las tablas. |
| AI Services / Document Intelligence | Enriquecimiento y extracción según la ruta de ingesta. | Que todas las opciones estén habilitadas o se usen en cada consulta. |
| Azure Speech | STT/TTS cuando se elige motor Azure. | Que el frontend esté configurado con clave y región. |
| App Service | Alojamiento de frontend y backends. | Que el paquete tenga la estructura de arranque esperada. |
| Azure Bot Service | Integración con canales y Direct Line. | Que Teams u otros canales estén dados de alta y probados. |
| SerpAPI | Búsqueda web de la herramienta compartida actual. | No es provisionado por las plantillas Azure revisadas. |
| Bing | Recurso heredado presente en Bicep. | No es el proveedor usado por `WebSearch_Tool` actualmente. |

Evidencia: [Bicep de servicios](../azuredeploy.bicep), [Bicep OpenAI/orquestación](../infra/main.bicep), [herramientas](../common/utils.py).

## 2. Variables: nombres, nunca valores

### Chat y recuperación

| Variables | Consumidor / propósito |
|---|---|
| `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_API_VERSION` | Conexión del modelo desde frontend y backend. |
| `GPT4o_DEPLOYMENT_NAME` | Despliegue de chat usado por el grafo y Search. |
| `AZURE_SEARCH_ENDPOINT`, `AZURE_SEARCH_KEY`, `AZURE_SEARCH_API_VERSION` | Peticiones REST a los índices. |
| `BLOB_SAS_TOKEN` | Construcción de enlaces a documentos; también se exige en la página Search. |
| `EMBEDDING_DEPLOYMENT_NAME` | Configuración de embeddings en el material y plantillas; verificar el índice/vectorizador. |
| `BLOB_CONNECTION_STRING` | Carga de archivos con las utilidades de Blob. |
| `FORM_RECOGNIZER_ENDPOINT`, `FORM_RECOGNIZER_KEY` | Extracción con Document Intelligence cuando se usa esa opción. |

### Agentes y memoria

| Variables | Consumidor / propósito |
|---|---|
| `AZURE_COSMOSDB_ENDPOINT`, `AZURE_COSMOSDB_KEY` | Cliente Cosmos DB. |
| `AZURE_COSMOSDB_NAME` | **Nombre de base de datos** esperado por el checkpointer, no nombre de cuenta. |
| `AZURE_COSMOSDB_CONTAINER_NAME` | Contenedor de checkpoints. |
| `SQL_SERVER_NAME`, `SQL_SERVER_DATABASE`, `SQL_SERVER_USERNAME`, `SQL_SERVER_PASSWORD` | Agente SQL mediante SQLAlchemy/pyodbc. |
| `SERPAPI_KEY` | `WebSearch_Tool` actual. |

### Frontend, bot y voz

| Variables | Consumidor / propósito |
|---|---|
| `FAST_API_SERVER` | URL base; el frontend añade `/stream`. |
| `BOT_DIRECTLINE_SECRET_KEY` | Web Chat de Bot Service. |
| `MicrosoftAppId`, `MicrosoftAppPassword`, `MicrosoftAppTenantId`, `MicrosoftAppType` | Autenticación/configuración del adaptador Bot Framework. |
| `SPEECH_ENGINE` | Selección entre `azure` y `openai`. |
| `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `AZURE_SPEECH_VOICE_NAME` | Motor Azure Speech, según configuración. |
| `AZURE_OPENAI_WHISPER_MODEL_NAME`, `AZURE_OPENAI_TTS_MODEL_NAME`, `AZURE_OPENAI_TTS_VOICE_NAME` | Despliegues/voz para la alternativa OpenAI. |
| `AGENT_PAGE_TITLE` | Título opcional de página, con valor predeterminado en el código. |

La [inicialización del frontend](../apps/frontend/app/__init__.py) valida variables de voz al importar el paquete, aunque el usuario todavía no haya activado el checkbox de audio. La configuración de arranque y la opción visual no son lo mismo.

El archivo de [credenciales](../credentials.env) existe y figura en Git, aunque también aparezca en [.gitignore](../.gitignore). No se inspeccionaron sus valores. Ignorar un archivo no elimina su seguimiento previo.

## 3. Dos rutas de despliegue documentadas

### Ruta A: workshop y despliegue por componentes

El [README principal](../README.md) plantea preparar Azure OpenAI, desplegar servicios base y ejecutar los notebooks. Los READMEs de [Bot Service](../apps/backend/botservice/README.md), [frontend](../apps/frontend/README.md) y [FastAPI](../apps/backend/fastapi/README.md) describen los paquetes ZIP y el despliegue a Web Apps.

El backend FastAPI se propone en un slot `staging` creado manualmente. El empaquetado descrito coloca archivos de arranque y dependencias en ubicaciones específicas; hay que conservar esa estructura al cambiar de mecanismo de publicación.

**Cuidado:** algunos botones de despliegue apuntan a otros repositorios remotos. No equivalen a desplegar exactamente este checkout local. Revisa el destino y los parámetros antes de utilizarlos.

### Ruta B: Azure Developer CLI

[azure.yaml](../azure.yaml) declara frontend y backend con host `appservice` y ejecuta:

1. `preup`: prepara directorios de empaquetado.
2. `preprovision`: crea o reutiliza una App Registration y añade una credencial.
3. Aprovisionamiento con [infra/main.bicep](../infra/main.bicep).
4. `postprovision`: configura SAS/Direct Line y crea un service principal.
5. `prepackage`: copia código/dependencias a directorios de destino.

No es una secuencia sin efectos secundarios: puede crear identidades y credenciales además de recursos facturables. Hay inconsistencias de rutas y parámetros verificadas en [Hallazgos](05-HALLAZGOS-Y-SIGUIENTES-PASOS.md). No se recomienda ejecutarla a ciegas.

## 4. Versiones y prerequisitos que debes alinear

- El [README principal](../README.md), las [notas de versión](../Latest_Release_Notes.md) y las plantillas de Web Apps orientan a Python 3.12.
- El [Dev Container](../.devcontainer/devcontainer.json) y el [workflow de apps](../.github/workflows/main_gptsmartsearch_apps.yml) usan Python 3.10.
- El [README de infraestructura](../infra/README.md) menciona Python 3.9, 3.10 o 3.11.
- Las [dependencias](../common/requirements.txt) no fijan versiones. Una instalación actual puede resolver APIs distintas de las usadas cuando se escribió el código.
- El agente SQL exige **ODBC Driver 17 for SQL Server**, además de `pyodbc`.
- Los modelos históricos por defecto en [infra/main.bicep](../infra/main.bicep) no coinciden con la configuración GPT-4o y embeddings de las aplicaciones/notebooks.

Esto no confirma ni descarta compatibilidad con una versión concreta; demuestra que no existe una única configuración reproducible en los archivos revisados.

## 5. Contrato HTTP de FastAPI

Fuente: [server.py](../apps/backend/fastapi/app/server.py).

| Método y ruta | Entrada | Salida / comportamiento |
|---|---|---|
| `GET /` | Ninguna. | Redirección a `/docs`. |
| `GET /docs` | Ninguna. | Documentación interactiva generada por FastAPI. |
| `POST /invoke` | `user_input`, `thread_id` opcional. | JSON con `final_answer`. |
| `POST /batch` | `questions`, `thread_id` opcional. | JSON con `answers`; procesamiento secuencial. |
| `POST /stream` | `user_input`, `thread_id` opcional. | Eventos SSE de texto, herramientas, metadatos, cierre o error. |

Ejemplo de forma de petición, sin datos sensibles:

```json
{
  "user_input": "@docsearch Resume la informacion disponible",
  "thread_id": "demo-roger-001"
}
```

Respuesta de `/invoke`:

```json
{
  "final_answer": "Texto generado por el flujo"
}
```

Ejemplo de petición a `/batch`:

```json
{
  "questions": ["Primera pregunta", "Pregunta de seguimiento"],
  "thread_id": "demo-roger-001"
}
```

Consideraciones de sesión:

- Reutilizar `thread_id` permite recuperar estado del mismo hilo.
- En `/invoke`, si no se envía, se genera un identificador, pero no se devuelve en el contrato `AskResponse`.
- En `/batch`, si no se proporciona hilo, cada pregunta recibe un identificador nuevo; si se proporciona, comparten hilo.
- `/batch` añade un texto de error a `answers` si una pregunta falla; no necesariamente falla el lote entero.
- `/stream` anuncia `run_id` en `metadata`.
- El [bot](../apps/backend/botservice/app/bot.py) utiliza el ID de conversación de Bot Framework.

No se comprobó ejecución real de estos contratos; son las formas definidas en el código.

## 6. Ejecución local: referencias, no promesa de arranque

La instrucción del [frontend](../apps/frontend/README.md) se basa en ejecutar Streamlit desde su carpeta. En PowerShell, una vez preparado el entorno y exportadas las variables:

```powershell
Set-Location .\apps\frontend
python -m streamlit run app\Home.py --server.port 8500 --server.address 127.0.0.1
```

Para FastAPI, el [README específico](../apps/backend/fastapi/README.md) pide habilitar el bloque local de rutas/carga de entorno en el servidor y ejecutar desde su carpeta:

```powershell
Set-Location .\apps\backend\fastapi\app
python server.py
```

Son alternativas que parten de la raíz del repositorio, no comandos para concatenar sin volver a la raíz. El servidor arranca por defecto en `127.0.0.1:8000`. **Antes de intentarlo, resuelve los bloqueos de importación y construcción del grafo.**

No se recomienda importar todo el archivo de credenciales con transformaciones improvisadas de shell: configura el entorno mediante un procedimiento revisado y evita imprimir secretos.

## 7. Lista de comprobación operacional

- [ ] Acordar versión Python y resolver/fijar dependencias.
- [ ] Validar rutas, módulos y parámetros Bicep antes de aprovisionar.
- [ ] Verificar nombres reales de despliegues de chat, embeddings y audio.
- [ ] Verificar campos, vectorizador, dimensiones y configuración semántica de los tres índices.
- [ ] Confirmar carga SQL y acceso ODBC.
- [ ] Alinear base/contenedor Cosmos DB, partición y retención.
- [ ] Configurar SerpAPI si se habilita búsqueda web.
- [ ] Probar un turno y una pregunta de seguimiento con el mismo hilo.
- [ ] Probar streaming sin duplicación de texto ni cierre prematuro.
- [ ] Verificar la estructura del ZIP y comando de arranque de cada aplicación.
- [ ] Verificar el slot si el frontend depende de `staging`.
- [ ] Revisar gestión de credenciales, identidad, acceso y tratamiento de datos antes de exponer una demo.
- [ ] Medir calidad, latencia y coste por pregunta con datos de referencia.

No se calcularon costes: dependen de región, SKU, consumo de modelos, volumen indexado y servicios externos. Hay costes de infraestructura que pueden existir aunque no se hagan preguntas.
