# PLAN.md — Planes comerciales del servicio

**Versión:** 0.2 (borrador para validar)
**Relacionado con:** `ARCHITECTURE.md`, `DESIGN.md`, `Legal.md`, `../README.md`

> **Todas las cifras son ilustrativas.** Salen de estimaciones de diseño (tokens, volumen) y de supuestos que **debes reemplazar con datos reales** de `llm_usage` y con las tarifas vigentes de tus proveedores (LLM, STT, VPS y Meta). No cotices sin hacerlo.

---

## 1. Idea general

Un mismo sistema, tres planes y complementos (*add-ons*). Lo que cambia por plan o por negocio se guarda en la base de datos y se lee en ejecución, sin despliegues distintos:

1. **Canal de WhatsApp:** Evolution por QR (no oficial) o **API oficial** (con coexistencia si se confirma compatibilidad).
2. **Modelo de IA:** económico o más potente.
3. **Límites de uso:** conversaciones y gasto de LLM al mes, número de recursos.
4. **Funciones:** consolas del humano, retraso de respuesta, resumen, reportes, soporte.
5. **Complementos:** transcripción de notas de voz y otros.

La configuración efectiva es: `defaults del código ← plans.features ← business_settings.config` (ver `ARCHITECTURE.md`, sección 4).

> Diseñado para **cualquier negocio que agende citas** (barberías, consultorios, estudios, etc.). Barbería es la primera vertical.

---

## 2. Planes propuestos

| | **Básico** | **Pro** | **Premium** |
|---|---|---|---|
| **Para quién** | Negocio pequeño que empieza | Negocio con flujo constante | Negocio que no puede perder su número ni la atención |
| **Canal** | Evolution (QR) | Evolution (QR) u oficial | **API oficial** (coexistencia si aplica) |
| **Riesgo de bloqueo del número** | Aceptado por el cliente | Aceptado si elige QR | Mitigado |
| **Modelo de IA** | Económico | Económico, prompt afinado | **Más potente** |
| **Conversaciones/mes (tope)** | ~150 | ~400 | ~1.000 |
| **Recursos (barberos/profesionales)** | 1 | hasta 3 | hasta 10 |
| **Servicios y agenda editables** | ✅ | ✅ | ✅ |
| **Excepciones de agenda** (festivos, vacaciones, bloqueos) | ✅ | ✅ | ✅ |
| **Agendar / cancelar / reagendar** | ✅ | ✅ | ✅ |
| **Recordatorios de cita** | ✅ 1 recordatorio | ✅ configurables | ✅ configurables (+ plantillas si es API oficial) |
| **Retraso de respuesta natural** (configurable) | ✅ | ✅ | ✅ |
| **Traspaso a humano** | ✅ | ✅ | ✅ |
| **Consolas del humano** | App de WhatsApp | App + **Telegram** (avisos y respuesta) | App + Telegram (+ Chatwoot cuando exista) |
| **Memoria / resumen de cliente** | Ventana corta | ✅ Resumen por cliente | ✅ Resumen por cliente |
| **Reportes** | ❌ | Mensual básico | Mensual detallado |
| **Revisión de citas y configuración** | Telegram (comandos) | Telegram / app | Telegram / app |
| **Soporte** | Correo, 48 h | 24 h | Prioritario, 8 h hábiles |
| **Precio mensual de referencia** | **~35 USD** | **~60 USD** | **~120 USD + costos de Meta** |

> Los planes son un punto de partida: cada función se activa o desactiva por negocio en `business_settings`, de modo que puedes ofrecer excepciones sin cambiar el código.

### 2.1 Complementos (*add-ons*)

| Complemento | Qué incluye | Cobro |
|---|---|---|
| **Notas de voz** | Transcripción de audios con tope mensual; se guarda la transcripción y se descarta el audio | Cargo fijo mensual o por consumo (por definir tras medir `llm_usage` con `kind = 'stt'`) |
| **Recursos adicionales** | Más barberos/profesionales sobre el tope del plan | Por recurso |
| **Chatwoot** | Bandeja de agentes (módulo listo, **no se ofrece aún**) | Por definir |
| **Conversaciones extra** | Bolsa adicional sobre el tope | Por bloque |

El complemento de voz es muy necesario en la práctica: se vende como complemento y no como parte de un plan específico, y puede activarse en cualquiera.

### 2.2 Consolas del humano por negocio

Se activan en `business_settings.config.handoff.consoles`:

| Escenario | Configuración |
|---|---|
| Un celular con **dos números** (WhatsApp personal + WhatsApp Business) | `whatsapp_app: true`; Telegram opcional solo para avisos |
| Responder desde otro lugar | `telegram: { enabled: true, can_reply: true }` |
| Integración única | Solo una consola en `true` |
| Varias a la vez | Cualquier combinación; todas escriben el mismo estado de pausa |

---

## 3. Estructura de costos y márgenes (ilustrativo)

