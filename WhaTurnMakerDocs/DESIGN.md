# DESIGN.md — Decisiones de diseño y reglas de comportamiento

**Versión:** 2.0 · Complemento de `ARCHITECTURE.md`. Aquí se documenta **qué se decidió, por qué y qué reglas debe cumplir el sistema** para ser confiable, honesto y rentable.

---

## 1. Decisiones clave

| # | Decisión | Motivo |
|---|---|---|
| D1 | **PostgreSQL desde el inicio** | Evolution ya requiere base relacional; concurrencia y restricciones avanzadas (`EXCLUDE`). |
| D2 | **Una instancia de Postgres, dos bases** (`evolution`, `app`) | Aislamiento de datos y credenciales con un solo servicio. |
| D3 | **Multi-tenant por `business_id`**, una instancia de Evolution por negocio | Un despliegue sirve a muchos negocios. El negocio se identifica por instancia (o `phone_number_id` en API oficial). |
| D4 | **Llave estable = `business_id` / número**, no la sesión | La sesión de WhatsApp puede caer en cualquier momento; los datos y respaldos no deben depender de ella. |
| D5 | **Backend FastAPI + worker ARQ** | Menor RAM que Java; async; trabajos diferidos (debounce, retraso de envío, recordatorios). |
| D6 | **Dos Redis**: caché de Evolution y cola (AOF, `noeviction`) | Evita que la expulsión de memoria borre trabajos. |
| D7 | **Bandeja de entrada y de salida en Postgres** (`messages`, `outbound_messages`) | Durabilidad e idempotencia; Redis solo transporta IDs. |
| D8 | **El LLM entiende, el código decide** | Los turnos, precios y reglas los resuelve código determinista. |
| D9 | **Catálogo y agenda editables por negocio** (`services`, `resources`, `schedule_exceptions`) | Cada negocio es distinto y cambia a diario. Diseño genérico para cualquier vertical de citas. |
| D10 | **Planes y funciones en BD** (`plans` + `business_settings` + `business_addons`) | Todo activable por plan y por negocio sin redespliegues. |
| D11 | **Canal desacoplado** (`channels/`) y **consolas del humano desacopladas** (`consoles/`) | QR hoy, API oficial/coexistencia mañana; WhatsApp app, Telegram y Chatwoot intercambiables. |
| D12 | **Prompts versionados en BD** (`prompt_versions`) | Rollback y trazabilidad; el prompt activo se asigna por negocio. |
| D13 | **Voz como complemento (*add-on*)**, no como plan | Es muy necesaria; se factura y mide aparte. |
| D14 | **Respaldos por número de recepción** | Restaurar un negocio sin depender de la sesión. |
| D15 | **Caddy como único punto expuesto** | HTTPS automático y superficie mínima. |
| D16 | **LLM por OpenRouter en pruebas**; segundo modelo implementado pero apagado | Flexibilidad para comparar modelos antes de producción. |

---

## 2. Principios de diseño

1. **El LLM redacta y entiende; el código decide.** Disponibilidad, validaciones y escrituras son deterministas. El modelo nunca escribe SQL ni inventa horarios o precios.
2. **La base de datos es la última defensa.** No-solapamiento de citas y unicidad de mensajes se garantizan con restricciones.
3. **Todo evento puede llegar repetido, tarde o desordenado.** Idempotencia en entrada y salida.
4. **Persistir primero, encolar después.** Si Redis se pierde, se recupera desde Postgres.
5. **Medir antes de cobrar.** Cada llamada al LLM (y a STT) se registra por negocio.
6. **Trato cercano, pero honesto.** El asistente puede sonar como el negocio, sin fingir ser una persona (ver sección 5).
7. **Si el bot duda, escala a un humano.**
8. **Genérico por diseño.** Barbería es la primera vertical; el modelo habla de *negocio, recurso, servicio y cita*.

---

## 3. Pipeline de mensajes

### 3.1 Entrada
- **Validación:** secreto del webhook y coincidencia de instancia (URL vs. payload).
- **Filtrado:** ignorar `fromMe` (salvo para detectar al humano), grupos, estados, reacciones y eventos no relevantes.
- **Idempotencia:** `UNIQUE (business_id, wa_message_id)`; si ya existe, se descarta.
- **Mensajes obsoletos:** si llegan tras una reconexión y superan `stale_message_max_age_min`, no se responden como nuevos (se registra y, según el caso, se avisa al dueño).
- **Casos de borde:** mensajes editados o borrados, respuestas citadas, ráfagas y orden de llegada.

