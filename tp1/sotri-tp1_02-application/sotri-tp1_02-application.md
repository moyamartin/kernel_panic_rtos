### ¿Cómo FreeRTOS asigna tiempo de procesamiento a cada Tarea en una aplicación?

FreeRTOS utiliza un `scheduler` que decide que tarea se ejecuta en cada momento.
El mecanismo dependera de la politica de planificacion configurada.

Hay tres tipos de configuraciones

- preemptivo con time-slicing (`configUSE_PREEMPTION = 1` y `configUSE_TIME_SLICING = 1`)

La planificacion preemptiva funciona de la siguiente manera:

    - para tares de igual prioridad, el scheduler va turnando las tareas (`Round Robing Scheduling`)
    - Si una tarea se encuentra en estado `RUNNING` y otra tarea pasa al estado `READY`, el CPU
    corre la tarea de mayor prioridad y la de menor prioridad pasa a `READY`.

- preemptivo sin time-slicing (`configUSE_PREEMPTION = 1` y  1configUSE_TIME_SLICING = 0`)

usa el mismo concepto que la configuracion anterior con respecto a las tareas de mayor prioridad,
pero no usa pedazos de tiempo para compartir el procesamiento entre tareas de menor o igual prioridad.
En este caso, el scheduler corre una nueva tarea si:

	- una tarea de mayor prioridad entra en el estado `READY`.
	- La tarea ejecutandose entra en el estado `BLOCKED` o `SUSPENDED`.

- Planificacion cooperativa

Ocurre el cambio de contexto solo cuando la tarea que corre entra en estado `BLOCKED` o cuando la tarea corriendo explicitamente llama `yield` (llama al re-scheduler de forma manual)

### Como FreeRTOS elige que Tarea debe ejecutarse en un momento dado?

FreeRTOS elige la tarea segun la prioridad de la misma. Es decir, ejecuta primero las tareas que estan en READY con mayor prioridad.

### Como la prioridad relativa de cada Tarea afecta el comportamiento del sistema?

El scheduler siempre va a priorizar la tarea con mayor prioridad, valga la redundancia. Supongamos que creamos dos Tasks: Task 1 con la mayor prioridad y Task 2 con menor prioridad.

La task 1 se ejecuta:
    1. Escribe en la terminal un par de lineas
    2. Eleva la prioridad de la Tarea 2.

A continuacion, por que Task 1 elevo la tarea de Task 2, se ejecuta la tarea 2. La tarea 2 muestra unos mensajes en la terminal y disminuye su prioridad. Por lo tanto, a continuacion se corre la Tarea 1 y asi sucesivamente.

Si ambas tareas tuviesen la misma prioridad, el scheduler les daria el mismo time-slice a ambas y
se ejecutarian una atras de la otra respectivamente. Si no cambiasen su prioridad en ningun momento,
la Task 1 seria ejecutada siempre provocando que el Thread 2 entre en inanicion.

### Cuales son los estados en los que puede encontrarse la Tarea?

- RUNNING: cuando una tarea se esta ejecutando
- READY: estado en el que la tarea esta lista para ser ejecutada.
- BLOCKED: La tarea se encuentra esperando por un evento externo o temporal.
- SUSPENDED: similar a BLOCKED pero solo puede salir de este estado si se llama a `vTaskResume()`.

### Como implementar Tareas?

Una tarea debe llevar la siguiente estructura:

```c
void vATaskFunction( void *pvParameters )
{
    for( ;; )
    {
        -- Task application code here. --
    }

    /* Tasks must not attempt to return from their implementing
       function or otherwise exit. In newer FreeRTOS port
       attempting to do so will result in an configASSERT() being
       called if it is defined. If it is necessary for a task to
       exit then have the task call vTaskDelete( NULL ) to ensure
       its exit is clean. */
    vTaskDelete( NULL );
}
```

el tipo de funcion `TaskFunction_t` se define como una funcion que
retorna `void` y toma como argumento un puntero a `void` como su
unico parametro. Todas las funciones que implementan una tarea
deben ser de este tipo. El parametro se utiliza para pasar cualquier
tipo de informacion a la misma.

Las tareas nunca deben retornar, por lo que se implementan como un
`super-loop`. Estas tareas deben ser `event-driven` asi las otras
tareas no entran en inanicion.

### Como crear una o mas instancias de una Tarea?

```c
    // Estructura para pasar datos personalizados a cada instancia
    typedef struct {
        int task_id;
        int delay_ms;
    } task_config_t;

    // Función de la tarea (se usa para todas las instancias)
    void vBlinkTask(void *pvParameters) {
        // 1. Extraer y castear el parámetro recibido
        task_config_t *config = (task_config_t *)pvParameters;
        
        printf("Iniciada Instancia %d\n", config->task_id);

        while (1) {
            // Código de tu tarea
            printf("Ejecutando Instancia %d\n", config->task_id);
            vTaskDelay(config->delay_ms / portTICK_PERIOD_MS);
        }
    }

    // Configuración global de los parámetros para cada instancia
    task_config_t config1 = { .task_id = 1, .delay_ms = 500 };
    task_config_t config2 = { .task_id = 2, .delay_ms = 1000 };

    xTaskCreate(ATask, "ATask_1", 1024, (void*)&config1, 1, NULL);
    xTaskCreate(ATask, "ATask_2", 2048, (void*)&config2, 1, NULL);
```

La tarea se instancia llamando a `xTaskCreate` como se puede observar
en el ejemplo anterior. La funcion `xTaskCreate` toma los siguientes
parametros:

```c
 BaseType_t xTaskCreate( TaskFunction_t pvTaskCode, <----------------- funcion de la tarea
                         const char * const pcName, <----------------- Nombre de la tarea, utilizado para debugging
                         const configSTACK_DEPTH_TYPE uxStackDepth, <- tamaño del stack de la tarea en Words
                         void *pvParameters, <------------------------ puntero a void que pasa cualquier tipo de informacion como parametro a la tarea
                         UBaseType_t uxPriority, <-------------------- 
                         TaskHandle_t *pxCreatedTask <---------------- handler de la task, que puede ser NULL
                       );
```

### Como eliminar una tarea?

Llamando a:

```c
void vTaskDelete( TaskHandle_t xTask );
```

donde xTask es el handler de una tarea.

## Paso 3

Se realizaron tres experimentos:

1- se incrementa la prioridad de `task_led`. En este caso, solo se ejecuta la tarea `task_led` y el sistema deja de responder ante las pulsaciones del boton azul.

2- se incrementa la prioridad de `task_btn`. En este caso, solo se ejecuta la tarea `task_btn` y el sistema solamente muestra los estados de esta task por la terminal y el LED no titila.

3- Se restauran ambas prioridades y el sistema vuelve a su normalidad.

## Paso 4

Al agregar las 3 tareas distintas de `task_btn` y eliminar una sola dentro del `super-loop` Tarea de `task_led`:

```c
    	if(h_task_btn_3 != NULL){
    		vTaskDelete(h_task_btn_3);
    		h_task_btn_3 = NULL;
    	}
```

Se puede observar que a veces `Task BTN 2` llega a mostrar por la terminal que fue ejecutada compartiendo la funcionalidad con `Task BTN 1` como se puede observar en el siguiente log:

```
[info]  Task BTN 1 - BTN HOVER
[info]  Task BTN 2 - BTN HOVER
[info]  Task LED - LED OFF
[info]  Task BTN 1 - BTN PRESSED
[info]  Task LED - LED BLINK
```

Por el otro lado, si eliminamos la guarda que verifica `h_task_btn_3` y no asignamos `NULL` a `h_task_btn_3`, se borran las tres tareas generadas de `task_btn` y solo se corre `task_led`.
