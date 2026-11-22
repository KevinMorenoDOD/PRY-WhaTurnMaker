# PRY-WhaTurnMaker — Documentación

Este repositorio conserva **solo la documentación** del proyecto.

La aplicación funcional (backend, worker, Docker, scripts y pruebas) está en:

```text
/home/pkar/Proyects/PRY-WhaTurnMaker-IA
```

Guía de arranque de la aplicación:

```text
/home/pkar/Proyects/PRY-WhaTurnMaker-IA/COMO_ARRANCAR.md
```

Arranque rápido:

```bash
cd /home/pkar/Proyects/PRY-WhaTurnMaker-IA
./start.sh
```

---

## Documentos

| Documento | Contenido |
|---|---|
| [`WTM-Docs/ARCHITECTURE.md`](./WTM-Docs/ARCHITECTURE.md) | Topología, multi-tenant, backend, configuración por capas, esquema PostgreSQL, flujo, cola, Compose y respaldos. |
| [`WTM-Docs/DESIGN.md`](./WTM-Docs/DESIGN.md) | Decisiones, reglas de comportamiento (pipeline, retraso, agenda, traspaso, voz), riesgos y fases. |
| [`WTM-Docs/PLAN.md`](./WTM-Docs/PLAN.md) | Planes Básico/Pro/Premium, complementos, márgenes y reglas de implementación. |
| [`WTM-Docs/Legal.md`](./WTM-Docs/Legal.md) | Protección de datos (Colombia), cartel con QR, subencargados, WhatsApp/Meta, contratos y checklist. |

---

## Resumen del producto

Asistente virtual para negocios que agendan citas (barberías, consultorios,
estudios, etc.) que atiende por WhatsApp, responde preguntas frecuentes del
negocio y agenda, cancela y reagenda citas usando un LLM con *tool calling*.
Un solo despliegue atiende a varios negocios (multi-tenant), cada uno con su
propio número, servicios, horarios y estilo.

El código y la operación viven en `PRY-WhaTurnMaker-IA`.

## Licencia

Por definir. Hasta decidirla, todos los derechos están reservados.
