# RxJS en TypeScript: operadores esenciales con ejemplos bancarios

## Objetivo

Esta guía explica los operadores de RxJS más utilizados en aplicaciones
frontend y backend, utilizando escenarios de un sistema bancario:
consulta de saldos, movimientos, transferencias, validación de
beneficiarios, pagos de tarjetas y notificaciones.

La intención es comprender:

-   Qué hace cada operador.
-   Cómo funciona sobre un `Observable`.
-   Cuándo conviene utilizarlo.
-   Qué errores de diseño conviene evitar.

Los ejemplos utilizan TypeScript y RxJS. Algunos servicios como
`accountApi` o `transferApi` son ficticios: representan servicios HTTP o
adaptadores de infraestructura.

> Importante en banca: RxJS ayuda a coordinar flujos asíncronos, pero no
> reemplaza las garantías del backend. Una operación financiera debe
> protegerse también con validaciones del servidor, autorización,
> transacciones de base de datos, idempotencia y controles antifraude.

------------------------------------------------------------------------

# 1. Conceptos básicos antes de los operadores

Un `Observable` representa un flujo de valores que puede emitirse a lo
largo del tiempo.

``` typescript
import { Observable, of } from 'rxjs';

const accountBalance$: Observable<number> = of(1500);

accountBalance$.subscribe(balance => {
  console.log(balance);
});
```

En este ejemplo:

-   `of(1500)` crea un Observable que emite el valor `1500`.
-   `subscribe()` se suscribe al flujo.
-   La función recibe el valor emitido.

La convención de colocar `$` al final de una variable, como
`accountBalance$`, es una convención común para indicar que contiene un
Observable. No es una regla obligatoria de TypeScript.

## Los tres tipos de notificación

Un Observable puede:

1.  Emitir valores mediante `next`.
2.  Terminar correctamente mediante `complete`.
3.  Terminar con un error mediante `error`.

``` typescript
accountBalance$.subscribe({
  next: balance => console.log(balance),
  error: error => console.error(error),
  complete: () => console.log('Consulta finalizada'),
});
```

## ¿Qué es un operador?

Un operador es una función que permite transformar, filtrar, combinar,
controlar o gestionar un flujo Observable.

Normalmente se utilizan dentro de `pipe()`:

``` typescript
import { map, filter } from 'rxjs';

const movements$ = of(100, -50, 200, -30);

const incomingMovements$ = movements$.pipe(
  filter(amount => amount > 0),
  map(amount => ({
    amount,
    type: 'INCOME',
  })),
);
```

-   `filter` deja pasar solo los valores que cumplen una condición.
-   `map` transforma cada valor emitido.
-   `pipe` permite encadenar operadores.

------------------------------------------------------------------------

# 2. Operadores de transformación

## 2.1. map()

### ¿Qué hace?

Transforma cada valor emitido por un Observable en otro valor.

No filtra ni elimina emisiones: transforma cada una.

### Ejemplo bancario

El backend devuelve el saldo en céntimos y la aplicación necesita
mostrarlo en soles.

``` typescript
import { map } from 'rxjs';

const balanceInCents$ = of(250050);

const balanceInSoles$ = balanceInCents$.pipe(
  map(cents => cents / 100),
);

balanceInSoles$.subscribe(balance => {
  console.log(balance); // 2500.5
});
```

También podemos transformar un movimiento recibido del backend:

``` typescript
interface MovementResponse {
  id: string;
  amount: number;
  currency: string;
}

interface MovementView {
  id: string;
  formattedAmount: string;
}

const movement$ = of<MovementResponse>({
  id: 'MOV-001',
  amount: 150.5,
  currency: 'PEN',
});

const movementView$ = movement$.pipe(
  map(movement => ({
    id: movement.id,
    formattedAmount: new Intl.NumberFormat('es-PE', {
      style: 'currency',
      currency: movement.currency,
    }).format(movement.amount),
  })),
);
```

### ¿Dónde utilizarlo?

-   Transformar respuestas HTTP.
-   Adaptar DTOs a modelos de presentación.
-   Convertir importes o fechas para mostrarlos.
-   Construir objetos que necesita un componente.

**Idea clave:** `map` transforma cada emisión; no sirve para realizar
por sí solo otra petición HTTP dependiente del valor.

------------------------------------------------------------------------

## 2.2. scan()

### ¿Qué hace?

Acumula valores a medida que el Observable emite. Es parecido a
`reduce`, pero `scan` va emitiendo cada resultado intermedio.

### Ejemplo bancario

Calcular el saldo acumulado a partir de una secuencia de movimientos.

