# LaRebel — Mapa de Flujo

## 1. Diagrama General

```
┌─────────────────────────────────────────────────────────────────┐
│                         ENTRADA                                  │
│  input_node → init variables → vav_datos_contacto               │
│               (lft_data, company,   (extrae datos               │
│                respuesta_final,      en background)             │
│                orquestador_value)                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                  prompt_orquestador
                  (clasifica consulta)
                         │
          ┌──────────────┴──────────────────┐
          │                                 │
   "Consulta general"            "Contratos colectivos"
          │                                 │
   load_lft_data                 ¿login verdadero?
   set_lftdata                   (vav_datos_contacto.login)
          │                         │              │
   categorizartipodedenuncia      TRUE            FALSE
          │                         │              │
   ┌──────┴──────────┐         VAV identif.   VAV empresa
   │   tipo_denuncia  │         usuario        (vav_company)
   └──┬──┬──┬──┬─────┘              │              │
      │  │  │  │               webhook            webhook
   LARE│PRO│SIQ│MUJ            validar emp        get CCT
   BEL │FED│AL │ERES                │              │
      │  │  │  │            ¿encontrado?      ¿empresa existe?
   prompt prompt prompt  RAG    │        │       │         │
   lare profe siqal pam_data  SÍ       NO      SÍ        NO
      │  │  │  │            (→ CCT)  (login  ¿RICH?   company
      └──┴──┴──┘            flujo   failed)    │      not exist
          │                    │         │    SÍ  NO     │
   respuesta_final          resp_fin resp_fin RICH  CCT resp_fin
          │                                   │     │
          └───────────────────────────────────┴─────┘
                         │
              prompt_node_analiza_consulta_resuelta
              (caseResolved: 0 o 1)
                         │
              limpiar_doble_asteriscos
              (post-procesamiento **)
                         │
              prompt_node_analiza_asesoramiento_humano
              (asesorHumano: 0 o 1)
                         │
              ¿pidió asesor humano?
              │                  │
             SÍ                 NO
              │                  │
         VAV contactNumber  code analytics
              │              http analytics
         code analytics           │
         http analytics      final_output
              │
         final_output
```

---

## 2. Tabla de Rutas del Flujo

| Ruta | Condición de activación | Nodos principales | Prompt utilizado |
|------|------------------------|-------------------|------------------|
| **A — Consulta General / LaRebel** | Usuario pregunta sobre derechos laborales sin mencionar CCT | orquestador → load_lft → categorizar → multicondicional (LAREBEL) → prompt_larebel | `la-rebel.md` |
| **B — Consulta General / SIQAL** | Usuario describe condición insegura y acepta denunciar ante SIQAL | orquestador → load_lft → categorizar → multicondicional (SIQAL) → prompt_siqal | `siqal.md` |
| **C — Consulta General / PROFEDET** | Usuario pide queja formal o representación legal ante PROFEDET | orquestador → load_lft → categorizar → multicondicional (PROFEDET) → prompt_profedet | `profedet.md` |
| **D — Consulta General / MUJERES** | Usuaria mujer + violencia/acoso/discriminación de género | orquestador → load_lft → categorizar → multicondicional (MUJERES) → RAG protocolo → prompt_mujeres | `mujeres.md` |
| **E — CCT sin login previo** | Usuario pide CCT, sin autenticación previa | orquestador (CCT) → VAV empresa → webhook CCT → VAV identificación → webhook validar empleado → CCT prompt | `contratos-colectivos.md` |
| **F — CCT login previo** | Usuario ya autenticado (login=true) solicita CCT | orquestador (CCT) → webhook CCT → Filtro CCT → prompt CCT | `contratos-colectivos.md` |
| **G — CCT empresa RICH** | Empleado de PRODUCTOS RICH autenticado | orquestador (CCT) → webhook CCT → if_rich=TRUE → prompt_flujo_prodrich | `flujo-rich.md` |
| **H — Empresa no encontrada** | Empresa no existe en la base de CCT (webhook devuelve vacío/404) | orquestador (CCT) → webhook CCT → cct incluye "404" → prompt company not exist | `company-not-exists.md` |
| **I — Empleado no encontrado** | Webhook de validación no encuentra al empleado | webhook validar empleado → output undefined/vacío → load_lft → prompt login failed | `login-failed.md` |
| **J — Empleado EXTERNO** | Webhook devuelve "EXTERNO" (no afiliado o empresa sin CCT) | webhook validar empleado → "EXTERNO" → load_lft → prompt login failed | `login-failed.md` |
| **K — Escalación a asesor humano** | Usuario pide explícitamente hablar con asesor o Alejandro Martínez | cualquier prompt detecta → analiza_asesoramiento_humano=1 → VAV contactNumber → analytics | (dentro de cualquier prompt) |

