# ADR-004 — Garantizar que todos los tipos de entrega son intercambiables en el sistema de seguimiento

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** L — Liskov Substitution Principle (LSP)

---

## Contexto

QuickBite maneja tres modalidades de entrega: **delivery a domicilio** (repartidor con GPS), **retiro en tienda** (el cliente va al restaurante) y **delivery express** (servicio premium con ruta optimizada). El equipo modeló esta jerarquía con herencia, haciendo que todas extendieran la clase `Entrega`:

**Código actual (con el problema):**

```python
# entrega.py — clase base
class Entrega:

    def __init__(self, pedido_id: int, direccion: str):
        self.pedido_id = pedido_id
        self.direccion = direccion
        self.estado = "PENDIENTE"

    def asignar_repartidor(self, repartidor_id: int) -> None:
        """Asigna un repartidor y actualiza la ruta GPS."""
        self.repartidor_id = repartidor_id
        print(f"Repartidor {repartidor_id} asignado al pedido {self.pedido_id}")

    def obtener_eta(self) -> int:
        """Retorna el tiempo estimado de llegada en minutos."""
        return 30  # estimado base

    def rastrear_posicion(self) -> dict:
        """Retorna la posición GPS del repartidor en tiempo real."""
        return {"lat": -12.046374, "lng": -77.042793}


# retiro_en_tienda.py
class RetiroEnTienda(Entrega):

    def asignar_repartidor(self, repartidor_id: int) -> None:
        # ❌ El retiro en tienda no tiene repartidor
        raise NotImplementedError("El retiro en tienda no requiere repartidor")

    def rastrear_posicion(self) -> dict:
        # ❌ No hay posición GPS que rastrear
        raise NotImplementedError("El retiro en tienda no tiene seguimiento GPS")


# delivery_express.py
class DeliveryExpress(Entrega):

    def obtener_eta(self) -> int:
        # ❌ Express tiene un algoritmo propio — pero rompemos el contrato
        # porque a veces retorna -1 cuando no hay repartidor disponible
        if not hasattr(self, "repartidor_id"):
            return -1  # ❌ viola la postcondición: debe ser un entero positivo
        return 15


# seguimiento_service.py — cliente que usa la jerarquía
class SeguimientoService:

    def mostrar_eta(self, entrega: Entrega) -> str:
        # ❌ Debe verificar el tipo concreto para evitar errores
        if isinstance(entrega, RetiroEnTienda):
            return "Recoge en tienda cuando esté listo"
        eta = entrega.obtener_eta()
        if eta == -1:  # ❌ conoce el comportamiento especial de DeliveryExpress
            return "Sin repartidor disponible aún"
        return f"Llega en aproximadamente {eta} minutos"

    def iniciar_seguimiento(self, entrega: Entrega) -> None:
        # ❌ Explota si la entrega es RetiroEnTienda
        posicion = entrega.rastrear_posicion()
        print(f"Posición actual: {posicion}")
```

**¿Cuál es el problema?**

`SeguimientoService` trata todas las entregas de forma uniforme, pero algunas subclases lanzan excepciones o retornan valores fuera del contrato prometido por la clase base:

- `RetiroEnTienda.asignar_repartidor()` lanza `NotImplementedError` → viola el contrato.
- `DeliveryExpress.obtener_eta()` puede retornar `-1` → viola la postcondición (debe ser entero ≥ 0).

El código cliente acumula `isinstance` defensivos, que es la señal clara de una violación LSP.

---

## Decisión

Reestructuramos la jerarquía separando las capacidades en **protocolos independientes**. Cada tipo de entrega implementa solo los protocolos que genuinamente soporta. `SeguimientoService` trabaja con los protocolos, no con la clase base.

**Código corregido:**