``` typescript
import { scan } from 'rxjs';

const movements$ = of(500, -100, -50, 200);

const accumulatedBalance$ = movements$.pipe(
  scan((balance, movement) => balance + movement, 1000),
);

accumulatedBalance$.subscribe(balance => {
  console.log(balance);
});

// 1500
// 1400
// 1350
// 1550
```

El saldo inicial es `1000`. Cada movimiento modifica el acumulado.

### ¿Dónde utilizarlo?

-   Construir estados progresivos.
-   Acumular resultados de un flujo.
-   Mantener contadores o indicadores.
-   Procesar eventos de una interfaz.

**Precaución bancaria:** este ejemplo es didáctico. No utilices el
acumulador del frontend como fuente oficial del saldo de una cuenta. El
saldo real debe provenir del backend o del sistema contable autorizado.

------------------------------------------------------------------------

## 2.3. startWith()

### ¿Qué hace?

Emite uno o más valores iniciales antes de comenzar a emitir los valores
del Observable original.

### Ejemplo bancario

Mostrar un estado inicial mientras se consulta el saldo.

``` typescript
import { startWith } from 'rxjs';

const balance$ = accountApi.getBalance().pipe(
  startWith({
    status: 'LOADING',
    balance: null,
  }),
);
```

En una implementación real conviene definir un tipo discriminado para
que los estados `LOADING`, `SUCCESS` y `ERROR` tengan propiedades
coherentes.

### ¿Dónde utilizarlo?

-   Mostrar estados iniciales de carga.
-   Inicializar formularios o filtros.
-   Dar un valor inicial a un flujo de interfaz.

------------------------------------------------------------------------

# 3. Operadores de filtrado y control de emisiones

## 3.1. filter()

### ¿Qué hace?

Permite que continúen únicamente los valores que cumplen una condición.

### Ejemplo bancario

Mostrar solamente movimientos de ingreso.

``` typescript
import { filter } from 'rxjs';

interface Movement {
  id: string;
  amount: number;
}

const movements$ = of(
  { id: '1', amount: 500 },
  { id: '2', amount: -100 },
  { id: '3', amount: 250 },
);

const incomeMovements$ = movements$.pipe(
  filter(movement => movement.amount > 0),
);
```

### ¿Dónde utilizarlo?

-   Filtrar movimientos por importe.
-   Ignorar eventos que no interesan.
-   Dejar pasar respuestas que cumplen una condición.
-   Evitar procesar valores inválidos en un flujo de interfaz.

**Diferencia con `map`:** `map` transforma; `filter` decide si una
emisión continúa.

------------------------------------------------------------------------

## 3.2. debounceTime()

### ¿Qué hace?

Espera un periodo de tiempo después de la última emisión. Si llega otro
valor antes de que termine ese periodo, reinicia la espera.

### Ejemplo bancario

Un buscador de beneficiarios por nombre o número de cuenta.

``` typescript
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs';

const beneficiaryResults$ = beneficiarySearchControl.valueChanges.pipe(
  debounceTime(400),
  distinctUntilChanged(),
  switchMap(query => beneficiaryApi.search(query)),
);
```

Si el usuario escribe rápidamente, no se consulta el backend por cada
tecla. Se espera a que deje de escribir durante 400 milisegundos.

### ¿Dónde utilizarlo?

-   Búsqueda de beneficiarios.
-   Búsqueda de movimientos.
-   Autocompletado de cuentas.
-   Filtros de transacciones.

**Precaución:** no uses `debounceTime` como mecanismo para controlar la
ejecución de una transferencia. Su propósito es regular emisiones, no
garantizar que una operación financiera se ejecute una sola vez.

------------------------------------------------------------------------

## 3.3. distinctUntilChanged()

### ¿Qué hace?

Evita emitir un valor cuando es igual al valor anterior, según el
comparador utilizado.

### Ejemplo bancario

Evitar repetir una consulta cuando el usuario vuelve a seleccionar el
mismo tipo de cuenta.

``` typescript
import { distinctUntilChanged } from 'rxjs';

const accountType$ = of(
  'SAVINGS',
  'SAVINGS',
  'CURRENT',
  'CURRENT',
  'SAVINGS',
);

const uniqueAccountTypes$ = accountType$.pipe(
  distinctUntilChanged(),
);
```

Emite:

``` text
SAVINGS
CURRENT
SAVINGS
```

### ¿Dónde utilizarlo?

-   Filtros de cuentas.
-   Formularios reactivos.
-   Búsquedas.
-   Parámetros de consulta que no han cambiado.

