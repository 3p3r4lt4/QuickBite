# ADR-001 — Separar el envío de notificaciones del servicio de pedidos

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** S — Single Responsibility Principle (SRP)

---

## Contexto

El `PedidoService` es responsable de registrar pedidos y, además, de enviar notificaciones al usuario (push, email, SMS) cada vez que el estado del pedido cambia (recibido → en preparación → en camino → entregado).

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
                "notification": {"title": "Pedido recibido", "body": f"Total: S/ {carrito['total']}"},
            },
        )
        return pedido_guardado

    def actualizar_estado(self, pedido_id, nuevo_estado):
        pedido = self.pedido_repo.actualizar_estado(pedido_id, nuevo_estado)

        # ❌ Lógica de notificación duplicada aquí también
        smtp = smtplib.SMTP("smtp.quickbite.pe", 587)
        # ... configuración SMTP repetida ...
        smtp.quit()

        return pedido
```

**¿Cuál es el problema?**

`PedidoService` tiene **dos razones para cambiar**:
- Si cambia la lógica de negocio del pedido (flujo de estados, validaciones de stock).
- Si cambia el canal de notificación (de Firebase a OneSignal, o se agrega Twilio SMS).

Esto también dificulta las pruebas: para testear `crear_pedido` hay que mockear SMTP, Firebase y la lógica de negocio al mismo tiempo.

---

## Decisión

Extraemos el envío de notificaciones a una clase dedicada `NotificacionService`. `PedidoService` delega en ella sin conocer los detalles del canal de envío.

**Código corregido:**

```python
# notificacion_service.py — responsabilidad única: enviar notificaciones
import smtplib
import requests


class NotificacionService:

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
from datetime import datetime


class PedidoService:

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
        self.notificaciones.notificar_pedido_recibido(usuario, pedido_guardado)
        return pedido_guardado

    def actualizar_estado(self, pedido_id: int, nuevo_estado: str, usuario: dict) -> dict:
        pedido = self.pedido_repo.actualizar_estado(pedido_id, nuevo_estado)

        # ✅ Delega el canal de comunicación al servicio especializado
        self.notificaciones.notificar_cambio_estado(usuario, pedido, nuevo_estado)
        return pedido
```

### Principio SOLID aplicado — SRP

> "Un módulo debe tener una, y solo una, razón para cambiar."

| Clase | Única razón de cambio |
|-------|----------------------|
| `PedidoService` | Reglas de negocio del pedido (estados, validaciones, flujo) |
| `NotificacionService` | Canales o contenido de las notificaciones (Firebase, Twilio, SendGrid) |

**Antes:** migrar de Firebase a OneSignal obligaba a modificar `PedidoService`.
**Después:** ese cambio solo afecta a `NotificacionService`.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Dejar todo en `PedidoService` y extraer solo la config SMTP/Firebase a variables de entorno | El problema persiste: la clase sigue teniendo dos razones de cambio |
| Usar eventos de dominio (Celery tasks) para desacoplar completamente | Válido a futuro como mejora, pero añade complejidad innecesaria en esta etapa |

---

## Consecuencias

### Positivas
- `PedidoService` se puede probar con un mock simple de `NotificacionService`, sin SMTP real ni Firebase.
- Agregar SMS via Twilio no toca la lógica de pedidos.
- `NotificacionService` puede reutilizarse desde `PagoService` (RF14: notificaciones de pago).

### Negativas / trade-offs
- Se añade una clase y una dependencia. Para el volumen de QuickBite está más que justificado.
- El flujo completo involucra dos clases; el equipo debe conocer la separación.
