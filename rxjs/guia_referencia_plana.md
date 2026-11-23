# Operadores RxJS — Guía de referencia

## 1. Operadores de transformación

Permiten transformar los valores emitidos por un Observable o convertirlos en nuevos Observables.

- **`map`**: transforma cada valor emitido por un Observable mediante una función y devuelve el resultado. Es útil para modificar estructuras de datos, extraer propiedades o realizar cálculos sin alterar el flujo de emisiones.

- **`switchMap`**: recibe una emisión y ejecuta un nuevo Observable interno. Si llega otra emisión antes de que termine el anterior, cancela la suscripción anterior y trabaja únicamente con la más reciente. Es ideal para buscadores, autocompletados y peticiones HTTP donde solo interesa el último resultado.

- **`exhaustMap`**: ejecuta la primera operación interna e ignora todas las nuevas emisiones mientras dicha operación no haya terminado. Cuando finaliza, vuelve a aceptar nuevas emisiones. Es útil para evitar múltiples envíos de formularios, pagos duplicados o ejecuciones repetidas por varios clics.

- **`concatMap`**: recibe las emisiones y las coloca en una cola para ejecutar sus operaciones internas secuencialmente. Cada operación espera a que termine la anterior antes de comenzar. Es útil cuando se necesita garantizar el orden de ejecución, como guardar registros consecutivamente.

- **`mergeMap`**: permite ejecutar múltiples operaciones internas concurrentemente sin esperar a que terminen las anteriores. Los resultados se emiten conforme cada operación finaliza, por lo que el orden de respuesta puede ser diferente al de entrada. Es útil para peticiones independientes que pueden ejecutarse en paralelo.

- **`scan`**: acumula progresivamente los valores emitidos, aplicando una función sobre el resultado anterior y el nuevo valor. Es similar a `reduce`, pero emite el acumulado después de cada nueva emisión. Es útil para contadores, estados acumulativos y cálculos progresivos.

- **`reduce`**: acumula todos los valores emitidos y devuelve un único resultado cuando el Observable termina. Es útil para calcular sumas, totales o resultados consolidados.

- **`toArray`**: recopila todas las emisiones de un Observable y las convierte en un único arreglo que se emite cuando el Observable completa. Es útil para agrupar resultados individuales.

## 2. Operadores de filtrado

Permiten controlar qué valores deben continuar dentro del flujo reactivo.

- **`filter`**: permite que continúen únicamente las emisiones que cumplen una condición determinada. Es útil para descartar valores nulos, registros inválidos o elementos que no cumplen ciertos criterios.

- **`take`**: recibe únicamente una cantidad determinada de emisiones y después completa automáticamente la suscripción. Es útil cuando solo se necesitan los primeros valores.

- **`takeUntil`**: mantiene activa una suscripción hasta que otro Observable emite un valor. Es especialmente útil para cancelar suscripciones cuando se destruye un componente Angular.

- **`takeWhile`**: permite recibir valores mientras se cumpla una condición. Cuando esta deja de cumplirse, completa la suscripción. Puede incluir opcionalmente el primer valor que incumple la condición.

- **`first`**: obtiene la primera emisión que cumpla una condición opcional y completa la suscripción. Si no encuentra ningún valor válido antes de completarse, puede generar un error.

- **`last`**: obtiene la última emisión de un Observable cuando este completa. Es útil cuando únicamente interesa el resultado final de un proceso.

- **`skip`**: ignora una cantidad determinada de emisiones iniciales y permite que continúen las siguientes. Es útil para omitir valores iniciales que no deben procesarse.

- **`distinctUntilChanged`**: evita emitir valores consecutivos que sean iguales según una comparación. Por defecto utiliza igualdad estricta (`===`), por lo que los objetos se comparan por referencia. Es útil para evitar actualizaciones o peticiones repetidas.

- **`distinct`**: permite emitir únicamente valores que no hayan aparecido anteriormente en el flujo. A diferencia de `distinctUntilChanged`, no se limita a comparar valores consecutivos.

- **`debounceTime`**: espera un intervalo de tiempo sin recibir nuevas emisiones antes de emitir el último valor. Si llega otro valor durante ese intervalo, reinicia la espera. Es ideal para buscadores y entradas de texto.

- **`throttleTime`**: limita la frecuencia de emisiones permitiendo, por defecto, la primera emisión de cada intervalo de tiempo e ignorando las siguientes durante ese periodo. Es útil para eventos frecuentes como scroll o movimientos del mouse.

- **`auditTime`**: espera un intervalo después de recibir una emisión y, al finalizar, emite el valor más reciente recibido durante ese intervalo. Es útil para procesar eventos frecuentes de manera controlada.

