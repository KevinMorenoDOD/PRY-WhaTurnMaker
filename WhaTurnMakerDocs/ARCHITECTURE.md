# ARCHITECTURE.md — Asistente IA para WhatsApp (gestión de citas)

**Versión:** 3.0
**Alcance:** SaaS multi-negocio para **cualquier negocio que agende citas** (la primera vertical son barberías).
**Stack:** Evolution API (WhatsApp) · FastAPI · PostgreSQL · Redis · ARQ (cola) · Caddy · LLM vía OpenRouter/API directa.

> Documentos relacionados: `DESIGN.md` (decisiones y reglas), `PLAN.md` (planes comerciales), `Legal.md` (aspectos legales), `README.md` (puesta en marcha).
> Lo que está **diferido a propósito** se lista en la sección 11.

---

## 1. Topología

```mermaid
graph TD
    Client[Cliente de WhatsApp] <--> EvoAPI[Evolution API<br/>1 instancia por negocio]

    subgraph VPS / WSL (Docker)
        Caddy[Caddy - HTTPS]
        EvoAPI -->|webhook| API[Backend FastAPI]
        API -->|guarda| PG[(PostgreSQL)]
        API -->|encola IDs| RQ[(Redis cola - AOF, noeviction)]
        RQ --> Worker[Worker ARQ]
        Worker <--> PG
        Worker -->|sendText / presencia| EvoAPI
        EvoAPI <--> PG
        EvoAPI <--> RE[(Redis Evolution - caché)]
        Caddy --> API
        Caddy --> EvoAPI
    end

    Worker <-->|API Key| LLM[OpenRouter / OpenAI / Anthropic]
    Worker -.->|opcional| TG[Telegram - consola del dueño]
    Worker -.->|módulo listo, no desplegado| CW[Chatwoot]
```

### Componentes

| Componente | Responsabilidad |
|---|---|
| **Evolution API** | Un solo despliegue con **una instancia por negocio** (cada una es una sesión de WhatsApp con su número). Entrega webhooks y envía mensajes. |
| **Backend FastAPI** | Recibe webhooks (los valida, filtra y guarda), expone la administración y encola trabajos. No ejecuta el LLM dentro de la petición. |
| **Worker ARQ** | Procesa conversaciones, llama al LLM, programa y envía respuestas, recordatorios y mantenimiento. Mismo código que el backend, otro proceso. |
| **PostgreSQL** | Una instancia, dos bases: `evolution` (la gestiona Evolution) y `app` (datos del negocio). |
| **Redis (cola)** | Broker de ARQ, locks por cliente y debounce. Con persistencia (AOF) y política `noeviction`. |
| **Redis (Evolution)** | Caché de Evolution. Puede expulsar claves. **Separado** del de la cola. |
| **Caddy** | Único punto expuesto (80/443), HTTPS automático. |
| **LLM** | Acceso por OpenRouter durante pruebas; proveedor final por definir antes de producción. |
| **Telegram (opcional)** | Consola del dueño: avisos y revisión de citas/configuración. Activable por negocio. |
| **Chatwoot** | Se deja una interfaz `HumanConsole` lista; **no se despliega**. |
| **n8n** | Opcional y fuera del flujo crítico. |

> **Alternativa a ARQ:** Procrastinate (cola sobre Postgres) elimina el segundo Redis y permite encolar dentro de la misma transacción que guarda el mensaje. Se descartó por tener menos ejemplos con FastAPI; puede reconsiderarse.

---

## 2. Multi-tenant: cómo se identifica el negocio

| Canal | Cómo se resuelve el negocio |
|---|---|
| Evolution (QR) | El webhook de cada instancia apunta a `POST /webhook/{instance}`. El backend busca `businesses.instance_name`. Además valida que el nombre de instancia del payload coincida con el de la URL y usa un secreto por negocio. |
| API oficial (Cloud API) | Se usa el `phone_number_id` del payload, guardado en `businesses.channel_config`. |
| Coexistencia (Cloud API + app Business) | Igual que la API oficial; los mensajes enviados desde la app llegan como *echoes* y se tratan como mensajes del humano. |