**Importante:** por defecto compara con igualdad estricta (`===`). Si se
emiten objetos nuevos con las mismas propiedades, debes proporcionar un
comparador o comparar una propiedad específica.

``` typescript
distinctUntilChanged(
  (previous, current) => previous.accountId === current.accountId
)
```

------------------------------------------------------------------------

## 3.4. take()

### ¿Qué hace?

Limita la cantidad de emisiones que se reciben y luego completa el
Observable.

### Ejemplo bancario

Obtener solamente la primera cotización disponible de un flujo que sigue
actualizándose.

``` typescript
import { take } from 'rxjs';

const exchangeRate$ = exchangeRateService.getLiveRate().pipe(
  take(1),
);
```

### ¿Dónde utilizarlo?

-   Obtener una cantidad limitada de emisiones.
-   Consumir una primera respuesta.
-   Completar una suscripción después de una cantidad conocida de
    valores.

Para una petición HTTP habitual de Angular, normalmente no necesitas
`take(1)`, porque el Observable HTTP emite una respuesta y completa por
sí mismo.

------------------------------------------------------------------------

## 3.5. takeUntil()

### ¿Qué hace?

Mantiene activa una suscripción hasta que otro Observable emite.

### Ejemplo bancario en Angular

Cancelar la escucha de actualizaciones de saldo cuando el componente se
destruye.

``` typescript
import { Subject, takeUntil } from 'rxjs';

class AccountComponent {
  private readonly destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.accountApi.balanceUpdates$
      .pipe(takeUntil(this.destroy$))
      .subscribe(balance => {
        console.log(balance);
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

En Angular moderno también puedes utilizar `takeUntilDestroyed()` desde
`@angular/core/rxjs-interop`.

### ¿Dónde utilizarlo?

-   Evitar suscripciones que sobreviven al componente.
-   Detener flujos al cerrar una pantalla.
-   Liberar recursos de streams y eventos.

------------------------------------------------------------------------

# 4. Operadores de aplanamiento: switchMap, mergeMap, concatMap y exhaustMap

Esta es una de las partes más importantes de RxJS.

Estos operadores se utilizan cuando un Observable emite un valor y, a
partir de ese valor, necesitamos iniciar otro Observable.

Por ejemplo:

-   El usuario selecciona una cuenta.
-   Se consulta el detalle de esa cuenta.
-   El usuario solicita sus movimientos.
-   Se consulta el historial.

La pregunta es: **¿qué debe ocurrir si llega una nueva emisión mientras
la operación anterior sigue en curso?**

Los cuatro operadores responden de manera diferente.

## 4.1. switchMap()

### ¿Qué hace?

Cuando recibe una nueva emisión, cambia al Observable más reciente y
deja de escuchar el Observable interno anterior.

### Ejemplo bancario

El usuario busca una cuenta por número. Si cambia la búsqueda, nos
interesa el resultado de la búsqueda más reciente.

``` typescript
import { switchMap } from 'rxjs';

const accountDetails$ = accountIdControl.valueChanges.pipe(
  switchMap(accountId =>
    accountApi.getAccountDetails(accountId)
  ),
);
```

Ejemplo de comportamiento:

``` text
Usuario escribe cuenta A → inicia consulta A
Usuario escribe cuenta B → deja de escuchar A e inicia B
Respuesta B → se procesa
```

### ¿Dónde utilizarlo?

-   Búsquedas y autocompletados.
-   Consultar detalles de una cuenta seleccionada.
-   Obtener movimientos de la cuenta actual.
-   Cargar información dependiente de una selección.

### Precaución importante

`switchMap` cancela la suscripción al Observable interno anterior. Si
ese Observable representa una petición HTTP de Angular, normalmente se
puede abortar la solicitud en el cliente.

Pero cancelar la petición desde el cliente **no garantiza que el
servidor no haya recibido o procesado la operación**.

Por eso no debes asumir que `switchMap` hace segura una transferencia
bancaria. Para operaciones financieras de escritura se necesitan
controles de idempotencia y garantías en el backend.

------------------------------------------------------------------------

## 4.2. mergeMap()

### ¿Qué hace?

Inicia un Observable interno por cada emisión y permite que esos
procesos se ejecuten concurrentemente.

### Ejemplo bancario

Consultar los detalles de varias cuentas de forma independiente.

``` typescript
import { mergeMap } from 'rxjs';

const accountIds$ = of('ACC-01', 'ACC-02', 'ACC-03');

