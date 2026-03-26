# LaRebel — Mapa de Flujo

## 1. Diagrama General

```
┌─────────────────────────────────────────────────────────────────┐
│                         ENTRADA                                  │
│  input_node → init variables → vav_datos_contacto               │
└────────────────────────┬───────────────────────────────────────┘
                         │
                  prompt_orquestador
                  (clasifica consulta)
                         │
          ┌──────────────┴──────────────────┐
          │                                 │
   "Consulta general"            "Contratos colectivos"
          │                                 │
   load_lft_data                 ¿login verdadero?
   set_lftdata                        │          │
          │                          TRUE      FALSE
   categorizartipodedenuncia          │            │
          │                      VAV identif.  VAV empresa
   ┌──────┴─────────┐           │            │
   │  tipo_denuncia │      webhook val.emp  webhook CCT
   └──┬──┬──┬──┬────┘        │            │
   LARE PRO SIQ MUJ       ¿encontrado?  ¿empresa existe?
   BEL  FED AL  ERES       │      │       │         │
      │  │  │   │        SÍ     NO      SÍ        NO
   prompt              (CCT) (login  ¿RICH?   company
      │  │  │   │          failed)   │   │   not exist
      └──┴──┴──┘                  SÍ   NO
          │                        RICH  CCT
          └─────────────────────────┘
                         │
              POST-PROCESAMIENTO
              (analiza resolución → limpia ** →
               detecta asesor humano → analytics)
                         │
                    final_output
```

---

## 2. Tabla de Rutas del Flujo

| Ruta | Condición de activación | Prompt utilizado |
|------|------------------------|------------------|
| **A — Consulta General / LaRebel** | Usuario pregunta sobre derechos laborales sin mencionar CCT | `la-rebel.md` |
| **B — Consulta General / SIQAL** | Usuario describe condición insegura y acepta denunciar | `siqal.md` |
| **C — Consulta General / PROFEDET** | Usuario pide queja formal o representación legal | `profedet.md` |
| **D — Consulta General / MUJERES** | Usuaria mujer + violencia/acoso/discriminación | `mujeres.md` |
| **E — CCT sin login previo** | Usuario pide CCT sin autenticación previa | `contratos-colectivos.md` |
| **F — CCT login previo** | Usuario ya autenticado solicita CCT | `contratos-colectivos.md` |
| **G — CCT empresa RICH** | Empleado de PRODUCTOS RICH autenticado | `flujo-rich.md` |
| **H — Empresa no encontrada** | Webhook devuelve 404 | `company-not-exists.md` |
| **I — Empleado no encontrado** | Webhook no encuentra al empleado | `login-failed.md` |
| **J — Empleado EXTERNO** | Webhook devuelve "EXTERNO" | `login-failed.md` |
| **K — Escalación a asesor humano** | Usuario pide explícitamente hablar con asesor | (dentro de cualquier prompt) |

---

## 3. Puntos de Decisión

| Nodo condición | Variable evaluada | Valor | Rama TRUE | Rama FALSE |
|----------------|------------------|-------|-----------|------------|
| `condition_if_consulta_general` | `orquestador_value` | "Consulta general" | load LFT → categorizar | flujo CCT |
| `condition_node_if_user_identified` | `vav_datos_contacto.login` | true | VAV identificación | VAV empresa |
| `if_company_not_exists_with_cct_content` | `cct` | "404" | prompt company not exist | VAV identificación |
| `condition_node_if_undefined` | `Respuesta consulta empleado` | "undefined" | login failed | check EXTERNO |
| `condition_node_if_externo` | `Respuesta consulta empleado` | "EXTERNO" | login failed | check RICH |
| `condition_node_if_rich` | `if_rich.output` | "TRUE" | flujo PRODRICH | flujo CCT estándar |
| `condition_asesorhumano` | `asesorHumano` | "1" | VAV contactNumber | analytics directo |
| `multicondicional_tipo_de_denuncia` | `tipo_denuncia` | PROFEDET/SIQAL/LAREBEL/MUJERES | prompt correspondiente | — |

---

## 4. Referencia de Tipos de Nodo

| Tipo | Descripción |
|------|-------------|
| `input` | Punto de entrada. Recibe el mensaje del usuario. |
| `output` | Punto de salida. Envía `respuesta_final` al usuario. |
| `promptNode` | Nodo de LLM. Usa `gpt-4o`. |
| `variableNode` | Operación sobre una variable: initialize, update o updatePersistent. |
| `conditionNode` | Bifurcación binaria (TRUE/FALSE). |
| `multiConditionNode` | Bifurcación múltiple. |
| `variableAcquisitionVerificationNode` | Extrae o solicita datos del usuario. Usa `gpt-4o-mini`. |
| `codeExecutionNode` | Ejecuta código JavaScript. Accede a variables via `process.env.TOOL_VARS`. |
| `httpNode` | Realiza petición HTTP a endpoint externo. |
| `rawDocumentNode` | Carga páginas de un documento RAG completo. |
| `chunkedDocumentRagNode` | Carga los N chunks más relevantes por búsqueda semántica. |

---

## 5. Flujo de Post-Procesamiento

Después de que cualquier prompt genera `respuesta_final`:

```
respuesta_final
      │
      ▼
prompt_node_analiza_consulta_resuelta (¿0 o 1?)
      │
      ▼
limpiar_doble_asteriscos (reemplaza ** por *)
      │
      ▼
prompt_node_analiza_asesoramiento_humano (¿0 o 1?)
      │
      ▼
condition_asesorhumano
      │              │
     "1"             "0"
      │               │
vav_contactnumber   (salta)
      │
      ▼
code_execution_node_analytics_body_request
      │
      ▼
http_node_analytics_for_test_agent (POST a n8n)
      │
      ▼
final_output
```

---

## 6. Normalización de Datos

```javascript
// Nombre y empresa: sin acentos, en mayúsculas
str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toUpperCase()

// Número de empleado: string, trim
str.toString().trim()
```

> ⚠️ **Bug conocido**: El nodo `if_rich` usa `company.toLowerCase()` pero `company` llega en MAYÚSCULAS. La comparación siempre fallará. Debe usarse `.toUpperCase()` y comparar contra `["PRODRICH", "PRODUCTOS RICH", "RICH'S", "RICHIS", "RICHS"]`.
