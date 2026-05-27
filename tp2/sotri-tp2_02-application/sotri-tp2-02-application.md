## ¿Cómo eliminar una Cola?

LLamando a:

```c
void vQueueDelete( QueueHandle_t xQueue );
```

donde `xQueue` es el `handler` de la cola a eliminar

## ¿Cómo crear una Cola?

Llamando a:

```c
QueueHandle_t xQueueCreate( UBaseType_t uxQueueLength,
                            UBaseType_t uxItemSize );
```

Crea una nueva cola y devuelve el `handler` para el cual la misma puede ser
referenciada. Para que la API este disponible,
`configSUPPORT_DYNAMIC_ALLOCATION` debe estar habilitada (= 1) en
`FreeRTOSConfig.h`.

Cada cola requiere RAM para guardar el estado de la cola y los elementos
contenidos dentro de la cola (el area de almacenamiento de la cola).
Si una cola es creada usando `xQueueCreate()` la RAM requerida es
automaticamente alocada en el heap de FreeRTOS. Si la cola es creada usando
`xQueueCreateStatic()` entonces la RAM la provee la aplicacion de forma estatica
durante el tiempo de compilacion.

Parametros:

- `uxQueueLength`: El numeor maximo de items que la cola puede alojar.
- `uxItemSize`: El tamaño, en bytes, del elemento que la cola va a guardar.

Devuelve:
- si la cola es creada de forma exitosa, un handler que apunta a la cola creada.
Si la memoria requerida para crear la cola NO puede ser alojada entonces
devuelve `NULL`.

## ¿Cómo gestiona una Cola los datos que contiene?

Una cola en FreeRTOS gestiona sus datos mediante un buffer de memoria tipo FIFO
(_First In, First Out_). Funcoona copiando los datos hacia y desde la cola
mediante un metodo _thread-safe_, lo que permite a las tareas comunicarse y
sincronizarse sin corromper la informacion

## ¿Cómo enviar datos a una Cola?

Para enviar datos a una cola se utiliza la siguiente macro:

```c
 BaseType_t xQueueSend(
                        QueueHandle_t xQueue,
                        const void * pvItemToQueue,
                        TickType_t xTicksToWait
                      );
```

Esta macro a su vez llama `xQueueGenericSend()`. La misma publica un objeto en
la cola. El objeto es encolado por copia, no por referencia. Esta funcion NO
debe ser llamada desde una  ISR. Para hacerlo desde una ISR, llamar
`xQueueSendFromISR()`.

Parametros:
- `xQueue`: _handler_ que apunta a cola a la cual el objeto va a ser enviado.
- `pvItemToQueue`: Puntero al objeto que va a ser encolado. El tamaño del objeto
fue definido cuando la cola fue creada, y esa cantidad de bytes van a ser
copiados dentro del almacenamiento de la cola.
- `xTicksToWait`: la maxima cantidad de tiempo que la tarea debe bloquearse
esperando a que haya espacio disponible en la cola, si es que la misma esta
llena. La misma va a retornar de forma inmediata si la cola esta llena y
`xTicksToWait` es igual a 0. El tiempo es definido en periodos del tick asi que
la constante `portTICK_PERIOD_MS` debe ser usada para convertir tiempo real en
ticks.

Si `INCLUDE_vTaskSuspend` es igual a 1 entonces especificando el tiempo de
bloqueo como `portMAX_DELAY` va a causar que la tarea se bloquee de forma
indefinida.

Devuelve:
- `pdPASS` si el objeto fue exitosamente encolado.
- `errQUEUE_FULL` si la operacion no fue exitos.

Ejemplo de uso:

```
struct AMessage
 {
    char ucMessageID;
    char ucData[ 20 ];
 } xMessage;

 unsigned long ulVar = 10UL;

 void vATask( void *pvParameters )
 {
 QueueHandle_t xQueue1, xQueue2;
 struct AMessage *pxMessage;

    /* Create a queue capable of containing 10 unsigned long values. */
    xQueue1 = xQueueCreate( 10, sizeof( unsigned long ) );

    /* Create a queue capable of containing 10 pointers to AMessage structures.
       These should be passed by pointer as they contain a lot of data. */
    xQueue2 = xQueueCreate( 10, sizeof( struct AMessage * ) );

    /* ... */

    if( xQueue1 != 0 )
    {
        /* Send an unsigned long. Wait for 10 ticks for space to become
           available if necessary. */
        if( xQueueSend( xQueue1,
                       ( void * ) &ulVar,
                       ( TickType_t ) 10 ) != pdPASS )
        {
            /* Failed to post the message, even after 10 ticks. */
        }
    }

    if( xQueue2 != 0 )
    {
        /* Send a pointer to a struct AMessage object. Don't block if the
           queue is already full. */
        pxMessage = & xMessage;
        xQueueSend( xQueue2, ( void * ) &pxMessage, ( TickType_t ) 0 );
    }

	/* ... Rest of task code. */
 }
```

