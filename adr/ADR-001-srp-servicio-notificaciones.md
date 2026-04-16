# ADR-001 — Separar el envío de notificaciones del servicio de pedidos

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** S — Single Responsibility Principle (SRP)

---

## Definición del principio

> "Un módulo debe tener **una, y solo una, razón para cambiar**."
> — Robert C. Martin

Una clase viola el SRP cuando múltiples actores con responsabilidades distintas (el área de negocio, el área técnica, el área legal) tienen motivos independientes para modificarla. Cada cambio en una responsabilidad es un riesgo de romper las demás.

---

## Analogía real — Odoo v13 (Sistema ERP Fiberlux)

Antes de ver el problema en QuickBite, analicemos un caso concreto del mundo real con Odoo v13.

### El problema en Odoo: `action_confirm()` con demasiadas responsabilidades

En el módulo de ventas de internet de Fiberlux, cuando el comercial confirma una cotización (previa aceptación del cliente con contrato firmado y plantilla económica adjunta), el método `action_confirm()` hace **todo** de una sola vez:

```python
# sale_order_internet.py (Odoo v13) — VIOLACIÓN del SRP
# Módulo: flx_sale_order_personalization

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def action_confirm(self):
        """
        ❌ Este método tiene 6+ razones para cambiar.
        Cada área del negocio puede requerir modificarlo.
        """
        # 1. Lógica propia de confirmación (herencia Odoo)
        result = super(SaleOrder, self).action_confirm()

        for order in self:
            # ❌ Responsabilidad 2: Gestión de proyectos
            # Si cambia el flujo de proyectos → toco este método
            project = self.env['project.project'].create({
                'name': f'Internet - {order.name}',
                'partner_id': order.partner_id.id,
                'user_id': order.user_id.id,
            })
            self.env['project.task'].create({
                'name': 'Instalación y configuración',
                'project_id': project.id,
                'sale_order_id': order.id,
            })

            # ❌ Responsabilidad 3: Generación de contrato
            # Si el área legal cambia el formato → toco este método
            contrato = self.env['flx.contrato'].create({
                'sale_order_id': order.id,
                'partner_id': order.partner_id.id,
                'fecha_inicio': fields.Date.today(),
                'plantilla_id': order.plantilla_contrato_id.id,
            })

            # ❌ Responsabilidad 4: Adjuntar documentos firmados
            # Si cambia el proceso de firma → toco este método
            if order.documento_firmado:
                order.documento_firmado.copy({
                    'res_model': 'flx.contrato',
                    'res_id': contrato.id,
                })

            # ❌ Responsabilidad 5: Solicitud de compras
            # Si cambia el flujo de compras → toco este método
            self.env['purchase.request'].create({
                'sale_order_id': order.id,
                'requested_by': order.user_id.id,
                'description': f'Equipos para pedido {order.name}',
            })

            # ❌ Responsabilidad 6: Notificación al cliente
            # Si cambia el proveedor de email → toco este método
            template = self.env.ref('flx_sale.email_confirmacion_internet')
            template.send_mail(order.id, force_send=True)
            order.message_post(
                body=f"Cotización {order.name} confirmada. Proyecto creado.",
                subtype_xmlid='mail.mt_note',
            )

        return result
```

**¿Cuántas personas tienen razón para abrir ese método?**

| Actor | Motivo para modificar `action_confirm()` |
|-------|------------------------------------------|
| Desarrollador backend | Cambio en el flujo de estados del pedido |
| Líder de proyectos | Cambia la estructura de tareas a crear |
| Área legal | Cambia el formato o campos del contrato |
| Área de compras | Cambia el proceso de solicitudes |
| Área de sistemas | Cambia el proveedor de email o notificaciones |
| Gerencia comercial | Cambia la plantilla económica adjunta |

**Ese único método tiene 6 razones de cambio. Viola el SRP.**

### La solución en Odoo v13 — delegar responsabilidades

