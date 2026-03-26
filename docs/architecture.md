# LaRebel — Arquitectura del Sistema

## 1. Visión General

**LaRebel** es un agente conversacional especializado en *derecho laboral mexicano*, operado por el **SNAC (Sindicato Nacional Alimenticio y del Comercio)**. Atiende a trabajadores afiliados y no afiliados mediante WhatsApp, ofreciendo:

- Asesoría general sobre derechos laborales (LFT 2026)
- Consulta de Contratos Colectivos de Trabajo (CCT) por empresa
- Canalización a SIQAL, PROFEDET y flujo especializado para mujeres
- Escalación a asesor humano del SNAC cuando el caso lo requiere

**Modelo LLM:** `gpt-4o` en todos los nodos de prompt
**Canales:** WhatsApp (sesión de 8 horas)
**Versión del flujo:** 120 (DRAFT)

---

## 2. Inventario de Nodos

| # | Label | Tipo | Descripción | Variables de entrada relevantes | Variables de salida |
|---|-------|------|-------------|--------------------------------|---------------------|
| 1 | `input_node` | input | Recibe el mensaje del usuario | — | `input_node.user_input`, `input_node.session_id` |
| 2 | `init_lftdata` | variableNode | Inicializa `lft_data` con valor "default" | — | `lft_data` |
| 3 | `init_company` | variableNode | Inicializa `company` con valor "null" | — | `company` |
| 4 | `create_respuestafinal` | variableNode | Inicializa `respuesta_final` con valor "Default" | — | `respuesta_final` |
| 5 | `vav_datos_contacto` | variableAcquisitionVerificationNode | Extrae datos de contacto del usuario sin solicitarlos activamente | `input_node.user_input` | `User_Name`, `Email`, `Phone`, `userIdentified`, `legalAdviceLead`, `caseResolved`, `workplace`, `login` (bool), `consulta_empleado`, `CURP` |
| 6 | `create_orquestadorvalue` | variableNode | Inicializa `orquestador_value` con "DEFAULT" | — | `orquestador_value` |
| 7 | `prompt_orquestador` | promptNode | Clasifica la consulta como "Consulta general" o "Contratos colectivos" | `input_node.user_input`, historial (5 mensajes) | `prompt_orquestador.output` |
| 8 | `set_orquestadorvalue` | variableNode | Actualiza `orquestador_value` con la salida del orquestador | `prompt_orquestador.output` | `orquestador_value` |
| 9 | `condition_if_consulta_general` | conditionNode | Ramifica: ¿`orquestador_value` incluye "Consulta general"? | `orquestador_value` | TRUE → carga LFT / FALSE → flujo CCT |
| 10 | `load_lft_data` (flujo general) | rawDocumentNode | Carga documento RAG `LFT_ENERO_2026` | — | `load_lft_data.content` |
| 11 | `set_lftdata` (flujo general) | variableNode | Asigna contenido LFT a `lft_data` | `load_lft_data.content` | `lft_data` |
| 12 | `categorizartipodedenuncia` | promptNode | Clasifica el caso como MUJERES / PROFEDET / SIQAL / LAREBEL | historial completo | `categorizartipodedenuncia.output` |
| 13 | `set_tipodenuncia` | variableNode | Asigna clasificación a `tipo_denuncia` | `categorizartipodedenuncia.output` | `tipo_denuncia` |
| 14 | `multicondicional_tipo_de_denuncia` | multiConditionNode | Enruta por valor de `tipo_denuncia` | `tipo_denuncia` | 4 ramas: PROFEDET / SIQAL / LAREBEL / MUJERES |
| 15 | `prompt_larebel` | promptNode | Respuesta principal de LaRebel (asesoría general) | `lft_data`, historial (5 msgs) | `prompt_larebel.output` |
| 16 | `update_respuesta_final_larebel` | variableNode | Actualiza `respuesta_final` con salida de LaRebel | `prompt_larebel.output` | `respuesta_final` |
| 17 | `prompt_siqal` | promptNode | Flujo de canalización SIQAL | historial (5 msgs) | `prompt_siqal.output` |
| 18 | `update_respuesta_final_siqal` | variableNode | Actualiza `respuesta_final` con salida SIQAL | `prompt_siqal.output` | `respuesta_final` |
| 19 | `prompt_profedet` | promptNode | Flujo de canalización PROFEDET | historial (5 msgs) | `prompt_profedet.output` |
| 20 | `update_respuesta_final_profedet` | variableNode | Actualiza `respuesta_final` con salida PROFEDET | `prompt_profedet.output` | `respuesta_final` |
| 21 | `load_chunked_protocoloatencionmujeres` | chunkedDocumentRagNode | Carga RAG `PROTOCOLO_ATENCION_MUJERES` (top 3 chunks) | query del input actual | `load_chunked_protocoloatencionmujeres.chunks` |
| 22 | `set_protocoloatencionmujeresdata` | variableNode | Asigna chunks del protocolo a `pam_data` | `chunks` | `pam_data` |
| 23 | `prompt_mujeres` | promptNode | Flujo especializado para mujeres trabajadoras | `pam_data`, historial (5 msgs) | `prompt_mujeres.output` |
| 24 | `update_respuesta_final` (MUJERES) | variableNode | Actualiza `respuesta_final` con salida MUJERES | `prompt_mujeres.output` | `respuesta_final` |
| 25 | `condition_node_if_user_identified` | conditionNode | ¿`vav_datos_contacto.login` es true? | `vav_datos_contacto.login` | TRUE → VAV identificación / FALSE → VAV empresa |
| 26 | `vav_company` | variableAcquisitionVerificationNode | Solicita el nombre de la empresa al usuario (modelo gpt-4o-mini) | `input_node.user_input` | `vav_company.company` |
| 27 | `variable_node_setcompany` | variableNode | Actualiza `company` con la empresa capturada | `vav_company.company` | `company` |
| 28 | `code_get_cct_by_company` | codeExecutionNode | Llama webhook para obtener datos del CCT por empresa | `company` (normalizado) | `code_get_cct_by_company.output` (JSON del CCT o "404" o "204") |
| 29 | `set_cct_content` | variableNode | Asigna respuesta del webhook a `cct` | `code_get_cct_by_company.output` | `cct` |
| 30 | `if_company_not_exists_with_cct_content` | conditionNode | ¿`cct` incluye "404"? | `cct` | TRUE → company not exist / FALSE → identificación |
| 31 | `vav_identificacin_usuario` | variableAcquisitionVerificationNode | Solicita nombre completo y número de empleado (gpt-4o-mini) | `input_node.user_input` | `nombre`, `numero_empleado` |
| 32 | `variable_node_update_useridentified` | variableNode | Establece `userIdentified = "1"` de forma persistente | — | (persistente en `vav_datos_contacto`) |
| 33 | `code_execution_consulta_empleado` | codeExecutionNode | Llama webhook para validar identidad del empleado | `nombre`, `numero_empleado`, `company` | `code_execution_consulta_empleado.output` |
| 34 | `variable_node_actualizar_vav_datos_contacto_consulta_empleado` | variableNode | Actualiza `consulta_empleado` de forma persistente | `code_execution_consulta_empleado.output` | (persistente en `vav_datos_contacto`) |
| 35 | `variable_node_respuesta_consulta_empleado` | variableNode | Inicializa `Respuesta consulta empleado` | `vav_datos_contacto.consulta_empleado` | `Respuesta consulta empleado` |
| 36 | `condition_node_if_undefined` | conditionNode | ¿`Respuesta consulta empleado` incluye "undefined"? | `Respuesta consulta empleado` | TRUE → load LFT (login failed) / FALSE → check EXTERNO |
| 37 | `condition_node_if_externo` | conditionNode | ¿`Respuesta consulta empleado` incluye "EXTERNO"? | `Respuesta consulta empleado` | TRUE → load LFT (login failed) / FALSE → check RICH |
| 38 | `if_rich` | codeExecutionNode | Verifica si la empresa es PRODUCTOS RICH (variantes) | `company` | `if_rich.output` ("TRUE" / "FALSE") |
| 39 | `condition_node_if_rich` | conditionNode | ¿`if_rich.output` incluye "TRUE"? | `if_rich.output` | TRUE → flujo PRODRICH / FALSE → flujo CCT estándar |
| 40 | `load_lft_data` (login failed) | rawDocumentNode | Carga LFT para respuesta de login fallido | — | `load_lft_data.content` |
| 41 | `set_lftdata` (login failed) | variableNode | Asigna LFT a `lft_data` (ruta login failed) | `load_lft_data.content` | `lft_data` |
| 42 | `prompt_node_procedimiento_login_failed` | promptNode | Responde cuando no se encuentran registros del empleado | `lft_data`, historial | `output` |
| 43 | `variable_node_update_respuesta_final` (login failed) | variableNode | Actualiza `respuesta_final` | output del nodo anterior | `respuesta_final` |
| 44 | `prompt_node_procedimiento_company_not_exist` | promptNode | Sugiere empresas similares cuando la empresa no se encuentra | historial | `output` |
| 45 | `variable_node_update_respuesta_final` (company not exist) | variableNode | Actualiza `respuesta_final` | output del nodo anterior | `respuesta_final` |
| 46 | `prompt_larebel_contratos_colectivos` | promptNode | LaRebel con contexto del CCT del empleado | `cct`, historial (5 msgs) | `output` |
| 47 | `variable_node_update_respuesta_final` (CCT) | variableNode | Actualiza `respuesta_final` | output del nodo anterior | `respuesta_final` |
| 48 | `prompt_flujo_prodrich` | promptNode | Flujo especial PRODRICH (trabajadores de PRODUCTOS RICH) | `cct`, historial (5 msgs) | `output` |
| 49 | `variable_node_updaterespuestafinalprodrich` | variableNode | Actualiza `respuesta_final` | output del nodo anterior | `respuesta_final` |
| 50 | `prompt_node_analiza_consulta_resuelta` | promptNode | Evalúa si la consulta fue resuelta (salida: "0" o "1") | `input_node.user_input`, `respuesta_final` | `output` |
| 51 | `variable_node_update_caseresolved` | variableNode | Actualiza `caseResolved` de forma persistente | output del nodo anterior | (persistente en `vav_datos_contacto`) |
| 52 | `limpiar_doble_asteriscos` | codeExecutionNode | Post-procesamiento: reemplaza `**` por `*` en `respuesta_final` | `respuesta_final` | `limpiar_doble_asteriscos.output` |
| 53 | `limpiar_respuestafinal` | variableNode | Actualiza `respuesta_final` con versión limpia | `limpiar_doble_asteriscos.output` | `respuesta_final` |
| 54 | `variable_contactnumber` | variableNode | Inicializa `contactNumber` con "---" | — | `contactNumber` |
| 55 | `prompt_node_analiza_asesoramiento_humano` | promptNode | Detecta si el usuario pidió asesor humano (salida: "0" o "1") | `input_node.user_input`, `respuesta_final` | `output` |
| 56 | `variable_node_asesorhumano` | variableNode | Asigna resultado a `asesorHumano` | output del nodo anterior | `asesorHumano` |
| 57 | `condition_asesorhumano` | conditionNode | ¿`asesorHumano` incluye "1"? | `asesorHumano` | TRUE → VAV contactNumber / FALSE → analytics |
| 58 | `vav_contactnumber` | variableAcquisitionVerificationNode | Solicita número telefónico para llamada del asesor (gpt-4o-mini) | `input_node.user_input`, `input_node.session_id` | `vav_contactnumber.contactNumber` |
| 59 | `variable_node_contactnumber` | variableNode | Actualiza `contactNumber` | `vav_contactnumber.contactNumber` | `contactNumber` |
| 60 | `code_execution_node_analytics_body_request` | codeExecutionNode | Construye el JSON del body para analytics | múltiples variables | `output` (JSON string) |
| 61 | `http_node_analytics_for_test_agent` | httpNode | Envía analytics al webhook n8n (POST) | `output` del nodo anterior | — |
| 62 | `final_output` | output | Envía `respuesta_final` al usuario | `respuesta_final` | — |