const accountDetails$ = accountIds$.pipe(
  mergeMap(accountId =>
    accountApi.getAccountDetails(accountId)
  ),
);
```

Las consultas pueden ejecutarse simultáneamente. Las respuestas pueden
llegar en un orden distinto al de las cuentas originales.

También puedes limitar la concurrencia:

``` typescript
mergeMap(
  accountId => accountApi.getAccountDetails(accountId),
  2
)
```

En este caso se permiten como máximo dos suscripciones internas activas
simultáneamente.

### ¿Dónde utilizarlo?

-   Consultas independientes de varias cuentas.
-   Procesamiento de operaciones que no necesitan conservar el orden.
-   Carga paralela de información.
-   Tareas concurrentes con un límite de concurrencia.

**Precaución:** no uses concurrencia ilimitada sin considerar la
capacidad del backend y del proveedor externo.

------------------------------------------------------------------------

## 4.3. concatMap()

### ¿Qué hace?

Procesa cada Observable interno en orden, esperando a que termine uno
antes de iniciar el siguiente.

### Ejemplo bancario

Procesar una secuencia de instrucciones que deben ejecutarse en orden.

``` typescript
import { concatMap } from 'rxjs';

const operations$ = of(
  { type: 'REGISTER', id: 'PAY-01' },
  { type: 'CONFIRM', id: 'PAY-01' },
  { type: 'NOTIFY', id: 'PAY-01' },
);

const result$ = operations$.pipe(
  concatMap(operation =>
    paymentApi.executeOperation(operation)
  ),
);
```

El orden será:

``` text
REGISTER → CONFIRM → NOTIFY
```

La siguiente operación no comienza hasta que termina la anterior.

### ¿Dónde utilizarlo?

-   Procesos secuenciales.
-   Operaciones que deben conservar el orden de llegada.
-   Guardado de cambios en secuencia.
-   Flujos donde no conviene ejecutar tareas simultáneamente.

**Precaución:** si las emisiones llegan más rápido de lo que se
procesan, se puede acumular una cola. Además, `concatMap` por sí solo no
proporciona atomicidad ni garantiza que un conjunto de operaciones
financieras se complete como una sola transacción.

------------------------------------------------------------------------

## 4.4. exhaustMap()

### ¿Qué hace?

Cuando ya existe un Observable interno activo, ignora las nuevas
emisiones hasta que el Observable actual termina.

### Ejemplo bancario

Evitar que un usuario dispare múltiples solicitudes desde el botón de
consulta de un estado de operación mientras la primera consulta sigue en
curso.

``` typescript
import { exhaustMap } from 'rxjs';

const status$ = refreshStatusButtonClicks$.pipe(
  exhaustMap(() =>
    paymentApi.getPaymentStatus('PAY-123')
  ),
);
```

Si el usuario hace clic varias veces mientras la consulta está activa,
se ignoran los clics adicionales.

### ¿Dónde utilizarlo?

-   Evitar múltiples solicitudes simultáneas desde un botón.
-   Evitar envíos repetidos de formularios durante una petición activa.
-   Procesos donde solo interesa atender la primera emisión mientras se
    ejecuta.

### Precaución bancaria

`exhaustMap` evita solicitudes concurrentes desde ese flujo del cliente,
pero no reemplaza la idempotencia del servidor. El usuario podría
repetir la operación desde otra pestaña, dispositivo o cliente.

------------------------------------------------------------------------

## 4.5. Comparación de los cuatro operadores

  Operador       Comportamiento
  -------------- -------------------------------------------------------
  `switchMap`    Cambia al último flujo y deja de escuchar el anterior
  `mergeMap`     Ejecuta varios flujos concurrentemente
  `concatMap`    Ejecuta los flujos en orden, uno después de otro
  `exhaustMap`   Ignora nuevas emisiones mientras hay un flujo activo

### Regla mental para recordarlos

-   `switchMap`: me interesa el más reciente.
-   `mergeMap`: quiero ejecutar varios al mismo tiempo.
-   `concatMap`: quiero respetar el orden.
-   `exhaustMap`: mientras estoy ocupado, ignoro nuevas solicitudes.

------------------------------------------------------------------------

# 5. Operadores para manejar errores y finalización

## 5.1. catchError()

### ¿Qué hace?

Permite interceptar un error del Observable y reemplazar el flujo
fallido por otro Observable.

### Ejemplo bancario

Si falla la consulta de movimientos, podemos devolver una respuesta
alternativa controlada o permitir que la interfaz muestre un estado de
error.

``` typescript
import { catchError, of } from 'rxjs';