```python
# sale_order_internet.py (Odoo v13) — APLICANDO SRP
# Cada responsabilidad vive en su propio servicio

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def action_confirm(self):
        """
        ✅ Una sola razón de cambio: el flujo de confirmación del pedido.
        Delega cada responsabilidad a su servicio especializado.
        """
        result = super(SaleOrder, self).action_confirm()

        for order in self:
            # ✅ Delega — no implementa
            self.env['flx.project.service'].crear_desde_pedido(order)
            self.env['flx.contrato.service'].generar(order)
            self.env['flx.compras.service'].crear_solicitud(order)
            self.env['flx.notificacion.service'].notificar_confirmacion(order)

        return result


# flx_notificacion_service.py — responsabilidad única: notificaciones
class FlxNotificacionService(models.AbstractModel):
    _name = 'flx.notificacion.service'
    _description = 'Servicio de notificaciones — única razón de cambio: canal/contenido'

    def notificar_confirmacion(self, order):
        """
        ✅ Solo este archivo se toca si cambia el proveedor de email
        o el contenido del mensaje. action_confirm() no se toca.
        """
        template = self.env.ref('flx_sale.email_confirmacion_internet')
        template.send_mail(order.id, force_send=True)
        order.message_post(
            body=f"Cotización {order.name} confirmada exitosamente.",
            subtype_xmlid='mail.mt_note',
        )

    def notificar_proyecto_creado(self, order, project):
        order.message_post(
            body=f"Proyecto '{project.name}' creado y asignado.",
            subtype_xmlid='mail.mt_note',
        )


# flx_project_service.py — responsabilidad única: gestión de proyectos
class FlxProjectService(models.AbstractModel):
    _name = 'flx.project.service'
    _description = 'Servicio de proyectos — única razón de cambio: estructura de proyectos'

    def crear_desde_pedido(self, order):
        """
        ✅ Solo este archivo se toca si cambia la estructura
        de proyectos o tareas. action_confirm() no se toca.
        """
        project = self.env['project.project'].create({
            'name': f'Internet - {order.name}',
            'partner_id': order.partner_id.id,
            'user_id': order.user_id.id,
        })
        self.env['project.task'].create({
            'name': 'Instalación y configuración',
            'project_id': project.id,
            'sale_order_id': order.id,
        })
        return project
```

**La señal de que el SRP está bien aplicado:** si el área legal llama para cambiar el contrato, el desarrollador abre `flx_contrato_service.py`. Si IT dice que cambian el servidor SMTP, el desarrollador abre `flx_notificacion_service.py`. **Nadie toca `action_confirm()` salvo que cambie el flujo de confirmación en sí.**

---

## El mismo problema en QuickBite

Con esa base, ahora el problema en QuickBite es exactamente el mismo patrón:

**Código actual (con el problema):**

```python
# pedido_service.py — tiene más de una responsabilidad
import smtplib
import requests
from datetime import datetime

class PedidoService:

    def __init__(self, pedido_repo):
        self.pedido_repo = pedido_repo

    def crear_pedido(self, usuario, carrito):
        pedido = {
            "usuario_id": usuario["id"],
            "items": carrito["items"],
            "total": carrito["total"],
            "estado": "RECIBIDO",
            "fecha": datetime.now().isoformat(),
        }
        pedido_guardado = self.pedido_repo.guardar(pedido)

        # ❌ Responsabilidad de notificación mezclada con lógica de pedido
        # Igual que action_confirm() en Odoo: dos cosas en un solo lugar
        smtp = smtplib.SMTP("smtp.quickbite.pe", 587)
        smtp.sendmail(
            "noreply@quickbite.pe",
            usuario["email"],
            f"Subject: Pedido #{pedido_guardado['id']} recibido\n\n"
            f"Tu pedido ha sido recibido. Total: S/ {carrito['total']}",
        )
        smtp.quit()

        # ❌ También llama a Firebase directamente
        requests.post(
            "https://fcm.googleapis.com/fcm/send",
            headers={"Authorization": "key=FIREBASE_KEY"},
            json={
                "to": usuario["fcm_token"],
                "notification": {
                    "title": "Pedido recibido",
                    "body": f"Total: S/ {carrito['total']}"
                },
            },
        )
        return pedido_guardado

    def actualizar_estado(self, pedido_id, nuevo_estado):
        pedido = self.pedido_repo.actualizar_estado(pedido_id, nuevo_estado)

        # ❌ Lógica de notificación duplicada aquí también
        # Mismo problema: si cambia el canal, toco este método también
        smtp = smtplib.SMTP("smtp.quickbite.pe", 587)
        # ... configuración SMTP repetida ...
        smtp.quit()

        return pedido
```