## ¿Cómo recibir datos a una Cola?

Para recibir datos de una cola se debe utilizar la funcion:

```c
BaseType_t xQueueReceive(
                          QueueHandle_t xQueue,
                          void *pvBuffer,
                          TickType_t xTicksToWait
);
```
Es una macro que llama a la funcion `xQueueGenericReceive()`.

Su objetivo es recibir un objeto de la cola. El objeto a recibir es copiado a
un buffer con el tamaño adecuado. La misma NO debe ser usada dentro de una ISR.
Para ello referirse a `xQueueReceiveFromISR`.

Parametros:
- `xQueue`: _handler_ que apunta a la cola de la cual se quiere recibir un
objeto.
- `pvBuffer`: puntero al buffer en el cual el objeto a recibir va a ser copiado.
- `xTicksToWait`: maxima cantidad de tiempo que la tarea debe bloquearse
esperando por un objeto a recibir si la cola se encuentra vacia. Si se pasa como
argumento `xTicksToWait` igual a 0 causara que la funcion retorne de forma
inmediata si la cola esta vacia. El tiempo es definido en periodos del tick
por lo que la constante `portTICK_PERIOD_MS` debe ser usada para convertir
el tiempo real. Si `INCLUDE_vTaskSuspend` es igual a 1 entonces usando
`portMAX_DELAY` causara que la tarea se bloquee de forma indefinida
(sin timeout).

Devuelve:
- `pdPASS` si el objeto fue recibido de la cola de forma exitosa.
- `errQUEUE_EMPTY` si la cola se encuentra vacia:

Ejemplo:

```c
/* Define a variable of type struct AMMessage. The examples below demonstrate
   how to pass the whole variable through the queue, and as the structure is
   moderately large, also how to pass a reference to the variable through a queue. */
struct AMessage
{
    char ucMessageID;
    char ucData[ 20 ];
} xMessage;

/* Queue used to send and receive complete struct AMessage structures. */
QueueHandle_t xStructQueue = NULL;

/* Queue used to send and receive pointers to struct AMessage structures. */
QueueHandle_t xPointerQueue = NULL;

void vCreateQueues( void )
{
    xMessage.ucMessageID = 0xab;
    memset( &( xMessage.ucData ), 0x12, 20 );
    
    /* Create the queue used to send complete struct AMessage structures. This can
       also be created after the schedule starts, but care must be task to ensure
       nothing uses the queue until after it has been created. */
    xStructQueue = xQueueCreate(
        /* The number of items the queue can hold. */
        10,
        /* Size of each item is big enough to hold the<br /> whole structure. */
        sizeof( xMessage ) );
        
    /* Create the queue used to send pointers to struct AMessage structures. */
    xPointerQueue = xQueueCreate(
        /* The number of items the queue can hold. */
        10,
        /* Size of each item is big enough to hold only a pointer. */
        sizeof( &xMessage ) );
                          
    if( ( xStructQueue == NULL ) || ( xPointerQueue == NULL ) )
    {
        /* One or more queues were not created successfully as there was not enough
           heap memory available. Handle the error here. Queues can also be created
           statically. */
    }
}

/* Task that writes to the queues. */
void vATask( void *pvParameters )
{
    struct AMessage *pxPointerToxMessage;
    
    /* Send the entire structure to the queue created to hold 10 structures. */
    xQueueSend( /* The handle of the queue. */
                xStructQueue,
                /* The address of the xMessage variable. sizeof( struct AMessage )
                    bytes are copied from here into the queue. */
                ( void * ) &xMessage,
                /* Block time of 0 says don't block if the queue is already full.
                   Check the value returned by xQueueSend() to know if the message
                   was sent to the queue successfully. */
                ( TickType_t ) 0 );
                             
    /* Store the address of the xMessage variable in a pointer variable. */
    pxPointerToxMessage = &xMessage;
    
    /* Send the address of xMessage to the queue created to hold 10 pointers. */
    xQueueSend( /* The handle of the queue. */
                xPointerQueue,
                /* The address of the variable that holds the address of xMessage.
                   sizeof( &xMessage ) bytes are copied from here into the queue. As the
                   variable holds the address of xMessage it is the address of xMessage
                   that is copied into the queue. */
                ( void * ) &pxPointerToxMessage,
                ( TickType_t ) 0 );
                
    /* ... Rest of task code goes here. */
}

/* Task that reads from the queues. */
void vADifferentTask( void *pvParameters )
{
    struct AMessage xRxedStructure, *pxRxedPointer;
    
    if( xStructQueue != NULL )
    {
        /* Receive a message from the created queue to hold complex struct AMessage
           structure. Block for 10 ticks if a message is not immediately available.
           The value is read into a struct AMessage variable, so after calling
           xQueueReceive() xRxedStructure will hold a copy of xMessage. */
        if( xQueueReceive( xStructQueue,
                           &( xRxedStructure ),
                           ( TickType_t ) 10 ) == pdPASS )
        {
            /* xRxedStructure now contains a copy of xMessage. */
        }
    }
   
    if( xPointerQueue != NULL )
    {
        /* Receive a message from the created queue to hold pointers. Block for 10
           ticks if a message is not immediately available. The value is read into a
           pointer variable, and as the value received is the address of the xMessage
           variable, after this call pxRxedPointer will point to xMessage. */
        if( xQueueReceive( xPointerQueue,
                          &( pxRxedPointer ),
                          ( TickType_t ) 10 ) == pdPASS )
        {
            /* *pxRxedPointer now points to xMessage. */
        }
    }
   
    /* ... Rest of task code goes here. */
}   
```

