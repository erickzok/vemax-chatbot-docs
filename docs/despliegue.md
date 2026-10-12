# Despliegue

[← Volver al README](../README.md)

## Infraestructura

Todo corre en **infraestructura propia** (un VPS administrado por mí), con Docker. No hay dependencias de una nube pública salvo las APIs de modelos.

| Servicio | Tecnología | Rol |
|---|---|---|
| `vemax-chatbot` | Python 3.11, FastAPI, uvicorn (imagen `python:3.11-slim`, Poetry) | Webhook del AgentBot, grafo LangGraph y endpoints `/chat` y `/health` |
| `redis` | Redis 7 con *healthcheck* | Servicio de soporte del bot (el bot espera a que esté sano para arrancar) |
| `wisphub-sync` | NestJS (API) | Endpoints `/bot/*`, consultas y sincronización manual |
| `wisphub-sync-worker` | NestJS (worker) | Crons: sincronización completa, sincronización diaria de OLTs de SmartOLT y resumen diario de asistencia humana |
| PostgreSQL | | Espejo de WispHub, pagos reportados, promesas, averías, solicitudes de venta |
| MongoDB | | Checkpoints de LangGraph (estado de cada conversación), mensajes, telemetría de costo y diagnóstico |
| Milvus | | FAQs vectorizadas |
| MinIO | API compatible con S3 | Audios TTS y medios estáticos (planes, flyers, métodos de pago) |
| Chatwoot | Fork propio: Rails, Sidekiq y Vue | CRM e inbox de WhatsApp |

## Decisiones operativas

- **Un solo proceso de uvicorn, a propósito**: el lock por conversación y el limitador de tokens viven en memoria. Está documentado que, para escalar a varios *workers*, ambos deben pasar a un lock distribuido (Redis o Mongo).
- **Timeout de 30 s** por request (middleware), con reintentos HTTP con espera progresiva hacia Chatwoot y el backend.
- **Configuración por variables de entorno** (pydantic-settings): modelos, límites de tokens por minuto, reintentos e IDs de Chatwoot. Los cambios de comportamiento sensibles tienen un interruptor (por ejemplo, agrupar el texto en el *caption*) para revertirlos sin desplegar.
- **El bot puede desplegarse antes que el backend**: si un endpoint nuevo aún no existe (404), la tool degrada de forma controlada.
- **Separación de secretos**: el bot solo conoce la API key de su backend y la de Chatwoot; las credenciales de WispHub y SmartOLT viven únicamente en el backend.

## Ciclo de desarrollo

- Diseño por módulo con *Spec-Driven Development*: cada módulo (chatwoot, media, graph, menú, mi servicio, pagos, ventas, soporte, faq, tts, usecases) tiene su `spec.md` y su `plan.md`.
- Cada rediseño importante (por ejemplo, la migración a multi-dominio con especialistas aislados) se documentó en un plan por fases: arquitectura objetivo, estado y tools, nodos y router, *standby* y reintentos, y migración.
- `pytest` en local antes de cada despliegue; el despliegue en el servidor lo hago con Docker Compose.
