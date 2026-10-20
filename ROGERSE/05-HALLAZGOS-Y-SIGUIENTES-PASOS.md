# 05 - Hallazgos y siguientes pasos

[Volver al índice](README.md)

## 1. Cómo interpretar este documento

Esta es una revisión de comprensión e integración basada en archivos locales. **No es una auditoría de seguridad ni una prueba de funcionamiento en Azure.**

- **Confirmado por lectura/comprobación local:** existe una discrepancia concreta entre archivos.
- **Por validar en ejecución:** el efecto depende de versiones, entorno o configuración desplegada.
- **Propuesto:** una mejora o prueba todavía no realizada.

Las prioridades indican orden sugerido para conseguir una demo fiable; no son niveles de vulnerabilidad.

## 2. Bloqueos y discrepancias de integración

| ID | Prioridad | Evidencia confirmada | Consecuencia y siguiente comprobación |
|---|---|---|---|
| H01 | P0 - arranque | [graph.py](../common/graph.py) importa `BING_PROMPT_TEXT`; [prompts.py](../common/prompts.py) define `WEBSEARCH_PROMPT_TEXT`, no el nombre importado. | La importación del grafo no puede resolverse con estas fuentes. Alinear el nombre y probar importación del backend. |
| H02 | P0 - construcción | Las cinco funciones `create_*search_agent` de [utils.py](../common/utils.py) exigen `name`; sus llamadas en [graph.py](../common/graph.py) no lo proporcionan. | Tras resolver H01, las llamadas no satisfacen las firmas. Pasar nombres coherentes y probar construcción de los cinco agentes. |
| H03 | P0 - infraestructura | [infra/main.bicep](../infra/main.bicep) y el [workflow Bicep](../.github/workflows/bicep-to-arm.yml) apuntan a un módulo de backend directamente bajo `apps/backend`; el archivo real está en [botservice](../apps/backend/botservice/azuredeploy-backend.bicep). | El módulo referenciado no existe en esa ruta. Corregir ambas superficies y compilar. |
| H04 | P0 - infraestructura | [infra/main.bicep](../infra/main.bicep) pasa `azureOpenAIModelName` al frontend; la [plantilla frontend](../apps/frontend/azuredeploy-frontend.bicep) declara `gpt4oDeploymentName`. Además, el módulo backend real exige `TenantId`, no pasado por el orquestador. | Aun corrigiendo H03, queda desalineado el contrato entre módulos. Revisar todos los parámetros y compilar el orquestador. |
| H05 | P1 - memoria | El [Bicep base](../azuredeploy.bicep) crea partición `/user_id` y TTL `1000`; el [checkpointer](../common/cosmosdb_checkpointer.py) crea/usa `/thread_id` y escribe documentos con `thread_id`, sin `user_id`. | Un contenedor existente no cambia de partición por `create_container_if_not_exists`. Alinear el diseño antes de crear datos y probar persistencia/relectura; no se comprobó el fallo exacto contra Cosmos DB. |
| H06 | P1 - destino de datos | La [plantilla backend](../apps/backend/botservice/azuredeploy-backend.bicep) asigna el nombre de cuenta a `AZURE_COSMOSDB_NAME`; el [checkpointer](../common/cosmosdb_checkpointer.py) lo interpreta como base. La [plantilla base](../azuredeploy.bicep) declara una base con otro nombre. | Puede intentar crear/usar otra base, en vez de la provisionada. Pasar explícitamente el nombre de base y verificar dónde se guardan los checkpoints. |
| H07 | P1 - modelos | [infra/main.bicep](../infra/main.bicep) usa valores por defecto históricos, como `gpt-4-32k`, versión `0613` y `text-embedding-ada-002`; apps/notebooks se orientan a GPT-4o y vectores de 3072 dimensiones. | No asumir que `azd` provisiona lo que consume el código. Validar modelos disponibles, nombres, cuota y dimensiones; no se certificó vigencia regional de esos modelos. |
| H08 | P1 - web | [WebSearch_Tool](../common/utils.py) usa SerpAPI y `SERPAPI_KEY`; las [plantillas backend](../apps/backend/botservice/azuredeploy-backend.bicep) configuran Bing. | El servicio Bing provisionado no satisface la herramienta actual. Elegir proveedor y alinear documentación/configuración. |
| H09 | P1 - empaquetado | Los hooks de [azure.yaml](../azure.yaml) copian la carpeta de backend sin seleccionar ni aplanar una implementación como lo describen los READMEs. El [workflow de apps](../.github/workflows/main_gptsmartsearch_apps.yml) también conserva ambas variantes. | Verificar que módulo de arranque, script, datos y dependencias queden en el lugar esperado. El empaquetado manual y el automatizado no son equivalentes por lectura de sus comandos. |
| H10 | P1 - entorno | [Dev Container](../.devcontainer/devcontainer.json)/[CI](../.github/workflows/main_gptsmartsearch_apps.yml) usan 3.10; [Web Apps](../apps/frontend/azuredeploy-frontend.bicep)/[notas](../Latest_Release_Notes.md) usan 3.12; [requirements](../common/requirements.txt) no fija versiones. | La ejecución no es reproducible a partir de un conjunto único de versiones. Establecer matriz y lock/pins antes de atribuir errores a Azure. |
| H11 | P1 - frontend FastAPI | El [Bicep frontend](../apps/frontend/azuredeploy-frontend.bicep) construye URL `staging`; el [README FastAPI](../apps/backend/fastapi/README.md) crea el slot manualmente, no la infraestructura revisada. | Puede haber frontend apuntando a un destino inexistente. Crear/configurar explícitamente la alternativa elegida y probarla. |

