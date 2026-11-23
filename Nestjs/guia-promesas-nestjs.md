# Guía Completa de Promesas (Promises) en NestJS y TypeScript

En el ecosistema de **NestJS** (desarrollado sobre Node.js y TypeScript), el manejo de operaciones asíncronas es fundamental. Aunque NestJS introduce conceptos avanzados como los **Observables de RxJS**, las **Promesas** nativas de JavaScript siguen siendo el estándar principal para interactuar con bases de datos (TypeORM, Prisma, Mongoose) y servicios externos.

A continuación, se detallan todos los métodos de la API estática de `Promise`, junto con patrones avanzados de control de flujo asíncrono diseñados para su aplicación en controladores, servicios, interceptores y pruebas en NestJS.

---

## 1. El Bloque Fundamental: `new Promise()` y el Patrón `async/await`

### `new Promise((resolve, reject) => { ... })`
Es el constructor base. Se utiliza para transformar funciones basadas en callbacks antiguos (estilo Node.js tradicional) en estructuras modernas y compatibles con `async/await`.

* **Caso de uso en NestJS:** Integración de librerías de terceros heredadas que no devuelven promesas de manera nativa (por ejemplo, procesamiento antiguo de imágenes, criptografía por callbacks o lectura de archivos tradicionales).

```typescript
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import * as fs from 'fs';

@Injectable()
export class ArchivoService {
  async leerArchivoLegacy(path: string): Promise<string> {
    return new Promise((resolve, reject) => {
      fs.readFile(path, 'utf8', (err, data) => {
        if (err) {
          return reject(new InternalServerErrorException('Error al leer el archivo'));
        }
        resolve(data);
      });
    });
  }
}
```

---

## 2. Métodos Combinatorios (Control de Múltiples Promesas)

Cuando manejas flujos paralelos, la elección del método define el comportamiento del sistema ante fallas microestructurales de la red o las bases de datos.

### `Promise.all(iterable)`
* **Comportamiento:** Ejecuta todas las promesas en paralelo. Espera a que **todas** se resuelvan con éxito. 
* **Manejo de Fallos:** Si **una sola** promesa se rechaza, `Promise.all` se cancela inmediatamente (efecto dominó) e ignora los resultados de las promesas restantes.
* **Caso de uso en NestJS:** Consultas de inicialización de un dashboard donde necesitas que todos los módulos carguen de manera obligatoria.

```typescript
@Injectable()
export class DashboardService {
  constructor(
    private usuariosService: UsuariosService,
    private metricasService: MetricasService,
  ) {}

  async obtenerDatosCompletos(): Promise<any> {
    // Se ejecutan de manera simultánea ahorrando tiempo de respuesta
    const [usuarios, metricas] = await Promise.all([
      this.usuariosService.contarActivos(),
      this.metricasService.obtenerIngresosMensuales(),
    ]);

    return { usuarios, metricas };
  }
}
```

### `Promise.allSettled(iterable)`
* **Comportamiento:** Espera a que **todas** las promesas terminen, sin importar si fueron exitosas (`fulfilled`) o fallidas (`rejected`). Devuelve un array de objetos detallando el estado individual de cada una.
* **Manejo de Fallos:** No interrumpe el flujo. Permite evaluar las excepciones por separado.
* **Caso de uso en NestJS:** Procesamiento por lotes o pasarelas de notificación multicanal. Si falla el SMS, igual necesitas procesar la respuesta del Email y del Push Notification.

```typescript
@Injectable()
export class NotificacionesService {
  async notificarMasivo(userId: string) {
    const tareas = [
      this.emailService.send(userId, 'Tu orden ha sido enviada'),
      this.smsService.send(userId, 'Tu orden ha sido enviada'),
      this.whatsappService.send(userId, 'Tu orden ha sido enviada')
    ];

    const resultados = await Promise.allSettled(tareas);

    // Filtrar cuáles fallaron para guardarlo en logs o reintentar
    resultados.forEach((resultado, indice) => {
      if (resultado.status === 'rejected') {
        console.error(`Canal ${indice} falló:`, resultado.reason);
      }
    });

    return resultados;
  }
}
```

### `Promise.any(iterable)`
* **Comportamiento:** Retorna el valor de la **primera promesa que se resuelva con éxito**.
* **Manejo de Fallos:** Ignora los rechazos individuales a menos que **todas** las promesas del array fallen, en cuyo caso lanza un error agregado (`AggregateError`).
* **Caso de uso en NestJS:** Estrategias de alta disponibilidad (High Availability). Consultar réplicas distribuidas de bases de datos o servidores DNS espejo: la primera que responda exitosamente provee la data.

```typescript
@Injectable()
export class ProveedorDataService {
  async obtenerConfiguracionGlobal() {
    const servidoresEspejo = [
      this.httpService.axiosRef.get('https://nodo-usa.api.com/config'),
      this.httpService.axiosRef.get('https://nodo-europa.api.com/config'),
      this.httpService.axiosRef.get('https://nodo-asia.api.com/config')
    ];

    // Obtiene la configuración del servidor geográfico más rápido en responder correctamente
    return await Promise.any(servidoresEspejo);
  }
}
```