- **`sampleTime`**: revisa periódicamente el último valor emitido y lo entrega si existe un valor nuevo desde el muestreo anterior. Es útil para monitorización y actualización periódica de datos.

## 3. Operadores de combinación

Permiten combinar múltiples Observables para trabajar con sus emisiones de manera conjunta.

- **`combineLatest`**: combina los valores más recientes de varios Observables. Espera hasta que todos hayan emitido al menos una vez y, posteriormente, genera una nueva combinación cuando cualquiera de ellos emite. Es útil para filtros combinados y estados dependientes.

- **`forkJoin`**: ejecuta o se suscribe a varios Observables y espera a que todos completen para emitir sus últimos valores en un único resultado. Es ideal para múltiples peticiones HTTP independientes. Si alguno nunca completa, `forkJoin` tampoco emitirá su resultado final.

- **`zip`**: combina las emisiones de varios Observables respetando su posición. Espera el primer valor de cada uno para generar la primera combinación, luego el segundo de cada uno, y así sucesivamente.

- **`merge`**: combina varios Observables en un único flujo y emite sus valores conforme llegan, sin garantizar un orden específico entre las fuentes. Es útil para unificar eventos independientes.

- **`concat`**: combina varios Observables ejecutándolos secuencialmente. No se suscribe al siguiente hasta que el anterior completa. Es útil cuando se necesita respetar un orden entre diferentes flujos.

- **`withLatestFrom`**: combina cada emisión del Observable principal con el último valor disponible de otros Observables. A diferencia de `combineLatest`, únicamente emite cuando lo hace el Observable principal.

- **`startWith`**: agrega uno o varios valores iniciales antes de las emisiones originales. Es útil para establecer estados iniciales, valores predeterminados o mostrar información antes de recibir datos.

- **`pairwise`**: agrupa las emisiones consecutivas en pares, proporcionando el valor anterior y el actual. Es útil para detectar cambios y comparar estados.

## 4. Operadores de manejo de errores

Permiten interceptar errores y definir cómo debe reaccionar el flujo.

- **`catchError`**: intercepta un error producido dentro de un Observable y permite devolver otro Observable como alternativa. Es útil para manejar fallos HTTP, proporcionar valores predeterminados o transformar errores.

- **`retry`**: vuelve a suscribirse al Observable de origen cuando ocurre un error, según la configuración establecida. Es útil para reintentar operaciones que pueden fallar temporalmente, como problemas de red. Debe utilizarse con precaución en operaciones que modifican datos.

- **`retryWhen`**: permite definir una estrategia personalizada de reintentos mediante un Observable de errores. Es útil para implementar esperas entre intentos, aunque está deprecado en RxJS 7 y se recomienda utilizar `retry` con configuración.

- **`throwError`**: crea un Observable que termina emitiendo un error. Es útil para propagar errores controlados dentro de una cadena reactiva.

- **`timeout`**: genera un error o cambia a un Observable alternativo cuando no se recibe una emisión dentro del tiempo configurado. Es útil para evitar esperas indefinidas.

## 5. Operadores de efectos secundarios y ciclo de vida

Permiten observar o controlar el comportamiento de un Observable sin necesariamente transformar sus valores.

- **`tap`**: ejecuta acciones secundarias por cada emisión, error o finalización, sin modificar los valores del flujo. Es útil para logs, depuración y métricas.

- **`finalize`**: ejecuta una función cuando la suscripción termina, ya sea por finalización normal, error o cancelación. Es útil para ocultar indicadores de carga y liberar recursos.

- **`delay`**: retrasa la entrega de las emisiones durante un intervalo determinado. Es útil para simular latencia o controlar cuándo se reciben resultados.

- **`defaultIfEmpty`**: emite un valor predeterminado cuando el Observable completa sin haber emitido ningún valor.

- **`repeat`**: vuelve a suscribirse al Observable cuando completa correctamente. Es útil para repetir procesos o consultas, teniendo cuidado de no generar ciclos innecesarios.

## 6. Operadores multicasting y compartición

Permiten compartir resultados o suscripciones entre múltiples consumidores.

- **`share`**: permite que varios suscriptores compartan una misma suscripción al Observable de origen mientras está activa. Es útil para evitar ejecuciones duplicadas de una misma operación.

- **`shareReplay`**: comparte una suscripción y conserva una cantidad configurable de emisiones anteriores para entregarlas a nuevos suscriptores. Es útil para reutilizar respuestas HTTP o compartir estados. Debe configurarse correctamente para evitar mantener suscripciones o datos innecesariamente.

## 7. Funciones y creadores de Observables

