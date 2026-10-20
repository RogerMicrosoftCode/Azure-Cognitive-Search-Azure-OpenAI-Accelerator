# ROGERSE - Entender este repositorio

## En una frase

Tienes un **acelerador educativo para construir una prueba de concepto de IA generativa en Azure**: permite buscar documentos, conversar con datos y coordinar cinco agentes especializados mediante un supervisor.

No es un modelo entrenado propio ni una aplicación empresarial lista para producción. Es una base de notebooks, componentes reutilizables, aplicaciones de demostración y plantillas de infraestructura.

## Documentos

| Orden | Documento | Qué encontrarás |
|---|---|---|
| 1 | [Análisis del repositorio](01-ANALISIS-DEL-REPOSITORIO.md) | Qué tienes, para qué sirve cada parte y cómo se conectan. |
| 2 | [Arquitectura y flujos](02-ARQUITECTURA-Y-FLUJOS.md) | Cinco dibujos Mermaid: arquitectura, ingesta, agentes, conversación y despliegue. |
| 3 | [Guía de notebooks](03-GUIA-NOTEBOOKS.md) | Los 16 notebooks, sus objetivos y una ruta de aprendizaje. |
| 4 | [Configuración y despliegue](04-CONFIGURACION-Y-DESPLIEGUE.md) | Servicios, variables, formas de ejecución y comprobaciones necesarias. |
| 5 | [Hallazgos y siguientes pasos](05-HALLAZGOS-Y-SIGUIENTES-PASOS.md) | Inconsistencias verificadas por lectura y una ruta para llevarlo a una demo fiable. |

## Lectura rápida: qué tienes

- **Experiencia de búsqueda documental:** recuperación híbrida y generación de respuestas con referencias.
- **Experiencia de chat:** interfaz Streamlit con dos alternativas de backend.
- **Cinco especialistas:** documentos, CSV, SQL, web y APIs.
- **Un supervisor:** selecciona el siguiente agente o termina la conversación.
- **Memoria persistente:** un checkpointer personalizado de LangGraph sobre Cosmos DB.
- **Multimodalidad:** notebook de visión/audio y capacidades de voz en el chat FastAPI.
- **Material de taller:** 16 notebooks y datasets de ejemplo.
- **Infraestructura como código:** Bicep y plantillas ARM, con automatización parcial.

## Lo más importante antes de ejecutarlo

La arquitectura está representada en el código, pero **no conviene asumir que todo arranca tal como está**. Encontré diferencias entre el notebook integrador, el grafo usado por las aplicaciones y las plantillas de despliegue.

Ejemplos concretos:

1. El grafo importa un prompt que no existe con ese nombre.
2. Sus llamadas de creación de agentes no envían un parámetro obligatorio.
3. La ruta del módulo Bicep del backend no coincide con el archivo existente.
4. La configuración de Cosmos DB no es coherente entre infraestructura y checkpointer.

El detalle y los criterios de comprobación están en [Hallazgos](05-HALLAZGOS-Y-SIGUIENTES-PASOS.md). No se modificó la aplicación para corregirlos: esta entrega es de análisis y documentación.

## Cómo ver los dibujos

Abre [Arquitectura y flujos](02-ARQUITECTURA-Y-FLUJOS.md) con una vista Markdown compatible con Mermaid, por ejemplo la de GitHub. Si tu vista de VS Code no lo renderiza, seguirás teniendo el diagrama como texto editable dentro del documento y su explicación debajo.

## Alcance y evidencia

- Fecha de análisis local: **19 de octubre de 2026**.
- Revisión local tomada como referencia: **`5c53f51`**, con fecha de commit **10 de julio de 2025**.
- Fuentes: código de las aplicaciones, utilidades compartidas, código y encabezados de notebooks, Bicep, configuración y workflows.
- Se contrastó el concepto de búsqueda híbrida con [Microsoft Learn](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview).
- **No se consultó la suscripción de Azure**, ni se comprobó que existan servicios, índices, despliegues de modelos o aplicaciones funcionando.
- No se leyeron valores de [credenciales locales](../credentials.env), ni se reprodujeron secretos o salidas de ejecución de notebooks.
- No se ejecutaron notebooks, despliegues ni llamadas a los datasets o APIs externas.
- Los archivos ZIP se inventariaron por nombre y tamaño; no se inspeccionó ni reprodujo su contenido.
- No se inspeccionaron los documentos binarios de presentación o diseño originales.

**Distinción de estados:** “implementado” significa que existe código; “declarado” significa que aparece en una plantilla; “operativo” requeriría una prueba de ejecución que no forma parte de este análisis.