### `Promise.race(iterable)`
* **Comportamiento:** Resuelve o rechaza la ejecución global tan pronto como **la primera promesa del grupo termine**, sin importar si fue un éxito o un fracaso.
* **Caso de uso en NestJS:** Implementación de cortocircuitos por tiempo límite (*Timeouts*) personalizados para APIs de terceros que tienden a quedarse colgadas.

```typescript
@Injectable()
export class GatewayExternoService {
  async ejecutarTransaccionConTimeout() {
    const peticionPago = this.bancoApi.procesar();
    
    const timeout = new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout de Red: El banco no respondió en 4 segundos')), 4000)
    );

    // Si el banco tarda más de 4s, el timeout gana la carrera y aborta el flujo con error
    return await Promise.race([peticionPago, timeout]);
  }
}
```

---

## 3. Métodos de Instanciación Inmediata

Ideales para optimizar rutas sincrónicas, mockear comportamientos en entornos de test o estructurar retornos limpios en Guards/Interceptors.

### `Promise.resolve(valor)`
* **Comportamiento:** Crea una promesa pre-resuelta con el argumento suministrado de forma instantánea.
* **Caso de uso en NestJS:** En pruebas unitarias empleando Jest, o en mecanismos de Interceptores donde los datos se leen directamente de la memoria caché interna sin tocar la red.

```typescript
// En un archivo de pruebas: usuario.service.spec.ts
const mockUsuariosRepository = {
  findOne: jest.fn().mockImplementation((id: string) => {
    return Promise.resolve({ id: '123', nombre: 'Carlos Santana', rol: 'Admin' });
  }),
};
```

### `Promise.reject(error)`
* **Comportamiento:** Crea una promesa pre-rechazada con el objeto o error suministrado.
* **Caso de uso en NestJS:** Forzar respuestas fallidas asíncronas en `Guards` de autorización antes de procesar llamadas costosas.

```typescript
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';

@Injectable()
export class ApiKeyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'];

    if (!apiKey) {
      // Retorna una promesa rechazada directamente
      return Promise.reject(new UnauthorizedException('API Key faltante'));
    }
    return true;
  }
}
```

---

## 4. Patrones de Control Avanzados (Más allá de la API nativa)

Existen situaciones asíncronas complejas en servidores NestJS que los métodos por defecto de JavaScript no resuelven de forma óptima. A continuación se presentan las soluciones de ingeniería para estos escenarios:

### Patrón A: Ejecución Secuencial (Con `Array.prototype.reduce`)
* **Problema:** `Promise.all` dispara todo a la vez. ¿Qué pasa si necesitas procesar un array de 1,000 registros en la base de datos de uno en uno consecutivamente para evitar la saturación del procesador?
* **Solución:** Utilizar `.reduce()` para encadenar las promesas secuencialmente de manera dinámica.

```typescript
@Injectable()
export class MigracionService {
  async procesarUsuariosSecuencialmente(usuarios: any[]) {
    // Resuelve las promesas en fila india, una detrás de la otra
    return usuarios.reduce(async (promesaAnterior, usuario) => {
      await promesaAnterior;
      return this.migrarRegistroIndividual(usuario);
    }, Promise.resolve());
  }

  private async migrarRegistroIndividual(usuario: any) {
    // Lógica interna de inserción en Base de Datos
  }
}
```

### Patrón B: Promesas en Lotes Controlados (Concurrencia Limitada)
* **Problema:** Necesitas procesar 500 peticiones HTTP. Procesarlas de una en una (secuencial) es muy lento, pero disparar las 500 en paralelo (`Promise.all`) tumbará tu base de datos o causará un bloqueo por Rate Limit en la API externa.
* **Solución:** Agrupar en pequeños lotes concurrentes (Chunks).

```typescript
@Injectable()
export class BatchService {
  async procesarEnLotes(items: any[], tamañoLote: number = 10) {
    const resultados = [];
    
    for (let i = 0; i < items.length; i += tamañoLote) {
      const lote = items.slice(i, i + tamañoLote);
      // Ejecuta de 10 en 10 concurrentemente
      const respuestasLote = await Promise.all(lote.map(item => this.procesarItem(item)));
      resultados.push(...respuestasLote);
    }
    
    return resultados;
  }

  private async procesarItem(item: any) { /* ... */ }
}
```

---

## Resumen de Comportamiento Estático

| Método | Concurrencia | Condición de Éxito Global | Reacción Ante Primer Fallo |
| :--- | :--- | :--- | :--- |
| **`Promise.all`** | Paralela | Cuando **todas** se resuelven de manera exitosa. | Rechaza instantáneamente rompiendo el flujo. |
| **`Promise.allSettled`** | Paralela | Siempre (cuando **todas** finalizan, sea como sea). | Ninguna. Continúa recopilando resultados. |
| **`Promise.any`** | Paralela | En cuanto **una sola** se resuelve con éxito. | Ignora el error y espera los demás hilos. |
| **`Promise.race`** | Paralela | En cuanto **una sola** termina (éxito o error). | Rechaza si la primera en terminar falla. |