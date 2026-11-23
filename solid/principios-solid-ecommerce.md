# Guía Práctica de Principios SOLID y Polimorfismo en NestJS

Esta guía explica los principios **SOLID** y el concepto de **Polimorfismo** utilizando el contexto de una **Tienda Virtual (E-commerce)**. El código está escrito en **TypeScript** estructurado bajo la arquitectura típica de **NestJS** (Controladores, Servicios y Repositorios).

---

## Introducción al Polimorfismo

Antes de entrar en SOLID, es vital entender el **Polimorfismo** (muchas formas). En NestJS/TypeScript, el polimorfismo nos permite definir una interfaz o clase abstracta común para múltiples tipos de objetos, de modo que nuestro código pueda interactuar con ellos de la misma manera, aunque cada objeto se comporte de forma diferente por dentro.

### Ejemplo de Polimorfismo en una Tienda Virtual

Imagina que calculamos el costo de envío de diferentes tipos de productos. En lugar de llenar el código con estructuras `if/else`, usamos polimorfismo a través de una interfaz.

```typescript
// Definimos la interfaz común
interface CalculadorEnvio {
  calcularCosto(peso: number): number;
}

// Implementación para Productos Nacionales
class EnvioNacional implements CalculadorEnvio {
  calcularCosto(peso: number): number {
    return peso * 1.5; // Tarifa fija por kilo
  }
}

// Implementación para Productos Internacionales
class EnvioInternacional implements CalculadorEnvio {
  calcularCosto(peso: number): number {
    return peso * 5.0 + 20; // Tarifa más arancel fijo
  }
}

// El servicio utiliza el polimorfismo de forma transparente
class ServicioLogistica {
  procesarEnvio(calculador: CalculadorEnvio, peso: number): number {
    // No le importa qué tipo de envío es, solo sabe que responde a 'calcularCosto'
    return calculador.calcularCosto(peso);
  }
}
```

---

## Principios SOLID

SOLID es un acrónimo de cinco principios de diseño orientado a objetos que nos ayudan a crear software mantenible, escalable y fácil de testear.

### 1. S - Single Responsibility Principle (Principio de Responsabilidad Única)
> *"Una clase debe tener una, y solo una, razón para cambiar."*

**Antipatrón (Clase Dios):** Un servicio de productos que se encarga de buscar en la base de datos, validar stock, enviar correos de alerta de bajo inventario y generar un PDF con la ficha técnica del producto.

#### Enfoque Correcto en NestJS:
Dividimos el código en múltiples clases donde cada una tiene una responsabilidad única bien delimitada.

```typescript
// 1. Encargado exclusivamente del acceso a datos
@Injectable()
export class ProductRepository {
  async findById(id: string): Promise<Product> {
    return /* lógica de BD */;
  }
  async updateStock(id: string, qty: number): Promise<void> { /* ... */ }
}

// 2. Encargado exclusivamente de las alertas de negocio
@Injectable()
export class InventoryAlertService {
  async notifyLowStock(productName: string): Promise<void> {
    // Lógica para enviar correos o mensajes a Slack
  }
}

// 3. El servicio principal que coordina las reglas del negocio de productos
@Injectable()
export class ProductService {
  constructor(
    private readonly productRepo: ProductRepository,
    private readonly alertService: InventoryAlertService,
  ) {}

  async purchaseProduct(id: string, quantity: number): Promise<void> {
    const product = await this.productRepo.findById(id);
    
    if (product.stock < quantity) {
      throw new BadRequestException('Stock insuficiente');
    }

    await this.productRepo.updateStock(id, product.stock - quantity);

    if (product.stock - quantity <= product.minThreshold) {
      await this.alertService.notifyLowStock(product.name);
    }
  }
}
```

---

### 2. O - Open/Closed Principle (Principio de Abierto/Cerrado)
> *"Las entidades de software deben estar abiertas para su extensión, pero cerradas para su modificación."*

Si queremos añadir un nuevo método de descuento (por ejemplo, Black Friday, Cupón de Influencer, Cupón de Bienvenida), no deberíamos estar modificando un archivo `discount.service.ts` agregando más bloques `switch` o `if`. En su lugar, debemos poder añadir nuevas estrategias sin tocar el código existente.

#### Enfoque Correcto (Aplicando Polimorfismo):

```typescript
// Definimos la estrategia base
export interface DiscountStrategy {
  apply(price: number): number;
}

// Implementación 1: Descuento Black Friday
export class BlackFridayDiscount implements DiscountStrategy {
  apply(price: number): number {
    return price * 0.70; // 30% de descuento
  }
}

// Implementación 2: Descuento Cupón Fijo
export class CouponDiscount implements DiscountStrategy {
  constructor(private readonly amount: number) {}
  apply(price: number): number {
    return Math.max(0, price - this.amount);
  }
}

// El servicio queda CERRADO a modificaciones, pero ABIERTO a recibir nuevas estrategias
@Injectable()
export class CheckoutService {
  calculateTotal(price: number, discountStrategy: DiscountStrategy): number {
    // Si mañana se crea un descuento "NavidadDiscount", esta línea no cambia en absoluto.
    return discountStrategy.apply(price);
  }
}
```

---

### 3. L - Liskov Substitution Principle (Principio de Sustitución de Liskov)
> *"Si S es un subtipo de T, los objetos de tipo T pueden ser reemplazados por objetos de tipo S sin alterar las propiedades del programa."*

En términos sencillos: las subclases o implementaciones nunca deben romper las expectativas o contratos definidos por la clase padre. No deben lanzar excepciones inesperadas o alterar drásticamente el comportamiento de los métodos heredados.

