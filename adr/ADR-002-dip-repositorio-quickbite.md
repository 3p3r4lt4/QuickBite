# ADR-002 — Inyectar dependencia del servicio de pagos en el módulo QuickBite

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** D — Dependency Inversion Principle (DIP)

---

## Contexto

El módulo de pagos de QuickBite depende directamente de una implementación concreta de pago externo (`StripePaymentGateway`) en lugar de abstraer el servicio con una interfaz de dominio.

**Código actual (con el problema):**

```java
@Service
public class PaymentService {

    // ❌ Dependencia concreta al gateway de pagos
    private final StripePaymentGateway paymentGateway;

    public PaymentService(StripePaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        return paymentGateway.charge(request);
    }
}
```

**¿Cuál es el problema?**

`PaymentService` (lógica de negocio) depende de `StripePaymentGateway`. Esto provoca:

- Acoplamiento directo al proveedor de pagos.
- Cambios en el proveedor de pago obligan a modificar la capa de negocio.

---

## Decisión

Definimos una abstracción propia `PaymentGateway` en la capa de dominio y hacemos que `PaymentService` dependa solo de ella. La implementación concreta de Stripe se sitúa en infraestructura y es inyectada por Spring como adaptador.

**Código corregido:**

```java
// PaymentGateway.java — interfaz propia en la capa de dominio
public interface PaymentGateway {

    PaymentResult charge(PaymentRequest request);
}


// PaymentService.java — depende únicamente de la abstracción del dominio
@Service
public class PaymentService {

    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;

    public PaymentService(PaymentGateway paymentGateway,
                          NotificationService notificationService) {
        this.paymentGateway = paymentGateway;
        this.notificationService = notificationService;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        PaymentResult result = paymentGateway.charge(request);
        if (result.isSuccess()) {
            notificationService.notifyPaymentSuccess(request.getOrderId());
        }
        return result;
    }
}
```

```java
// StripePaymentGateway.java — adaptador para Stripe
@Service
public class StripePaymentGateway implements PaymentGateway {

    private final StripeClient stripeClient;

    public StripePaymentGateway(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }

    @Override
    public PaymentResult charge(PaymentRequest request) {
        StripeChargeResponse response = stripeClient.charge(request.getAmount(), request.getCardToken());
        return new PaymentResult(response.isSuccessful(), response.getTransactionId(), response.getErrorMessage());
    }
}
```



### Principio SOLID aplicado — DIP

> "Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones."

**Dirección de dependencias:**

```
ANTES (incorrecta):
  PaymentService  ──→ StripePaymentGateway 

DESPUÉS (correcta):
  PaymentService     ──→ PaymentGateway  (abstracción del dominio)
  StripePaymentGateway ──→ PaymentGateway  (abstracción del dominio)
```

El servicio de pago y el adaptador Stripe dependen de la misma abstracción, permitiendo cambiar la tecnología sin afectar la lógica de negocio.

## Consecuencias

### Positivas
- `PaymentService` se vuelve independiente del proveedor de pagos.
- Cambiar de Stripe a otro proveedor solo requiere implementar `PaymentGateway` y registrar el bean correspondiente.
- La lógica de negocio mantiene una dirección de dependencia correcta.

### Negativas / trade-offs
- Se agrega una capa de abstracción extra para el módulo de pagos.

---

## Diagrama de capas resultante

```
┌──────────────────────────────────────────────┐
│             Capa de Dominio                  │
│                                              │
│   PaymentService                             │
│        │                                     │
│        └──→  PaymentGateway  (interfaz)      │
└─────────────────────┬────────────────────────┘
                      │ implementa
┌─────────────────────▼────────────────────────┐
│          Capa de Infraestructura             │
│                                              │
│   StripePaymentGateway                       │
│            │
└──────────────────────────────────────────────┘
```
