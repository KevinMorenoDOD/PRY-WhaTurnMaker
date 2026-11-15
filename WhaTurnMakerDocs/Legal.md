# Legal.md — Aspectos legales del SaaS de asistente IA para WhatsApp

**Versión:** 2.0 · **Fecha de revisión:** 11 de noviembre de 2026
**Jurisdicción de referencia:** Colombia
**Documentos relacionados:** `ARCHITECTURE.md`, `DESIGN.md`, `PLAN.md`, `README.md`

> **Aviso importante:** este documento es una guía de trabajo para organizar el proyecto. **No es asesoría legal** y no reemplaza a un abogado. Las normas y las políticas de las plataformas cambian; cada punto marcado con ⚠️ debe validarse con un profesional (idealmente especializado en protección de datos y contratos de tecnología) antes de vender. Al final hay una lista de fuentes consultadas.

---

## 1. Resumen ejecutivo

| Tema | Qué implica | Prioridad |
|---|---|---|
| **Protección de datos (Ley 1581 de 2012)** | Almacenas teléfonos y conversaciones: son datos personales. El negocio es *Responsable*; tú, *Encargado*. | 🔴 Crítica |
| **Contrato de transmisión de datos** | Debe existir entre cada negocio y tú, con alcance, instrucciones, subencargados y obligaciones. | 🔴 Crítica |
| **Autorización del titular (cartel con QR)** | Se obtiene antes del primer mensaje con un cartel y un mensaje prellenado, con un respaldo en el primer contacto. | 🔴 Crítica |
| **Proveedores en el exterior** (LLM, OpenRouter, Telegram, STT, hosting) | Transmisión internacional de datos: contrato y revisión de términos. | 🔴 Crítica |
| **Términos de WhatsApp / Meta** | Evolution por QR no es oficial: riesgo de bloqueo. La API oficial restringe bots de propósito general (el tuyo, de citas, encaja). | 🔴 Crítica |
| **Honestidad del asistente** | Declarar que es un asistente virtual y responder con verdad si preguntan en serio. | 🟠 Alta |
| **Contrato SaaS con tus clientes** | Responsabilidades, riesgo del canal, SLA, retención, terminación. | 🟠 Alta |
| **Notas de voz** | Tratamiento del audio y minimización. | 🟠 Alta |
| **Respaldos** | Cifrado, retención y eliminación por negocio. | 🟠 Alta |
| **Recordatorios y publicidad** | Distinguir recordatorio de cita de mercadeo. | 🟠 Alta |
| **Datos sensibles y menores** | Negocios de salud o con menores requieren asesoría específica. | 🟠 Alta |
| **Formalización y tributario** | Persona natural o SAS, facturación electrónica, IVA. | 🟡 Media |
| **Propiedad intelectual y licencias** | Revisar licencias de Evolution API, n8n y dependencias. | 🟡 Media |

---

## 2. Roles según la Ley 1581 de 2012

| Rol | Quién es en tu proyecto | Qué hace |
|---|---|---|
| **Titular** | El cliente final que escribe por WhatsApp | Dueño de sus datos y de sus derechos. |
| **Responsable** | **El negocio** (barbería, clínica, etc.) | Decide para qué se usan los datos y debe obtener la autorización. |
| **Encargado** | **Tú** (proveedor del SaaS) | Trata los datos por cuenta del negocio, siguiendo sus instrucciones. |
| **Subencargados** (en la práctica) | Ver tabla 3 | Terceros que tocan datos para prestar el servicio. |

**Consecuencias**
- El negocio decide la finalidad (agendar y atender). **No puedes usar los datos para fines propios** (entrenar modelos, mercadeo propio, venta de bases, analítica para otros) sin base legal y autorización.
- La ley fija deberes distintos para el Responsable (art. 17) y el Encargado (art. 18), y ambos pueden ser sancionados por la SIC.
- La SIC ha señalado en conceptos que los sub-encargados también quedan sujetos a la ley; tu contrato con el negocio debe **declarar expresamente** a los subencargados. ⚠️

---

## 3. Mapa de subencargados y datos que reciben