---

## 3. Variables Globales

| Variable | Tipo | Valor inicial | Actualizada por | Descripción |
|----------|------|--------------|-----------------|-------------|
| `respuesta_final` | string | "Default" | `update_respuesta_final_*` (varios nodos) | Respuesta final que se envía al usuario |
| `orquestador_value` | string | "DEFAULT" | `set_orquestadorvalue` | Clasificación del orquestador: "Consulta general" o "Contratos colectivos" |
| `company` | string | "null" | `variable_node_setcompany` | Nombre de empresa del usuario (normalizado en mayúsculas sin acentos) |
| `lft_data` | string | "default" | `set_lftdata` (×2) | Contenido del documento LFT_ENERO_2026 |
| `cct` | string | — | `set_cct_content` | Contenido del CCT de la empresa del usuario (o "404" / "204") |
| `tipo_denuncia` | string | — | `set_tipodenuncia` | Clasificación: MUJERES / PROFEDET / SIQAL / LAREBEL |
| `pam_data` | string | — | `set_protocoloatencionmujeresdata` | Chunks del Protocolo de Atención a Mujeres (RAG) |
| `contactNumber` | string | "---" | `variable_node_contactnumber` | Teléfono del usuario para llamada del asesor humano |
| `asesorHumano` | string | — | `variable_node_asesorhumano` | "1" si el usuario pidió asesor humano, "0" si no |
| `userIdentified` | string | — | `variable_node_update_useridentified` | "1" si el usuario se autenticó con nombre y número de empleado |
| `caseResolved` | string | — | `variable_node_update_caseresolved` | "1" si la consulta fue resuelta, "0" si no |
| `Respuesta consulta empleado` | string | — | `variable_node_respuesta_consulta_empleado` | Resultado del webhook de validación de identidad del empleado |