const movements$ = accountApi.getMovements().pipe(
  catchError(error => {
    console.error('No fue posible consultar los movimientos', error);

    return of([]);
  }),
);
```

En este ejemplo, si falla la consulta, el Observable emite una lista
vacía y completa.

### ¿Dónde utilizarlo?

-   Manejar errores HTTP.
-   Devolver valores alternativos cuando tenga sentido.
-   Transformar errores técnicos en resultados controlados.
-   Mantener funcionando un flujo cuando una consulta secundaria falla.

### Precaución

No conviertas cualquier error financiero en un resultado exitoso.

Por ejemplo, si falla una transferencia, devolver
`of({ success: true })` sería gravísimo. Tampoco debes interpretar una
lista vacía como prueba de que no hubo movimientos si realmente falló la
consulta.

Una alternativa es devolver un estado explícito:

``` typescript
type MovementsState =
  | { status: 'SUCCESS'; movements: Movement[] }
  | { status: 'ERROR'; message: string };
```

------------------------------------------------------------------------

## 5.2. finalize()

### ¿Qué hace?

Ejecuta una función cuando el Observable termina, ya sea por
finalización, error o cancelación de la suscripción.

### Ejemplo bancario

Desactivar un indicador de carga al terminar una consulta.

``` typescript
import { finalize } from 'rxjs';

const balance$ = accountApi.getBalance().pipe(
  finalize(() => {
    console.log('Finalizó la consulta del saldo');
  }),
);
```

### ¿Dónde utilizarlo?

-   Desactivar indicadores de carga.
-   Liberar recursos.
-   Registrar el cierre de una operación técnica.
-   Ejecutar limpieza al terminar una suscripción.

**Importante:** `finalize` no significa que una transferencia haya sido
exitosa. Se ejecuta también si hubo error o cancelación. Para conocer el
resultado debes observar las emisiones y los errores.

------------------------------------------------------------------------

## 5.3. retry()

### ¿Qué hace?

Vuelve a suscribirse al Observable cuando ocurre un error, hasta
alcanzar el número de reintentos configurado.

### Ejemplo bancario

Reintentar una consulta de cotización que falló temporalmente.

``` typescript
import { retry } from 'rxjs';

const exchangeRate$ = exchangeRateApi.getRate().pipe(
  retry(2),
);
```

Esto permite hasta dos reintentos adicionales después del intento
inicial.

### ¿Dónde utilizarlo?

-   Consultas de información que pueden fallar temporalmente.
-   Errores transitorios de red.
-   Servicios de lectura que admiten reintentos.

### Precaución bancaria

No apliques `retry` indiscriminadamente a operaciones que modifican
dinero.

Imagina que envías una transferencia y el servidor la procesa, pero la
respuesta se pierde por un problema de red. Si reintentas sin
idempotencia, podrías ejecutar la operación dos veces.

Para reintentar una operación financiera, el backend debe soportar una
clave de idempotencia y garantizar que repetir la misma solicitud no
duplique el efecto.

------------------------------------------------------------------------

## 5.4. timeout()

### ¿Qué hace?

Establece un tiempo máximo de espera para recibir una emisión. Si se
excede, genera un error.

### Ejemplo bancario

Una consulta a un proveedor de tipo de cambio no debería quedar
esperando indefinidamente.

``` typescript
import { timeout } from 'rxjs';

const exchangeRate$ = exchangeRateApi.getRate().pipe(
  timeout({ first: 5000 }),
);
```

Si no llega una primera emisión en cinco segundos, se produce un error.

### ¿Dónde utilizarlo?

-   Consultas a APIs externas.
-   Integraciones con proveedores.
-   Flujos que deben responder dentro de un tiempo esperado.

Un timeout no demuestra que la operación remota haya fallado. El
servidor podría haberla procesado y la respuesta simplemente no llegó a
tiempo.

------------------------------------------------------------------------

# 6. Operadores para combinar Observables

## 6.1. forkJoin()

### ¿Qué hace?

Espera a que todos los Observables de entrada completen y emite una sola
vez con sus últimos valores.

### Ejemplo bancario

Al entrar al resumen de una cuenta necesitamos consultar:

-   Datos de la cuenta.
-   Saldo.
-   Últimos movimientos.

``` typescript
import { forkJoin } from 'rxjs';