| Subencargado | Qué datos recibe | Ubicación típica | Nota |
|---|---|---|---|
| **Proveedor de LLM** (y **OpenRouter** si se usa como intermediario) | Texto de la conversación necesario para responder | Exterior | OpenRouter enruta a terceros: revisar su política de retención y de enrutamiento antes de producción. |
| **Meta / WhatsApp** | Mensajes del canal | Exterior | Subencargado inherente al canal. |
| **Telegram** (opcional) | Avisos y respuestas del dueño: nombre, parte del teléfono, resumen | Exterior | Los chats de bots no tienen cifrado de extremo a extremo; minimizar. |
| **Proveedor de transcripción (STT)** (si hay voz) | Audio de la nota de voz | Exterior | Ver sección 8. |
| **Proveedor de VPS / hosting** | Toda la base de datos | Según región elegida | Documentar región. |
| **Almacenamiento de respaldos** | Respaldos cifrados | Según proveedor | Cifrado obligatorio. |
| **Chatwoot** (futuro) | Conversaciones | Según despliegue | Módulo listo, no desplegado. |
| **Evolution API** | — | Auto-alojado | Es software tuyo en tu VPS, no un tercero; revisar su licencia. |

La lista se **versiona** y debe notificarse al negocio cuando cambie. Cada vez que se active o desactive una consola (Telegram, Chatwoot) o un complemento (voz), actualiza el anexo del contrato. ⚠️

---

## 4. Contrato de transmisión de datos personales

Cuando el negocio te permite tratar datos por su cuenta, la normativa reglamentaria (Decreto 1377 de 2013, compilado en el Decreto 1074 de 2015) exige un **contrato de transmisión** que señale el alcance del tratamiento.

**Contenido mínimo recomendado**
1. Partes y roles (Responsable / Encargado).
2. **Finalidad y alcance:** agendar citas, recordatorios y atención por WhatsApp; nada más.
3. Datos tratados (teléfono, nombre, mensajes, transcripciones, citas) y categorías de titulares.
4. Obligaciones del Encargado: tratar según los principios de la ley, salvaguardar la seguridad y guardar confidencialidad.
5. **Instrucciones del Responsable** y prohibición de uso propio.
6. **Subencargados autorizados** (tabla 3) y obligación de notificar cambios.
7. **Transmisión internacional:** países y proveedores (sección 6).
8. **Medidas de seguridad** mínimas (sección 11).
9. **Incidentes de seguridad:** plazo para avisar (recomendado: de inmediato, máximo 24–72 h desde la detección). ⚠️
10. **Atención de derechos de los titulares** (sección 9).
11. **Retención, respaldos y eliminación/devolución** al terminar (incluidos los respaldos por número).
12. Auditoría razonable, responsabilidad y ley aplicable.

> Inclúyelo como anexo del contrato SaaS (sección 14). ⚠️ Pide a un abogado que lo redacte o revise.

---

## 5. Autorización del titular: cartel con QR y primer contacto

### 5.1 Qué exige la norma (resumen)
- La autorización debe pedirse **a más tardar al momento de recolectar** los datos, informando qué datos se recolectan y para qué.
- Puede darse **por escrito, de forma oral o mediante conductas inequívocas**; **el silencio nunca es conducta inequívoca**.
- El Responsable debe **conservar prueba** de la autorización.
- El aviso y la política pueden difundirse por cualquier medio (documento, formato electrónico, verbal u otra tecnología).

### 5.2 Estrategia en capas

1. **Cartel con QR en el local** (mostrador, vitrina, redes del negocio). El QR abre WhatsApp con un **mensaje prellenado**, por ejemplo:
   `https://wa.me/57XXXXXXXXXX?text=Hola%2C%20acepto%20la%20pol%C3%ADtica%20de%20datos%20y%20quiero%20agendar%20una%20cita`
   El titular lee el aviso **antes** de recolectarse cualquier dato y luego realiza un acto voluntario (escanear y enviar). El primer mensaje queda como prueba.
