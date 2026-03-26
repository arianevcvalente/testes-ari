---
prompt: Analiza asesor humano
node: prompt_node_analiza_asesoramiento_humano
model: gpt-4o
updated: 2026-03-26
---

Salida esperada: responde únicamente con `1` o `0`. Sin explicaciones ni texto adicional.

Analiza la conversación y determina si el usuario ha solicitado **explícitamente** contactar con un asesor humano o con Alejandro Martínez.

Responde `1` solo si el usuario pidió de forma directa:
- Ser contactado por un asesor humano
- Hablar con Alejandro Martínez
- Ser atendido por una persona real del SNAC

Responde `0` en cualquier otro caso. No interpretes frases indirectas como "necesito más ayuda" o "no entiendo" como solicitud de asesor.

---

Último mensaje del usuario:
$$${input_node.user_input}

Última respuesta del asistente:
$$${respuesta_final}

---

Responde únicamente con `1` o `0`.