**Llave estable:** `businesses.id`, con `phone_number` como identificador natural **único**. El `instance_name` es un atributo editable (si se recrea la sesión, el negocio y sus datos siguen intactos). Servicios, recursos, citas, respaldos y configuración se asocian a `business_id`/número, **nunca a la sesión**.

Refuerzos: toda consulta filtra por `business_id` (repositorios obligan el parámetro), pruebas automáticas de aislamiento y, opcionalmente, *Row Level Security* en Postgres.

---

## 3. Estructura del backend

```
src/
├── config/
│   ├── settings.py            # variables de entorno
│   ├── database.py            # SQLAlchemy async
│   └── llm_config.py          # cliente LLM (principal y de respaldo)
│
├── api/
│   ├── webhook.py             # POST /webhook/{instance}
│   └── admin.py               # negocios, servicios, excepciones, pausa, métricas
│
├── channels/                  # abstracción del canal
│   ├── base.py                # parse_incoming / send_message / set_presence
│   ├── evolution.py           # Evolution API (QR)
│   └── cloud_api.py           # WhatsApp Cloud API (stub hasta Fase 4)
│
├── consoles/                  # consolas del humano (varias pueden estar activas)
│   ├── base.py                # interfaz HumanConsole: notify() / receive_reply()
│   ├── whatsapp_app.py        # el dueño responde desde la app (fromMe / echoes)
│   ├── telegram.py            # opcional por negocio
│   └── chatwoot.py            # módulo listo, desactivado
│
├── workers/
│   ├── settings.py            # WorkerSettings de ARQ (funciones y cron)
│   ├── process_conversation.py
│   ├── send_reply.py
│   ├── notify_owner.py
│   ├── summarize_customer.py
│   └── cron.py                # recordatorios, recuperación, retención, respaldos
│
├── services/
│   ├── message_pipeline.py    # filtrado, idempotencia, chequeos previos
│   ├── context_builder.py     # prompt activo + ejemplos de voz + resumen + historial
│   ├── llm_service.py         # tool calling, reintentos, fallback, registro de uso
│   ├── booking_service.py     # huecos, reservas provisionales, citas
│   ├── reminder_service.py
│   ├── stt_service.py         # notas de voz (complemento)
│   ├── settings_service.py    # configuración efectiva = plan + negocio
│   └── usage_service.py       # topes y consumo
│
├── repositories/              # acceso a datos (siempre con business_id)
├── models/                    # entidades y esquemas Pydantic
└── migrations/                # Alembic
```

Regla de dependencias: `api/workers → services → repositories → models`. Los servicios hablan con `channels/` y `consoles/` solo por sus interfaces.

---

## 4. Configuración por capas

La configuración efectiva de un negocio se calcula así y se valida con Pydantic (con caché corta):

```
defaults del código  ←  plans.features  ←  business_settings.config
```

Ejemplo de `business_settings.config` (JSON):

```json
{
  "reply_delay": {
    "mode": "human_like",
    "read_s": [2, 6],
    "chars_per_second": 25,
    "typing_cap_s": 12,
    "typing_presence": true,
    "split_long_messages": true
  },
  "handoff": {
    "customer_message_variants": ["Dame un momento 🙌", "Un momento y te confirmo"],
    "no_response_timeout_min": 15,
    "pause_hours": 12,
    "consoles": {
      "whatsapp_app": { "enabled": true },
      "telegram":     { "enabled": true, "chat_id": null, "can_reply": true },
      "chatwoot":     { "enabled": false }
    }
  },
  "reminders": { "enabled": true, "offsets_hours": [24, 2] },
  "stale_message_max_age_min": 120,
  "channel": { "type": "evolution_qr", "coexistence": false }
}
```

Esto cubre los casos de un celular con dos números, integración única o varias consolas activas a la vez: cada una se enciende o apaga aquí.

---

## 5. Esquema PostgreSQL (base `app`)