2. **Perfil de WhatsApp Business** con enlace a la política (respaldo; no sustituye al cartel).
3. **Respaldo en el primer contacto:** si el primer mensaje del cliente **no** coincide con el prellenado (llegó por Google Maps, Instagram o un referido), el saludo del asistente incluye una línea con el aviso y el enlace, una sola vez.
4. **Registro de la prueba:** `customers.consent_at`, `consent_source` (`qr_poster` | `first_message_notice`), `consent_message_id` y `policy_version`.

### 5.3 Contenido del cartel (validar con abogado) ⚠️
- Nombre del negocio (Responsable) y medio de contacto.
- Datos que se tratan (teléfono y mensajes) y para qué (agendar y atender).
- Que quien responde es un **asistente virtual con IA**.
- Derechos del titular (consultar, corregir, pedir eliminación) y enlace a la política completa.
- Frase de aceptación: "Al escanear y enviar el mensaje autorizas este tratamiento."

### 5.4 Texto de respaldo (primer contacto)
> "Hola, soy [Nombre], el asistente virtual de [Negocio] 😊. Uso IA para ayudarte a agendar. Al escribirnos autorizas el tratamiento de tus datos según esta política: [enlace]. Si prefieres hablar con una persona, dímelo."

### 5.5 Límites
- Si el negocio quiere enviar **promociones** u otros fines, necesita autorización **específica** para esa finalidad; no se cubre con esta frase.
- Para segmentos de salud o con menores, este mecanismo **no basta por sí solo** (sección 10). ⚠️
- La ley colombiana no tiene, que yo sepa, una base alternativa tipo "necesario para prestar el servicio" que permita tratar sin autorización; confírmalo con tu abogado.

---

## 6. Transmisión internacional de datos

LLM, OpenRouter, Telegram, STT y quizá el VPS implican enviar datos fuera de Colombia.

- **Transmisión:** el Responsable envía datos a un Encargado en el exterior que los trata por su cuenta (tu caso con estos proveedores).
- **Transferencia:** el receptor es otro Responsable.
- La ley (art. 26) prohíbe, por regla general, transferir datos a países sin nivel adecuado de protección, con excepciones; para transmisiones, la normativa reglamentaria contempla el contrato de transmisión.

**Qué hacer**
1. **Mapear** dónde viajan los datos (tabla 3).
2. **Verificar el país de destino:** según la Circular Externa 005 de 2017 de la SIC, Estados Unidos aparece entre los países con nivel adecuado. ⚠️ Confirma el listado vigente.
3. **Revisar los términos de cada proveedor:** contrato de tratamiento de datos (DPA), si usan los datos de la API para entrenar (confírmalo por escrito), retención y opciones de retención cero. Con **OpenRouter**, revisa además a qué proveedores finales enruta y qué políticas aplican; configura el enrutamiento para excluir los que entrenen con tus datos.
4. **Cláusulas contractuales modelo** de la SIC (Circular Externa 003 de 2025): uso facultativo, útiles cuando el destino no tiene reconocimiento de nivel adecuado.
5. **Declararlo** en el contrato con el negocio y en la política de tratamiento.
6. **Minimizar** lo enviado: sin teléfono completo ni datos innecesarios al LLM (nombre de pila e identificador interno). En Telegram, nombre de pila y últimos 4 dígitos, con enlace para abrir el chat completo.

---

## 7. Inteligencia artificial: transparencia y honestidad

- La SIC expidió la **Circular Externa 002 de 2024** con lineamientos para el tratamiento de datos personales en sistemas de IA (transparencia, responsabilidad demostrada, evaluación de riesgos). ⚠️ Revísala con tu abogado.
- **No he verificado el estado actual de proyectos de ley sobre IA en Colombia.** Revísalo antes de lanzar y periódicamente.

**Regla de honestidad (decisión de diseño)**
- El asistente puede tener nombre y hablar con el estilo del negocio, en primera persona y con trato cercano.
- El **cartel, el perfil y el aviso** indican que es un asistente virtual.
- Si el cliente pregunta en serio si habla con una persona o un bot, **responde con verdad**, con naturalidad y ofreciendo pasar con una persona del equipo.
- **Nunca** afirma ser humano ni inventa una vida personal.
- Esto protege la autorización informada, evita reclamos por engaño y es coherente con los lineamientos de transparencia.