## ¿Qué significa bloquearse en una Cola?

Significa que la tarea permanecera en el estado `BLOCKED` hasta que un evento
especifico en la cola suceda. Como por ejemplo:

- si la tarea esta leyendo de la cola, se desbloqueara cuando haya un objeto
disponible en ella.
- si la tarea esta escribiendo en la cola, se desbloqueara cuando la memoria
de la cola este libre.

## ¿Cómo bloquearse en varias Cola?

Utilizando lo que se llama conjunto de colas. Los conjuntos de cola son una 
caracteristica de FreeRTOS que permite a una tarea bloquearse (quedarse en
espera) al recibir datos de varias colas simultaneamente. Las colas se agrupan
en conjuntos; de esta forma, en lugar de bloquearse en una cola, la tarea se
bloquea en conjunto.

Para crear un conjunto de colas, se utiliza la funcion `xQueueCreateSet()`.
Una vez creado el conjunto, el mismo puede ser referenciado a traves de la
variable `QueueSetHandle_t`.

Para agregar un miembro al conjunto de colas, se utilizala funcion
`xQueueAddToSet()`.

Para verificar si alguno de los miembros esta listo para ser leido, se
utiliza la funcion `xQueueSelectFromSet()`.

## ¿Cómo sobrescribir datos en una Cola?

Para sobreescribir datos en una cola, el desarrollador debe usar la funcion
`xQueueOverwrite()`. La misma envia datos a la cola, y si la misma se encuentra
llena, sobreescribe los datos en la misma.

## ¿Cómo vaciar una Cola?

Para vaciar una cola se debe utilizar la funcion `xQueueReset` y devuelve la cola
a su estado original vacio.

## ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?

Las prioridades de las tareas al escribir y leer una cola afectan quien accede primero
a ella. Por ejemplo, si el productor (el que escribe) tiene mayor prioridad que el
consumidor (lector), enviara datos hasta que la cola se llene, en ese momento el lector
podra leer un solo objeto de la cola e inmediatamente el productor volvera a enviar datos
(la cola tiene espacio disponible). Por lo contrario, si el consumidor tiene mayor prioridad
que el productor, la cola permanecera vacia ya que la lectura sera de forma inmediata apenas
se agregue un dato en la misma. 

Como regla general se da el segundo caso. La prioridad del consumidor es mayor que la del
productor. Si el lector tiene menos prioridad, podria no ejecutarse a tiempo y causar
que el sistema no funcione como se espera.

