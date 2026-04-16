````md
# ADR-003: OCP en módulo Repartidores — Asignación de pedidos

## Estado

Propuesto

---

## Contexto

El módulo **Repartidores** de QuickBite administra la disponibilidad de los repartidores y la asignación de pedidos generados por los clientes.

Actualmente, cuando un pedido queda listo para despacho, el sistema selecciona un repartidor usando una única clase con lógica condicional embebida.

Código actual:

```java
public class AsignadorRepartidores {

    public Repartidor asignar(String tipo, List<Repartidor> repartidores) {

        if (tipo.equals("DISTANCIA")) {
            return repartidorMasCercano(repartidores);

        } else if (tipo.equals("RATING")) {
            return repartidorMejorCalificado(repartidores);

        } else if (tipo.equals("CARGA")) {
            return repartidorMenorCarga(repartidores);

        } else {
            throw new RuntimeException("Tipo no soportado");
        }
    }
}
````

Este diseño presenta problemas:

* Cada nueva regla requiere modificar la clase principal.
* Aumenta el número de condicionales.
* Mayor riesgo de introducir errores.
* Baja mantenibilidad.
* Difícil realizar pruebas unitarias aisladas.

El negocio indicó que próximamente se agregarán nuevas reglas como:

* Repartidor motorizado.
* Repartidor premium.
* Repartidor con mayor tasa de aceptación.
* Repartidor ecológico (bicicleta).
* Prioridad nocturna.

Por ello se requiere aplicar **Open/Closed Principle (OCP)**.

---

## Decisión

Se reemplazará la lógica condicional por el patrón **Strategy**.

Se definirá una interfaz común para los criterios de asignación:

```java
public interface EstrategiaAsignacion {
    Repartidor asignar(List<Repartidor> repartidores);
}
```

Cada regla será una clase independiente.

---

## Implementación propuesta

### Estrategia por distancia

```java
public class AsignacionPorDistancia implements EstrategiaAsignacion {

    public Repartidor asignar(List<Repartidor> repartidores) {
        return repartidores.stream()
            .min(Comparator.comparing(Repartidor::getDistancia))
            .orElse(null);
    }
}
```

### Estrategia por rating

```java
public class AsignacionPorRating implements EstrategiaAsignacion {

    public Repartidor asignar(List<Repartidor> repartidores) {
        return repartidores.stream()
            .max(Comparator.comparing(Repartidor::getRating))
            .orElse(null);
    }
}
```

### Clase principal refactorizada

```java
public class AsignadorRepartidores {

    private EstrategiaAsignacion estrategia;

    public AsignadorRepartidores(EstrategiaAsignacion estrategia) {
        this.estrategia = estrategia;
    }

    public Repartidor asignar(List<Repartidor> repartidores) {
        return estrategia.asignar(repartidores);
    }
}
```

### Uso

```java
EstrategiaAsignacion estrategia =
        new AsignacionPorDistancia();

AsignadorRepartidores service =
        new AsignadorRepartidores(estrategia);

Repartidor r = service.asignar(lista);
```

---

## Consecuencias

### Positivas

* Nuevas reglas se agregan sin modificar código existente.
* Menor acoplamiento.
* Código más limpio.
* Mejor testabilidad.
* Mayor escalabilidad.

### Negativas

* Incremento de clases.
* Requiere conocer patrón Strategy.
* Puede necesitar fábrica de estrategias en el futuro.

---

## Ejemplo de extensión futura

Nueva necesidad del negocio:

```java
public class AsignacionMotorizado
        implements EstrategiaAsignacion {
}
```

No se modifica la clase principal.

---

## Principio aplicado

**Open/Closed Principle**

> Las clases deben estar abiertas para extensión, pero cerradas para modificación.

```
```