**Otras buenas prácticas**
- Traspaso a humano disponible en cualquier momento.
- Registrar las acciones del bot (herramientas ejecutadas) para poder explicarlas.
- No dar asesoría médica, legal ni financiera; ceñirse a la información del negocio.
- Protección contra *prompt injection* (ver `DESIGN.md`).

---

## 8. Notas de voz

- Un audio contiene la voz de una persona: es un dato personal y puede revelar más de lo que el cliente quería compartir.
- **Minimización:** se transcribe y se guarda **solo la transcripción**; el archivo de audio se descarta tras procesarlo (no se almacena ni se incluye en respaldos).
- El proveedor de transcripción es un **subencargado**: revisa si retiene audio o lo usa para entrenar, y su ubicación.
- Informar en el aviso que los audios se transcriben automáticamente.
- Complemento opcional por negocio: si no está activo, el bot no procesa audios (solo responde con cortesía).

---

## 9. Derechos de los titulares y retención

Los titulares pueden **conocer, actualizar, rectificar y suprimir** sus datos y revocar la autorización. El contrato debe fijar tus plazos para apoyar al negocio (p. ej. 5 días hábiles).

**Funciones a implementar**
- Búsqueda por teléfono: todos los datos de un titular.
- **Exportar** conversación y citas de un titular.
- **Borrar o anonimizar** mensajes, resumen y datos de un titular.
- Registro de solicitudes atendidas (`audit_log`).
- **Baja de clientes (opt-out, "BAJA"):** contemplada en el esquema (`customers.opted_out`); se desarrollará más adelante.
- **Retención:** política escrita (6 o 12 meses para mensajes; luego borrar o dejar solo el resumen) coherente con la finalidad.
- **Al terminar el contrato:** eliminar o devolver los datos del negocio en un plazo definido (p. ej. 30 días), incluidos los respaldos según su ciclo de rotación.

---

## 10. Datos sensibles y menores

- **Datos sensibles** (salud, vida sexual, origen racial, biometría, etc.) tienen tratamiento reforzado: autorización explícita y no se puede condicionar un servicio a entregarlos.
- Barberías y negocios similares rara vez los manejan, pero **clínicas, odontólogos, psicólogos o centros estéticos** sí. Si los atiendes:
  - instruir al bot para **no solicitar** información clínica y derivar al personal si el cliente la ofrece,
  - evitar guardarla en resúmenes o enviarla innecesariamente al LLM,
  - consultar a un abogado **antes** de ofrecer el servicio a ese segmento. ⚠️
- **Menores de edad:** su tratamiento tiene restricciones y debe respetar su interés superior. Define con el abogado cómo se obtiene la autorización del representante legal.

---

## 11. Seguridad, respaldos e incidentes

| Medida | Estado en el diseño |
|---|---|
| HTTPS en todo lo expuesto (Caddy) | ✅ Definido |
| Postgres, Redis y Evolution sin exposición pública | ✅ Definido |
| Webhook con secreto por negocio y verificación de instancia; API keys; admin autenticado | ✅ Definido |
| Telegram: solo `chat_id` autorizado y secreto del webhook | ✅ Definido |
| Credenciales separadas por base; secretos del canal cifrados | ✅ Definido |
| **Respaldos cifrados** (`pg_dump` + exportación por número) fuera del VPS, con restauración probada | 🔲 Por implementar |
| Cifrado de disco/volúmenes del VPS | 🔲 Recomendado |
| SSH con llaves y firewall | 🔲 Por implementar |
| Registro de accesos y acciones (`audit_log`) | 🔲 Por implementar |
| **Aislamiento por `business_id`** con pruebas automáticas | 🔲 Obligatorio |
| Secretos fuera del repositorio | 🔲 Verificar |
| Plan de respuesta a incidentes | 🔲 Por redactar |

