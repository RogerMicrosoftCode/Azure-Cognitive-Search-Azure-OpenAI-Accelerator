# 03 - Guía de notebooks y ruta de aprendizaje

[Volver al índice](README.md)

## 1. Inventario completo

Hay **16 notebooks**, no 15: el número 09 aparece en dos variantes.

| Notebook | Qué enseña o prepara | Dependencia práctica |
|---|---|---|
| [01 - Carga documental](../01-Load-Data-ACogSearch.ipynb) | Carga a Blob y pipeline de extracción, OCR, división y embeddings; índice `srch-index-files`. | Storage, AI Search, servicios de IA y embeddings configurados. |
| [02 - CSV one-to-many](../02-LoadCSVOneToMany-ACogSearch.ipynb) | Convierte datos tabulares/publicaciones en documentos recuperables; índice `srch-index-csv`. | Blob e infraestructura de búsqueda/embeddings. |
| [03 - Consultas y OpenAI](../03-Quering-AOpenAI.ipynb) | Búsqueda en varios índices, prompts, embeddings, citas, streaming y ejercicios de fundamentación. | Índices previos con datos y modelo de chat. |
| [04 - Documentos complejos](../04-Complex-Docs.ipynb) | Extracción PDF, chunks y carga al índice `srch-index-books`. | PDFs de ejemplo, embeddings y Search; Document Intelligence si se elige esa vía. |
| [05 - Memoria](../05-Adding_Memory.ipynb) | Diferencia entre historial en contexto y persistencia de conversaciones. | Modelo y Cosmos DB para la parte persistente. |
| [06 - Primer RAG](../06-First-RAG.ipynb) | Herramienta de búsqueda, estado, agente LangGraph y checkpointer. | Índices/documentos y Cosmos DB para memoria. |
| [07 - CSV QA](../07-TabularDataQA.ipynb) | Agente que usa Python/Pandas para calcular sobre CSV. | Dataset tabular y entorno Python compatible. |
| [08 - SQL QA](../08-SQLDB_QA.ipynb) | Carga de datos a SQL, conexión ODBC y agente con herramientas SQL. | Base SQL, permisos y driver ODBC. |
| [09 - BingChatClone](../09-BingChatClone.ipynb) | Variante de búsqueda web basada en Bing y lectura de páginas. | Integración externa de Bing; comprobar disponibilidad antes de usarla. |
| [09 - WebSearch QA](../09-WebSearch_QA.ipynb) | Variante de buscador web y recuperación de páginas. | Proveedor web y credenciales correspondientes; es la ruta más alineada con la herramienta compartida actual. |
| [10 - API Search](../10-API-Search.ipynb) | Uso de OpenAPI y herramientas HTTP para consultar APIs. | Especificación y acceso al servicio externo. |
| [11 - Multimodalidad](../11-Adding_Multi-modality.ipynb) | Ejemplos de visión y audio no realtime. | Modelos y servicios que correspondan al ejercicio. |
| [12 - Smart Agent](../12-Smart_Agent.ipynb) | Integra los cinco especialistas con `create_supervisor`; ejemplos sync/async. | Especialistas, fuentes y memoria ya preparados. |
| [13 - Construcción de apps](../13-Building-Apps.ipynb) | Introduce backend Bot Service y frontend Streamlit. | Infraestructura y lógica del acelerador. |
| [14 - API Bot Service](../14-BotService-API.ipynb) | Consumo programático del bot y continuidad de conversación. | Bot desplegado y Direct Line configurado. |
| [15 - API FastAPI](../15-FastAPI-API.ipynb) | Alternativa HTTP nativa y consumo de `/invoke`, `/batch`, `/stream`. | Backend FastAPI operativo. |

## 2. Ruta sugerida para entenderlo, sin ejecutar todavía

1. Lee el [análisis general](01-ANALISIS-DEL-REPOSITORIO.md).
2. Mira la [arquitectura](02-ARQUITECTURA-Y-FLUJOS.md).
3. Revisa los notebooks 01, 03 y 06 para entender datos -> recuperación -> respuesta -> agente.
4. Revisa 07, 08, 09-WebSearch y 10 para entender por qué existen especialistas diferentes.
5. Revisa 12 para entender la coordinación.
6. Revisa 13 a 15 para entender cómo se convierte el experimento en una aplicación.
7. Vuelve a [graph.py](../common/graph.py) y compara la implementación de las aplicaciones con el notebook 12.

Esta ruta de lectura no implica que puedas omitir las cargas requeridas al ejecutar el grafo completo.

## 3. Ruta mínima para una demostración documental

**Propuesta, no procedimiento ya validado en este entorno:**

1. Seleccionar un conjunto pequeño de documentos propios autorizados.
2. Preparar el entorno y confirmar modelos/dimensiones.
3. Ejecutar una ingesta y probar recuperación antes de probar generación.
4. Construir un RAG con una única fuente, siguiendo los patrones de los notebooks 03 y 06.
5. Evaluar preguntas con respuestas esperadas y citas.
6. Añadir interfaz solamente después de comprobar el flujo.

La página [Search](../apps/frontend/app/pages/1_Search.py) y el [grafo compartido](../common/graph.py) tienen tres nombres de índice configurados explícitamente. Una demo de un solo índice requiere ajustar esa selección; no basta con cargar uno y dejar los demás inexistentes.

## 4. Ruta hacia el escenario completo

1. Preparar los índices de 01, 02 y 04.
2. Verificar memoria con 05/06.
3. Preparar CSV, SQL, web y API con 07 a 10.
4. Probar el supervisor del notebook 12.
5. Resolver los [bloqueos del grafo usado por apps](05-HALLAZGOS-Y-SIGUIENTES-PASOS.md).
6. Probar primero un backend y después el otro.
7. Incorporar voz o visión cuando el escenario textual sea fiable.

El constructor actual intenta crear **los cinco agentes** al inicializar el backend. Las fuentes no están implementadas como módulos opcionales que se activen solamente cuando llegue una pregunta.

## 5. Cómo saber si aprendiste cada capa

| Capa | Pregunta que deberías poder responder |
|---|---|
| Ingesta | ¿Dónde se crean los índices y cómo se generan los vectores? |
| Recuperación | ¿Qué diferencia hay entre texto, vectores y ranking semántico? |
| RAG | ¿Qué contexto recibe el modelo y de dónde salen las referencias? |
| Herramientas | ¿Qué puede ejecutar cada especialista y sobre qué datos? |
| Supervisor | ¿Qué pasa con una pregunta sin `@`? |
| Memoria | ¿Qué identifica una conversación y dónde se guarda? |
| API | ¿Qué cambia entre respuesta completa, lote y streaming? |
| Despliegue | ¿Qué prepara Bicep y qué sigue dependiendo de notebooks o pasos manuales? |

## 6. Precauciones al ejecutar

- Los notebooks pueden crear recursos, cargar datos y consumir APIs de pago.
- No ejecutes todas las celdas sin revisar operaciones de creación, borrado o sobrescritura.
- No compartas notebooks con salidas que contengan credenciales o datos sensibles.
- No uses material de demostración como si fuera información empresarial propia o evidencia de calidad del modelo.
- Para CSV y SQL, comprueba los resultados contra un cálculo o consulta de referencia.
- Para respuestas documentales, comprueba que la cita realmente respalde la afirmación.
- La visión demostrada en el notebook 11 no acredita una interfaz de carga de imágenes de extremo a extremo en las aplicaciones.
