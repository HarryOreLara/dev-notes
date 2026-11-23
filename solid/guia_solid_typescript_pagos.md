# SOLID en TypeScript: diseño de un sistema de pagos

## Objetivo

Esta guía explica los cinco principios SOLID mediante un mismo contexto:
una aplicación que procesa pagos por distintos medios, como tarjeta,
transferencia bancaria y billetera digital.

Los ejemplos utilizan TypeScript y clases sencillas. La idea es entender
**qué problema resuelve cada principio, cómo reconocer una mala
implementación y cómo refactorizarla**. El código es ilustrativo: en una
aplicación real habría que agregar validaciones, persistencia, manejo de
errores, seguridad e integración con los proveedores de pago.

## El escenario

Imagina que desarrollamos un backend para una tienda. Cuando un cliente
compra, el sistema debe:

1.  Validar el importe.
2.  Procesar el pago con el medio elegido.
3.  Registrar el resultado.
4.  Enviar una notificación al cliente.

Al inicio podríamos escribir todo en una sola clase. Eso parece
sencillo, pero con el tiempo aparecen muchos medios de pago, reglas y
servicios externos. SOLID ayuda a organizar esas responsabilidades y
dependencias.

------------------------------------------------------------------------

# S --- Single Responsibility Principle (SRP)

## Principio de responsabilidad única

**Una clase debe tener una sola responsabilidad y, por tanto, una sola
razón principal para cambiar.**

No significa que una clase solo pueda tener un método. Significa que sus
métodos deben pertenecer a una misma responsabilidad.

### Una implementación problemática

``` typescript
class PaymentService {
  process(amount: number, cardNumber: string): void {
    // Validar el importe
    if (amount <= 0) {
      throw new Error('Importe inválido');
    }

    // Procesar el pago con tarjeta
    console.log(`Cobrar ${amount} a la tarjeta ${cardNumber}`);

    // Guardar el pago en la base de datos
    console.log('Guardando pago en la base de datos');

    // Enviar correo al cliente
    console.log('Enviando correo de confirmación');
  }
}
```

Esta clase mezcla varias responsabilidades:

-   Validación de reglas del pago.
-   Ejecución del cobro.
-   Persistencia de información.
-   Notificación al cliente.

Si cambia la forma de enviar correos, tendríamos que modificar
`PaymentService`. Si cambia la base de datos, también. Si cambia la
regla de validación, nuevamente habría que modificarla.

La clase tiene demasiadas razones para cambiar.

### Una implementación aplicando SRP

Separamos cada responsabilidad:

``` typescript
class PaymentValidator {
  validate(amount: number): void {
    if (amount <= 0) {
      throw new Error('El importe debe ser mayor que cero');
    }
  }
}

class PaymentRepository {
  save(paymentId: string, amount: number): void {
    console.log(`Guardando ${paymentId}: ${amount}`);
  }
}

class PaymentNotifier {
  sendConfirmation(email: string): void {
    console.log(`Enviando confirmación a ${email}`);
  }
}

class PaymentProcessor {
  process(amount: number): string {
    console.log(`Procesando pago por ${amount}`);
    return 'payment-123';
  }
}
```

Un servicio de aplicación puede coordinar estas piezas:

``` typescript
class PaymentService {
  constructor(
    private readonly validator: PaymentValidator,
    private readonly processor: PaymentProcessor,
    private readonly repository: PaymentRepository,
    private readonly notifier: PaymentNotifier,
  ) {}

  execute(amount: number, email: string): void {
    this.validator.validate(amount);

    const paymentId = this.processor.process(amount);

    this.repository.save(paymentId, amount);

    this.notifier.sendConfirmation(email);
  }
}
```

`PaymentService` todavía coordina el caso de uso de pagar, pero no
implementa por sí mismo los detalles de validación, cobro, persistencia
o notificación.

### ¿Cuándo aplicar SRP?

Cuando una clase acumula tareas de naturaleza distinta o cambia por
motivos que pertenecen a áreas diferentes.

**Idea clave:** separar responsabilidades relacionadas, no crear una
clase por cada línea de código. Evita tanto las clases gigantes como la
fragmentación innecesaria.

------------------------------------------------------------------------

# O --- Open/Closed Principle (OCP)

## Principio abierto/cerrado

**Las entidades de software deben estar abiertas a la extensión, pero
cerradas a la modificación.**

