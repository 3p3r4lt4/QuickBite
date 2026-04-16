# ADR-002 — Introducir estrategias extensibles para el procesamiento de pagos

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** O — Open/Closed Principle (OCP)

---

## Contexto

Cuando un usuario realiza un pedido, el sistema procesa el cobro mediante una pasarela de pago externa. Actualmente esa lógica vive dentro de `PagoService` con una cadena de condicionales según el método de pago elegido (tarjeta, yape, plin, efectivo):

**Código actual (con el problema):**

```python
# pago_service.py — lógica de pago acoplada con condicionales
import requests


class PagoService:

    def procesar_pago(self, pedido: dict, metodo: str, datos_pago: dict) -> dict:
        total = pedido["total"]

        # ❌ Cada nuevo método de pago exige modificar este bloque
        if metodo == "CULQI_TARJETA":
            response = requests.post(
                "https://api.culqi.com/v2/charges",
                headers={"Authorization": "Bearer CULQI_KEY"},
                json={"amount": int(total * 100), "currency_code": "PEN",
                      "source_id": datos_pago["token_culqi"]},
            )
            if response.status_code != 201:
                raise Exception("Pago con tarjeta rechazado")
            return {"estado": "APROBADO", "referencia": response.json()["id"]}

        elif metodo == "STRIPE":
            response = requests.post(
                "https://api.stripe.com/v1/payment_intents",
                auth=("STRIPE_SECRET", ""),
                data={"amount": int(total * 100), "currency": "pen",
                      "payment_method": datos_pago["payment_method_id"],
                      "confirm": True},
            )
            if response.json().get("status") != "succeeded":
                raise Exception("Pago con Stripe rechazado")
            return {"estado": "APROBADO", "referencia": response.json()["id"]}

        elif metodo == "EFECTIVO":
            # Sin integración externa
            return {"estado": "PENDIENTE", "referencia": f"EFECTIVO-{pedido['id']}"}

        else:
            raise ValueError(f"Método de pago no soportado: {metodo}")
```

**¿Cuál es el problema?**

Cada vez que se agrega un nuevo método (Yape, Plin, BBVA, BCP Yape Empresas), hay que **modificar** `PagoService`. Esto:
- Rompe código que ya funciona y supera pruebas de regresión.
- Mezcla la orquestación del pago con los detalles de cada pasarela.
- Hace el archivo cada vez más grande e imposible de mantener.

---

## Decisión

Introducimos el protocolo `EstrategiaPago` con un único método `procesar()`. Cada pasarela tiene su propia clase que implementa ese protocolo. `PagoService` usa la estrategia que recibe, sin importarle cuál es.

**Código corregido:**