H01/H02 se comprobaron además mediante una inspección textual de definiciones y llamadas, sin importar módulos ni ejecutar código del proyecto. H03 se comprobó con existencia de ruta local. Las demás discrepancias se sustentan en los archivos enlazados, no en un incidente observado en Azure.

## 3. Comportamientos que necesitan pruebas focalizadas

### A. Streaming y texto final

El [servidor SSE](../apps/backend/fastapi/app/server.py) acumula y emite fragmentos `data`; cuando se cumple su condición de finalización, emite el texto acumulado en `end`. El [cliente SSE](../apps/frontend/app/helpers/streamlit_helpers.py) entrega al visor tanto `data` como `end`.

**Riesgo funcional a probar:** duplicación de respuesta si llega el mismo texto en ambos eventos. Además, el cierre depende de eventos `on_chain_end` llamados `LangGraph`, que pueden incluir grafos anidados. No se ha reproducido un cierre prematuro; debe comprobarse con la versión seleccionada.

Prueba propuesta: capturar eventos, comparar concatenación de tokens con la respuesta esperada y verificar una sola respuesta visible y un cierre al final del grafo exterior.

### B. Continuidad y retención

La [plantilla Cosmos DB](../azuredeploy.bicep) usa TTL de 1000 segundos, aproximadamente 16 minutos y 40 segundos, para elementos sin un override. No es una garantía de memoria indefinida. La retención efectiva depende del contenedor usado y de las escrituras.

Prueba propuesta: guardar y recuperar un hilo, reiniciar el backend, comprobar otro hilo independiente y verificar el comportamiento de expiración esperado.

### C. Búsqueda en múltiples índices

[get_search_results](../common/utils.py) espera campos semánticos y respuestas con `value`. Cuando un índice responde de manera distinta o falla, el tratamiento no constituye una validación completa de cada respuesta HTTP.

Prueba propuesta: tres índices válidos, uno inexistente, uno vacío y un resultado por debajo del umbral; comprobar que el error se distingue de una búsqueda sin resultados.

### D. Construcción del supervisor

El [supervisor manual](../common/graph.py) utiliza salida estructurada Pydantic; el estado del grafo espera actualizaciones accesibles por `next`. Su contrato debe comprobarse con la versión fijada de LangGraph.

Prueba propuesta: invocar el supervisor con un estado mínimo, comprobar la forma devuelta, el nodo siguiente y `FINISH` después de un especialista.

### E. Misma pregunta, dos rutas

La [página Search](../apps/frontend/app/pages/1_Search.py) y el [agente documental](../common/graph.py) usan umbrales y tamaños de recuperación diferentes. El [notebook 12](../12-Smart_Agent.ipynb) tiene otra construcción de supervisor.

