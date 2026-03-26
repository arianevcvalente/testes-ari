# LaRebel — Arquitectura del Sistema

## 1. Visión General

**LaRebel** es un agente conversacional especializado en *derecho laboral mexicano*, operado por el **SNAC (Sindicato Nacional Alimenticio y del Comercio)**.

**Modelo LLM:** `gpt-4o` en todos los nodos de prompt  
**Canales:** WhatsApp (sesión de 8 horas)  
**Versión del flujo:** 120 (DRAFT)

---

## 2. Inventario de Nodos

| # | Label | Tipo | Descripción |
|---|-------|------|-------------|
| 1 | `input_node` | input | Recibe el mensaje del usuario |
| 2 | `init_lftdata` | variableNode | Inicializa `lft_data` con valor "default" |
| 3 | `init_company` | variableNode | Inicializa `company` con valor "null" |
| 4 | `create_respuestafinal` | variableNode | Inicializa `respuesta_final` con valor "Default" |
| 5 | `vav_datos_contacto` | variableAcquisitionVerificationNode | Extrae datos de contacto del usuario |
| 6 | `create_orquestadorvalue` | variableNode | Inicializa `orquestador_value` con "DEFAULT" |
| 7 | `prompt_orquestador` | promptNode | Clasifica la consulta |
| 8 | `set_orquestadorvalue` | variableNode | Actualiza `orquestador_value` |
| 9 | `condition_if_consulta_general` | conditionNode | Ramifica: Consulta general vs CCT |
| 10 | `load_lft_data` | rawDocumentNode | Carga documento RAG LFT_ENERO_2026 |
| 11 | `set_lftdata` | variableNode | Asigna contenido LFT a `lft_data` |
| 12 | `categorizartipodedenuncia` | promptNode | Clasifica caso: MUJERES/PROFEDET/SIQAL/LAREBEL |
| 13 | `set_tipodenuncia` | variableNode | Asigna clasificación a `tipo_denuncia` |
| 14 | `multicondicional_tipo_de_denuncia` | multiConditionNode | Enruta por valor de `tipo_denuncia` |
| 15 | `prompt_larebel` | promptNode | Respuesta principal LaRebel |
| 16 | `update_respuesta_final_larebel` | variableNode | Actualiza `respuesta_final` |
| 17 | `prompt_siqal` | promptNode | Flujo SIQAL |
| 18 | `update_respuesta_final_siqal` | variableNode | Actualiza `respuesta_final` |
| 19 | `prompt_profedet` | promptNode | Flujo PROFEDET |
| 20 | `update_respuesta_final_profedet` | variableNode | Actualiza `respuesta_final` |
| 21 | `load_chunked_protocoloatencionmujeres` | chunkedDocumentRagNode | Carga RAG PROTOCOLO_ATENCION_MUJERES |
| 22 | `set_protocoloatencionmujeresdata` | variableNode | Asigna chunks a `pam_data` |
| 23 | `prompt_mujeres` | promptNode | Flujo MUJERES |
| 24 | `update_respuesta_final` (MUJERES) | variableNode | Actualiza `respuesta_final` |
| 25 | `condition_node_if_user_identified` | conditionNode | ¿login es true? |
| 26 | `vav_company` | variableAcquisitionVerificationNode | Solicita nombre de empresa |
| 27 | `variable_node_setcompany` | variableNode | Actualiza `company` |
| 28 | `code_get_cct_by_company` | codeExecutionNode | Llama webhook para obtener CCT |
| 29 | `set_cct_content` | variableNode | Asigna respuesta a `cct` |
| 30 | `if_company_not_exists_with_cct_content` | conditionNode | ¿`cct` incluye "404"? |
| 31 | `vav_identificacin_usuario` | variableAcquisitionVerificationNode | Solicita nombre y número de empleado |
| 32 | `variable_node_update_useridentified` | variableNode | Establece `userIdentified = "1"` |
| 33 | `code_execution_consulta_empleado` | codeExecutionNode | Valida identidad del empleado |
| 34 | `variable_node_actualizar_vav_datos_contacto_consulta_empleado` | variableNode | Actualiza `consulta_empleado` |
| 35 | `variable_node_respuesta_consulta_empleado` | variableNode | Inicializa `Respuesta consulta empleado` |
| 36 | `condition_node_if_undefined` | conditionNode | ¿`Respuesta consulta empleado` incluye "undefined"? |
| 37 | `condition_node_if_externo` | conditionNode | ¿`Respuesta consulta empleado` incluye "EXTERNO"? |
| 38 | `if_rich` | codeExecutionNode | Verifica si empresa es PRODUCTOS RICH |
| 39 | `condition_node_if_rich` | conditionNode | ¿`if_rich.output` incluye "TRUE"? |
| 40 | `load_lft_data` (login failed) | rawDocumentNode | Carga LFT para login fallido |
| 41 | `set_lftdata` (login failed) | variableNode | Asigna LFT a `lft_data` |
| 42 | `prompt_node_procedimiento_login_failed` | promptNode | Responde cuando no hay registros |
| 43 | `variable_node_update_respuesta_final` (login failed) | variableNode | Actualiza `respuesta_final` |
| 44 | `prompt_node_procedimiento_company_not_exist` | promptNode | Sugiere empresas similares |
| 45 | `variable_node_update_respuesta_final` (company not exist) | variableNode | Actualiza `respuesta_final` |
| 46 | `prompt_larebel_contratos_colectivos` | promptNode | LaRebel con contexto CCT |
| 47 | `variable_node_update_respuesta_final` (CCT) | variableNode | Actualiza `respuesta_final` |
| 48 | `prompt_flujo_prodrich` | promptNode | Flujo PRODRICH |
| 49 | `variable_node_updaterespuestafinalprodrich` | variableNode | Actualiza `respuesta_final` |
| 50 | `prompt_node_analiza_consulta_resuelta` | promptNode | Evalúa si consulta fue resuelta |
| 51 | `variable_node_update_caseresolved` | variableNode | Actualiza `caseResolved` |
| 52 | `limpiar_doble_asteriscos` | codeExecutionNode | Reemplaza `**` por `*` |
| 53 | `limpiar_respuestafinal` | variableNode | Actualiza `respuesta_final` limpia |
| 54 | `variable_contactnumber` | variableNode | Inicializa `contactNumber` con "---" |
| 55 | `prompt_node_analiza_asesoramiento_humano` | promptNode | Detecta si pidió asesor humano |
| 56 | `variable_node_asesorhumano` | variableNode | Asigna resultado a `asesorHumano` |
| 57 | `condition_asesorhumano` | conditionNode | ¿`asesorHumano` incluye "1"? |
| 58 | `vav_contactnumber` | variableAcquisitionVerificationNode | Solicita número telefónico |
| 59 | `variable_node_contactnumber` | variableNode | Actualiza `contactNumber` |
| 60 | `code_execution_node_analytics_body_request` | codeExecutionNode | Construye JSON de analytics |
| 61 | `http_node_analytics_for_test_agent` | httpNode | Envía analytics al webhook |
| 62 | `final_output` | output | Envía `respuesta_final` al usuario |

