# Arquitectura General — QuickBite (Sistema de Delivery de Comidas)

---

## Descripción

Plataforma digital de delivery de comidas construida en **Python,Java (FastAPI + Django REST Framework)** sobre una arquitectura de **microservicios** con cuatro capas: Presentación, API Gateway, Negocio y Datos. Conecta tres actores principales —Clientes, Restaurantes y Repartidores— e integra servicios externos de pago, mapas y notificaciones.

---

## Diagrama de Contexto

```
  ┌──────────────────┐                                          ┌──────────────────────┐
  │  Cliente         │ ──▶                                 ┌───▶│  Culqi / Stripe      │
  │  (Web / Móvil)   │        ┌─────────────────────────┐  │    │  Pasarela de Pagos   │
  └──────────────────┘        │                         │  │    └──────────────────────┘
                              │      Q U I C K B I T E  │  │    ┌──────────────────────┐
  ┌──────────────────┐        │                         │──┼───▶│  Google Maps API     │
  │  Restaurante     │ ──▶   │   API Gateway (Nginx)   │  │    │  GPS / Rutas / ETA   │
  │  (Panel Web)     │        │   Microservicios Python │  │    └──────────────────────┘
  └──────────────────┘        │   Puerto 8000           │  │    ┌──────────────────────┐
                              │                         │──┼───▶│  Firebase / Twilio   │
  ┌──────────────────┐        │                         │  │    │  SendGrid            │
  │  Repartidor      │ ──▶   │                         │  │    │  Notificaciones      │
  │  (App Móvil)     │        └─────────────────────────┘  │    └──────────────────────┘
  └──────────────────┘                                      │    ┌──────────────────────┐
                                                            └───▶│  CDN                 │
  ┌──────────────────┐                                           │  Assets / Imágenes   │
  │  Administrador   │ ──▶                                       └──────────────────────┘
  │  (Dashboard)     │
  └──────────────────┘
```

---

## Capas del sistema

```
┌──────────────────────────────────────────────────────────────────┐
│                      Capa de Presentación                        │
│   Web App (React)  ·  App Móvil (iOS / Android)                  │
│   Panel Restaurante  ·  App Repartidor  ·  Dashboard Admin       │
├──────────────────────────────────────────────────────────────────┤
│                       API Gateway                                │
│   Nginx  ·  Autenticación JWT / OAuth 2.0  ·  Rate Limiting      │
│   Logging centralizado  ·  Enrutamiento a microservicios         │
├──────────────────────────────────────────────────────────────────┤
│                     Capa de Negocio                              │
│                                                                  │
│   ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│   │  PedidoService  │  │   PagoService   │  │UsuarioService  │  │
│   │  (SRP / DIP)    │  │   (OCP / SRP)   │  │  (ISP)         │  │
│   └─────────────────┘  └─────────────────┘  └────────────────┘  │
│                                                                  │
│   ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│   │SeguimientoSrv.  │  │NotificacionSrv. │  │CatalogoService │  │
│   │  (LSP)          │  │  (SRP)          │  │                │  │
│   └─────────────────┘  └─────────────────┘  └────────────────┘  │
│                                                                  │
│   Protocolos (ABC): PedidoRepositorio · EstrategiaPago           │
│   EntregaRastreable · EntregaAsignable · PerfilClienteService    │
│   AdministracionUsuarioService · ContactoRepartidorService       │
├──────────────────────────────────────────────────────────────────┤
│                      Capa de Datos                               │
│   PostgreSQL 17 (fuente de verdad)  ·  Redis (caché / Celery)   │
│   SQLAlchemyPedidoRepositorio  ·  InMemoryRepositorio (tests)   │
│   Celery Workers  ·  Apache Airflow (ETL / reportes)             │
└──────────────────────────────────────────────────────────────────┘
```

**Regla de dependencias:** las capas superiores dependen de las inferiores **solo a través de protocolos abstractos (ABC)**. La infraestructura (SQLAlchemy, Firebase, Culqi) nunca es importada directamente por la capa de negocio.

---

