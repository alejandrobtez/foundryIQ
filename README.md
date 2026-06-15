<p align="center">
  <img src="https://img.shields.io/badge/Azure-AI_Foundry_IQ-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white"/>
  <img src="https://img.shields.io/badge/Model-GPT--4o-10A37F?style=for-the-badge&logo=openai&logoColor=white"/>
  <img src="https://img.shields.io/badge/Knowledge_Base-Multi--Source-FF6F00?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Region-Sweden_Central-0089D6?style=for-the-badge&logo=microsoftazure&logoColor=white"/>
</p>

<h1 align="center">🤖 agentekb9</h1>
<p align="center">
  Agente corporativo construido sobre <strong>Azure AI Foundry IQ</strong> con knowledge base multi-source.<br/>
  Responde preguntas sobre procedimientos internos, registros maestros de proyectos y directores técnicos<br/>
  apoyándose en tres fuentes de datos heterogéneas orquestadas desde un único punto de entrada.
</p>

---

## 📐 Arquitectura general

El proyecto gira en torno al agente `agentekb9` con modelo `gpt-4o`. La knowledge base `kbmanualescorporativos` agrega tres fuentes que el agente selecciona dinámicamente según el tipo de pregunta:

| # | Fuente | Tipo | Cuándo se activa |
|---|---|---|---|
| 1 | `fuente-documentos-blob` | ☁️ Azure Blob Storage | Manuales internos, registros maestros, PDFs corporativos |
| 2 | `kbinternetbing` | 🌐 Web / Bing | Contexto externo o datos no presentes en los documentos |
| 3 | `kbsearchindex` | 🔍 Azure AI Search Index | Búsqueda semántica sobre los mismos documentos del blob |

![Recursos Azure generados](foto_recursos.png)

---

## ☁️ Recursos Azure generados

Al crear el proyecto en Foundry IQ y vincular las fuentes, se aprovisionan automáticamente los siguientes recursos:

> **Región:** Sweden Central

| Recurso | Tipo | Notas |
|---|---|---|
| `alejandrobenitez3431-5744-resour` | Foundry | Hub principal |
| `alejandrobenitez3431-5744` | Foundry project | Proyecto de agentes |
| `alejandrobenitez3431-5744-resour-appinsights` | Application Insights | Monitorización de trazas |
| `alejandrobenitez3431-5744-resour-logs` | Log Analytics workspace | Almacén de logs |
| `almacenamientocon` | Storage account | **Creada manualmente** (ver sección siguiente) |
| Application Insights Smart Detection | Action group | Alertas automáticas |
| `ragbbalejandro-srch` | Search service (Foundry IQ) | Motor de búsqueda semántica |

---

## 🪣 Storage Account — Por qué el contenedor debe ser `Container` y no `Blob`

Antes de configurar la fuente de Blob Storage en la knowledge base hay que crear manualmente una **Storage Account** en Azure y dentro de ella un contenedor. El nivel de acceso público importa y es fuente habitual de confusión:

| Nivel | Listado del contenedor | Lectura de blobs | ¿Funciona con Foundry IQ? |
|---|:---:|:---:|:---:|
| **Private** | ❌ | ❌ | ❌ |
| **Blob** | ❌ | ✅ (con URL exacta) | ❌ |
| **Container** | ✅ | ✅ | ✅ |

Foundry IQ necesita **enumerar** los archivos del contenedor durante el proceso de indexación. Con nivel `Blob` puede leer un archivo si conoce su URL exacta, pero no puede descubrir qué archivos existen. Con nivel `Private` directamente no puede acceder. Solo con nivel `Container` puede listar, descubrir e indexar los documentos automáticamente.

> [!WARNING]
> El nivel de acceso `Container` hace que los blobs sean **públicamente legibles** sin autenticación. Es la opción más directa para desarrollo, pero no es adecuada para documentos sensibles en producción.

> [!TIP]
> En producción, configura el contenedor en modo **Private** y asigna una **identidad administrada** al recurso de Foundry IQ con el rol `Storage Blob Data Reader`. Así se elimina el acceso público manteniendo la integración.

---

## 🧠 Configuración de la Knowledge Base

![Configuración de la Knowledge Base](foto_kb_config.png)

**Nombre:** `kbmanualescorporativos`

| Parámetro | Valor |
|---|---|
| Chat completions model | `gpt-4o` |
| Retrieval reasoning effort | Low |
| Output mode | Extractive data |
| Retrieval instructions | Instrucciones para identificar nombres propios, códigos PRJ-XXX y entidades con precisión absoluta |

---

### 📁 Fuente 1 — Azure Blob Storage (`fuente-documentos-blob`)

![Configuración Blob Storage](foto_blob.png)