---

## 4. Webhooks Externos

> ⚠️ **Problema de seguridad**: Las URLs están hardcodeadas en los nodos de código. Si este agente se exporta o comparte, quedan expuestas. Se recomienda moverlas a variables de entorno.

| Endpoint | Método | Nodo | Descripción | Body enviado |
|----------|--------|------|-------------|--------------|
| `http://n8n-alb-1431437514.mx-central-1.elb.amazonaws.com/webhook/validate-identity` | POST | `code_execution_consulta_empleado` | Valida identidad del empleado contra base de datos | `{ name, employeeNumber, company }` (normalizados) |
| `http://n8n-alb-1431437514.mx-central-1.elb.amazonaws.com/webhook/validate-company` | POST | `code_get_cct_by_company` | Devuelve datos del CCT de la empresa o vacío si no existe | `{ company }` (normalizado) |
| `http://n8n-alb-1431437514.mx-central-1.elb.amazonaws.com/webhook/snac-analytics-for-test-agent` | POST | `http_node_analytics_for_test_agent` | Registra métricas de la sesión | `{ userIdentified, legalAdviceLead, caseResolved, company, workplace, email, sessionId, phoneNumber, contactNumber, lastMessageDateTime }` |

---

## 5. RAG Documents

| Nombre RAG | Tipo de nodo | Usado en | Descripción |
|------------|-------------|----------|-------------|
| `LFT_ENERO_2026` | rawDocumentNode | `load_lft_data` (×2) | Ley Federal del Trabajo actualizada enero 2026 (5 entradas por página) |
| `PROTOCOLO_ATENCION_MUJERES` | chunkedDocumentRagNode | `load_chunked_protocoloatencionmujeres` | Protocolo de atención a mujeres trabajadoras (top 3 chunks por relevancia) |