Prueba propuesta: comparar fuentes recuperadas y respuestas para un conjunto fijo de preguntas. No atribuir diferencias automáticamente al modelo.

## 4. Consideraciones previas a una exposición empresarial

Sin presentar esto como auditoría, hay decisiones operativas visibles que requieren revisión:

- El archivo de [credenciales](../credentials.env) figura versionado pese a estar ignorado. No se inspeccionó si contiene valores reales. Si existen, deben gestionarse según el proceso corporativo y no copiarse a estos documentos.
- Los agentes CSV, SQL y API disponen de herramientas de ejecución/consulta. Deben establecerse permisos mínimos, límites de datos y aislamiento adecuados antes de usarlos con información empresarial.
- La [plantilla base](../azuredeploy.bicep) y las [plantillas de aplicaciones](../apps/backend/botservice/azuredeploy-backend.bicep) no representan un diseño privado/endurecido por defecto.
- Los [logs del backend](../apps/backend/fastapi/app/server.py) incluyen preguntas y respuestas. Revisar tratamiento de datos y nivel de logging.
- El [frontend Bot Service](../apps/frontend/app/pages/2_BotService_Chat.py) inyecta configuración Direct Line en el contenido del navegador; diseñar su mecanismo de acceso antes de abrirlo a usuarios.
- No se observó instrumentación empresarial completa ni pruebas automatizadas que acrediten disponibilidad, calidad o separación entre usuarios.
- Las cifras de ahorro de compresión de las [notas de versión](../Latest_Release_Notes.md) son afirmaciones del material original, no mediciones reproducidas en este análisis.

## 5. Plan recomendado, por etapas

### Etapa 1 - Hacer coherente la base

**Trabajo propuesto:**

1. Resolver H01/H02 y el contrato de actualización del supervisor.
2. Elegir una versión Python y fijar dependencias compatibles.
3. Unificar nombres de agentes, prompts y modelos entre notebooks y apps.
4. Resolver H03/H04 y compilar Bicep antes de crear recursos.
5. Elegir base, partición y retención Cosmos DB.

**Salida verificable:** el backend importa, construye el grafo y las plantillas compilan en la configuración elegida.

### Etapa 2 - Una sola experiencia fiable

**Recomendación:** empezar por RAG documental y un único canal. Evita estrenar a la vez SQL, CSV, web, API y voz.

**Salida verificable:** un conjunto pequeño de preguntas con documentos conocidos, respuestas respaldadas y pruebas de “no tengo evidencia suficiente”.

La selección de un subconjunto de agentes requiere un cambio explícito: el constructor actual no ofrece activación por capacidades.

### Etapa 3 - Memoria y streaming

1. Verificar aislamiento funcional entre hilos.
2. Verificar persistencia tras reinicio y retención.
3. Probar contrato SSE, errores y desconexiones.
4. Elegir una estructura de paquete y un arranque reproducible.

**Salida verificable:** conversación de varios turnos, una sola respuesta visible y diagnóstico claro de errores.

### Etapa 4 - Ampliar herramientas y preparar operación

1. Incorporar una fuente adicional cada vez.
2. Añadir pruebas de herramientas y enrutamiento.
3. Revisar controles de acceso, secretos, conectividad y logs.
4. Medir recuperación, fundamentación, latencia y coste.
5. Incorporar voz/visión si realmente aporta valor al caso.

**Salida verificable:** checklist aprobado para una demo controlada; la decisión de producción requiere una evaluación posterior.

## 6. Qué se hizo y qué no se hizo

**Hecho:** lectura estructural de código, inventario local, inspección de encabezados y configuraciones relevantes de los 16 notebooks, contraste entre módulos y documentación, y creación de estos documentos con cinco diagramas.

**No hecho:** correcciones de código, instalación de dependencias, importación de los backends, pruebas de integración, despliegue, consulta del tenant, auditoría de seguridad o mediciones de calidad/coste.

**Conclusión:** tienes una buena base para aprender y construir una POC multiagente, pero la siguiente inversión debe ser alinear el ensamblado y validar un flujo reducido antes de ampliar el escenario.