**¿Cuál es el problema?**

`PedidoService` tiene **dos razones para cambiar**:
- Si cambia la lógica de negocio del pedido (flujo de estados, validaciones de stock).
- Si cambia el canal de notificación (de Firebase a OneSignal, o se agrega Twilio SMS).

Esto también dificulta las pruebas: para testear `crear_pedido` hay que mockear SMTP, Firebase **y** la lógica de negocio al mismo tiempo. En Odoo sería equivalente a necesitar un servidor de correo real para correr el test de confirmación de una venta.

---

## Decisión

Extraemos el envío de notificaciones a una clase dedicada `NotificacionService`. `PedidoService` delega en ella sin conocer los detalles del canal de envío.

**Código corregido:**

```python
# notificacion_service.py — responsabilidad única: enviar notificaciones
# Equivalente a flx.notificacion.service en Odoo v13
import smtplib
import requests


class NotificacionService:
    """
    Única razón de cambio: si cambia el canal (Firebase → OneSignal)
    o el contenido de las notificaciones.
    PedidoService no se toca ante estos cambios.
    """

    def __init__(self, smtp_host: str, smtp_port: int, firebase_key: str):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port
        self.firebase_key = firebase_key

    def notificar_pedido_recibido(self, usuario: dict, pedido: dict) -> None:
        """Notifica al usuario que su pedido fue recibido (email + push)."""
        self._enviar_email(
            destinatario=usuario["email"],
            asunto=f"Pedido #{pedido['id']} recibido — QuickBite",
            cuerpo=f"Tu pedido ha sido recibido. Total: S/ {pedido['total']}",
        )
        self._enviar_push(
            fcm_token=usuario["fcm_token"],
            titulo="¡Pedido recibido!",
            cuerpo=f"Preparando tu pedido #{pedido['id']}",
        )

    def notificar_cambio_estado(self, usuario: dict, pedido: dict, nuevo_estado: str) -> None:
        """Notifica al usuario un cambio de estado en su pedido."""
        mensajes = {
            "EN_PREPARACION": "Tu pedido está siendo preparado 🍳",
            "EN_CAMINO":      "Tu repartidor está en camino 🛵",
            "ENTREGADO":      "¡Pedido entregado! Buen provecho 🎉",
            "CANCELADO":      f"Tu pedido #{pedido['id']} fue cancelado.",
        }
        cuerpo = mensajes.get(nuevo_estado, f"Estado actualizado: {nuevo_estado}")
        self._enviar_push(
            fcm_token=usuario["fcm_token"],
            titulo="Actualización de tu pedido",
            cuerpo=cuerpo,
        )

    # --- métodos privados de infraestructura ---

    def _enviar_email(self, destinatario: str, asunto: str, cuerpo: str) -> None:
        smtp = smtplib.SMTP(self.smtp_host, self.smtp_port)
        smtp.sendmail("noreply@quickbite.pe", destinatario, f"Subject: {asunto}\n\n{cuerpo}")
        smtp.quit()

    def _enviar_push(self, fcm_token: str, titulo: str, cuerpo: str) -> None:
        requests.post(
            "https://fcm.googleapis.com/fcm/send",
            headers={"Authorization": f"key={self.firebase_key}"},
            json={"to": fcm_token, "notification": {"title": titulo, "body": cuerpo}},
            timeout=5,
        )


# pedido_service.py — responsabilidad única: gestionar pedidos
# Equivalente al action_confirm() limpio de Odoo: delega, no implementa
from datetime import datetime


class PedidoService:
    """
    Única razón de cambio: si cambia la lógica de negocio del pedido
    (flujo de estados, validaciones de stock, reglas de negocio).
    Los canales de notificación no le conciernen.
    """

    def __init__(self, pedido_repo, notificacion_service: NotificacionService):
        self.pedido_repo = pedido_repo
        self.notificaciones = notificacion_service  # ✅ delega sin conocer detalles

    def crear_pedido(self, usuario: dict, carrito: dict) -> dict:
        pedido = {
            "usuario_id": usuario["id"],
            "items": carrito["items"],
            "total": carrito["total"],
            "estado": "RECIBIDO",
            "fecha": datetime.now().isoformat(),
        }
        pedido_guardado = self.pedido_repo.guardar(pedido)

        # ✅ Delega sin conocer cómo se envía la notificación
        # Como action_confirm() llamando a flx.notificacion.service en Odoo
        self.notificaciones.notificar_pedido_recibido(usuario, pedido_guardado)
        return pedido_guardado

    def actualizar_estado(self, pedido_id: int, nuevo_estado: str, usuario: dict) -> dict:
        pedido = self.pedido_repo.actualizar_estado(pedido_id, nuevo_estado)

        # ✅ Delega el canal de comunicación al servicio especializado
        self.notificaciones.notificar_cambio_estado(usuario, pedido, nuevo_estado)
        return pedido
```