**Violación Común:** Crear un `DigitalProduct` que hereda de `Product`, pero al ejecutar el método `shipToAddress()` lanza un error `new Error("Los productos digitales no se envían")`. Esto rompe el polimorfismo y destruye la confianza en la clase padre.

#### Enfoque Correcto en NestJS:

Separamos los conceptos en la jerarquía o mediante interfaces correctas para asegurar que la sustitución de tipos sea siempre segura.

```typescript
// Interfaz base para productos físicos que requieren inventario físico
export interface ShippableProduct {
  name: string;
  weight: number;
  getDimensions(): string;
}

// Producto que cumple perfectamente con el contrato
export class PhysicalBook implements ShippableProduct {
  constructor(public name: string, public weight: number) {}
  getDimensions() { return '20x15x3 cm'; }
}

// Producto digital NO implementa ShippableProduct si no puede cumplir con sus reglas.
// En su lugar, implementa otra abstracción adecuada.
export interface DownloadableProduct {
  name: string;
  getFileSize(): number;
}

export class EBook implements DownloadableProduct {
  constructor(public name: string, private size: number) {}
  getFileSize() { return this.size; }
}
```

---

### 4. I - Interface Segregation Principle (Principio de Segregación de Interfaces)
> *"Los clientes no deberían ser forzados a depender de interfaces que no utilizan."*

Es mucho mejor tener muchas interfaces pequeñas y especializadas que una sola interfaz gigantesca y genérica.

**Antipatrón:** Una interfaz `ProductOperations` que contiene `saveToDb()`, `renderHtml()`, `generateInvoicePdf()`, `syncWithAmazon()`. Un desarrollador que solo necesita un renderizador HTML sencillo se ve obligado a codificar métodos de sincronización con Amazon vacíos o lanzando errores.

#### Enfoque Correcto:

```typescript
// Interfaces atómicas y específicas
export interface DatabasePersistable {
  save(): Promise<void>;
}

export interface HtmlRenderable {
  renderCardHtml(): string;
}

export interface MarketplaceSyncable {
  syncStock(marketplaceId: string): Promise<void>;
}

// Un producto estándar de la tienda solo usa lo que necesita
export class CustomTShirt implements DatabasePersistable, HtmlRenderable {
  save() { /* ... */ return Promise.resolve(); }
  renderCardHtml() { return '<div>Camiseta personalizada</div>'; }
  // No arrastra métodos de sincronización con marketplaces externos si no los necesita
}
```

---

### 5. D - Dependency Inversion Principle (Principio de Inversión de Dependencias)
> *"Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones. Las abstracciones no deben depender de los detalles."*

Este principio es el corazón de la **Inyección de Dependencias (DI)** que NestJS implementa de forma nativa a través de sus proveedores (`@Injectable()`). El servicio de negocio (alto nivel) no debe acoplarse directamente a un cliente de base de datos específico como Mongoose o TypeORM (bajo nivel). Debe depender de un Token o Interfaz intermedio.

#### Enfoque Correcto en NestJS:

```typescript
// 1. Definimos la abstracción (Contrato)
export interface OrderRepositoryInterface {
  saveOrder(order: any): Promise<void>;
}

// Crear un Token único para la inyección en NestJS
export const ORDER_REPOSITORY_TOKEN = Symbol('OrderRepositoryInterface');

// 2. Implementación de bajo nivel (Detalle técnico: TypeORM)
@Injectable()
export class TypeORMOrderRepository implements OrderRepositoryInterface {
  async saveOrder(order: any): Promise<void> {
    console.log('Guardando orden en PostgreSQL usando TypeORM...');
  }
}

// 3. Módulo de alto nivel (Regla de negocio)
@Injectable()
export class CheckoutService {
  constructor(
    // Inyectamos usando el Token abstracto, no la clase concreta de TypeORM
    @Inject(ORDER_REPOSITORY_TOKEN)
    private readonly orderRepository: OrderRepositoryInterface,
  ) {}

  async completeCheckout(cart: any): Promise<void> {
    // Procesar lógica de negocio de la orden...
    await this.orderRepository.saveOrder(cart);
  }
}

// 4. Configuración del Módulo de NestJS (AppModule / ProductModule)
@Module({
  providers: [
    CheckoutService,
    {
      provide: ORDER_REPOSITORY_TOKEN,
      useClass: TypeORMOrderRepository, // Aquí es donde se define el detalle físico de la BD
    },
  ],
})
export class OrderModule {}
```

Si el día de mañana decides migrar toda la base de datos de PostgreSQL (TypeORM) a MongoDB (Mongoose), el código dentro de tu `CheckoutService` **no cambia un solo caracter**. Solo creas la nueva clase `MongooseOrderRepository` y reconfiguras el campo `useClass` en tu archivo `@Module`.

---

## Conclusión

| Principio | Resumen en una frase para E-commerce | Beneficio Principal |
| :--- | :--- | :--- |
| **S** - Responsabilidad Única | El servicio de Catálogo solo gestiona datos, el envío de correos va a otro servicio. | Código fácil de testear. |
| **O** - Abierto/Cerrado | Agrega cupones navideños creando un archivo nuevo, sin romper los cupones actuales. | Escalabilidad sin bugs colaterales. |
| **L** - Sustitución de Liskov | No trates a un curso online descargable igual que a una lavadora física que requiere camión de carga. | Jerarquías de código limpias y confiables. |
| **I** - Segregación de Interfaces | Si un producto no se sincroniza con MercadoLibre, su interfaz no debe obligarlo a programarlo. | Desacoplamiento de componentes. |
| **D** - Inversión de Dependencias | Tu negocio depende de "Guardar la compra", no le importa si es en MySQL, Mongo o un archivo de texto. | Modularidad total e independencia de librerías de terceros. |