**Respaldos por número:** facilitan restaurar un negocio sin depender de la sesión de WhatsApp y también **eliminar** sus datos al terminar el contrato. Definir ciclo de rotación y comunicarlo en el contrato (los respaldos conservan datos hasta que rotan).

**Incidentes (mínimo):** detectar → contener → evaluar alcance → avisar al negocio afectado (que es quien debe reportar a la SIC si corresponde) → corregir → documentar. Para los Responsables obligados a inscribirse en el RNBD, los incidentes deben reportarse a la SIC dentro de los 15 días hábiles siguientes a su detección.

---

## 12. Registro Nacional de Bases de Datos (RNBD)

- Según el Decreto 090 de 2018, la obligación de registro recae en las **sociedades y entidades sin ánimo de lucro con activos totales superiores a 100.000 UVT** y en las personas jurídicas de naturaleza pública.
- **La mayoría de tus clientes (micro y pequeños negocios) y probablemente tú, al inicio, no estarán obligados.** Pero estar exento del registro **no te exime** del resto de la Ley 1581.
- ⚠️ Revisa el valor de la UVT vigente y los activos de clientes grandes por si les aplica.

---

## 13. WhatsApp / Meta

### 13.1 Evolution API por QR (no oficial)
- Funciona simulando un cliente de WhatsApp Web; **no es un canal autorizado por Meta** para automatización empresarial a escala.
- **Riesgo:** bloqueo del número sin previo aviso.
- **Mitigaciones:** aceptación **escrita** del riesgo por el cliente; excluir o limitar tu responsabilidad por bloqueos; **número dedicado** al negocio; canal desacoplado (migración a API oficial); evitar mensajes masivos, ráfagas y envíos a quien no escribió primero.

### 13.2 API oficial
- Desde el **15 de enero de 2026**, los términos de la WhatsApp Business Solution **prohíben a proveedores de IA distribuir asistentes de propósito general**. Meta aclaró que **no afecta** a negocios que usan IA para atención al cliente, reservas o seguimiento.
- Tu producto (agendamiento y atención de un negocio concreto) encaja, **siempre que el alcance sea acotado**: el bot rechaza conversación abierta y temas ajenos al negocio.
- Mensajes proactivos requieren consentimiento y, fuera de la ventana de 24 h, plantillas aprobadas. ⚠️ Lee los términos vigentes (WhatsApp Business Solution Terms y Business Messaging Policy).

### 13.3 Coexistencia (app Business + API en el mismo número)
- Permite responder desde la app WhatsApp Business mientras el bot opera por la API; los mensajes se reflejan en ambos lados.
- La información que encontré proviene sobre todo de proveedores comerciales; **valida las condiciones directamente con Meta** y confirma la compatibilidad con Evolution antes de ofrecerla. ⚠️
- Meta pasa a ser subencargado y el contrato debe reflejarlo.

---

## 14. Consumidores, publicidad y recordatorios

- Lo que el bot informa (precios, horarios, políticas de cancelación) debe ser **veraz y verificable** (Estatuto del Consumidor, Ley 1480 de 2011). Por eso el bot toma precios y reglas **solo de la base de datos** (`services`), nunca los inventa, y conserva el historial.
- **Recordatorios de citas** ya solicitadas son distintos de la **publicidad**. Las promociones requieren autorización específica y un mecanismo de baja.
- **Horarios de contacto:** la Ley 2300 de 2023 regula horarios y canales para contactar consumidores con fines de cobranza, ventas u ofertas. ⚠️ Verifica con tu abogado si aplica a recordatorios y mercadeo, y programa los envíos proactivos dentro de las franjas permitidas.
- Las normas 1480/2300 **no se verificaron** en esta revisión; confirma el texto vigente.

---

## 15. Contrato SaaS con tus clientes

Cláusulas que no deberían faltar:

1. **Objeto y alcance** (qué hace y qué no el asistente).
2. **Roles de datos** y anexo de transmisión (secciones 3 y 4).
3. **Riesgo del canal** (QR) aceptado por escrito.
4. **Limitaciones de la IA:** puede equivocarse; el negocio es responsable de los servicios, precios, horarios y excepciones que configura; no es asesoría profesional.
5. **Cartel con QR y política de datos:** el negocio se compromete a exhibirlos y a no usar los datos para otros fines.
6. **SLA** razonable (p. ej. 99 % mensual), mantenimientos y dependencia de terceros (WhatsApp, LLM, VPS).
7. **Precio, pago, facturación e impuestos;** ajuste por costos de LLM, voz o API de WhatsApp.
8. **Uso aceptable:** prohibir spam, contenido ilegal, suplantación y mercadeo sin consentimiento.
9. **Propiedad:** los datos son del negocio; el software, prompts base y mejoras son tuyos; licencia de uso al cliente. Precisar de quién son los **ejemplos de voz** (mensajes del dueño) usados en el prompt.
10. **Confidencialidad.**
11. **Limitación de responsabilidad** (tope razonable) e indemnidad por mal uso.
12. **Duración, terminación, devolución/eliminación de datos** (incluyendo respaldos).
13. **Soporte:** canales y tiempos.
14. **Subcontratistas** y notificación de cambios.
15. **Ley aplicable y solución de controversias.**
16. **Cláusulas por plan y complementos** (ver sección 16).

> ⚠️ Pide a un abogado que redacte la plantilla **una vez**; luego la reutilizas con cada cliente.

---

## 16. Implicaciones legales por plan y complemento

| Tema | Básico | Pro | Premium |
|---|---|---|---|
| Riesgo de bloqueo del número | Cláusula obligatoria | Si elige QR | Reducida |
| Meta como subencargado | Sí | Sí | Sí |
| LLM / OpenRouter | Sí | Sí | Sí |
| Telegram | Si activa | Sí | Sí |
| STT (complemento de voz) | Si activa | Si activa | Si activa |
| Recordatorios | Servicio de cita ya solicitada | Igual | Igual (+ plantillas en API oficial) |
| Segmentos sensibles | No sin asesoría | No sin asesoría | Solo con asesoría específica |

---

## 17. Propiedad intelectual y licencias

- **Código propio:** define titularidad si trabajas con socios o freelancers (cesión escrita).
- **Evolution API:** revisa su licencia y condiciones vigentes antes de usarla comercialmente. ⚠️
- **n8n:** *Sustainable Use License*; restringe ciertos usos comerciales. Está fuera del flujo crítico; valida su licencia si lo incluyes.
- **Dependencias de Python:** inventario de licencias (`pip-licenses` o similar); evita copyleft fuerte si distribuyes el software.
- **Salidas del LLM:** revisa los términos de cada proveedor sobre titularidad y uso.
- **Marca:** verifica disponibilidad y regístrala ante la SIC.

---

## 18. Formalización, impuestos y facturación

⚠️ Consulta a un contador antes de facturar.

- Decide **persona natural** o **sociedad (p. ej. SAS)**; la sociedad separa tu patrimonio del riesgo del negocio (datos y bloqueos del canal).
- RUT, registro mercantil y **facturación electrónica** ante la DIAN.
- **IVA** sobre servicios de software; retenciones que aplicarán tus clientes.
- **Costos en dólares** (LLM, VPS, voz): considera la tasa de cambio.
- Seguro de responsabilidad civil/ciberseguridad: evaluar al crecer.

---

## 19. Documentos por preparar

| Documento | Para qué | Quién |
|---|---|---|
| Contrato SaaS / términos de servicio | Relación comercial | Abogado (una vez) |
| Anexo de transmisión de datos + lista de subencargados | Ley 1581 como Encargado | Abogado |
| Plantilla de política de tratamiento del negocio | El Responsable la publica | Abogado + tú |
| **Texto del cartel con QR** y mensaje prellenado | Autorización informada | Abogado + tú |
| Texto de respaldo del primer contacto | Quienes no pasan por el cartel | Abogado + tú |
| Política interna de seguridad, incidentes y respaldos | Responsabilidad demostrada | Tú (con apoyo) |
| Política de retención y eliminación | Coherencia con el sistema | Tú + abogado |
| Aceptación de riesgo del canal QR | Proteger tu responsabilidad | Abogado |
| DPA/términos de LLM, OpenRouter, Telegram, STT, VPS | Respaldo de la cadena | Revisar los de cada proveedor |

