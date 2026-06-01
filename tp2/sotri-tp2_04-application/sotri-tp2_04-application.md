## ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción?

- xQueueSendFromISR()
- xQueueSendToBackFromISR()
- xQueueSendToFronFromISR()
- xQueueReceiveFromISR()
- xSemaphoreGiveFromISR()

Estas funciones se caracterizan por:

- Ser mas simples que sus versiones completas o alternativas. Su implementacion es mas sencilla por
lo cual su ejecucion es mas rapida.

- Estan diseñadas para ser ejecutadas de forma segura dentro de una ISR pero no fuera de ellas.

- No hacen un cambio de contexto, pero devuelven si se debe hacer un cambio de contexto o no.

## ¿Métodos para delegar el procesamiento de interrupciones a una Tarea?

El procesamiento diferido de interrupciones normalmente implica registrar el motivo de la interrupcion,
borrarla dentro de la rutina de ISR y luego desbloquear una tarea del RTOS para que el procesamiento
requerido por lai nterrupcion pueda ser realizado por la tarea desbloqueada, en lugar de dentro de la
ISR. Existen dos metodos para aplazar las interrupciones:

1. Gestion centralizada de interrupciones diferidas

Cada interrupcion que utiliza este metodo se ejecuta en el contexto de la misma tarea demonio del RTOS.
Esta tarea demonio es creada por FreeRTOS y es conocida como _timer service_task.

Para delegar el procesamiento de la interrupcion a la tarea demonio de FreeRTOS hay que pasar el puntero
a la funcion que procesa la interrupcion como el parametro `xFunctionToPend` cuando se llama a
`xTimerPendFunctionCallFromISR()`. 

Como desventaja:

- todas los handlers de interrupciones se ejecutan en el mixmo contexto que la tarea demonio del RTOS 
y, por lo tanto, tienen la misma prioridad.

- `xTimerPendFunctionCallFromISR()` envia punteros al handler de interrupciones delegadas a la tarea
demonion a traves de la cola de la misma. Por lo tanto, el demonio procesa las interrupciones en el
orden en el que las recibe. y no necesariamente segun el orden de prioridades.

- Escribiendo y leyendo de la cola agrega latencia.

2. Delego del Manejo de la interrupcion controlado por la aplicacion

Cada interrupcion que usa este metodo se ejecuta en el contexto de una tarea creada por el desarrollador
de la aplicacion. Por ejemplo, desde una ISR se puede notificar a otra tarea usando `vTaskNotifyGiveFromISR()`
indicandole que haga el procesamiento de la interrupcion.

Como beneficio esto tiene:

- baja latencia (los punteros a funcion no son enviados a traves de una cola)
- Posibilidad de asignar diferentes prioridades a cada tarea usada para delegar interrupciones.
Esto permite que cada tarea matchee con la prioridad de la interrupcion.

3. ¿Cómo usar una cola para transferir datos dentro y fuera de una rutina de servicio de interrupción?

Para transferir datos dentro de una ISR, debemos usar la funcion

```c
BaseType_t xQueueSendFromISR(
                                QueueHandle_t xQueue,
                                const void *pvItemToQueue,
                                BaseType_t *pxHigherPriorityTaskWoken
);
```

parametros:

- `xQueue`

    El handler a la cola en el cual el objeto va a ser enviado.

- `pvItemToQueue`
    
    Puntero al objeto que va a ser enviado a traves de la cola.

- `pxHigherPriorityTaskWoken`

    `xQueueSendFromISR()` asignara a `*pxHigherPriorityWoken` el valor
    `pdTrue` si enviando el objeto a la cola causa que otra tarea se
    desbloquee, y la tarea desbloqueada tiene una prioridad mayor que
    la tarea actual. Si `xQueueSendFromISR()` pone este valor en `pdTRUE`
    entonces se necesita un cambio de contexto antes de que la interrupcion
    finalice.

Ejemplo de uso para una entrada/salida buffereada:

```
void vBufferISR( void )
{
    char cIn;
    BaseType_t xHigherPriorityTaskWoken;

    /* We have not woken a task at the start of the ISR. */
    xHigherPriorityTaskWoken = pdFALSE;

    /* Loop until the buffer is empty. */
    do
    {
        /* Obtain a byte from the buffer. */
        cIn = portINPUT_BYTE( RX_REGISTER_ADDRESS );

       /* Post the byte. */
       xQueueSendFromISR( xRxQueue, &cIn, &xHigherPriorityTaskWoken );

    } while( portINPUT_BYTE( BUFFER_COUNT ) );

    /* Now the buffer is empty we can switch context if necessary. */
    if( xHigherPriorityTaskWoken )
    {
        /* Actual macro used here is port specific. */
        taskYIELD_FROM_ISR ();
    }
}
```

¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?

FreeRTOS permite el anidamiento de interrupciones siempre y cuando la arquitectura en la que se quiere
correr el RTOS lo permita. En el caso de Cortex-M4, la misma define dos contantes en `FreeRTOSConfig.h`:

1. `configMAX_SYSCALL_INTERRUPT_PRIORITY`: Define la prioridad maxima de interrupcion desde la cual se
pueden llamar funciones de la API de FreeRTOS.
2.  Interrupciones con una prioridad estrictamente superior (numericamente menor en la arquitectura Cortex-M)
 a este limite nunca seran retrasadas por el RTOS, pero tienen estrictamente prohibido usar cualquier funcion
de la API de FreeRTOS.

## Actividad 3

En esta actividad se implemento la comunicacion entre `task_btn` y `app_it` a traves de un Semaforo binario
aplicando los siguientes cambios:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{

	// Check which version of the gpio triggered this callback
	if (GPIO_Pin == BTN_A_PIN)
	{
+		BaseType_t xHigherPriorityTaskWoken = pdFALSE;
+		xSemaphoreGiveFromISR(h_it_btn_bin_sem, &xHigherPriorityTaskWoken);
+		portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
	}
}
```

Adicionalmente, se implemento el siguiente cambio en `task_btn` para recibir el semaforo de la interrupcion.

```
void task_btn_statechart(void)
{
	/* Get Events to excite Task */
	/// We first check if we can take the semaphore, otherwise the task enters in blocked state
	/// once the button is pressed, the ISR gives the semaphore and this task enters running state
	/// and excites the FSM.
	///
	/// Once the FSM goes out of ST_BTN_UP, the system needs to keep checking the value of the pin
	/// for debouncing it.
	if((ST_BTN_UP == task_btn_dta.state && pdTRUE == xSemaphoreTake(h_it_btn_bin_sem, portMAX_DELAY)) ||
			(ST_BTN_UP != task_btn_dta.state && BTN_PRESSED == HAL_GPIO_ReadPin(task_btn_dta.gpio_port, task_btn_dta.pin)))
	{
		task_btn_dta.event = EV_BTN_DOWN;
	}
	else
	{
		task_btn_dta.event = EV_BTN_UP;
	}
```

El comportamiento sigue siendo igual, nada mas que ahora `task_btn` permanece bloqueada hasta que la interrupcion
llama a `xSemaphoreGiveFromISR`, en ese instante se desbloquea y procede a procesar la maquina de estados.