### 3.2 Debounce y orden por cliente
- Al llegar un mensaje se encola `process_conversation` con unos segundos de retraso; si llegan más, se difiere de nuevo y se procesan **juntos**.
- Lock en Redis por `(business_id, customer_id)`: una sola ejecución a la vez por cliente. Si no se obtiene, el job se reencola.

### 3.3 Chequeos previos (sin LLM)
Estado del negocio (`trial`/`active` o respuesta de cortesía), topes del plan, `bot_paused`, palabras de traspaso, antigüedad y tipo de mensaje.

### 3.4 Salida con retraso configurable
- El retraso busca naturalidad, no lentitud: **lectura** (2 a 6 s) + **escritura** proporcional al largo del texto (≈25 caracteres/segundo, con tope), con variación aleatoria y presencia "escribiendo…".
- Se descuenta lo que ya tardó el LLM: `espera = max(0, objetivo − transcurrido)`.
- Mensajes largos se parten en 2 o 3 con pausas cortas.
- No se usan esperas de minutos en el flujo de agenda: el cliente se impacienta y se cruzan mensajes. Si un negocio lo quiere, es configurable (`reply_delay`) y por defecto no se activa.
- Los mensajes del **humano no llevan retraso artificial**.
- **Salida idempotente:** `pending → sending` atómico; un reintento no duplica el envío. Si Evolution falla, reintenta con *backoff*; si no se recupera, queda `failed` y se avisa al dueño.

### 3.5 Dos puntos de control de pausa
Antes de llamar al LLM y antes de enviar. Si el humano tomó la conversación entretanto, la respuesta se cancela.

---

## 4. Agenda y recordatorios

### 4.1 Asignación de turnos (determinista)
- **Recursos:** un negocio tiene uno o varios (barberos, profesionales, salas). La restricción anti-solapamiento es **por recurso**.
- **Huecos:** reglas semanales − excepciones (`closed`) + aperturas extra − citas activas; duración del servicio más `buffer_min`.
- **Reserva provisional** (`held`, ~10 minutos) al proponer un hueco; **confirmación explícita** del cliente antes de pasar a `confirmed`.
- Un cron expira las reservas vencidas; `booking_service` también las expira en la misma transacción antes de reservar.
- **Zonas horarias:** UTC en base, conversión con `businesses.timezone`; se inyecta fecha y hora actual local al prompt.
- **Autorización:** un cliente solo ve y modifica **sus** citas; el `customer_id` lo inyecta el backend.

### 4.2 Datos editables por el negocio
`services` (duración, precio, buffer, activo), `resources`, `availability_rules`, `schedule_exceptions` (festivos, vacaciones, bloqueos puntuales). Se editan desde el panel o Telegram y cada cambio queda en `audit_log`. El LLM solo lee estos datos.

### 4.3 Herramientas expuestas al LLM

| Herramienta | Función |
|---|---|
| `list_services()` | Servicios activos con duración y precio. |
| `check_availability(service, date, resource?)` | Huecos libres calculados por código. |
| `hold_slot(service, resource, start)` | Reserva provisional. |
| `book_appointment(hold_id)` | Confirma tras aceptación explícita. |
| `cancel_appointment(id)` | Cancela una cita propia. |
| `reschedule_appointment(id, ...)` | Cambia la cita en una transacción. |
| `request_human(category, reason)` | Activa el traspaso. |

### 4.4 Recordatorios (desde el inicio)
- Se calculan al confirmar la cita (`reminder_at`, según `reminders.offsets_hours`) y los dispara un cron cada minuto desde la base de datos (sobreviven a reinicios y se ajustan al reagendar).
- La respuesta del cliente a un recordatorio ("sí confirmo", "cancelo") entra al flujo normal.
- Con **QR** no hay plantillas; con la **API oficial**, fuera de la ventana de 24 h se necesita plantilla aprobada y opt-in. Un recordatorio de una cita ya solicitada es distinto de la publicidad (ver `Legal.md`).

---

## 5. Voz del asistente y honestidad