Aunque no todos son operadores, son herramientas fundamentales de RxJS.

- **`of`**: crea un Observable que emite los valores proporcionados y después completa. Es útil para pruebas, respuestas simuladas y valores estáticos.

- **`from`**: convierte estructuras compatibles, como arreglos, promesas e iterables, en Observables.

- **`interval`**: crea un Observable que emite números consecutivos periódicamente según el intervalo configurado. Es útil para temporizadores y tareas periódicas.

- **`timer`**: crea un Observable que emite después de un tiempo determinado y opcionalmente continúa emitiendo de manera periódica.

- **`fromEvent`**: convierte eventos, como clics o pulsaciones de teclado, en emisiones de un Observable. Es útil para manejar eventos del navegador de forma reactiva.

- **`defer`**: crea un nuevo Observable mediante una función cada vez que alguien se suscribe. Es útil para ejecutar lógica de inicialización en el momento de la suscripción.

- **`EMPTY`**: representa un Observable que completa inmediatamente sin emitir valores. Es útil para finalizar una rama de ejecución sin producir resultados.

- **`NEVER`**: representa un Observable que no emite valores ni completa. Es útil en escenarios específicos de control de flujos.

- **`throwError`**: crea un Observable que termina con un error. Se utiliza frecuentemente junto con `catchError` y operadores de transformación.

---

## 8. Comparación de los cuatro operadores de aplanamiento principales

| Operador | Comportamiento | Caso de uso |
|---|---|---|
| `switchMap` | Cancela la suscripción interna anterior y procesa la nueva. | Buscadores y autocompletado. |
| `exhaustMap` | Ignora nuevas emisiones mientras exista una operación interna activa. | Evitar envíos duplicados. |
| `concatMap` | Encola las operaciones internas y las ejecuta en orden. | Guardar datos secuencialmente. |
| `mergeMap` | Ejecuta operaciones internas concurrentemente. | Peticiones HTTP independientes. |

### Ejemplo práctico

Supongamos que un usuario presiona un botón cinco veces rápidamente y cada clic genera una petición HTTP que tarda dos segundos.

- **`switchMap`**: inicia una petición por clic, pero cancela la suscripción a la anterior cuando llega el siguiente. Solo mantiene activa la última.
- **`exhaustMap`**: procesa el primer clic e ignora los otros cuatro mientras la primera petición siga activa.
- **`concatMap`**: procesa los cinco clics en orden, esperando que cada petición termine antes de iniciar la siguiente.
- **`mergeMap`**: inicia las cinco peticiones sin esperar a que terminen las anteriores, permitiendo su ejecución concurrente.

**Importante:** cancelar una suscripción con `switchMap` no garantiza que una operación ya iniciada en el servidor se revierta.

## 9. Recomendaciones para Angular

| Necesidad | Operador recomendado |
|---|---|
| Buscar mientras el usuario escribe | `debounceTime` + `distinctUntilChanged` + `switchMap` |
| Evitar múltiples clics de envío | `exhaustMap` |
| Guardar cambios respetando el orden | `concatMap` |
| Ejecutar peticiones independientes en paralelo | `mergeMap` |
| Cargar varios recursos y esperar todos | `forkJoin` |
| Combinar estados reactivos | `combineLatest` |
| Cancelar suscripciones al destruir componentes | `takeUntil` o `takeUntilDestroyed` de Angular |
| Manejar errores HTTP | `catchError` |
| Reintentar errores temporales | `retry` |
| Ejecutar lógica al finalizar o cancelar | `finalize` |
| Evitar emisiones consecutivas duplicadas | `distinctUntilChanged` |
| Compartir resultados entre suscriptores | `shareReplay` |
| Registrar valores para depuración | `tap` |

## 10. Regla rápida para recordar

- **`switchMap`** → Me interesa solamente lo último.
- **`exhaustMap`** → Me interesa lo primero y no quiero interrupciones.
- **`concatMap`** → Me interesan todos, pero en orden.
- **`mergeMap`** → Me interesan todos y pueden ejecutarse en paralelo.
- **`debounceTime`** → Espero a que dejen de llegar valores.
- **`distinctUntilChanged`** → No quiero valores consecutivos repetidos.
- **`combineLatest`** → Quiero combinar los valores actuales.
- **`forkJoin`** → Quiero esperar a que todos terminen.
- **`catchError`** → Quiero controlar un error.
- **`finalize`** → Quiero ejecutar una limpieza al terminar.
- **`takeUntil`** → Quiero detenerme cuando ocurra otro evento.
- **`shareReplay`** → Quiero compartir y reutilizar emisiones anteriores.
