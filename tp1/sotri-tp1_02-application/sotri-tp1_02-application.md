### ¿Cómo FreeRTOS asigna tiempo de procesamiento a cada Tarea?

FreeRTOS utiliza un *scheduler* (planificador) que decide qué tarea se ejecuta en cada momento. El mecanismo exacto depende de la política de planificación configurada. 

Existen tres configuraciones principales:

1. **Preemptivo con time-slicing** (`configUSE_PREEMPTION = 1` y `configUSE_TIME_SLICING = 1`)
   La planificación preemptiva funciona de la siguiente manera:
   - Para tareas de **igual prioridad**, el *scheduler* alterna entre ellas utilizando particiones de tiempo (*Round-Robin Scheduling*).
   - Si una tarea se encuentra en estado `RUNNING` y otra tarea de **mayor prioridad** pasa al estado `READY`, el CPU interrumpe la tarea actual y ejecuta la de mayor prioridad. La tarea interrumpida regresa al estado `READY`.

2. **Preemptivo sin time-slicing** (`configUSE_PREEMPTION = 1` y `configUSE_TIME_SLICING = 0`)
   Mantiene el concepto de interrupción por prioridad del caso anterior, pero **no** utiliza *quantums* de tiempo para alternar entre tareas de la misma prioridad. El *scheduler* solo realiza un cambio de contexto si:
   - Una tarea de mayor prioridad entra en el estado `READY`.
   - La tarea en ejecución cede el control al entrar en estado `BLOCKED` o `SUSPENDED`.

3. **Planificación cooperativa** (`configUSE_PREEMPTION = 0`)
   El cambio de contexto ocurre **únicamente** cuando la tarea en ejecución entra en estado `BLOCKED`, o cuando llama explícitamente a `taskYIELD()` para invocar al *scheduler* de forma manual.

---

### ¿Cómo elige FreeRTOS qué Tarea debe ejecutarse?

La selección se basa estrictamente en la **prioridad**. El *scheduler* siempre ejecutará la tarea en estado `READY` que posea la prioridad más alta disponible.

---

### ¿Cómo afecta la prioridad relativa al comportamiento del sistema?

El *scheduler* prioriza la ejecución de las tareas de mayor jerarquía. Supongamos que tenemos la **Task 1** (alta prioridad) y la **Task 2** (baja prioridad):

1. Se ejecuta la Task 1, imprime mensajes en la terminal y luego eleva la prioridad de la Task 2.
2. Como ahora la Task 2 tiene mayor prioridad, el *scheduler* desaloja a la Task 1 y comienza a ejecutar la Task 2.
3. La Task 2 imprime sus mensajes y disminuye su propia prioridad. El *scheduler* devuelve el control a la Task 1, y el ciclo se repite.

Si ambas tareas tuvieran la **misma prioridad** (y el *time-slicing* estuviera activo), el *scheduler* les asignaría el mismo tiempo de procesamiento y se ejecutarían de forma alternada. Por el contrario, si la Task 1 nunca se bloqueara ni redujera su prioridad, la Task 2 jamás obtendría tiempo de CPU, entrando en un estado de **inanición** (*starvation*).

---

### Estados en los que puede encontrarse una Tarea

- **RUNNING:** La tarea tiene el control del CPU y se está ejecutando.
- **READY:** La tarea está lista para ejecutarse, esperando que el *scheduler* le asigne tiempo de CPU.
- **BLOCKED:** La tarea está pausada, esperando un evento temporal (como un *delay*) o externo (como un semáforo o la llegada de datos).
- **SUSPENDED:** Similar a *Blocked*, pero la tarea ignora cualquier evento. Solo puede salir de este estado si otra tarea llama explícitamente a `vTaskResume()`.

---

### ¿Cómo implementar Tareas?

Una tarea en FreeRTOS debe seguir una estructura de *super-loop* (bucle infinito). Además, debe estar diseñada para ser orientada a eventos (*event-driven*) para ceder el procesador y evitar la inanición de otras tareas.

```c
void vATaskFunction(void *pvParameters)
{
    // Código de inicialización de la tarea

    for(;;)
    {
        // Código de la aplicación de la tarea
    }

    /* 
     * Las tareas nunca deben retornar de su función mediante "return". 
     * Si por algún motivo una tarea debe finalizar, debe eliminarse 
     * a sí misma llamando a vTaskDelete(NULL) para asegurar una salida limpia. 
     */
    vTaskDelete(NULL);
}
```

