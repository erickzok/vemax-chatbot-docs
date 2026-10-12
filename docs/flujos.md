# Flujos de negocio

[← Volver al README](../README.md)

Todo empieza con un menú de bienvenida de 4 opciones (Mi servicio, Pagos y facturación, Contratar servicio, Soporte técnico). Si el cliente escribe directamente su problema ("hola, no tengo internet"), el classifier lo envía al especialista sin pasar por el menú.

## 1. Soporte técnico ("no tengo internet / no tengo cable")

```mermaid
flowchart TD
    A[Cliente reporta una falla] --> B[Identificar servicio y categoría<br/>luz roja, intermitencia o lentitud,<br/>suspensión, otro]
    B --> C[Pedir DNI y validar formato<br/>descartar DNIs de relleno]
    C --> D[Diagnóstico en vivo vía SmartOLT<br/>estado de la ONU + estado del cliente]
    D -->|Avería general en la zona| AG[Informar la avería general<br/>con imagen y etiqueta]
    D -->|Servicio suspendido por deuda| P[Derivar al flujo de pagos]
    D -->|Pago reportado pero sin servicio| PV[Derivar a un asesor<br/>pago en verificación]
    D -->|ONU con problema| R1[Proponer reinicio remoto<br/>y pedir confirmación]
    R1 -->|Cliente confirma| R2[Reiniciar la ONU] --> R3{¿Volvió el servicio?}
    R3 -->|No| S1[Proponer resincronización<br/>y pedir confirmación] --> S2[Resincronizar] --> S3{¿Volvió?}
    R3 -->|Sí| OK[Cerrar el caso]
    S3 -->|Sí| OK
    S3 -->|No| EV[Pedir desde cuándo, daño visible,<br/>foto y ubicación del equipo]
    EV --> REP[Reporte técnico al equipo<br/>+ etiqueta falla-tecnica-escalada]
```

Reglas en código:
- La remediación remota se marca como **agotada durante 24 h** por DNI: si el cliente vuelve a escribir, no se le hace reiniciar otra vez el equipo y se deriva directo.
- La categoría la infiere el LLM, pero si la omite hay un respaldo con expresiones regulares (por ejemplo, "pagué" → suspensión, "luz roja" → luz roja).
- Si el mismo paso se repite 3 veces sin avanzar, se deriva automáticamente.
- El cliente puede consultar el estado de un ticket con su número.

## 2. Pagos y facturación

```mermaid
flowchart TD
    A[Cliente: ya pagué / quiero pagar / necesito más tiempo] --> B{¿Qué necesita?}
    B -->|Cómo pagar| M[obtener_metodo_pago<br/>cuentas e instrucciones con imagen]
    B -->|Ya pagó| F{¿Envió la foto<br/>del comprobante?}
    F -->|No| PF[Pedir la foto]
    F -->|Sí| V[Visión: ¿pago realizado o pantalla previa?<br/>extraer monto, fecha y hora, método,<br/>número de operación]
    V -->|Sin fecha y hora de la operación| CF[Pedir la constancia real]
    V -->|Comprobante válido| DUP{Deduplicación<br/>método + número de operación}
    DUP -->|Ya registrado| YA[Informar que ya está registrado]
    DUP -->|Nuevo| REG[Registrar el pago reportado]
    REG -->|Servicio suspendido| PP[Promesa de pago con reactivación<br/>48 h de verificación]
    REG -->|Servicio activo| NT[Avisar al equipo para validar]
    B -->|Pide más tiempo sin haber pagado| PR[Prórroga: fecha concreta obligatoria<br/>máximo 4 días, una cada 25 días]
```

Reglas en código:
- Sin foto no hay registro: se cortaron de raíz los "pagos fantasma", por ejemplo un "9k" sobre un televisor interpretado como un pago de S/ 9000.
- Fechas en hora de Perú, nunca en la hora del servidor.
- DNI inexistente: 3 intentos y luego se deriva a un asesor.
- Si el backend no responde al verificar duplicados, el registro sigue (*fail-open*) y se avisa al equipo: nunca se deja al cliente sin respuesta por una caída interna.

## 3. Ventas (contratar servicio)

```mermaid
flowchart TD
    A[Prospecto quiere contratar] --> Z[Elegir zona con botones<br/>3 grupos de cobertura]
    Z --> L[Localidad dentro del grupo<br/>coincidencia difusa con el catálogo]
    L -->|Sin cobertura| NC[Informarlo con amabilidad]
    L -->|Con cobertura| PL[Planes de la zona con ficha e imagen]
    PL --> E[Elige plan] --> D[DNI → validación del nombre del titular]
    D --> DAT[Dirección, referencia, tipo de vivienda,<br/>2 teléfonos de contacto]
    DAT --> OP[Opcionales: correo, fotos de DNI,<br/>recibo y fachada]
    OP --> CONF[Resumen y confirmación explícita]
    CONF --> SV[Solicitud de venta en el backend<br/>+ aviso al equipo comercial]
```

- La cobertura y los canales usan catálogos con coincidencia difusa (difflib y normalización de acentos), no búsqueda vectorial: con nombres propios de zonas, los embeddings fallaban.
- Un sub-router determinista despacha los pasos mecánicos del formulario sin pasar por el LLM.

## 4. Mi servicio (clientes existentes)

- Consultar el plan, el estado del servicio y la fecha de corte con el DNI.
- Ver las facturas pendientes.
- Pedir un cambio de plan (subir o bajar): se confirma y se registra la solicitud con la etiqueta `cambio-plan-solicitado` para el asesor.
- Consultar la grilla de canales de cable (`buscar_canal`).

## 5. Transversales

- **FAQs**: `buscar_faq` sobre Milvus, y como último recurso automático antes de un "no sé".
- **Formato de respuesta**: el cliente puede pedir respuestas en audio (TTS) o volver a texto.
- **Cancelar el caso**: si hay un caso en curso, lo cancela; si no lo hay, se interpreta como una solicitud de baja y se deriva a una persona.
- **Asistencia humana**: en cualquier punto, con el motivo y un resumen del caso.