## Microservicios principales

```
                        ┌──────────────────────┐
                        │     API Gateway       │
                        │  JWT · Rate Limit     │
                        └──────────┬───────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  MS Usuarios     │   │  MS Pedidos      │   │  MS Pagos        │
│  RF01 RF02 RF03  │   │  RF06 RF07       │   │  RF08 RF09 RF16  │
│  Puerto 8001     │   │  Puerto 8002     │   │  Puerto 8003     │
└──────────────────┘   └──────────────────┘   └──────────────────┘
          │                        │                        │
          └────────────────────────┼────────────────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  MS Catálogo     │   │  MS Seguimiento  │   │  MS Notif.       │
│  RF04 RF05       │   │  RF10 RF11 RF12  │   │  RF14            │
│  Puerto 8004     │   │  RF13            │   │  Puerto 8006     │
│  Redis Cache     │   │  WebSocket       │   │  FCM/Twilio/SG   │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

---

## Flujo principal — Ciclo de un pedido

```
Cliente                API Gateway            MS Pedidos         MS Pagos
   │                        │                      │                  │
   │── POST /pedidos ───────▶│                      │                  │
   │                        │── JWT válido? ───────▶│                  │
   │                        │                      │── crear_pedido() │
   │                        │                      │── guardar() ─────▶ PostgreSQL
   │                        │                      │                  │
   │                        │                      │── notif_recibido()──▶ Firebase
   │                        │                      │                  │
   │── POST /pagos ─────────▶│                      │                  │
   │                        │──────────────────────────────────────────▶│
   │                        │                      │  procesar_pago() │
   │                        │                      │  EstrategiaCulqi │──▶ Culqi API
   │                        │                      │                  │
   │◀── 200 OK (aprobado) ───│                      │◀─ APROBADO ──────│
   │                        │                      │── estado=APROBADO│
   │                        │                      │── notif_pago() ──────▶ SendGrid
```

---

## Stack tecnológico

| Componente | Tecnología | RNF asociado |
|------------|------------|--------------|
| Backend | Python 3.12 · FastAPI / DRF | RNF11, RNF12 |
| Base de datos principal | PostgreSQL 17 | RNF16 |
| Caché | Redis 7 | RNF05 |
| Mensajería async | Celery + Redis | RNF08 |
| ETL / Reportes | Apache Airflow | RF15 |
| Infraestructura | Docker · Nginx · Linode | RNF07, RNF09 |
| CI/CD | GitHub Actions | RNF11 |
| Pasarelas de pago | Culqi · Stripe | RNF06, RNF15 |
| Notificaciones | Firebase FCM · Twilio · SendGrid | RF14, RNF15 |
| Mapas / GPS | Google Maps Platform | RF10, RF11, RNF04 |
| CDN | Cloudflare / Linode CDN | RNF10 |
| Auth | OAuth 2.0 / JWT | RNF01 |
| Monitoreo | APM (Datadog / New Relic) | RNF13 |

---

## Decisiones registradas

| ADR | Decisión | Principio | Impacto principal |
|-----|----------|-----------|-------------------|
| [ADR-001](../adr/ADR-001-srp-servicio-notificaciones.md) | Separar `NotificacionService` del `PedidoService` | SRP | Una sola razón de cambio por clase |
| [ADR-002](../adr/ADR-002-ocp-pasarelas-pago.md) | Patrón Strategy para pasarelas de pago | OCP | Agregar Yape/Plin sin tocar código existente |
| [ADR-003](../adr/ADR-003-dip-repositorio-pedidos.md) | Protocolo abstracto `PedidoRepositorio` | DIP | Tests unitarios sin PostgreSQL ni Docker |
| [ADR-004](../adr/ADR-004-lsp-tipos-entrega.md) | Jerarquía de entregas con protocolos separados | LSP | Sustitución segura sin `NotImplementedError` |
| [ADR-005](../adr/ADR-005-isp-gestion-usuarios.md) | Segregar interfaz de usuarios por actor | ISP | Cada actor depende solo de lo que usa |