---

## Principio SOLID aplicado — SRP

> "Un módulo debe tener una, y solo una, razón para cambiar."

### Tabla de responsabilidades — comparativa Odoo vs QuickBite

| Sistema | Clase | Única razón de cambio |
|---------|-------|----------------------|
| **Odoo v13** | `SaleOrder.action_confirm()` | Solo el flujo de confirmación del pedido |
| **Odoo v13** | `FlxNotificacionService` | Solo el canal o contenido de notificaciones |
| **Odoo v13** | `FlxProjectService` | Solo la estructura de proyectos y tareas |
| **Odoo v13** | `FlxContratoService` | Solo la generación y formato del contrato |
| **QuickBite** | `PedidoService` | Reglas de negocio del pedido (estados, validaciones, flujo) |
| **QuickBite** | `NotificacionService` | Canales o contenido de las notificaciones (Firebase, Twilio, SendGrid) |

### El impacto concreto del cambio

**Odoo — Antes (violación):**
```
IT dice: "Cambiamos de servidor SMTP"
  → Desarrollador abre action_confirm()
  → Está al lado del código de proyectos, contratos y compras
  → Riesgo de romper la creación de proyectos al tocar el correo
```

**Odoo — Después (SRP aplicado):**
```
IT dice: "Cambiamos de servidor SMTP"
  → Desarrollador abre flx_notificacion_service.py
  → Solo hay código de notificaciones
  → Cero riesgo sobre proyectos, contratos o compras
```

**QuickBite — Antes (violación):**
```
Product Owner dice: "Migramos de Firebase a OneSignal"
  → Desarrollador abre pedido_service.py
  → Está al lado del código de negocio del pedido
  → Riesgo de romper la lógica de estados al tocar las notificaciones
```

**QuickBite — Después (SRP aplicado):**
```
Product Owner dice: "Migramos de Firebase a OneSignal"
  → Desarrollador abre notificacion_service.py
  → Solo hay código de notificaciones
  → PedidoService no se toca, sus tests no fallan
```


## Consecuencias

### Positivas
- `PedidoService` se puede probar con un mock simple de `NotificacionService`, sin SMTP real ni Firebase. En Odoo: `action_confirm()` se testea sin servidor de correo.
- Agregar SMS via Twilio no toca la lógica de pedidos. En Odoo: agregar WhatsApp Business no toca `action_confirm()`.
- `NotificacionService` puede reutilizarse desde `PagoService` (RF14). En Odoo: `flx.notificacion.service` se reutiliza desde el módulo de facturación.
- Cada área del negocio tiene su propio archivo para modificar: el área legal toca `flx.contrato.service`, IT toca `flx.notificacion.service`.

### Negativas / trade-offs
- Se añaden clases y dependencias adicionales. Para el volumen de QuickBite y Fiberlux está completamente justificado.
- El flujo completo involucra múltiples clases; el equipo debe conocer la separación y la convención de nombres.