const dashboard$ = forkJoin({
  account: accountApi.getAccount(),
  balance: accountApi.getBalance(),
  movements: accountApi.getRecentMovements(),
});
```

Cuando las tres peticiones completan, obtenemos un objeto:

``` typescript
{
  account: { /* datos */ },
  balance: { /* datos */ },
  movements: [ /* movimientos */ ]
}
```

### ¿Dónde utilizarlo?

-   Cargar varias peticiones HTTP independientes para una pantalla.
-   Obtener información inicial de un dashboard.
-   Combinar respuestas que completan.

### Precauciones

-   Si uno de los Observables falla, `forkJoin` falla, salvo que
    gestiones ese error.
-   Si uno nunca completa, `forkJoin` nunca emite.
-   No es adecuado para streams infinitos, como un WebSocket de
    cotizaciones en vivo.

------------------------------------------------------------------------

## 6.2. combineLatest()

### ¿Qué hace?

Combina los valores más recientes de varios Observables y emite cada vez
que cualquiera de ellos emite, una vez que todos hayan emitido al menos
una vez.

### Ejemplo bancario

Mostrar movimientos filtrados por cuenta y rango de fechas. Si cambia
cualquiera de los filtros, se actualiza el resultado.

``` typescript
import { combineLatest, switchMap } from 'rxjs';

const filteredMovements$ = combineLatest({
  accountId: selectedAccountId$,
  dateRange: selectedDateRange$,
}).pipe(
  switchMap(({ accountId, dateRange }) =>
    movementApi.getMovements(accountId, dateRange)
  ),
);
```

### ¿Dónde utilizarlo?

-   Combinar filtros de búsqueda.
-   Actualizar pantallas según varios controles.
-   Combinar estado de sesión y selección de cuenta.
-   Mantener una vista sincronizada con varios flujos.

**Diferencia con `forkJoin`:**

-   `forkJoin`: espera a que todos completen y emite una vez.
-   `combineLatest`: emite cada vez que cambia alguno, después de que
    todos hayan emitido al menos una vez.

------------------------------------------------------------------------

## 6.3. withLatestFrom()

### ¿Qué hace?

Cuando el Observable principal emite, combina ese valor con el último
valor disponible de otro Observable.

A diferencia de `combineLatest`, los cambios del Observable secundario
no disparan por sí solos una nueva emisión.

### Ejemplo bancario

Cuando el usuario pulsa consultar, enviamos la cuenta actualmente
seleccionada.

``` typescript
import { withLatestFrom, switchMap } from 'rxjs';

const accountStatement$ = statementButtonClicks$.pipe(
  withLatestFrom(selectedAccountId$),
  switchMap(([_, accountId]) =>
    accountApi.getStatement(accountId)
  ),
);
```

La consulta se dispara por el clic. Cambiar la cuenta por sí solo no
genera una consulta.

### ¿Dónde utilizarlo?

-   Acciones de botones que necesitan el estado más reciente.
-   Confirmaciones que requieren la cuenta seleccionada.
-   Envío de formularios junto con datos actuales de otros flujos.

El Observable secundario debe haber emitido al menos una vez para que
`withLatestFrom` pueda producir una salida.

------------------------------------------------------------------------

# 7. Operadores para compartir y reutilizar flujos

## 7.1. shareReplay()

### ¿Qué hace?

Comparte una suscripción a un Observable entre varios suscriptores y
puede reproducir emisiones anteriores a los nuevos suscriptores.

### Ejemplo bancario

Compartir una consulta de configuración de una cuenta que utilizan
varios componentes.

``` typescript
import { shareReplay } from 'rxjs';

const accountConfiguration$ = accountApi.getConfiguration().pipe(
  shareReplay({
    bufferSize: 1,
    refCount: true,
  }),
);
```

Con `bufferSize: 1`, se conserva el último valor para reproducirlo a
nuevos suscriptores.

### ¿Dónde utilizarlo?

-   Evitar solicitudes duplicadas cuando varios consumidores comparten
    el mismo flujo.
-   Reutilizar datos de configuración.
-   Compartir información de referencia que cambia poco.

### Precaución

`shareReplay` no es una solución universal de caché.

Debes pensar en:

-   Cuándo se invalida el dato.
-   Si puede quedar información obsoleta.
-   Cuánto tiempo debe mantenerse.
-   Qué ocurre al cambiar de usuario o sesión.
-   Si se están compartiendo datos sensibles entre contextos.

Nunca compartas accidentalmente información bancaria entre usuarios o
sesiones diferentes.

------------------------------------------------------------------------

# 8. Ejemplo integrado: formulario de transferencia

Vamos a combinar varios operadores en un escenario habitual de frontend.

El usuario selecciona una cuenta de origen, escribe el número de cuenta
de destino y consulta si el beneficiario existe.

``` typescript
import {
  debounceTime,
  distinctUntilChanged,
  filter,
  switchMap,
  catchError,
  of,
  map,
  startWith,
} from 'rxjs';

type BeneficiaryState =
  | { status: 'LOADING' }
  | { status: 'SUCCESS'; beneficiary: Beneficiary }
  | { status: 'ERROR'; message: string };

