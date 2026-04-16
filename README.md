# Architecture Decision Records — QuickBite

**Sistema:** QuickBite — Sistema de Delivery de Comidas
**Versión:** 1.0 | **Fecha:** Abril 2026
**Autores:** Eduardo Peralta · Lennin Cubas · Christian Echevaria
**Curso:** Arquitectura de Software — Módulo 1 | Tecsup

---

## ¿Qué es un ADR?

Un **Architecture Decision Record (ADR)** documenta una decisión arquitectónica significativa: el contexto que la motivó, la decisión tomada, las alternativas descartadas y las consecuencias esperadas.

---

## Índice de decisiones

| ADR | Título | Principio SOLID | Estado |
|-----|--------|-----------------|--------|
| [ADR-001](./ADR-001-srp-servicio-notificaciones.md) | Separar el envío de notificaciones del servicio de pedidos | **S** — SRP | ✅ Aceptado |
| [ADR-002](./ADR-002-ocp-pasarelas-pago.md) | Estrategias extensibles para el procesamiento de pagos | **O** — OCP | ✅ Aceptado |
| [ADR-003](./ADR-003-dip-repositorio-pedidos.md) | Abstraer la persistencia de pedidos mediante interfaz | **D** — DIP | ✅ Aceptado |
| [ADR-004](./ADR-004-lsp-tipos-entrega.md) | Tipos de entrega intercambiables en el sistema de seguimiento | **L** — LSP | ✅ Aceptado |
| [ADR-005](./ADR-005-isp-gestion-usuarios.md) | Segregar la interfaz de usuarios según el rol del actor | **I** — ISP | ✅ Aceptado |

---

## Mapa de principios SOLID → componentes QuickBite

```
S — SRP │ PedidoService  ──┤  NotificacionService (Firebase/SendGrid/Twilio)
O — OCP │ PagoService    ──┤  EstrategiaPago: Culqi | Stripe | Yape | Efectivo
D — DIP │ PedidoService  ──┤  PedidoRepositorio (ABC) ← SQLAlchemy | InMemory
L — LSP │ EntregaBase    ──┤  DeliveryDomicilio | DeliveryExpress | RetiroEnTienda
I — ISP │ UsuarioService ──┤  PerfilCliente | Administración | Restaurante | Repartidor
```

---

## Tecnologías del sistema

- **Backend:** Python (FastAPI o Django REST Framework) ,java21 spring framework 3
- **Base de datos:** PostgreSQL 17 (fuente de verdad — RNF16)
- **Caché:** Redis (catálogo — RNF05)
- **Mensajería:** Celery + Redis (tareas asíncronas)
- **Pasarelas:** Culqi / Stripe (RF08) · Yape/Plin (RF16)
- **Notificaciones:** Firebase FCM · SendGrid · Twilio (RF14)
- **Mapas / GPS:** Google Maps Platform (RF10, RF11)
- **Infraestructura:** Docker · Nginx · Linode · GitHub Actions CI/CD