```python
# protocolos_entrega.py — capacidades separadas por protocolo
from abc import ABC, abstractmethod


class EntregaBase(ABC):
    """Contrato base: solo lo que TODAS las modalidades de entrega comparten."""

    @abstractmethod
    def obtener_estado(self) -> str:
        """Retorna el estado actual: PENDIENTE | EN_PREPARACION | LISTO | ENTREGADO."""
        ...

    @abstractmethod
    def cancelar(self) -> bool:
        """Cancela la entrega. Retorna True si fue posible cancelar."""
        ...


class EntregaRastreable(ABC):
    """Capacidad de seguimiento GPS en tiempo real (RF10, RF11)."""

    @abstractmethod
    def rastrear_posicion(self) -> dict:
        """Retorna {'lat': float, 'lng': float} con la posición actual del repartidor."""
        ...

    @abstractmethod
    def obtener_eta(self) -> int:
        """
        Retorna el tiempo estimado de llegada en minutos.
        Postcondición garantizada: valor siempre >= 0.
        """
        ...


class EntregaAsignable(ABC):
    """Capacidad de ser asignada a un repartidor (RF12)."""

    @abstractmethod
    def asignar_repartidor(self, repartidor_id: int) -> None:
        """Asigna el repartidor disponible al pedido."""
        ...

    @abstractmethod
    def liberar_repartidor(self) -> None:
        """Libera al repartidor cuando finaliza o cancela."""
        ...


# delivery_domicilio.py — tiene GPS, tiene repartidor
class DeliveryDomicilio(EntregaBase, EntregaRastreable, EntregaAsignable):
    """Entrega estándar a domicilio con repartidor y seguimiento GPS."""

    def __init__(self, pedido_id: int, direccion: str, gps_service):
        self.pedido_id = pedido_id
        self.direccion = direccion
        self.gps_service = gps_service
        self._estado = "PENDIENTE"
        self._repartidor_id = None

    def obtener_estado(self) -> str:
        return self._estado

    def cancelar(self) -> bool:
        if self._estado not in ("PENDIENTE", "EN_PREPARACION"):
            return False
        self._estado = "CANCELADO"
        return True

    def rastrear_posicion(self) -> dict:
        # ✅ Siempre retorna coordenadas válidas
        return self.gps_service.posicion_actual(self._repartidor_id)

    def obtener_eta(self) -> int:
        # ✅ Postcondición garantizada: siempre >= 0
        minutos = self.gps_service.calcular_eta(self._repartidor_id, self.direccion)
        return max(0, minutos)

    def asignar_repartidor(self, repartidor_id: int) -> None:
        self._repartidor_id = repartidor_id
        self._estado = "EN_CAMINO"

    def liberar_repartidor(self) -> None:
        self._repartidor_id = None


# delivery_express.py — tiene GPS y repartidor, ETA garantizado
class DeliveryExpress(EntregaBase, EntregaRastreable, EntregaAsignable):
    """Entrega express con ruta optimizada y ETA reducido (≤ 20 min)."""

    ETA_MAXIMO = 20

    def __init__(self, pedido_id: int, direccion: str, gps_service, route_optimizer):
        self.pedido_id = pedido_id
        self.direccion = direccion
        self.gps_service = gps_service
        self.route_optimizer = route_optimizer
        self._estado = "PENDIENTE"
        self._repartidor_id = None

    def obtener_estado(self) -> str:
        return self._estado

    def cancelar(self) -> bool:
        if self._estado == "ENTREGADO":
            return False
        self._estado = "CANCELADO"
        return True

    def rastrear_posicion(self) -> dict:
        return self.gps_service.posicion_actual(self._repartidor_id)

    def obtener_eta(self) -> int:
        # ✅ Postcondición garantizada: siempre entre 0 y ETA_MAXIMO
        ruta = self.route_optimizer.ruta_optima(self._repartidor_id, self.direccion)
        return min(max(0, ruta.minutos_estimados), self.ETA_MAXIMO)

    def asignar_repartidor(self, repartidor_id: int) -> None:
        self._repartidor_id = repartidor_id
        self._estado = "EN_CAMINO"

    def liberar_repartidor(self) -> None:
        self._repartidor_id = None


# retiro_en_tienda.py — NO es rastreable, NO es asignable a repartidor
class RetiroEnTienda(EntregaBase):
    """
    Modalidad de retiro presencial.
    No implementa EntregaRastreable ni EntregaAsignable porque no aplica.
    ✅ No lanza excepciones — simplemente no declara esas capacidades.
    """

    def __init__(self, pedido_id: int, restaurante_id: int):
        self.pedido_id = pedido_id
        self.restaurante_id = restaurante_id
        self._estado = "PENDIENTE"

    def obtener_estado(self) -> str:
        return self._estado

    def cancelar(self) -> bool:
        if self._estado == "ENTREGADO":
            return False
        self._estado = "CANCELADO"
        return True


# seguimiento_service.py — trabaja con protocolos, sin isinstance defensivos
class SeguimientoService:

    def mostrar_eta(self, entrega: EntregaBase) -> str:
        # ✅ Solo actúa si la entrega genuinamente soporta rastreo
        if isinstance(entrega, EntregaRastreable):
            eta = entrega.obtener_eta()
            return f"Llega en aproximadamente {eta} minutos"
        return "Recoge en tienda cuando tu pedido esté listo"

    def iniciar_seguimiento(self, entrega: EntregaBase) -> dict | None:
        # ✅ No lanza excepción: pregunta si puede, no asume que puede
        if isinstance(entrega, EntregaRastreable):
            return entrega.rastrear_posicion()
        return None  # retiro en tienda no tiene seguimiento

    def asignar_repartidor(self, entrega: EntregaBase, repartidor_id: int) -> bool:
        # ✅ Solo asigna si la modalidad lo soporta
        if isinstance(entrega, EntregaAsignable):
            entrega.asignar_repartidor(repartidor_id)
            return True
        return False


# --- PRUEBAS UNITARIAS SIN EXCEPCIONES INESPERADAS ---

# test_lsp_entrega.py
def test_delivery_domicilio_cumple_contrato_rastreable(mock_gps):
    entrega = DeliveryDomicilio(pedido_id=1, direccion="Av. Larco 123", gps_service=mock_gps)
    entrega.asignar_repartidor(42)

    eta = entrega.obtener_eta()
    assert eta >= 0, "ETA debe ser siempre >= 0"  # ✅ postcondición garantizada

    posicion = entrega.rastrear_posicion()
    assert "lat" in posicion and "lng" in posicion  # ✅ estructura correcta


def test_retiro_en_tienda_no_lanza_excepcion():
    entrega = RetiroEnTienda(pedido_id=2, restaurante_id=10)

    # ✅ No lanza excepción: simplemente no implementa los protocolos
    assert not isinstance(entrega, EntregaRastreable)
    assert not isinstance(entrega, EntregaAsignable)
    assert entrega.obtener_estado() == "PENDIENTE"


def test_delivery_express_eta_siempre_positivo(mock_gps, mock_optimizer):
    entrega = DeliveryExpress(
        pedido_id=3, direccion="Jr. de la Unión 500",
        gps_service=mock_gps, route_optimizer=mock_optimizer,
    )
    entrega.asignar_repartidor(99)
    assert entrega.obtener_eta() >= 0  # ✅ ya no retorna -1
```