const beneficiaryState$ = destinationAccountControl.valueChanges.pipe(
  map(value => value.trim()),
  debounceTime(400),
  distinctUntilChanged(),
  filter(accountNumber => accountNumber.length >= 10),
  switchMap(accountNumber =>
    beneficiaryApi.findByAccount(accountNumber).pipe(
      map(beneficiary => ({
        status: 'SUCCESS',
        beneficiary,
      }) as BeneficiaryState),
      catchError(() =>
        of({
          status: 'ERROR',
          message: 'No se pudo consultar el beneficiario',
        } as BeneficiaryState)
      ),
      startWith({
        status: 'LOADING',
      } as BeneficiaryState),
    )
  ),
);
```

### ¿Qué hace cada operador en este flujo?

1.  `map`: limpia espacios del número ingresado.
2.  `debounceTime`: espera a que el usuario deje de escribir.
3.  `distinctUntilChanged`: evita consultar si el valor no cambió.
4.  `filter`: permite consultar solo cuando el número tiene la longitud
    mínima.
5.  `switchMap`: consulta el beneficiario más reciente y deja de
    escuchar la consulta anterior.
6.  `map`: transforma la respuesta en un estado de éxito.
7.  `catchError`: convierte el error en un estado explícito.
8.  `startWith`: emite el estado de carga mientras se realiza la
    consulta.

El resultado es un flujo que la interfaz puede observar para mostrar un
indicador de carga, los datos del beneficiario o un mensaje de error.

**Seguridad:** conocer el beneficiario en el frontend no autoriza una
transferencia. El backend debe volver a validar la cuenta, la
titularidad, los permisos, los límites y las reglas de riesgo al
ejecutar la operación.

------------------------------------------------------------------------

# 9. ¿Qué operador utilizar según el escenario?

  Necesidad                                               Operador recomendado
  ------------------------------------------------------- ------------------------
  Transformar una respuesta                               `map`
  Filtrar movimientos                                     `filter`
  Esperar a que termine la escritura                      `debounceTime`
  Evitar valores consecutivos repetidos                   `distinctUntilChanged`
  Consultar solo el último criterio                       `switchMap`
  Ejecutar consultas concurrentes                         `mergeMap`
  Ejecutar procesos en orden                              `concatMap`
  Ignorar nuevos clics mientras hay una petición activa   `exhaustMap`
  Manejar errores                                         `catchError`
  Ejecutar limpieza al finalizar                          `finalize`
  Reintentar un flujo                                     `retry`
  Limitar el tiempo de espera                             `timeout`
  Obtener varias respuestas que completan                 `forkJoin`
  Reaccionar a cambios de varios flujos                   `combineLatest`
  Combinar una acción con el estado más reciente          `withLatestFrom`
  Compartir y reproducir el último valor                  `shareReplay`
  Acumular valores progresivamente                        `scan`
  Completar después de N emisiones                        `take`
  Detener un flujo cuando otro emite                      `takeUntil`
  Emitir un valor inicial                                 `startWith`

------------------------------------------------------------------------

# 10. Errores frecuentes al utilizar RxJS en banca

## Error 1: utilizar switchMap para garantizar que no se duplique una transferencia

`switchMap` cancela la suscripción anterior en el cliente, pero no
garantiza que el backend no haya procesado la solicitud.

**Solución:** idempotencia en el backend, controles de concurrencia y
validaciones de negocio.

## Error 2: utilizar retry en una operación financiera sin idempotencia

Si el servidor procesa una transferencia y la respuesta se pierde, el
cliente podría repetir el cobro.

**Solución:** utilizar claves de idempotencia y diseñar el endpoint para
que una repetición de la misma operación no genere un segundo efecto.

## Error 3: confundir finalize con éxito

`finalize` se ejecuta en caso de éxito, error o cancelación.

**Solución:** gestionar explícitamente los estados de éxito y error.

## Error 4: usar forkJoin con un Observable que nunca completa

Un WebSocket o un stream de actualizaciones en tiempo real puede no
completar nunca.

**Solución:** utilizar operadores adecuados para streams continuos, como
`combineLatest`, o limitar el flujo cuando corresponda.

## Error 5: acumular suscripciones sin liberarlas

En componentes que escuchan flujos de larga duración pueden quedar
suscripciones activas después de abandonar la pantalla.

**Solución:** utilizar `takeUntil`, `takeUntilDestroyed` en Angular o
mecanismos equivalentes de gestión del ciclo de vida.

## Error 6: asumir que un Observable se ejecuta automáticamente

Muchos Observables son fríos (*cold*): su ejecución comienza al
suscribirse. Por ejemplo, una petición HTTP de Angular normalmente se
ejecuta al suscribirse.

**Solución:** comprender cuándo se crea el flujo y cuándo se produce
realmente la suscripción. Evitar suscripciones duplicadas que puedan
disparar solicitudes duplicadas.

------------------------------------------------------------------------

# 11. Plan recomendado para aprender los operadores

No intentes memorizar todos los operadores de una sola vez. Prioriza
estos grupos:

### Nivel 1 --- Fundamentales

-   `map`
-   `filter`
-   `tap`
-   `catchError`
-   `finalize`

### Nivel 2 --- Los más importantes para flujos asíncronos

-   `switchMap`
-   `mergeMap`
-   `concatMap`
-   `exhaustMap`

### Nivel 3 --- Formularios y pantallas

-   `debounceTime`
-   `distinctUntilChanged`
-   `combineLatest`
-   `withLatestFrom`
-   `startWith`

### Nivel 4 --- Control y composición

-   `forkJoin`
-   `take`
-   `takeUntil`
-   `retry`
-   `timeout`
-   `shareReplay`
-   `scan`

También es importante aprender `tap()`: permite ejecutar efectos
secundarios como logs o métricas sin transformar las emisiones. No debe
utilizarse para cambiar el valor que circula por el Observable.

------------------------------------------------------------------------

# 12. Ejercicios prácticos

Intenta resolver estos escenarios antes de consultar una solución:

1.  **Consulta de saldo:** el backend devuelve un saldo y necesitas
    convertirlo a un formato de presentación. ¿Qué operador utilizarías?
2.  **Buscador de beneficiarios:** el usuario escribe un número de
    cuenta y quieres esperar 400 ms antes de consultar. Si cambia el
    valor, solo interesa la última búsqueda. ¿Qué operadores
    combinarías?
3.  **Dashboard bancario:** necesitas cargar simultáneamente la
    información del cliente, sus cuentas y sus tarjetas. Las tres
    peticiones HTTP completan. ¿Qué operador utilizarías?
4.  **Botón de consulta:** el usuario hace varios clics mientras una
    consulta sigue activa y solo quieres atender el primero. ¿Qué
    operador utilizarías?
5.  **Procesamiento secuencial:** necesitas ejecutar tres operaciones en
    orden y no iniciar la siguiente hasta que termine la anterior. ¿Qué
    operador utilizarías?
6.  **Consulta de cotización:** el servicio externo puede fallar
    temporalmente. ¿Qué operador permite reintentar? ¿Qué precauciones
    tomarías?
7.  **Filtros de movimientos:** tienes un Observable de cuenta
    seleccionada y otro de rango de fechas. Cada cambio debe volver a
    consultar los movimientos. ¿Qué operadores utilizarías?
8.  **Transferencia:** la respuesta HTTP se pierde después de que el
    servidor posiblemente procesó la operación. ¿Por qué `retry` sin
    idempotencia puede ser peligroso?

## Respuestas para comprobar tu razonamiento

1.  `map`.
2.  `debounceTime`, `distinctUntilChanged` y `switchMap`.
3.  `forkJoin`.
4.  `exhaustMap`.
5.  `concatMap`.
6.  `retry`, con una política adecuada y considerando si la operación es
    segura para reintentar.
7.  `combineLatest` y `switchMap`.
8.  Porque el servidor podría haber ejecutado la transferencia y un
    reintento podría duplicar el efecto. Se necesita idempotencia en el
    backend.

------------------------------------------------------------------------

# Conclusión

RxJS permite trabajar con flujos asíncronos de manera declarativa. En
una aplicación bancaria puede ayudar a coordinar consultas, formularios,
filtros, actualizaciones de saldo, estados de pantalla y llamadas a
servicios externos.

Las ideas más importantes que debes dominar son:

-   `map` transforma.
-   `filter` filtra.
-   `tap` ejecuta efectos secundarios sin transformar.
-   `switchMap` prioriza el flujo más reciente.
-   `mergeMap` permite concurrencia.
-   `concatMap` conserva el orden.
-   `exhaustMap` ignora nuevas emisiones mientras hay una operación
    activa.
-   `catchError` gestiona errores.
-   `forkJoin` espera a que varios flujos completen.
-   `combineLatest` reacciona a los últimos valores de varios flujos.

**La clave no es utilizar muchos operadores, sino elegir el operador
correcto según el comportamiento que necesita el negocio.** En banca,
además, debes distinguir siempre entre controlar el flujo en el cliente
y garantizar la integridad de una operación financiera en el servidor.