En otras palabras, deberíamos poder incorporar nuevos comportamientos
sin tener que reescribir continuamente el código estable que ya
funciona.

### Una implementación problemática

``` typescript
type PaymentMethod = 'CARD' | 'BANK_TRANSFER' | 'WALLET';

class PaymentService {
  pay(method: PaymentMethod, amount: number): void {
    if (method === 'CARD') {
      console.log(`Procesando tarjeta: ${amount}`);
    } else if (method === 'BANK_TRANSFER') {
      console.log(`Procesando transferencia: ${amount}`);
    } else if (method === 'WALLET') {
      console.log(`Procesando billetera digital: ${amount}`);
    }
  }
}
```

¿Qué sucede si mañana agregamos un pago con QR?

Debemos modificar `PaymentMethod`, agregar otra condición y volver a
modificar `PaymentService`. Con cada nuevo medio, esa clase crece y
aumenta el riesgo de afectar los métodos existentes.

### Una implementación aplicando OCP

Definimos un contrato común:

``` typescript
interface PaymentMethod {
  pay(amount: number): void;
}
```

Creamos una implementación por medio de pago:

``` typescript
class CardPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Procesando pago con tarjeta: ${amount}`);
  }
}

class BankTransferPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Procesando transferencia bancaria: ${amount}`);
  }
}

class WalletPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Procesando pago con billetera digital: ${amount}`);
  }
}
```

El servicio trabaja con la abstracción:

``` typescript
class PaymentService {
  execute(paymentMethod: PaymentMethod, amount: number): void {
    paymentMethod.pay(amount);
  }
}
```

Ahora podemos añadir otro medio sin modificar `PaymentService`:

``` typescript
class QrPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Procesando pago con QR: ${amount}`);
  }
}
```

Uso:

``` typescript
const service = new PaymentService();

service.execute(new CardPayment(), 100);
service.execute(new BankTransferPayment(), 200);
service.execute(new QrPayment(), 150);
```

### ¿Qué conseguimos?

`PaymentService` no necesita saber si está trabajando con tarjeta,
transferencia, billetera o QR. Solo necesita que el objeto cumpla el
contrato `PaymentMethod`.

### ¿Cuándo aplicar OCP?

Cuando se prevé que una parte del sistema tendrá nuevas variantes:
medios de pago, reglas de descuentos, métodos de envío, proveedores o
formatos.

**Idea clave:** extender mediante nuevas implementaciones de
abstracciones, en lugar de agregar condiciones por cada variante. No
significa que nunca debamos modificar código; significa evitar
modificaciones repetitivas en componentes que deberían permanecer
estables.

------------------------------------------------------------------------

# L --- Liskov Substitution Principle (LSP)

## Principio de sustitución de Liskov

**Los objetos de una clase derivada deben poder sustituir a los objetos
de su clase base sin alterar el comportamiento correcto esperado por el
programa.**

No basta con que una clase hija pueda heredar de otra desde el punto de
vista sintáctico. También debe respetar el contrato y las expectativas
de la clase base.

### Un ejemplo de diseño incorrecto

Supongamos que definimos este contrato:

``` typescript
abstract class Payment {
  abstract pay(amount: number): void;

  refund(paymentId: string): void {
    console.log(`Reembolsando pago ${paymentId}`);
  }
}
```

Parece que todos los pagos pueden procesarse y reembolsarse. Pero
imaginemos que una modalidad de transferencia de nuestro sistema no
admite reembolsos automáticos mediante el proveedor.

``` typescript
class NonRefundableTransfer extends Payment {
  pay(amount: number): void {
    console.log(`Procesando transferencia: ${amount}`);
  }

  refund(paymentId: string): void {
    throw new Error('Este medio no permite reembolsos');
  }
}
```

El problema es que el contrato de `Payment` promete que `refund()` está
disponible, pero una subclase rompe esa expectativa lanzando un error.

Un consumidor que recibe cualquier `Payment` podría asumir que puede
llamar a `refund()`. Al recibir `NonRefundableTransfer`, ese supuesto
deja de ser válido.

### Una implementación más adecuada

Separamos las capacidades:

``` typescript
interface Payable {
  pay(amount: number): void;
}

interface Refundable {
  refund(paymentId: string): void;
}
```

Un pago con tarjeta puede implementar ambas capacidades:

``` typescript
class CardPayment implements Payable, Refundable {
  pay(amount: number): void {
    console.log(`Cobro con tarjeta: ${amount}`);
  }

  refund(paymentId: string): void {
    console.log(`Reembolso de tarjeta: ${paymentId}`);
  }
}
```

Una transferencia que no admite reembolso automático implementa
solamente `Payable`:

``` typescript
class BankTransferPayment implements Payable {
  pay(amount: number): void {
    console.log(`Transferencia bancaria: ${amount}`);
  }
}
```

Y cada consumidor declara lo que realmente necesita:

``` typescript
class PaymentExecutor {
  execute(payment: Payable, amount: number): void {
    payment.pay(amount);
  }
}

class RefundService {
  execute(payment: Refundable, paymentId: string): void {
    payment.refund(paymentId);
  }
}
```

Ahora `RefundService` no acepta accidentalmente un medio de pago que no
ofrece esa capacidad.

### ¿Cuándo aplicar LSP?

Cuando utilizamos herencia o implementaciones intercambiables y una
clase derivada cambia las reglas, lanza errores inesperados o no cumple
lo prometido por su abstracción.

**Idea clave:** una implementación debe respetar el contrato de aquello
que sustituye. No fuerces a todas las variantes a ofrecer capacidades
que algunas no pueden cumplir.

------------------------------------------------------------------------

# I --- Interface Segregation Principle (ISP)

## Principio de segregación de interfaces

**Los clientes no deben verse obligados a depender de métodos que no
utilizan.**

Es preferible tener contratos pequeños y específicos en lugar de una
interfaz enorme que obligue a todas las clases a implementar operaciones
innecesarias.

### Una implementación problemática

``` typescript
interface PaymentOperations {
  pay(amount: number): void;
  refund(paymentId: string): void;
  generateReceipt(paymentId: string): void;
  cancel(paymentId: string): void;
}
```

Imaginemos que una transferencia bancaria de nuestro sistema puede
realizar pagos, pero no ofrece cancelación ni reembolso automático.

Si implementa esta interfaz, tendrá que declarar métodos que no puede
soportar:

``` typescript
class BankTransfer implements PaymentOperations {
  pay(amount: number): void {
    console.log(`Transferencia: ${amount}`);
  }

  refund(paymentId: string): void {
    throw new Error('No soportado');
  }

  generateReceipt(paymentId: string): void {
    console.log(`Generando comprobante: ${paymentId}`);
  }

  cancel(paymentId: string): void {
    throw new Error('No soportado');
  }
}
```

La interfaz obliga a implementar capacidades que no corresponden a esa
clase.

### Una implementación aplicando ISP

Dividimos el contrato en interfaces específicas:

``` typescript
interface Payable {
  pay(amount: number): void;
}

interface Refundable {
  refund(paymentId: string): void;
}

interface ReceiptGeneratable {
  generateReceipt(paymentId: string): void;
}

interface Cancellable {
  cancel(paymentId: string): void;
}
```

Cada medio implementa únicamente lo que necesita:

``` typescript
class CardPayment
  implements Payable, Refundable, ReceiptGeneratable, Cancellable {

  pay(amount: number): void {
    console.log(`Cobro con tarjeta: ${amount}`);
  }

  refund(paymentId: string): void {
    console.log(`Reembolso: ${paymentId}`);
  }

  generateReceipt(paymentId: string): void {
    console.log(`Comprobante: ${paymentId}`);
  }

  cancel(paymentId: string): void {
    console.log(`Cancelación: ${paymentId}`);
  }
}

class BankTransfer implements Payable, ReceiptGeneratable {
  pay(amount: number): void {
    console.log(`Transferencia: ${amount}`);
  }

  generateReceipt(paymentId: string): void {
    console.log(`Comprobante de transferencia: ${paymentId}`);
  }
}
```

### Diferencia entre ISP y LSP

Estos dos principios están relacionados, pero no son iguales:

-   **ISP:** evita diseñar contratos demasiado grandes que obliguen a
    implementar métodos innecesarios.
-   **LSP:** garantiza que las implementaciones respeten el
    comportamiento prometido por sus contratos.

En el ejemplo, ISP nos ayuda a dividir `PaymentOperations`. LSP nos
ayuda a asegurar que cada clase que implementa `Payable` o `Refundable`
cumpla correctamente lo que esos contratos prometen.

### ¿Cuándo aplicar ISP?

Cuando una interfaz tiene muchos métodos y algunas implementaciones
terminan lanzando errores de "no soportado", dejando métodos vacíos o
implementando operaciones que no tienen sentido para ellas.