---

## 6. Problemas Conocidos

| # | Problema | Severidad | Nodo afectado | Descripción |
|---|---------|-----------|---------------|-------------|
| 1 | URLs internas hardcodeadas | Alta | `code_execution_consulta_empleado`, `code_get_cct_by_company`, `http_node_analytics_for_test_agent` | URLs del load balancer interno de AWS expuestas en el código. Riesgo de exposición si el agente se exporta. |
| 2 | Detección de "404" por string | Media | `if_company_not_exists_with_cct_content` | La condición verifica si `cct` *incluye* la cadena "404". Si un CCT legítimo contiene ese número, el flujo enrutará incorrectamente. Usar un campo separado de status code. |
| 3 | `console.log(undefined)` | Baja | `code_execution_consulta_empleado` | Cuando `data.code === 0` el nodo registra `undefined`. Esto puede causar que `code_execution_consulta_empleado.output` sea vacío o literal "undefined". |
| 4 | Normalización inconsistente en `if_rich` | Media | `if_rich` | El nodo normaliza con `toUpperCase()` en otros lugares, pero `if_rich` compara con `company.toLowerCase()`. Si `company` llega ya en mayúsculas (como lo procesa el resto del sistema), la comparación siempre fallará. |
| 5 | Duplicación masiva de blocos de prompt | Media | Todos los promptNodes | Los blocos de "Escalación a asesor humano", "SNAC Y SU PAPEL CLAVE", "REGLA SUPREMA SOBRE FORMATO" y "Cierre de conversación" están copiados textualmente en 7-8 prompts. Cualquier cambio requiere actualizar todos manualmente. |
| 6 | Inconsistencia de año en Login Failed | Baja | `prompt_node_procedimiento_login_failed` | El prompt original mencionaba "LFT actualizada hasta febrero de 2025" y "conocimiento hasta 2025". Corregido en la versión .md de este repositorio a 2026. |
| 7 | Doble mapeo en Filtro CCT | Baja | `filtro-cct.md` | `CCT-INICIAL-GAMESA-TUXPAN-CFCRL.pdf` aparece mapeado a dos empresas distintas: SABRITAS ORIZABA y GAMESA TUXPAN. Verificar cuál es el archivo correcto para cada empresa. |

---

## 7. Bloques Compartidos

Los siguientes bloques aparecen idénticos en múltiples prompts. Están marcados con `<!-- BLOCO COMPARTILHADO: nombre -->` en los archivos .md de `/prompts/`:

| Bloque | Aparece en |
|--------|-----------|
| `escalacion-asesor-humano` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, company-not-exists, contratos-colectivos |
| `snac-papel-clave` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, company-not-exists, contratos-colectivos |
| `regla-suprema-formato` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, company-not-exists, contratos-colectivos |
| `reglas-salida-numerica` | la-rebel, siqal, profedet, mujeres, flujo-rich, login-failed, contratos-colectivos |
| `cierre-conversacion` | la-rebel, siqal, profedet, mujeres, flujo-rich, company-not-exists |

> **Nota**: El flujo PRODRICH (`flujo-rich.md`) tiene una variante más restrictiva del bloque de formato: prohíbe tanto negritas como itálicas. Esta diferencia es *intencional* — los trabajadores de campo de RICH reciben la respuesta en un canal que no renderiza markdown.