```python
# estrategia_pago.py — protocolo (contrato cerrado a modificación)
from abc import ABC, abstractmethod


class EstrategiaPago(ABC):
    """
    Contrato que toda pasarela de pago debe cumplir.
    Cerrado a modificación, abierto a extensión.
    """

    @abstractmethod
    def procesar(self, monto: float, datos_pago: dict) -> dict:
        """
        Procesa el cobro y retorna un dict con:
          - estado: 'APROBADO' | 'PENDIENTE' | 'RECHAZADO'
          - referencia: identificador único de la transacción
        """
        ...


# estrategia_culqi.py
import requests


class PagoCulqi(EstrategiaPago):
    """Pasarela Culqi — tarjeta de crédito/débito (RNF15)."""

    CULQI_URL = "https://api.culqi.com/v2/charges"

    def __init__(self, api_key: str):
        self.api_key = api_key

    def procesar(self, monto: float, datos_pago: dict) -> dict:
        response = requests.post(
            self.CULQI_URL,
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "amount": int(monto * 100),  # centavos
                "currency_code": "PEN",
                "source_id": datos_pago["token_culqi"],
            },
            timeout=10,
        )
        if response.status_code != 201:
            return {"estado": "RECHAZADO", "referencia": None}
        return {"estado": "APROBADO", "referencia": response.json()["id"]}


# estrategia_stripe.py
import requests


class PagoStripe(EstrategiaPago):
    """Pasarela Stripe — internacional (RNF15)."""

    STRIPE_URL = "https://api.stripe.com/v1/payment_intents"

    def __init__(self, secret_key: str):
        self.secret_key = secret_key

    def procesar(self, monto: float, datos_pago: dict) -> dict:
        response = requests.post(
            self.STRIPE_URL,
            auth=(self.secret_key, ""),
            data={
                "amount": int(monto * 100),
                "currency": "pen",
                "payment_method": datos_pago["payment_method_id"],
                "confirm": "true",
            },
            timeout=10,
        )
        resultado = response.json()
        if resultado.get("status") != "succeeded":
            return {"estado": "RECHAZADO", "referencia": None}
        return {"estado": "APROBADO", "referencia": resultado["id"]}


# estrategia_efectivo.py
class PagoEfectivo(EstrategiaPago):
    """Pago en efectivo al repartidor — sin integración externa."""

    def procesar(self, monto: float, datos_pago: dict) -> dict:
        pedido_id = datos_pago.get("pedido_id", "SIN-ID")
        return {"estado": "PENDIENTE", "referencia": f"EFECTIVO-{pedido_id}"}


# ✅ Nueva estrategia añadida sin tocar ninguna clase existente
# estrategia_yape.py
import requests


class PagoYape(EstrategiaPago):
    """Yape — billetera digital (nueva pasarela, cero modificaciones en código existente)."""

    YAPE_URL = "https://api.yape.com.pe/v1/cobros"

    def __init__(self, merchant_id: str, secret: str):
        self.merchant_id = merchant_id
        self.secret = secret

    def procesar(self, monto: float, datos_pago: dict) -> dict:
        response = requests.post(
            self.YAPE_URL,
            json={
                "merchant_id": self.merchant_id,
                "monto": monto,
                "telefono": datos_pago["telefono_yape"],
                "concepto": "Pedido QuickBite",
            },
            headers={"X-Secret": self.secret},
            timeout=10,
        )
        if response.status_code == 200 and response.json().get("aprobado"):
            return {"estado": "APROBADO", "referencia": response.json()["codigo_operacion"]}
        return {"estado": "RECHAZADO", "referencia": None}


# pago_service.py — cerrado a modificación respecto al método de cobro
class PagoService:

    def __init__(self, pago_repo, notificacion_service):
        self.pago_repo = pago_repo
        self.notificaciones = notificacion_service

    def procesar_pago(self, pedido: dict, estrategia: EstrategiaPago, datos_pago: dict) -> dict:
        """
        Orquesta el pago usando la estrategia inyectada.
        No sabe qué pasarela es — solo la usa.  ✅
        """
        resultado = estrategia.procesar(pedido["total"], datos_pago)

        transaccion = {
            "pedido_id": pedido["id"],
            "metodo": type(estrategia).__name__,
            "monto": pedido["total"],
            "estado": resultado["estado"],
            "referencia": resultado["referencia"],
        }
        self.pago_repo.guardar(transaccion)
        return transaccion


# pago_factory.py — selecciona la estrategia según el método elegido por el usuario
class PagoFactory:

    def __init__(self, culqi_key: str, stripe_key: str, yape_merchant: str, yape_secret: str):
        self._estrategias = {
            "CULQI_TARJETA": PagoCulqi(culqi_key),
            "STRIPE":        PagoStripe(stripe_key),
            "EFECTIVO":      PagoEfectivo(),
            "YAPE":          PagoYape(yape_merchant, yape_secret),
        }

    def obtener(self, metodo: str) -> EstrategiaPago:
        estrategia = self._estrategias.get(metodo)
        if not estrategia:
            raise ValueError(f"Método de pago no soportado: {metodo}")
        return estrategia
```

**¿Cómo se usa en conjunto?**

```python
# En el caso de uso / controlador
factory = PagoFactory(culqi_key=CULQI_KEY, stripe_key=STRIPE_KEY, ...)
estrategia = factory.obtener(request.data["metodo_pago"])
transaccion = pago_service.procesar_pago(pedido, estrategia, request.data["datos_pago"])
```

### Principio SOLID aplicado — OCP

> "Las entidades de software deben estar abiertas para extensión y cerradas para modificación."

**Antes:** agregar Yape → modificar `PagoService` (riesgo de regresión en Culqi y Stripe).
**Después:** agregar Yape → crear `PagoYape` (cero riesgo sobre código existente).

```
Agregar nueva pasarela de pago:
  ANTES → modificar PagoService    ← toca código ya probado en producción
  AHORA → crear nueva clase        ← no toca nada existente
```

**¿Qué está "cerrado"?** La clase `PagoService` y el protocolo `EstrategiaPago`.
**¿Qué está "abierto"?** El conjunto de implementaciones concretas de `EstrategiaPago`.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Guardar credenciales de pasarelas en BD y seleccionar con condicional | No soporta diferencias en la firma de cada API (Culqi usa token, Stripe usa PaymentMethod, Yape usa teléfono) |
| Usar herencia en lugar de composición | La herencia genera jerarquías frágiles. La composición mediante el protocolo ABC es más flexible y testeable en Python |

---

## Consecuencias

### Positivas
- Cada pasarela tiene su propio test unitario aislado y su propio mock HTTP.
- Integrar una nueva pasarela (RF08, RF16) no requiere tocar ni revisar las clases existentes.
- `PagoService` permanece estable ante cambios regulatorios o de contrato con proveedores.

### Negativas / trade-offs
- `PagoFactory` sigue siendo un punto de modificación al agregar un nuevo método. Aceptable: el cambio es una sola línea en el diccionario.
- Se crean múltiples clases pequeñas. Para el scope actual de QuickBite (RF08, RF16) está completamente justificado por la variedad de pasarelas peruanas.