---

## 20. Checklist por fases

**Antes del primer cliente**
- [ ] Definir figura legal (persona natural/SAS), RUT y facturación.
- [ ] Contrato SaaS + anexo de datos revisados por abogado.
- [ ] Política de tratamiento del negocio y **cartel con QR** (mensaje prellenado).
- [ ] Texto de respaldo del primer contacto y campos de consentimiento implementados.
- [ ] Revisar y aceptar términos/DPA del LLM y de OpenRouter (enrutamiento y retención).
- [ ] Definir región del VPS y documentar la ubicación de los datos.
- [ ] Cláusula de riesgo del canal QR aceptada por escrito.
- [ ] Respaldos cifrados fuera del VPS y SSH endurecido.
- [ ] Prompt con regla de honestidad y alcance limitado al negocio.

**Durante la operación**
- [ ] Exportación y borrado por titular; registro de solicitudes.
- [ ] Retención aplicada con tarea programada.
- [ ] Plan de incidentes probado.
- [ ] Revisión semestral de normativa, políticas de Meta y subencargados.
- [ ] Pruebas automáticas de aislamiento entre negocios.
- [ ] Baja de clientes (BAJA) implementada.

**Al crecer**
- [ ] Evaluar API oficial o coexistencia para clientes de mayor riesgo.
- [ ] Verificar obligación de RNBD por cliente.
- [ ] Revisar segmentos sensibles (salud, menores) con asesoría.
- [ ] Seguro de responsabilidad y certificaciones de seguridad.

---

## 21. Cambios técnicos que implica lo legal

1. `customers`: `consent_at`, `consent_source`, `consent_message_id`, `policy_version`, `opted_out` (ya en el esquema).
2. Endpoints de administración: exportar y borrar por teléfono.
3. Tarea de **retención** programada y de **eliminación por negocio** (incluye respaldos por número).
4. `audit_log` para solicitudes de titulares y acciones administrativas.
5. **Minimización** en llamadas al LLM, avisos de Telegram y respaldos; **descartar audios** tras transcribir.
6. System prompt: asistente virtual, alcance limitado, honestidad si preguntan, sin asesoría profesional ni datos sensibles.
7. Lógica de **primer contacto:** si el primer mensaje coincide con el prellenado, marcar `qr_poster`; si no, enviar el aviso y marcar `first_message_notice`.
8. Versionado de la **política de tratamiento** por negocio (`policy_version`).
9. Pruebas de aislamiento multi-tenant en CI.

---

## 22. Fuentes consultadas

Revisa siempre la versión oficial y vigente:

- Ley Estatutaria 1581 de 2012 (deberes del Responsable y del Encargado, arts. 17 y 18; transferencia, art. 26).
- Decreto 1377 de 2013, compilado en el Decreto 1074 de 2015 (autorización, aviso de privacidad, contrato de transmisión).
- Decreto 090 de 2018 y Circular Externa 003 de 2018 de la SIC (alcance del RNBD; reporte de incidentes).
- Circular Externa 005 de 2017 de la SIC (países con nivel adecuado).
- Circular Externa 002 de 2024 de la SIC (IA y datos personales).
- Circular Externa 003 de 2025 de la SIC (cláusulas contractuales modelo).
- Concepto SIC sobre obligaciones del tercero subcontratado.
- Términos de WhatsApp Business Solution y reportes de prensa sobre la prohibición de chatbots de propósito general desde el 15 de enero de 2026.
- Publicaciones de proveedores sobre la coexistencia de WhatsApp (informativas; validar con Meta).
- Ley 1480 de 2011 y Ley 2300 de 2023: **no verificadas** en esta revisión.

> Revisa este documento cada vez que cambie el canal (QR → Cloud API), el proveedor de LLM o STT, el país de alojamiento, las consolas activas o el tipo de negocio atendido.