- El asistente habla en primera persona con el **estilo del negocio** (ejemplos reales de mensajes del dueño en `prompt_versions` tipo `style_examples`) y puede tener nombre propio.
- El cartel con QR, el perfil y el aviso de datos indican que es un **asistente virtual**.
- **Regla firme:** si el cliente pregunta en serio si habla con una persona o con un bot, el asistente responde con honestidad (con naturalidad y ofreciendo pasar con una persona). Nunca afirma ser humano ni inventa una vida personal.
- Alcance limitado al negocio: no es un asistente de propósito general (también lo exigen los términos de WhatsApp Business).
- Prompts versionados: un único prompt activo por negocio y tipo; cambiar de versión es cambiar el registro activo. Se conserva el historial y un conjunto de conversaciones de referencia para detectar regresiones.

---

## 6. Traspaso a humano

### 6.1 Disparadores (en capas)
1. **Reglas deterministas:** palabras como "humano", "asesor", "persona", "queja", "reclamo".
2. **LLM** con `request_human(category, reason)`; categorías cerradas: `specific_advice`, `complaint`, `out_of_scope`, `urgent`, `explicit_request`.
3. **Salvaguardas en código:** dos o tres turnos sin resolver, o falta de información.
4. **Eco del humano:** un mensaje `fromMe` (o *echo* en coexistencia) que el backend no envió pausa el bot de ese cliente.

### 6.2 Qué ocurre
- El bot envía un mensaje breve y configurable (p. ej. *"Dame un momento 🙌"*, con variantes), marca `bot_paused`, registra el traspaso en `handoffs` y avisa por las **consolas activas**.
- Si nadie responde en `no_response_timeout_min`, el bot avisa al cliente de la demora (una sola vez). Fuera de horario indica que se verá a primera hora.
- Reanudación: vence `paused_until`, comando del dueño o administración. El resumen incluye lo que habló el humano.