El prototipo de la funcion (`TaskFunction_t`) debe retornar `void` y recibir un único parámetro de tipo `void*`.
Este parámetro es sumamente útil para pasar estructuras de configuración a la tarea en el momento de su creación.

### ¿Cómo crear una o más instancias de una Tarea?

Para instanciar tareas se utiliza la función `xTaskCreate()`. Aprovechando el parámetro `pvParameters`, podemos usar una misma función base para instanciar múltiples tareas con comportamientos distintos:

```c
// 1. Estructura para pasar datos personalizados a cada instancia
typedef struct {
    int task_id;
    int delay_ms;
} task_config_t;

// 2. Función base de la tarea
void vBlinkTask(void *pvParameters) {
    // Extraer y castear el parámetro recibido
    task_config_t *config = (task_config_t *)pvParameters;
    
    printf("Iniciada Instancia %d\n", config->task_id);

    while (1) {
        printf("Ejecutando Instancia %d\n", config->task_id);
        vTaskDelay(pdMS_TO_TICKS(config->delay_ms)); // pdMS_TO_TICKS es la macro recomendada
    }
}

// 3. Configuración global (deben mantener su alcance de memoria durante la ejecución)
task_config_t config1 = { .task_id = 1, .delay_ms = 500 };
task_config_t config2 = { .task_id = 2, .delay_ms = 1000 };

// 4. Creación de las instancias
xTaskCreate(vBlinkTask, "Blink_1", 1024, (void*)&config1, 1, NULL);
xTaskCreate(vBlinkTask, "Blink_2", 2048, (void*)&config2, 1, NULL);
```

Los parámetros que recibe xTaskCreate son los siguientes:

| Parámetro | Descripción |
| --- | --- |
| `pvTaskCode` | Puntero a la función que implementa la tarea (vBlinkTask). |
| `pcName` | Nombre descriptivo (cadena de texto) utilizado exclusivamente para debugging. |
| `uxStackDepth` | Tamaño del stack asignado a la tarea (en _words_, no en _bytes) |
| `pvParameters` | Puntero a `void*` para pasar argumentos a la tarea al momento de su creación. |
| `uxPriority` | Prioridad de la tarea |
| `pxCreatedTask` | Puntero al _handle_ de la tarea para gestionarla luego (puede ser `NULL`) |



### ¿Cómo eliminar una Tarea?

Se elimina mediante la siguiente API, pasando el _handle_ de la tarea a destruir:

```c
void vTaskDelete(TaskHandle_t xTask);
```

(NOTA: Pasar `NULL` como argumento elimina la tarea desde la cual se está llamando a la función).

## Experimentos realizados

### Paso 3: Modificación de Prioridades

Se evaluó el comportamiento del sistema al alterar las prioridades relativas:

1. **Incremento de prioridad en `task_led`**: Solo se ejecutó la tarea `task_led`. El sistema entró en inanición respecto a otras entradas y dejó de responder a las pulsaciones del botón azul.
2. **Incremento de prioridad en `task_btn`**: Solo se ejecutó la tarea task_btn. El sistema únicamente reportó los estados del botón por terminal, impidiendo que el LED titilara.
3. **Restauración de prioridades**: Al igualar las prioridades a su estado original, el sistema recuperó la normalidad, compartiendo el tiempo de CPU mediante time-slicing.

### Paso 4: Creación y Eliminación Dinámica

Se instanciaron 3 tareas derivadas de `task_btn` y se configuró una condición para eliminar solo la tercera instancia desde el super-loop de `task_led`:

```c
if (h_task_btn_3 != NULL) {
    vTaskDelete(h_task_btn_3);
    h_task_btn_3 = NULL;
}
```

Bajo esta condición, se observa que `Task BTN 2` llega a ejecutarse y reportar por terminal, compartiendo funcionalidad con `Task BTN 1`:

```
[info]  Task BTN 1 - BTN HOVER
[info]  Task BTN 2 - BTN HOVER
[info]  Task LED - LED OFF
[info]  Task BTN 1 - BTN PRESSED
[info]  Task LED - LED BLINK
```

Por el contrario, si se omite la validación del puntero (la guarda `if(h_task_btn_3 != NULL)`) y no se asigna `NULL` al handle tras la eliminación, el comportamiento falla catastróficamente provocando que se borren las tres tareas generadas y dejando únicamente a `task_led` en ejecución.