---

## 3. Variables Globales

| Variable | Valor inicial | Descripción |
|----------|--------------|-------------|
| `respuesta_final` | "Default" | Respuesta final que se envía al usuario |
| `orquestador_value` | "DEFAULT" | Clasificación: "Consulta general" o "Contratos colectivos" |
| `company` | "null" | Nombre de empresa del usuario |
| `lft_data` | "default" | Contenido del documento LFT_ENERO_2026 |
| `cct` | — | Contenido del CCT (o "404"/"204") |
| `tipo_denuncia` | — | MUJERES / PROFEDET / SIQAL / LAREBEL |
| `pam_data` | — | Chunks del Protocolo de Atención a Mujeres |
| `contactNumber` | "---" | Teléfono para llamada del asesor humano |
| `asesorHumano` | — | "1" si pidió asesor humano, "0" si no |
| `userIdentified` | — | "1" si se autenticaron los datos |
| `caseResolved` | — | "1" si la consulta fue resuelta |
| `Respuesta consulta empleado` | — | Resultado del webhook de validación |

---

## 4. Webhooks Externos

> ⚠️ **Problema de seguridad**: Las URLs están hardcodeadas. Se recomienda moverlas a variables de entorno.

| Endpoint | Método | Nodo | Descripción |
|----------|--------|------|-------------|
| `http://n8n-alb-1431437514.mx-central-1.elb.amazonaws.com/webhook/validate-identity` | POST | `code_execution_consulta_empleado` | Valida identidad del empleado |
| `http://n8n-alb-1431437514.mx-central-1.elb.amazonaws.com/webhook/validate-company` | POST | `code_get_cct_by_company` | Devuelve CCT de la empresa |
| `http://n8n-alb-1431437514.mx-central-1.elb.amazonaws.com/webhook/snac-analytics-for-test-agent` | POST | `http_node_analytics_for_test_agent` | Registra métricas de sesión |

---

## 5. RAG Documents

| Nombre RAG | Tipo | Usado en | Descripción |
|------------|------|----------|-------------|
| `LFT_ENERO_2026` | rawDocumentNode | `load_lft_data` (×2) | Ley Federal del Trabajo enero 2026 |
| `PROTOCOLO_ATENCION_MUJERES` | chunkedDocumentRagNode | `load_chunked_protocoloatencionmujeres` | Protocolo de atención a mujeres trabajadoras |

---

## 6. Problemas Conocidos

| # | Problema | Severidad | Nodo afectado | Descripción |
|---|---------|-----------|---------------|-------------|
| 1 | URLs internas hardcodeadas | Alta | webhooks | URLs del ALB interno de AWS expuestas en el código |
| 2 | Detección de "404" por string | Media | `if_company_not_exists_with_cct_content` | Si un CCT contiene "404", enrutará incorrectamente |
| 3 | `console.log(undefined)` | Baja | `code_execution_consulta_empleado` | Cuando `data.code === 0` registra `undefined` |
| 4 | Normalización inconsistente en `if_rich` | Media | `if_rich` | Usa `toLowerCase()` pero `company` llega en MAYÚSCULAS |
| 5 | Duplicación de bloques de prompt | Media | Todos los promptNodes | 5 bloques copiados textualmente en 7-8 prompts |
| 6 | Inconsistencia de año en Login Failed | Baja | `prompt_node_procedimiento_login_failed` | Original decía 2025, corregido a 2026 |
| 7 | Doble mapeo en Filtro CCT | Baja | `filtro-cct.md` | `CCT-INICIAL-GAMESA-TUXPAN-CFCRL.pdf` mapeado a dos empresas |

---

## 7. Bloques Compartidos

| Bloque | Aparece en |
|--------|-----------|
| `escalacion-asesor-humano` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, company-not-exists, contratos-colectivos |
| `snac-papel-clave` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, company-not-exists, contratos-colectivos |
| `regla-suprema-formato` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, company-not-exists, contratos-colectivos |
| `reglas-salida-numerica` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, contratos-colectivos |
| `cierre-conversacion` | la-rebel, siqal, profedet, mujeres, flujo-rich, company-not-exists |

> **Nota**: O flujo PRODRICH tem uma variante mais restritiva do bloco de formato: proibe tanto negritas como itálicas. Isso é *intencional* — os trabalhadores de campo da RICH recebem a resposta em um canal que não renderiza markdown.