### Principio SOLID aplicado — LSP

> "Los subtipos deben poder sustituir a sus tipos base sin alterar la corrección del programa."
> — Barbara Liskov, 1987

**Antes:** sustituir `Entrega` por `RetiroEnTienda` lanzaba `NotImplementedError`; `DeliveryExpress` rompía la postcondición retornando `-1`.

**Después:** cada subtipo cumple completamente el contrato de los protocolos que declara implementar.

```
ANTES:
  Entrega.rastrear_posicion()  → RetiroEnTienda lanza NotImplementedError  ❌
  Entrega.obtener_eta()        → DeliveryExpress retorna -1                ❌

DESPUÉS:
  EntregaRastreable.rastrear_posicion() → solo quien puede hacerlo lo declara  ✅
  EntregaRastreable.obtener_eta()       → garantizado >= 0 en todos los impl.  ✅
```

**Señal de alerta LSP en QuickBite:** `isinstance` en cascada en `SeguimientoService`, métodos que lanzan `NotImplementedError`, y retornos centinela (`-1`, `None`) donde se esperaba un valor real.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Verificar el tipo con `isinstance` antes de cada operación en el servicio | Funciona pero obliga al cliente a conocer los tipos concretos; el problema LSP se traslada al código cliente |
| Una clase base con métodos opcionales que retornan `None` por defecto | Los `None` silenciosos son peor que las excepciones; producen errores difíciles de rastrear en producción |

---

## Consecuencias

### Positivas
- Ninguna operación lanza `NotImplementedError`. El contrato siempre se cumple.
- `SeguimientoService` puede trabajar con cualquier modalidad sin condiciones defensivas extrañas.
- Añadir una nueva modalidad (ej. `DroneDelivery`) solo requiere decidir qué protocolos implementa.
- Las pruebas son predecibles: no hay rutas de excepción ocultas (RF10, RF11).

### Negativas / trade-offs
- La jerarquía se vuelve más horizontal (más protocolos ABC). Requiere mayor atención al diseño inicial.
- Los `isinstance` en `SeguimientoService` siguen siendo necesarios para el despacho; se puede eliminar con el patrón Visitor si la jerarquía crece significativamente.