**Idea clave:** contratos pequeños, enfocados y coherentes.

------------------------------------------------------------------------

# D --- Dependency Inversion Principle (DIP)

## Principio de inversión de dependencias

**Los módulos de alto nivel no deben depender de módulos de bajo nivel.
Ambos deben depender de abstracciones. Las abstracciones no deben
depender de los detalles; los detalles deben depender de las
abstracciones.**

Este principio es especialmente importante en aplicaciones backend y en
NestJS.

### Una implementación problemática

Imaginemos que nuestro servicio de pagos depende directamente de una
implementación concreta de Niubiz:

``` typescript
class NiubizGateway {
  charge(amount: number): void {
    console.log(`Cobrando ${amount} mediante Niubiz`);
  }
}

class PaymentService {
  private readonly gateway = new NiubizGateway();

  execute(amount: number): void {
    this.gateway.charge(amount);
  }
}
```

¿Qué sucede si mañana queremos utilizar otro proveedor, por ejemplo
Stripe?

`PaymentService` está acoplado directamente a `NiubizGateway`. Para
cambiar de proveedor debemos modificar el servicio.

Además, probarlo de forma aislada resulta más difícil porque crea
internamente su dependencia concreta.

### Una implementación aplicando DIP

Primero definimos una abstracción:

``` typescript
interface PaymentGateway {
  charge(amount: number): void;
}
```

Después, cada proveedor implementa el contrato:

``` typescript
class NiubizGateway implements PaymentGateway {
  charge(amount: number): void {
    console.log(`Cobro mediante Niubiz: ${amount}`);
  }
}

class StripeGateway implements PaymentGateway {
  charge(amount: number): void {
    console.log(`Cobro mediante Stripe: ${amount}`);
  }
}
```

Nuestro servicio depende de la abstracción:

``` typescript
class PaymentService {
  constructor(
    private readonly gateway: PaymentGateway
  ) {}

  execute(amount: number): void {
    this.gateway.charge(amount);
  }
}
```

Podemos elegir el proveedor desde fuera:

``` typescript
const niubizService = new PaymentService(new NiubizGateway());
niubizService.execute(100);

const stripeService = new PaymentService(new StripeGateway());
stripeService.execute(200);
```

`PaymentService` ya no conoce los detalles de Niubiz ni Stripe. Solo
conoce `PaymentGateway`.

### ¿Por qué es importante para las pruebas?

Podemos inyectar una implementación falsa:

``` typescript
class FakePaymentGateway implements PaymentGateway {
  charge(amount: number): void {
    console.log(`Simulando cobro: ${amount}`);
  }
}

const testService = new PaymentService(new FakePaymentGateway());

testService.execute(50);
```

Así podemos probar la lógica del servicio sin llamar a un proveedor
real.

### ¿Cuándo aplicar DIP?

Cuando una clase de negocio depende directamente de bases de datos, APIs
externas, proveedores de pagos, sistemas de correo u otros detalles de
infraestructura.

**Idea clave:** el servicio de negocio debe depender de un contrato; los
proveedores concretos deben adaptarse a ese contrato.

------------------------------------------------------------------------

# 6. Cómo se relacionan los cinco principios

Los principios no son reglas aisladas. Se complementan:

  -----------------------------------------------------------------------
  Principio                           Pregunta que ayuda a responder
  ----------------------------------- -----------------------------------
  SRP                                 ¿Esta clase tiene demasiadas
                                      responsabilidades?

  OCP                                 ¿Puedo agregar una variante sin
                                      modificar continuamente el código
                                      existente?

  LSP                                 ¿Puedo sustituir esta
                                      implementación sin romper las
                                      expectativas?

  ISP                                 ¿Estoy obligando a implementar
                                      operaciones innecesarias?

  DIP                                 ¿Mi lógica de negocio depende de
                                      detalles concretos?
  -----------------------------------------------------------------------

Una forma sencilla de recordarlos en el sistema de pagos:

-   **SRP:** separar el procesamiento, la persistencia y las
    notificaciones.
-   **OCP:** agregar un nuevo medio de pago sin llenar el servicio de
    `if/else`.
-   **LSP:** garantizar que cada implementación respete su contrato.
-   **ISP:** separar las capacidades de pagar, reembolsar, cancelar y
    generar comprobantes.
