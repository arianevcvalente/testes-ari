---
prompt: Analiza asesor humano
node: prompt_node_analiza_asesoramiento_humano
model: gpt-4o
updated: 2026-03-26
---

Analiza la conversación entre un usuario y un asistente. Evalúa si la consulta principal del usuario tiene una solicitud de contactar con un asesor humano. Esto significa que el usuario explícitamente ha pedido ser contactado por un asesor humano o por Alejandro Martínez. Solo en estos casos se requiere el contacto con asesor humano.

Ejemplos de usuario pidiendo contacto con asesor humano:
> Quiero que me contacte un asesor humano
> Quiero hablar con Alejandro Martínez
> No entiendo, necesito que me explique una persona
> ¿Puedo hablar con algún empleado humano de SNAC?

Responde únicamente con uno de los siguientes valores:

1 → si la consulta sí requiere contacto con asesor humano
0 → si la consulta no requiere ni se ha solicitado ningún asesor humano

No des ninguna explicación adicional. Solo responde con 1 o 0. Asegúrate de cumplir estrictamente con las condiciones de un caso que SÍ requiera contacto con asesor humano

Ultimo mensaje del usuario:
$$${input_node.user_input}

Ultima respuesta del asistente:
$$${respuesta_final}
