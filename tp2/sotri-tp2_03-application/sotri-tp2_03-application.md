## ¿Cómo crear y usar semáforos binarios y semáforos contadores?

Los semaforos binarios se tuilizan tanto para la exclusion mutua como para la
sincronizacion.

Los semaforos binarios y los mutex son muy similares, pero presentan algunas
diferencias sutiles:
- los mutex incluyen un mecanismo de herencia de prioridad, mientras que los 
semaforos binarios no.

Las funciones de la API de semaforos permiten especificar un tiempo de bloqueo.
Este tiempo indica el numero maximo de ciclos que una tarea debe permanecer
bloqueada al intentar acceder un semaforo, en caso de que este no este
disponible de inmediato. Si varias tareas se bloquean en el mismo semaforo,
la tarea con mayor prioridad sera la que se desbloquee la proxima vez que el
semaforo este disponible.

Un semaforo binario puede representarse como una cola que solo puede contener
un elemento. En este caso, la cola solo puede estar llena o vacia. Las tareas
e interrupciones que utilizan la cola no necesitan saber que contiene, solo
saber si esta vacio o llena.

El mismo se puede crear utilizando la funcion:

```c
SemaphoreHandle_t xSemaphoreCreateBinary( void );
```

Por el otro lado, se encuentran los semaforos cotnadores. Asi como los semaforos
binarios pueden considerarse colas de longitud uno, los semaforos de conteno
pueden considerarse colas de longitud mayor a uno. Nuevamente, a los usuarios
del semaforo no les interesan los datos almacenados en la cola, sino si la 
misma se encuentra vacia o no.

Para crear uno, se utiliza la funcion

```c
SemaphoreHandle_t xSemaphoreCreateCounting( UBaseType_t uxMaxCount,
                                            UBaseType_t uxInitialCount);
```

## Uso del semáforo binario para sincronización entre tareas

El patron correcto de sincronizacion con semaforo binario es
**productor/consumidor**:

1. El **productor** (`task_btn`) escribe los datos compartidos y luego llama a
   `xSemaphoreGive` para señalizar.
2. El **consumidor** (`task_led`) llama a `xSemaphoreTake` bloqueandose hasta
   recibir la señal, y recien entonces lee los datos.

El semaforo nunca se devuelve desde el consumidor: cada `Give` del productor
habilita exactamente un `Take` del consumidor.

`task_btn` ahora llama `xSemaphoreGive` cada vez que event pasa a `EV_BTN_UP` o `EV_BTN_DOWN`.
`task_led` ahora llama `xSemaphoreTake` y togglea `task_led_dta.event` cuando el semaforo
devuelve `pdTrue`, que solo se habilita cuando `task_btn` llama `xSemaphoreGive`

