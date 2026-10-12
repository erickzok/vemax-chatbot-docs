# Guardrails y evaluación

[← Volver al README](../README.md)

Principio general: **el LLM interpreta y conversa; el código valida y ejecuta.** Cada vez que un error de producción mostró que el prompt no bastaba, la regla pasó a código y se cubrió con un test.

## 1. Guardrails

| Riesgo | Mitigación |
|---|---|
| Acción irreversible sin consentimiento | Confirmación explícita antes de reiniciar o resincronizar la ONU, cerrar una venta o cambiar de plan. El "sí" se valida en código, no lo interpreta el LLM. |
| Pago registrado sin haberse pagado | La foto del comprobante es obligatoria; además se exige la **fecha y hora de la operación** (una pantalla previa al pago no la tiene). En producción, 282 de 291 registros exitosos la traían. |
| Pago duplicado | Deduplicación por `(método, número de operación)` en el backend (respuesta 409) y lock por conversación para que dos mensajes simultáneos no registren dos veces. |
| Bucles sin salida | Contador de intentos por paso: 3 repeticiones sin avance → derivación automática. La remediación remota se bloquea 24 h por DNI. |
| Derivar a un humano sin necesidad | `verificar_derivacion`: el classifier contrasta la decisión; si el pedido es de un dominio de negocio, se reintenta una vez con el especialista correcto. |
| El especialista rechaza un tema que no es suyo | `dominio_fallback` detecta el rechazo y reenruta en el mismo turno (regex → modelo económico → modelo de razonamiento solo si discrepan). |
| "No sé" del modelo | `faq_fallback` consulta las FAQs antes de responder; si tampoco hay respuesta, deriva. |
| DNI falso o de relleno | Validación de formato (7 a 12 dígitos), detección de valores de relleno y verificación de existencia en el backend. |
| El bot habla encima de un asesor | Silencio cuando la conversación tiene un asesor asignado o `bot_paused`; autoasignación cuando un asesor escribe; filtro de ecos del propio bot. |
| Historial corrupto | Reparación de `tool_calls` huérfanos y ventanas que siempre empiezan en un mensaje del cliente (si no, OpenAI responde 400). |
| Ráfagas y límites de la API | Limitador propio de tokens por minuto, al 80 % del límite de la cuenta, y reintentos con espera del SDK. Origen: un pico real de 292 010 tokens/min contra un límite de 200 000. |
| Audio ininteligible o alucinado | Juez LLM que descarta muletillas y textos que Whisper inventa sobre el silencio; en ese caso se le pide al cliente que repita. |
| Mensajes fuera de horario | La derivación informa el horario de atención y deja la conversación etiquetada para el equipo. |

## 2. Evaluación

### Tests automatizados

- **44 archivos y 707 casos con pytest** (`pytest-asyncio`), con dependencias externas simuladas. Un *fixture* automático en `conftest` impide que cualquier test llame a OpenAI, así que la suite es determinista y corre sin red.
- Cobertura por componente:
  - **Grafo**: classifier, router de entrada, verificación de dominio, ToolNodes por especialista, `dominio_fallback`, `faq_fallback`, verificación de derivación.
  - **Especialistas**: soporte técnico, pagos, ventas, mi servicio e historial.
  - **Tools**: pagos (incluidos casos excepcionales y cobertura de pago), soporte técnico, fallas, leads, planes, cambio de plan, canales, métodos de pago, menú, cancelación, asistencia humana y DNI de relleno.
  - **Infraestructura**: webhook y cliente de Chatwoot, anti-eco, reintentos HTTP, limitador TPM, presupuesto de tokens, uso del caché de prompts, *standby*, remediación agotada, prompts de audio y juez de audio.

### Regresión sobre casos reales

Cada incidente de producción queda documentado junto al código con el hilo original y se valida repitiendo la llamada al modelo 5 veces antes y después del cambio. Ejemplo: "¿Puedo cancelar lo del internet por Yape?" se interpretaba como una solicitud de baja 5 de 5 veces; tras pasarle al especialista la intención interpretada por el classifier, llamó a `obtener_metodo_pago` 5 de 5 veces.

### Observabilidad

- **Costo por turno** (`llm_usage` en MongoDB): modelo, tokens de entrada, de caché y de salida, número de llamadas y costo en USD con la tabla de precios por modelo, incluido el descuento del caché de prompts. Agregados por día y por cliente.
- **Diagnóstico por turno** (con TTL): mensaje de entrada, cada tool con sus argumentos y su resultado, la respuesta final y una alerta automática de "tool saltada" cuando el modelo responde algo que debió salir de una tool.
- **Scripts**: medición de latencia del classifier, consumo TPM real, visor de conversaciones y reparación de hilos con `tool_calls` huérfanos.
- **Envío por turno**: se registra cuántos mensajes de WhatsApp generó cada turno para medir el ahorro de agrupar el texto en el *caption* de la imagen (Meta cobra cada mensaje de servicio).