-   **DIP:** depender de `PaymentGateway`, no directamente de Niubiz o
    Stripe.

# 7. ¿Cómo se aplica esto en NestJS?

NestJS utiliza TypeScript y un sistema de inyección de dependencias. Los
principios SOLID encajan naturalmente con su arquitectura.

Por ejemplo, podemos declarar un token para una abstracción:

``` typescript
export const PAYMENT_GATEWAY = Symbol('PAYMENT_GATEWAY');
```

Definir un contrato:

``` typescript
export interface PaymentGateway {
  charge(amount: number): Promise<void>;
}
```

Crear una implementación:

``` typescript
@Injectable()
export class NiubizGateway implements PaymentGateway {
  async charge(amount: number): Promise<void> {
    console.log(`Cobro con Niubiz: ${amount}`);
  }
}
```

Inyectarla en el servicio:

``` typescript
@Injectable()
export class PaymentService {
  constructor(
    @Inject(PAYMENT_GATEWAY)
    private readonly gateway: PaymentGateway,
  ) {}

  async execute(amount: number): Promise<void> {
    await this.gateway.charge(amount);
  }
}
```

Y registrar la implementación en el módulo:

``` typescript
@Module({
  providers: [
    PaymentService,
    {
      provide: PAYMENT_GATEWAY,
      useClass: NiubizGateway,
    },
  ],
  exports: [PaymentService],
})
export class PaymentModule {}
```

Si más adelante queremos utilizar Stripe, podemos crear `StripeGateway`
y cambiar el proveedor registrado en el módulo. `PaymentService` no
necesita conocer esa decisión.

**Importante:** las interfaces de TypeScript desaparecen en runtime. Por
eso no podemos utilizar directamente `PaymentGateway` como token de
inyección. Utilizamos un token concreto, como
`Symbol('PAYMENT_GATEWAY')`, o una clase abstracta.

# 8. Errores frecuentes al aprender SOLID

1.  **Crear una clase para absolutamente todo.** SRP no significa
    dividir el programa en cientos de clases diminutas.
2.  **Confundir OCP con no modificar nunca el código.** El objetivo es
    reducir cambios repetitivos y proteger las partes estables.
3.  **Pensar que herencia automáticamente significa LSP.** Una subclase
    puede heredar correctamente desde el punto de vista sintáctico y aun
    así romper el contrato.
4.  **Crear una interfaz gigante para ISP.** Si algunas implementaciones
    no soportan varios métodos, revisa si el contrato está mezclando
    capacidades.
5.  **Confundir DIP con inyección de dependencias.** La inyección de
    dependencias es una técnica que puede ayudar a aplicar DIP, pero no
    son lo mismo. Puedes inyectar una clase concreta y seguir acoplado a
    ella.

# 9. Ejercicio práctico

Ahora intenta diseñar tú mismo una pequeña ampliación del sistema.

El negocio solicita incorporar:

-   Pago con tarjeta.
-   Transferencia bancaria.
-   Pago con billetera digital.
-   Reembolsos solo para algunos medios.
-   Registro de cada operación.
-   Notificación al cliente.

Antes de escribir código, responde:

1.  ¿Qué clases crearías para cumplir SRP?
2.  ¿Qué contrato común utilizarías para procesar pagos y cómo
    agregarías un nuevo medio siguiendo OCP?
3.  ¿Cómo evitarías que un medio sin soporte de reembolsos rompa LSP?
4.  ¿Cómo separarías las capacidades de pago y reembolso para cumplir
    ISP?
5.  ¿Cómo harías que `PaymentService` dependa de una abstracción y no
    directamente de un proveedor para cumplir DIP?

Una buena práctica es implementar primero una solución sencilla y luego
revisar qué problemas aparecen. Refactorizar conscientemente es una de
las mejores maneras de aprender SOLID.

------------------------------------------------------------------------

## Conclusión

SOLID no es un framework ni una librería. Son cinco principios de diseño
que ayudan a construir software más mantenible, extensible, comprobable
y menos acoplado.

No necesitas aplicar los cinco de manera mecánica en cada clase. Debes
reconocer los problemas que intentan resolver y utilizarlos cuando
aporten valor al diseño.

Si recuerdas una sola idea de esta guía, que sea esta:

**Diseña tus clases para que cada una tenga una responsabilidad clara,
utiliza contratos coherentes, permite extender el comportamiento y evita
que la lógica de negocio dependa de detalles de infraestructura.**