### 3.1 Supuestos de costo variable (por negocio al mes)

| Concepto | Básico | Pro | Premium |
|---|---|---|---|
| LLM (según modelo y volumen) | ~3 USD | ~8 USD | ~35–60 USD (modelo potente, a validar) |
| Infraestructura compartida (VPS, respaldos) | ~3 USD | ~3 USD | ~4 USD |
| Transcripción de voz (si lo activa) | Medir | Medir | Medir |
| WhatsApp (Meta) | 0 | 0 u oficial | Tarifa vigente de Meta para Colombia: **por definir** |
| Pasarela de pago / impuestos | A calcular | A calcular | A calcular |

### 3.2 Margen bruto aproximado (sin impuestos ni costos de Meta/voz)

| | Básico | Pro | Premium |
|---|---|---|---|
| Precio | 35 | 60 | 120 |
| Costos variables (estimados) | ~6 | ~11 | ~45–70 |
| **Margen bruto estimado** | **~29 USD (83 %)** | **~49 USD (82 %)** | **~50–75 USD (42–62 %)** |

**Lectura:** Premium es el más sensible. Para protegerlo: tope de gasto de LLM por plan, usar el modelo potente solo donde aporta (el resto con el económico) y trasladar los costos de Meta y de voz.

### 3.3 Cómo fijar el precio de verdad

```
precio_mínimo = (costo_LLM + costo_STT + costo_infra + costo_Meta + comisión_pasarela) / (1 − margen_objetivo)
```

1. Recoge **4 semanas** de datos reales en `llm_usage` por negocio (y en staging con OpenRouter para comparar modelos).
2. Calcula costo medio y percentil 90 por conversación.
3. Define precio con un margen objetivo (p. ej. 70 % o más) y topes que protejan el percentil alto.
4. Revisa cada trimestre: los precios de LLM, STT y Meta cambian.

---

## 4. Implementación técnica de los planes

### 4.1 Modelo de datos (resumen; esquema completo en `ARCHITECTURE.md`)

- `plans`: precio, modelo, modelo de respaldo, tipo de canal, topes (`max_conversations_month`, `max_llm_cost_usd_month`, `max_resources`) y `features` (jsonb).
- `businesses`: `plan_id`, `channel_type`, `channel_config` (secretos cifrados), `status`.
- `business_settings.config` (jsonb): sobrescribe lo del plan (retraso de respuesta, traspaso, recordatorios, consolas).
- `business_addons`: complementos activos con su configuración.

Ejemplo de funciones de un plan:

```json
{
  "reply_delay": { "mode": "human_like" },
  "reminders": { "enabled": true, "max_offsets": 1 },
  "handoff": { "consoles_allowed": ["whatsapp_app"] },
  "summary": false,
  "reports": false
}
```

### 4.2 Lógica en el backend

- **Fábrica de canales:** `get_channel(business)` devuelve `EvolutionAdapter` o `CloudApiAdapter`.
- **Selección de modelo:** `llm_service` usa `plan.llm_model`; el modelo de respaldo está **implementado pero apagado**.
- **Control de uso:** antes de cada llamada se suma `llm_usage` del mes y se compara con los topes.
- **Consolas:** `notify_owner` recorre las consolas activas y permitidas por el plan.
- **Retraso de respuesta:** parámetros de `reply_delay` leídos de la configuración efectiva.
- **Complementos:** `stt_service` solo actúa si `business_addons` tiene `voice_transcription` activo.
- **Estado del negocio:** si `status` no es `trial`/`active`, el bot responde con cortesía y no llama al LLM.

### 4.3 Qué pasa al llegar a los límites

| Situación | Comportamiento recomendado |
|---|---|
| 80 % del tope | Aviso al dueño con opción de subir de plan o comprar bolsa extra. |
| 100 % de conversaciones | Mensaje de cortesía al cliente y aviso al dueño; **o** seguir con modelo económico y cobrar excedente (**decidir**). |
| Tope de gasto de LLM | Degradar al modelo económico o pausar y escalar a humano; nunca dejar el gasto abierto. |
| Falta de pago (`past_due`) | Gracia de ~5 días; luego `suspended`. |
| Suspensión | El bot informa que el servicio no está disponible; los datos se conservan según contrato. |

### 4.4 Cambio de plan

- **Sin cambiar de canal:** inmediato (cambia `plan_id` y se recalculan las funciones).
- **A la API oficial:** proceso controlado: alta/verificación en Meta (WABA), registro o migración del número (evaluar coexistencia), plantillas, pruebas en staging, ventana de cambio, avisar al cliente, actualizar `channel_config` y apagar la sesión QR.
- **Bajar de plan:** al cierre del ciclo, validando que el uso cabe en los límites inferiores.

---

## 5. Implicaciones legales por plan (resumen; detalle en `Legal.md`)