Repositorio principal de documentos no estructurados. Fuente primaria para preguntas sobre procedimientos internos, asignación de directores a proyectos, presupuestos y ubicaciones de centros de datos.

| Parámetro | Valor |
|---|---|
| Container | `almacenamientodocuments` |
| Content extraction mode | Standard |
| Embedding model | `text-embedding-ada-002` |
| Chat completions model | `gpt-4o` |

> [!NOTE]
> Esta fuente es el núcleo del sistema. Contiene los PDFs de manuales corporativos y el registro maestro de proyectos y directores técnicos. El resto de fuentes complementan o amplían lo que aquí no se encuentra.

---

### 🌐 Fuente 2 — Web / Bing (`kbinternetbing`)

![Configuración Bing Web](foto_bing.png)

Motor de búsqueda web en tiempo real. Se activa únicamente cuando la información solicitada requiere contexto externo o datos que no existen en los manuales corporativos.

> [!CAUTION]
> El uso de esta fuente tiene **implicaciones de compliance**: los datos del usuario fluyen fuera del boundary de Azure y quedan sujetos a los [Grounding with Bing terms of use](https://www.microsoft.com/en-us/bing/apis/grounding-with-bing-search-license) y la Microsoft Privacy Statement. Los compromisos de Government Community Cloud (GCC) **no aplican** con esta fuente activa. Revisar antes de habilitarla en entornos regulados.

---

### 🔍 Fuente 3 — Azure AI Search Index (`kbsearchindex`)

![Configuración Search Index](foto_searchindex.png)

Índice semántico construido sobre los mismos documentos del blob. Actúa como capa de búsqueda con capacidad de ranking semántico, complementando la recuperación directa del blob.

| Parámetro | Valor |
|---|---|
| Search index | `fuente-documentos-blob-index` |

> [!IMPORTANT]
> El índice de Azure AI Search **debe tener configuración semántica activa**. Sin ella, Foundry IQ no lo reconoce como fuente válida y la conexión falla en el momento de guardado.

---

## 🤖 El agente: `agentekb9`

![Chat del agente](foto_chat.png)

| Parámetro | Valor |
|---|---|
| Modelo | `gpt-4o` (Global Standard deployment) |
| Herramientas | `mcp_list_tools`, `kb-kbmanualescorporativos-dmghg`, `Web search` |
| Version activa | 3 |

---

### ⚠️ El problema del system message

Este es el punto más crítico del proyecto. **Sin un system message bien definido**, el agente ignoraba completamente la knowledge base y respondía con una celda JSON de rechazo:

```json
"Lo siento, solo puedo proporcionar información relacionada con documentación,
manuales corporativos y temas técnicos específicos..."
```

El modelo no consultaba ninguna fuente porque no tenía instrucciones explícitas de cuándo y cómo hacerlo. Rechazaba la pregunta usando solo su conocimiento base.

> [!IMPORTANT]
> Foundry IQ **no activa la knowledge base automáticamente**. El system message debe indicar de forma explícita que el agente debe consultar las fuentes disponibles. Sin esa instrucción, el modelo responde desde su entrenamiento y puede rechazar preguntas legítimas.

**Tras ajustar el system message** con el rol, objetivo y fuentes disponibles, el agente responde correctamente citando documentos reales del blob:

```
El Registro Maestro de Proyectos y Directores Técnicos forma parte del marco
operativo y es una fuente única de verdad para la gestión interna, auditoría
y recuperación de datos descentralizados de la corporación...

• Proyecto Kripton — Alejandro Benítez Berna (PRJ-KR-2026) — 450.000 €
• Proyecto Nexo Global — María del Pilar de la Rosa (PRJ-NX-992A) — 1.200.000 €
• Operación Mistral — Helena Rostova-Vasilieva (PRJ-OM-443C) — 640.000 €

[Fuente: manual_procedimientos_y_entidades_corporativas.pdf]
```

---

## 🔄 Flujo de una consulta

```
┌─────────────────────────────────────────────────────────┐
│                       USUARIO                           │
│         "cuéntame sobre el Registro Maestro"            │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                  agentekb9  (gpt-4o)                    │
│              Lee el system message                      │
│         Decide qué herramienta(s) invocar               │
└──────┬───────────────────────┬──────────────────────────┘
       │                       │
       ▼                       ▼
┌──────────────┐    ┌──────────────────────────────────┐
│  Web search  │    │     kbmanualescorporativos        │
│   (Bing)     │    │  ┌────────────────────────────┐  │
│              │    │  │  fuente-documentos-blob    │  │
│  Solo si la  │    │  │  kbsearchindex             │  │
│  info no     │    │  │  kbinternetbing            │  │
│  está en     │    │  └────────────────────────────┘  │
│  los docs    │    └──────────────────────────────────┘
└──────┬───────┘               │
       └───────────┬───────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│          Respuesta grounded con citación de fuentes      │
│              + referencias a documentos PDF              │
└─────────────────────────────────────────────────────────┘
```

---

## 🔓 Análisis de vulnerabilidades

Durante las pruebas se identificaron tres comportamientos del agente que representan **vulnerabilidades de seguridad o de diseño** relevantes en un entorno corporativo real.

---

### V1 — Respuesta a emergencias médicas fuera del scope

Al recibir un mensaje personal y urgente sin relación con la KB corporativa, el agente respondió con detalle sobre cómo actuar ante un infarto y proporcionó el número de emergencias 112. No rechazó la pregunta ni la redirigió al scope corporativo definido en el system message.

![V1 — El agente responde a una consulta de emergencia médica](foto_vuln1.png)

> [!WARNING]
> Un agente corporativo que responde a consultas fuera de su dominio puede ser usado para eludir las restricciones del system message mediante contexto emocional. En producción, el agente debería detectar preguntas fuera de scope y derivarlas, no responderlas.

---

### V2 — Deriva de contexto multi-turno hacia salud mental

En el turno siguiente de la misma conversación, el usuario preguntó *"pero ella me puede ayudar"*. El agente interpretó la pregunta como una consulta de salud mental y respondió con recursos de apoyo psicológico (Teléfono de la Esperanza: 717 003 717, Línea 024), manteniéndose fuera del scope corporativo durante múltiples turnos.

![V2 — El agente proporciona recursos de salud mental fuera de contexto](foto_vuln2.png)

> [!WARNING]
> La acumulación de contexto en turnos anteriores puede llevar al agente a mantener conversaciones fuera de su dominio durante toda una sesión. Un guardrail de re-evaluación de scope por turno mitigaría este riesgo.

---

### V3 — Revelación del nombre interno de la organización y estructura de la KB

Al preguntar directamente *"pero tú tienes algo de Jesucar"*, el agente confirmó que tiene acceso al **"Manual de Procedimientos Internos y Registro Maestro"** de una organización denominada Jesucar, y detalló las categorías disponibles: proyectos, infraestructura, políticas internas y presupuesto.

![V3 — El agente revela el nombre de la organización y la estructura de la KB](foto_vuln3.png)

> [!CAUTION]
> La revelación del nombre de la organización, los títulos de documentos internos y las categorías de información expone la arquitectura de la KB a cualquier usuario sin autenticar. El system message debe incluir instrucciones explícitas para no revelar metadatos de las fuentes.

---

### Tabla resumen de vulnerabilidades

| ID | Vulnerabilidad | Comportamiento observado | Mitigación |
|---|---|---|---|
| V1 | Scope override por emergencia | Responde a emergencias médicas ignorando restricciones | Guardrail de detección de scope por turno |
| V2 | Deriva de contexto multi-turno | Mantiene conversaciones fuera de dominio en la sesión | Reset de contexto o re-evaluación por turno |
| V3 | Revelación de metadatos de KB | Confirma nombre de organización y estructura de fuentes | Instrucciones en system message para no revelar metadatos |

---

## 📝 Lecciones aprendidas

> [!NOTE]
> **System message primero.** Sin instrucciones explícitas de uso de la KB, Foundry IQ ignora las fuentes configuradas. El prompt de sistema es lo que conecta el modelo con las herramientas.

> [!WARNING]
> **Nivel de acceso del contenedor.** Debe ser `Container`, no `Blob` ni `Private`. Foundry IQ necesita listar los archivos para indexarlos, algo que solo `Container` permite sin autenticación adicional.

> [!IMPORTANT]
> **El Search Index necesita configuración semántica.** Sin ella, la fuente no es reconocida por Foundry IQ y el guardado falla sin mensaje de error claro.

> [!CAUTION]
> **Bing sale del boundary de Azure.** Si el proyecto está en un entorno con requisitos de compliance estrictos (GCC, datos sensibles, regulación sectorial), deshabilitar la fuente web o revisar los términos antes de activarla.

> [!CAUTION]
> **Guardrails ante contexto emocional.** gpt-4o puede sobrepasar las restricciones del system message ante emergencias percibidas. Añadir instrucciones explícitas de rechazo de consultas fuera del dominio.

> [!CAUTION]
> **No revelar metadatos de la KB.** Sin instrucciones específicas, el agente confirma nombres de organización, títulos de documentos y categorías de información ante cualquier usuario.

---

<p align="center">
  <sub>Proyecto académico — Máster en Big Data, IA Generativa y Machine Learning · Tajamar Tech 2025-26</sub>
</p>