Todos los timestamps son `timestamptz` (UTC). La zona horaria vive en `businesses.timezone`. Los cambios de esquema se hacen solo con Alembic.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- ───────── Planes y negocios ─────────
CREATE TABLE plans (
    id                       text PRIMARY KEY,           -- 'basic' | 'pro' | 'premium'
    name                     text NOT NULL,
    price_usd                numeric(8,2) NOT NULL,
    llm_model                text NOT NULL,
    llm_fallback_model       text,                        -- implementado, apagado por ahora
    channel_type             text NOT NULL CHECK (channel_type IN ('evolution_qr','cloud_api')),
    max_conversations_month  int,
    max_llm_cost_usd_month   numeric(8,2),
    max_resources            int,
    features                 jsonb NOT NULL DEFAULT '{}',
    created_at               timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE businesses (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name             text NOT NULL,
    vertical         text NOT NULL DEFAULT 'general',    -- barberia, clinica, estetica...
    phone_number     text NOT NULL UNIQUE,               -- llave natural estable
    instance_name    text UNIQUE,                        -- sesión de Evolution (editable)
    timezone         text NOT NULL DEFAULT 'America/Bogota',
    plan_id          text REFERENCES plans(id),
    channel_type     text NOT NULL DEFAULT 'evolution_qr'
                     CHECK (channel_type IN ('evolution_qr','cloud_api')),
    channel_config   jsonb,                              -- secretos cifrados / phone_number_id
    owner_notify_phone text,
    status           text NOT NULL DEFAULT 'active'
                     CHECK (status IN ('trial','active','past_due','suspended','cancelled')),
    plan_started_at  timestamptz,
    created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE business_settings (
    business_id  uuid PRIMARY KEY REFERENCES businesses(id) ON DELETE CASCADE,
    config       jsonb NOT NULL DEFAULT '{}',
    updated_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE business_addons (               -- complementos (p. ej. 'voice_transcription')
    business_id  uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    addon        text NOT NULL,
    is_active    boolean NOT NULL DEFAULT true,
    config       jsonb NOT NULL DEFAULT '{}',
    PRIMARY KEY (business_id, addon)
);

-- ───────── Prompts versionados ─────────
CREATE TABLE prompt_versions (
    id           bigserial PRIMARY KEY,
    business_id  uuid REFERENCES businesses(id) ON DELETE CASCADE,   -- NULL = base global
    kind         text NOT NULL CHECK (kind IN ('system','style_examples')),
    version      int  NOT NULL,
    content      text NOT NULL,
    is_active    boolean NOT NULL DEFAULT false,
    created_by   text,
    created_at   timestamptz NOT NULL DEFAULT now(),
    UNIQUE (business_id, kind, version)
);
CREATE UNIQUE INDEX ux_prompt_active ON prompt_versions
    (COALESCE(business_id, '00000000-0000-0000-0000-000000000000'::uuid), kind)
    WHERE is_active;

-- ───────── Catálogo y agenda (editables por cada negocio) ─────────
CREATE TABLE services (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id   uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    name          text NOT NULL,
    description   text,
    duration_min  int  NOT NULL CHECK (duration_min > 0),
    buffer_min    int  NOT NULL DEFAULT 0 CHECK (buffer_min >= 0),
    price         numeric(12,2),
    currency      text NOT NULL DEFAULT 'COP',
    is_active     boolean NOT NULL DEFAULT true,
    sort_order    int NOT NULL DEFAULT 0
);

CREATE TABLE resources (                      -- barbero, profesional, sala, puesto...
    id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id  uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    name         text NOT NULL,
    kind         text NOT NULL DEFAULT 'professional',
    is_active    boolean NOT NULL DEFAULT true
);

CREATE TABLE resource_services (              -- qué recurso presta qué servicio
    resource_id  uuid NOT NULL REFERENCES resources(id) ON DELETE CASCADE,
    service_id   uuid NOT NULL REFERENCES services(id)  ON DELETE CASCADE,
    PRIMARY KEY (resource_id, service_id)
);

CREATE TABLE availability_rules (             -- horario semanal (pausa de almuerzo = 2 filas)
    id          bigserial PRIMARY KEY,
    resource_id uuid NOT NULL REFERENCES resources(id) ON DELETE CASCADE,
    weekday     smallint NOT NULL CHECK (weekday BETWEEN 1 AND 7),   -- 1 = lunes
    start_time  time NOT NULL,
    end_time    time NOT NULL,
    CHECK (end_time > start_time)
);

CREATE TABLE schedule_exceptions (            -- festivos, vacaciones, bloqueos, aperturas extra
    id           bigserial PRIMARY KEY,
    business_id  uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    resource_id  uuid REFERENCES resources(id) ON DELETE CASCADE,    -- NULL = todo el negocio
    kind         text NOT NULL CHECK (kind IN ('closed','extra_open')),
    starts_at    timestamptz NOT NULL,
    ends_at      timestamptz NOT NULL,
    reason       text,
    created_at   timestamptz NOT NULL DEFAULT now(),
    CHECK (ends_at > starts_at)
);

-- ───────── Clientes y mensajería ─────────
CREATE TABLE customers (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id         uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    phone_number        text NOT NULL,
    display_name        text,
    summary             text,
    bot_paused          boolean NOT NULL DEFAULT false,
    paused_until        timestamptz,
    consent_at          timestamptz,
    consent_source      text,              -- 'qr_poster' | 'first_message_notice' | ...
    consent_message_id  text,
    policy_version      text,
    opted_out           boolean NOT NULL DEFAULT false,   -- funcionalidad diferida (BAJA)
    created_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (business_id, phone_number)
);

CREATE TABLE messages (                       -- bandeja de entrada/historial durable
    id             bigserial PRIMARY KEY,
    business_id    uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    customer_id    uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    direction      text NOT NULL CHECK (direction IN ('in','out')),
    role           text NOT NULL CHECK (role IN ('user','assistant','human')),
    wa_message_id  text,
    content_type   text NOT NULL DEFAULT 'text',          -- text | audio | image | ...
    content        text NOT NULL,                         -- para audio: la transcripción
    correlation_id uuid NOT NULL DEFAULT gen_random_uuid(),
    processed_at   timestamptz,                           -- NULL = pendiente de procesar
    created_at     timestamptz NOT NULL DEFAULT now(),
    UNIQUE (business_id, wa_message_id)
);
CREATE INDEX idx_messages_customer_time ON messages (customer_id, created_at DESC);
CREATE INDEX idx_messages_pending ON messages (business_id, customer_id) WHERE processed_at IS NULL;

CREATE TABLE outbound_messages (              -- bandeja de salida con envío programado
    id             bigserial PRIMARY KEY,
    business_id    uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    customer_id    uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    correlation_id uuid NOT NULL,
    content        text NOT NULL,
    status         text NOT NULL DEFAULT 'pending'
                   CHECK (status IN ('pending','sending','sent','failed','cancelled')),
    scheduled_at   timestamptz NOT NULL,
    sent_at        timestamptz,
    wa_message_id  text,
    attempts       int NOT NULL DEFAULT 0,
    last_error     text,
    created_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_outbound_due ON outbound_messages (status, scheduled_at);

-- ───────── Citas ─────────
CREATE TABLE appointments (
    id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id       uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    customer_id       uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    resource_id       uuid NOT NULL REFERENCES resources(id),
    service_id        uuid NOT NULL REFERENCES services(id),
    starts_at         timestamptz NOT NULL,
    ends_at           timestamptz NOT NULL,
    status            text NOT NULL DEFAULT 'held'
                      CHECK (status IN ('held','confirmed','cancelled','expired','completed','no_show')),
    hold_expires_at   timestamptz,                        -- reserva provisional
    reminder_at       timestamptz,                        -- próximo recordatorio
    reminder_sent_at  timestamptz,
    notes             text,
    created_at        timestamptz NOT NULL DEFAULT now(),
    CHECK (ends_at > starts_at),
    EXCLUDE USING gist (
        resource_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    ) WHERE (status IN ('held','confirmed'))
);
CREATE INDEX idx_appt_business_time ON appointments (business_id, starts_at);
CREATE INDEX idx_appt_reminders ON appointments (reminder_at) WHERE status = 'confirmed' AND reminder_sent_at IS NULL;

-- ───────── Traspasos, consumo y auditoría ─────────
CREATE TABLE handoffs (
    id           bigserial PRIMARY KEY,
    business_id  uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    customer_id  uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    category     text NOT NULL,        -- specific_advice | complaint | out_of_scope | urgent | explicit_request
    reason       text,
    created_at   timestamptz NOT NULL DEFAULT now(),
    resolved_at  timestamptz
);

CREATE TABLE llm_usage (
    id            bigserial PRIMARY KEY,
    business_id   uuid NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    customer_id   uuid REFERENCES customers(id) ON DELETE SET NULL,
    kind          text NOT NULL DEFAULT 'chat',        -- chat | summary | stt
    model         text NOT NULL,
    tokens_in     int NOT NULL DEFAULT 0,
    tokens_out    int NOT NULL DEFAULT 0,
    audio_seconds int NOT NULL DEFAULT 0,
    cost_usd      numeric(10,6),
    created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_usage_business_time ON llm_usage (business_id, created_at);

CREATE TABLE audit_log (
    id           bigserial PRIMARY KEY,
    business_id  uuid REFERENCES businesses(id) ON DELETE SET NULL,
    actor        text NOT NULL,                        -- 'owner:telegram', 'admin', 'system'
    action       text NOT NULL,
    detail       jsonb,
    created_at   timestamptz NOT NULL DEFAULT now()
);
```

**Notas de diseño del esquema**
- **Doble reserva:** la restricción `EXCLUDE` es **por recurso** (no por negocio). Cubre `held` y `confirmed`.
- **Reservas provisionales:** Postgres no permite usar `now()` en la restricción, por lo que un cron marca `held → expired` al vencer `hold_expires_at`, y `booking_service` expira las reservas vencidas del recurso dentro de la misma transacción antes de intentar reservar.
- Para negocios con un solo profesional, se crea un único recurso implícito.
- `services`, `resources`, `availability_rules` y `schedule_exceptions` se editan desde el panel/Telegram del negocio; el LLM solo los **lee**.

---

## 6. Flujo del mensaje (extremo a extremo)

```mermaid
sequenceDiagram
    participant C as Cliente
    participant E as Evolution API
    participant A as Backend (webhook)
    participant DB as PostgreSQL
    participant Q as Redis/ARQ
    participant W as Worker
    participant L as LLM

    C->>E: mensaje
    E->>A: webhook
    A->>A: valida secreto, filtra, deduplica
    A->>DB: INSERT messages (processed_at = NULL)
    A->>Q: encola process_conversation (diferido: debounce)
    A-->>E: 200 OK
    Q->>W: process_conversation(customer)
    W->>DB: lock por cliente, lee mensajes pendientes
    W->>W: chequeos: estado, topes, pausa, tipo de mensaje
    W->>L: contexto + herramientas
    L-->>W: tool calls / respuesta
    W->>DB: booking_service (huecos, reserva, cita)
    W->>DB: INSERT outbound_messages (scheduled_at = ahora + retraso)
    W->>Q: encola send_reply (run_at)
    Q->>W: send_reply
    W->>W: ¿pausa? ¿mensajes nuevos?
    W->>E: presencia "escribiendo" + sendText
    W->>DB: sent, wa_message_id, llm_usage
```

**Paso a paso**
1. **Webhook:** valida el secreto, resuelve el negocio, filtra (`fromMe`, grupos, estados, reacciones) y descarta duplicados por `wa_message_id`. Responde 200 al instante; no hace trabajo pesado.
2. **Guardado durable:** el mensaje va a `messages` con `processed_at = NULL`. Esa es la bandeja de entrada. Redis solo transporta IDs.
3. **Encolado y debounce:** `process_conversation` se encola con unos segundos de retraso; si llegan más mensajes del mismo cliente, se difiere de nuevo.
4. **Chequeos previos (código, sin LLM):** estado del negocio (`status`), topes del plan, `bot_paused`, palabras de traspaso, antigüedad del mensaje (`stale_message_max_age_min`), tipo de mensaje (audio → transcripción si hay complemento de voz; imagen u otros → cortesía o traspaso).
5. **Contexto:** prompt activo y ejemplos de voz (`prompt_versions`), resumen del cliente, últimos N mensajes, servicios, fecha y hora local del negocio, excepciones próximas.
6. **LLM con herramientas** (bucle con límite de iteraciones): `list_services`, `check_availability`, `hold_slot`, `book_appointment`, `cancel_appointment`, `reschedule_appointment`, `request_human`. Los parámetros se validan y `customer_id` lo inyecta el backend.
7. **Respuesta a la bandeja de salida** (`outbound_messages`), con `scheduled_at = inicio + max(0, retraso_objetivo − tiempo_ya_transcurrido)`.
8. **Envío (`send_reply`):** toma la fila con cambio atómico `pending → sending` (idempotente), revisa de nuevo pausa y mensajes nuevos, envía presencia "escribiendo" y el texto, guarda el `wa_message_id` **antes** de considerar enviado y marca `sent`.
9. **Posproceso:** `llm_usage`, `processed_at`, actualización del resumen y programación de recordatorios.

**Dos puntos de control de pausa:** antes de llamar al LLM y antes de enviar. Si el dueño tomó la conversación mientras se generaba la respuesta, ésta se cancela.

---

## 7. Asignación de turnos (determinista)

La conversación sirve para entender *qué quiere* el cliente. La asignación la decide `booking_service`:

1. **Cálculo de huecos** por recurso: reglas semanales − excepciones (`closed`) + aperturas extra − citas existentes (`held`/`confirmed`), usando la duración del servicio más `buffer_min`.
2. **Reserva provisional** (`held`, con `hold_expires_at`, p. ej. 10 minutos) al proponer o elegir un hueco.
3. **Confirmación explícita** del cliente → `confirmed`, se calcula `reminder_at`.
4. Si dos personas compiten, la restricción de Postgres rechaza a la segunda y el bot ofrece otras opciones.
5. **Cancelar y reagendar:** solo el propio cliente; reagendar es una transacción (libera y reserva).

---

## 8. Cola (ARQ)

| Tarea | Función |
|---|---|
| `process_conversation` | Agrupa mensajes pendientes, arma contexto, llama al LLM. |
| `send_reply` | Envía con retraso y presencia, idempotente. |
| `notify_owner` | Avisos de traspaso a las consolas activas. |
| `summarize_customer` | Actualiza `customers.summary`. |
| `cron: scan_reminders` (cada minuto) | Encola recordatorios desde `appointments.reminder_at`. |
| `cron: expire_holds` | `held → expired`. |
| `cron: recover_pending` | Reencola mensajes con `processed_at IS NULL` y salidas vencidas si Redis se perdió. |
| `cron: retention_cleanup`, `cron: backups` | Mantenimiento. |

Reglas: **Postgres primero, cola después**; lock por `(business_id, customer_id)` en Redis (si no se obtiene, el job se reencola); reintentos con *backoff* y tiempo límite; los recordatorios nacen de la base de datos (no de jobs a días vista) para sobrevivir a reinicios y reagendamientos.

---

## 9. Despliegue (Docker Compose)

Mismo `docker-compose.yml` en WSL y VPS. **Verifica en la documentación de Evolution API los nombres de variables y la versión; fija versiones, no uses `latest`.**

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["80:80", "443:443"]
    volumes: [./Caddyfile:/etc/caddy/Caddyfile, caddy_data:/data]
    depends_on: [backend, evolution-api]
    restart: unless-stopped

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes: [pg_data:/var/lib/postgresql/data, ./initdb:/docker-entrypoint-initdb.d]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
    restart: unless-stopped

  redis-evolution:
    image: redis:7-alpine
    restart: unless-stopped

  redis-queue:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory-policy", "noeviction"]
    volumes: [redis_queue_data:/data]
    restart: unless-stopped

  evolution-api:
    image: atendai/evolution-api:<VERSION_FIJA>      # verificar imagen y versión
    environment:
      AUTHENTICATION_API_KEY: ${EVOLUTION_API_KEY}
      DATABASE_PROVIDER: postgresql
      DATABASE_CONNECTION_URI: postgresql://evolution:${EVOLUTION_DB_PASSWORD}@postgres:5432/evolution
      CACHE_REDIS_ENABLED: "true"
      CACHE_REDIS_URI: redis://redis-evolution:6379/6
    depends_on:
      postgres: { condition: service_healthy }
      redis-evolution: { condition: service_started }
    restart: unless-stopped

  backend:
    build: ./backend
    command: uvicorn src.main:app --host 0.0.0.0 --port 3000
    env_file: .env
    depends_on:
      postgres: { condition: service_healthy }
      redis-queue: { condition: service_started }
    restart: unless-stopped

  worker:
    build: ./backend
    command: arq src.workers.settings.WorkerSettings
    env_file: .env
    depends_on:
      postgres: { condition: service_healthy }
      redis-queue: { condition: service_started }
    restart: unless-stopped

volumes: { pg_data: {}, redis_queue_data: {}, caddy_data: {} }
```

**Caddyfile**

```
api.tudominio.com {
    handle /webhook/*  { reverse_proxy backend:3000 }
    handle /admin/*    { reverse_proxy backend:3000 }
    handle /health     { reverse_proxy backend:3000 }
}
evo.tudominio.com {
    reverse_proxy evolution-api:8080
}
```

Los puertos 5432, 6379 y 8080 no se publican. Para pruebas, Evolution apunta su webhook a `http://backend:3000/webhook/{instance}` por la red interna.

---

## 10. Respaldos, recursos y operación

**Respaldos dependientes del número, no de la sesión**
- `pg_dump` diario de `app` y `evolution`.
- **Exportación lógica por negocio**: un archivo por `phone_number` y fecha (`backups/<phone_number>/<AAAA-MM-DD>.json.gz.enc`) con servicios, recursos, reglas, excepciones, clientes, citas, prompts y configuración. Si una sesión de Evolution se cae o se pierde, el negocio se restaura con su número y solo hay que volver a escanear el QR.
- Respaldo del volumen de Evolution (sesiones) como ayuda extra, sin depender de él.
- Todos los respaldos **cifrados** y enviados fuera del VPS; restauración probada periódicamente. Los respaldos por número también facilitan la eliminación de datos de un negocio (ver `Legal.md`).

**Recursos:** VPS de 4 GB recomendado (Evolution con varias instancias + Postgres + 2 Redis + backend + worker). Mide el consumo con 5 a 7 instancias activas.

**Observabilidad:**
- `correlation_id` por mensaje, presente en el webhook, el job, las llamadas al LLM y el envío.
- Alerta cuando una instancia de Evolution se desconecta, cuando hay mensajes pendientes antiguos o envíos `failed`.
- `/health`, monitoreo externo y revisión semanal de `llm_usage`.

**Entornos:** WSL (desarrollo), **staging con número de pruebas** antes de producción, y producción. El canal tiene un adaptador falso y un simulador de conversaciones para probar agenda sin WhatsApp real.

---

## 11. Diferido a propósito (decidido, no olvidado)

| Tema | Estado |
|---|---|
| Baja de clientes (opt-out "BAJA") | Contemplado en el esquema (`customers.opted_out`); la funcionalidad se desarrolla más adelante. |
| Condición de carrera del traspaso (eco de mensajes del bot como `fromMe`) | Se resuelve al llegar a esa fase. Mitigación prevista: guardar el `wa_message_id` antes de enviar. |
| Mensajes del humano directo a Evolution (sin pasar por el backend) y su puente con Telegram | Por diseñar al desarrollar las consolas. El backend registrará los mensajes del humano vía el webhook para historial y pausa. |
| Chatwoot | Interfaz `HumanConsole` lista; no se despliega. |
| Segundo modelo (fallback) | Implementado pero apagado; pruebas con OpenRouter antes de producción. |
| Aplicación / chat de Telegram para revisar citas y configuración | Por definir (empezar con Telegram). |
| Coexistencia con la API oficial | Verificar compatibilidad con Evolution y con Meta antes de ofrecerla. |
