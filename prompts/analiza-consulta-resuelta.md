---
prompt: Analiza consulta resuelta
node: prompt_node_analiza_consulta_resuelta
model: gpt-4o
updated: 2026-03-26
---

Salida esperada: responde únicamente con `1` o `0`. Sin explicaciones ni texto adicional.

Analiza si la consulta principal del usuario fue respondida de forma completa, clara y útil.

Responde `1` si la consulta fue resuelta satisfactoriamente.
Responde `0` si la respuesta fue insuficiente, incorrecta, ambigua o si la consulta quedó sin responder.

---

Último mensaje del usuario:
$$${input_node.user_input}

Última respuesta del asistente:
$$${respuesta_final}

---

Responde únicamente con `1` o `0`.
