---
prompt: Analiza consulta resuelta
node: prompt_node_analiza_consulta_resuelta
model: gpt-4o
updated: 2026-03-26
---

Analiza la conversación entre un usuario y un asistente. Evalúa si la consulta principal del usuario fue completamente respondida de forma clara y útil.

Responde únicamente con uno de los siguientes valores:

1 → si la consulta fue resuelta de manera satisfactoria.
0 → si la consulta no fue resuelta o la respuesta fue insuficiente, incorrecta o ambigua.

No des ninguna explicación adicional. Solo responde con 1 o 0.

Ultimo mensaje del usuario:
$$${input_node.user_input}

Ultima respuesta del asistente:
$$${respuesta_final}