---

## 3. Puntos de Decisión

| Nodo condición | Variable evaluada | Operador | Valor | Rama TRUE | Rama FALSE |
|----------------|------------------|----------|-------|-----------|------------|
| `condition_if_consulta_general` | `orquestador_value` | includes | "Consulta general" | load LFT → categorizar | flujo CCT |
| `condition_node_if_user_identified` | `vav_datos_contacto.login` | is_true | — | VAV identificación usuario | VAV empresa |
| `if_company_not_exists_with_cct_content` | `cct` | includes | "404" | prompt company not exist | VAV identificación |
| `condition_node_if_undefined` | `Respuesta consulta empleado` | includes | "undefined" | load LFT → login failed | check EXTERNO |
| `condition_node_if_externo` | `Respuesta consulta empleado` | includes | "EXTERNO" | load LFT → login failed | check RICH |
| `condition_node_if_rich` | `if_rich.output` | includes | "TRUE" | flujo PRODRICH | flujo CCT estándar |
| `condition_asesorhumano` | `asesorHumano` | includes | "1" | VAV contactNumber | analytics directo |
| `multicondicional_tipo_de_denuncia` | `tipo_denuncia` | includes (×4) | PROFEDET / SIQAL / LAREBEL / MUJERES | nodo de prompt correspondiente | — |

---

## 4. Referencia de Tipos de Nodo

| Tipo | Descripción |
|------|-------------|
| `input` | Punto de entrada. Recibe el mensaje del usuario y el session_id. |
| `output` | Punto de salida. Envía la variable `respuesta_final` al usuario. |
| `promptNode` | Nodo de LLM. Ejecuta un prompt con contexto de conversación. Usa `gpt-4o` salvo indicación. |
| `variableNode` | Operación sobre una variable: `initialize` (crea), `update` (sobreescribe) o `updatePersistent` (persiste entre turnos). |
| `conditionNode` | Bifurcación binaria (TRUE/FALSE) basada en una condición sobre una variable. |
| `multiConditionNode` | Bifurcación múltiple: evalúa N condiciones y activa la primera que se cumple. |
| `variableAcquisitionVerificationNode` | Nodo inteligente que extrae o solicita datos del usuario. Tiene un prompt de adquisición configurable. Usa `gpt-4o-mini` por defecto. |
| `codeExecutionNode` | Ejecuta código JavaScript (Node.js). Accede a variables del flujo via `process.env.TOOL_VARS`. La salida es lo que se pasa a `console.log()`. |
| `httpNode` | Realiza una petición HTTP (GET/POST) a un endpoint externo. |
| `rawDocumentNode` | Carga páginas de un documento RAG completo (sin chunking). |
| `chunkedDocumentRagNode` | Carga los N chunks más relevantes de un RAG basado en el input actual (búsqueda semántica). |

---

## 5. Flujo de Post-Procesamiento (cada turno)

Después de que cualquier prompt genera `respuesta_final`, el flujo siempre ejecuta:

```
respuesta_final
      │
      ▼
prompt_node_analiza_consulta_resuelta
(¿fue la consulta resuelta? → 0 o 1)
      │
      ▼
variable_node_update_caseresolved
(persiste caseResolved)
      │
      ▼
limpiar_doble_asteriscos
(regex: reemplaza ** por *)
      │
      ▼
limpiar_respuestafinal
(actualiza respuesta_final limpia)
      │
      ▼
variable_contactnumber
(inicializa contactNumber = "---")
      │
      ▼
prompt_node_analiza_asesoramiento_humano
(¿pidió asesor humano? → 0 o 1)
      │
      ▼
condition_asesorhumano
      │                │
     "1"              "0"
      │                │
vav_contactnumber   (salta)
      │
variable_node_contactnumber
      │
      ▼
code_execution_node_analytics_body_request
(construye JSON de analytics)
      │
      ▼
http_node_analytics_for_test_agent
(POST a n8n)
      │
      ▼
final_output
(envía respuesta al usuario)
```

---

## 6. Normalización de Datos

Los webhooks externos reciben datos normalizados de la siguiente forma:

```javascript
// Nombre y empresa: sin acentos, en mayúsculas
str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toUpperCase()

// Número de empleado: string, trim
str.toString().trim()
```

> ⚠️ **Bug conocido**: El nodo `if_rich` usa `company.toLowerCase()` para comparar con variantes de "prodrich", pero `company` llega normalizado en MAYÚSCULAS desde `variable_node_setcompany`. La comparación siempre fallará. Debe usarse `.toUpperCase()` y comparar contra `["PRODRICH", "PRODUCTOS RICH", "RICH'S", "RICHIS", "RICHS"]`.