| Tema | Básico | Pro | Premium |
|---|---|---|---|
| Cláusula de riesgo de bloqueo del número | **Obligatoria** | Si elige QR | Reducida |
| Meta/WhatsApp como subencargado | Sí | Sí | Sí (Cloud API) |
| LLM / OpenRouter como subencargado | Sí | Sí | Sí |
| Telegram como subencargado | Si activa | Sí | Sí |
| Proveedor de STT (si activa voz) | Si activa | Si activa | Si activa |
| Cartel con QR y aviso de datos | Sí | Sí | Sí |
| Recordatorios proactivos | Servicio de una cita ya solicitada | Igual | Igual (+ plantillas aprobadas en API oficial) |
| Segmentos sensibles (salud, menores) | No ofrecer sin asesoría | No ofrecer sin asesoría | Solo con asesoría específica |

---

## 6. Operación comercial

### 6.1 Onboarding por plan

| Paso | Básico | Pro | Premium |
|---|---|---|---|
| Datos del negocio: servicios, recursos, horarios, excepciones, tono (ejemplos de mensajes) | ✅ | ✅ | ✅ |
| Contrato SaaS + anexo de transmisión de datos | ✅ | ✅ | ✅ |
| Vincular número (QR) | ✅ | ✅ | — |
| Alta y verificación en Meta (WABA / coexistencia) | — | Opcional | ✅ |
| Cartel con QR de WhatsApp (mensaje prellenado) y política de datos | ✅ | ✅ | ✅ |
| Configurar consolas del humano (WhatsApp app / Telegram) | App | App + Telegram | Según plan |
| Prueba en **staging** (consulta, reserva, cancelación, traspaso, audio si aplica) | ✅ | ✅ | ✅ |
| Capacitación | Guía escrita | Guía + 30 min | Sesión + seguimiento |

### 6.2 Facturación y cobro

- **Inicio (3 a 7 clientes):** cobro manual mensual (transferencia o enlace de pago) con factura electrónica y apoyo de un contador.
- **Después:** cobro recurrente con pasarela y estados automáticos `active → past_due → suspended`.
- Definir moneda (COP o USD), IVA y cómo se cobran los costos de Meta y de voz.
- Periodo de prueba opcional (7 a 14 días) con tope de conversaciones bajo.

### 6.3 Indicadores

- Costo de LLM y de STT por negocio y por conversación (media y percentil 90).
- Conversaciones vs. tope; porcentaje de uso de recursos.
- Tasa de traspaso a humano y categorías más frecuentes (`handoffs`).
- Citas confirmadas, no-shows y efectividad de recordatorios.
- Errores del LLM/canal, mensajes `failed`, desconexiones de instancias.
- Bloqueos de números (clientes con QR).
- Margen real por plan.

---

## 7. Ya definido y diferido

| Tema | Estado |
|---|---|
| Recordatorios | **Desde el MVP**, genéricos para cualquier tipo de cita. |
| Voz | Complemento (no plan específico). |
| Baja de clientes (BAJA) | Contemplado en el esquema; se desarrolla después. |
| Condición de carrera del traspaso | Se revisa al llegar a esa fase. |
| Chatwoot | Módulo listo; no se ofrece ni se despliega. |
| Segundo modelo | Implementado pero apagado; pruebas con OpenRouter antes de producción. |
| Aplicación y/o Telegram para revisar citas y configuración | Por definir; empezar por Telegram. |

---

## 8. Hoja de ruta de planes

1. **MVP (Fases 1–2):** Básico y Pro con QR, tabla `plans` desde el inicio, `business_settings`, `business_addons`, `llm_usage`, recordatorios y complemento de voz.
2. **Fase 3 (primeros clientes):** control de topes y avisos, cobro manual, Telegram como consola.
3. **Fase 4:** `CloudApiAdapter`, onboarding en Meta, plan **Premium**, y evaluación de coexistencia.
4. **Después:** reportes, panel web, cobro recurrente, Chatwoot.

> No prometas Premium hasta tener el `CloudApiAdapter` probado de punta a punta y haber medido el costo real del modelo potente.

---

## 9. Decisiones pendientes

- [ ] Nombres, precios finales y moneda de cobro.
- [ ] Definición exacta de "conversación" para los topes.
- [ ] Qué ocurre al pasar el tope: ¿pausar, degradar el modelo o cobrar excedente?
- [ ] Modelo y proveedor de LLM por plan; prueba comparativa con conversaciones reales.
- [ ] Cobro del complemento de voz (fijo o por consumo) y proveedor de STT.
- [ ] Cobro de los costos de Meta (incluidos con tope o por consumo).
- [ ] Periodo de prueba: duración y límites.
- [ ] Política de reembolsos, permanencia mínima y preaviso de cancelación.
- [ ] Pasarela de pago y facturación electrónica.
- [ ] Si el plan Pro ofrece la API oficial de entrada o solo el QR.