### 6.3 Consolas (varias pueden estar activas)
| Consola | Estado |
|---|---|
| App de WhatsApp / WhatsApp Business del dueño | Activa por defecto (eventos `fromMe`/*echo*). |
| Telegram | Opcional por negocio y por plan: solo avisos, o avisos + respuesta. |
| Chatwoot | Interfaz lista; **no se despliega**. |

Los mensajes del humano se envían **de inmediato** (sin retraso) por Evolution y el webhook alimenta el estado de pausa y el historial.

---

## 7. Notas de voz (complemento)

- Complemento por negocio (`business_addons: voice_transcription`), con consumo propio en `llm_usage` (`kind = 'stt'`, `audio_seconds`).
- Se transcribe y se guarda **la transcripción**; el archivo de audio se descarta tras procesarlo (minimización de datos).
- Si el negocio no tiene el complemento, el bot responde con cortesía que por ahora no puede escuchar audios y ofrece texto o traspaso.

---

## 8. Seguridad

- Solo Caddy expone puertos. Postgres, Redis y Evolution quedan en la red interna.
- Webhook con secreto compartido (por negocio) y verificación de instancia; API key en Evolution; `/admin` autenticado.
- Telegram: solo el `chat_id` autorizado del dueño; secreto del webhook de Telegram.
- Credenciales distintas por base; secretos del canal cifrados.
- **Prompt injection:** el contenido del cliente es dato no confiable; el modelo solo actúa por herramientas con parámetros validados y límites por cliente.
- **Aislamiento multi-tenant:** `business_id` obligatorio en repositorios; pruebas automáticas de fuga entre negocios.
- Límite de gasto diario/mensual por negocio ante bucles o abuso.

---

## 9. Resiliencia

- Reintentos con *backoff* y *timeouts* en LLM y Evolution.
- Modelo de respaldo: **implementado, apagado** por ahora.
- Mensaje de contingencia si el LLM falla y cola de fallidos revisable.
- Recuperación: cron que reencola mensajes sin `processed_at` y salidas vencidas.
- Alerta si una instancia de Evolution se desconecta; el negocio puede restaurarse por número desde su respaldo.

---

## 10. Pruebas y entornos

- Adaptador de canal falso + simulador de conversaciones (agenda, cancelación, traspaso, ráfagas, duplicados, mensajes tardíos).
- Conjunto de conversaciones de referencia para comparar prompts y modelos (OpenRouter).
- Pruebas prioritarias: idempotencia, doble reserva, zonas horarias, reservas provisionales, aislamiento entre negocios, herramientas con parámetros inválidos.
- **Staging con número de pruebas** antes de cualquier cliente real.

---

## 11. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Evolution por QR es no oficial: bloqueo del número | Alto | Aceptación escrita del cliente; número dedicado; canal desacoplado; plan Premium con API oficial. |
| Caída de sesión de Evolution | Alto | Alerta de desconexión; respaldo por número; reescaneo del QR. |
| Costo de tokens/voz crece | Medio | Resumen + ventana corta; topes por plan; `llm_usage`; complemento de voz medido aparte. |
| Alucinaciones (precios, horarios) | Medio | Datos solo desde BD; confirmación explícita; traspaso ante duda. |
| Percepción de engaño | Medio | Cartel/perfil que declaran asistente virtual; respuesta honesta si preguntan. |
| Pérdida de Redis | Medio | Persistencia primero en Postgres; cron de recuperación. |
| VPS caído o disco lleno | Alto | Healthchecks, alertas, respaldos externos probados. |
| Cambios de versión en Evolution/OpenRouter | Medio | Fijar versiones; probar en staging. |
| Licencias (Evolution, n8n) | Bajo/Medio | Revisar antes de comercializar. |

---

## 12. Costos (estimación orientativa)

> Cifras aproximadas; verifica precios vigentes de cada proveedor y reemplázalas con datos de `llm_usage`.

**Supuestos por negocio:** ~8 clientes/día, ~20 respuestas del bot por cliente (≈4.800/mes), ~3.000 tokens de entrada y ~200 de salida por respuesta.

| Concepto | Estimación mensual |
|---|---|
| LLM económico | ~2–6 USD por negocio |
| LLM más potente | 35–60 USD o más por negocio (a validar) |
| Transcripción de voz | Por consumo; **medir** |
| VPS 4 GB compartido | ~10–25 USD en total |
| WhatsApp Cloud API (Meta) | Según tarifa vigente para Colombia; **por definir** |

Detalle de planes y márgenes en `PLAN.md`.

---

## 13. Plan por fases

**Fase 0 — Entorno (WSL):** Compose con Postgres, 2 Redis, Evolution, backend y worker; migraciones; adaptador de canal falso.

**Fase 1 — MVP funcional:** webhook (validación, filtrado, idempotencia), bandejas de entrada/salida, `process_conversation`, LLM con herramientas, `services`/`resources`/`availability_rules`, reservas provisionales y citas, envío con retraso, `llm_usage`, prompts versionados, **recordatorios**.

**Fase 2 — Robustez y operación:** debounce y locks completos, cancelar/reagendar, traspaso (WhatsApp app + Telegram), excepciones de agenda editables, resumen del cliente, complemento de voz, Caddy, respaldos por número, alertas.

**Fase 3 — Producción con 3 a 7 negocios:** VPS de 4 GB, staging aprobado, onboarding por negocio, panel/Telegram de administración, contrato y anexo de datos firmados.

**Fase 4 — Evolución:** API oficial/coexistencia (plan Premium), baja de clientes (BAJA), refinar condición de carrera del traspaso, modelo de respaldo operativo, Chatwoot si se necesita, panel web.

---

## 14. Diferido a propósito

| Tema | Estado |
|---|---|
| Baja de clientes (BAJA) | Esquema listo (`opted_out`); se desarrolla más adelante. |
| Condición de carrera del traspaso | Se revisa al llegar a esa fase (guardar `wa_message_id` antes de enviar). |
| Puente Telegram ↔ Evolution para mensajes del humano | Por diseñar. |
| Chatwoot | Módulo listo, no desplegado. |
| Segundo modelo | Implementado, apagado. |
| App/Telegram de administración | Por definir (empezar con Telegram). |

---

## 15. Pendientes por definir

- [ ] Proveedor y modelo de LLM definitivos (tras pruebas con OpenRouter) y proveedor de STT.
- [ ] Ventana de debounce, tamaño del historial y `stale_message_max_age_min`.
- [ ] Política de retención final (6 o 12 meses) y de respaldos.
- [ ] Definición exacta de "conversación" para los topes del plan.
- [ ] Dominio y subdominios para Caddy; proveedor de respaldos externos.
- [ ] Alcance del panel/Telegram de administración (qué se edita y quién).
