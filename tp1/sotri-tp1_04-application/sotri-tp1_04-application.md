### ¿Cómo implementar el procesamiento periódico mediante una Tarea?

Para implementar un procesamiento periódico mediante una Tarea,
se utiliza dentro del `super-loop` la función
`vTaskDelayUntil()`.

Esta función permite retrasar una tarea hasta un tiempo específico y
puede ser utilizada para asegurar que una tarea se ejecute con una frecuencia
periódica constante.

A diferencia de `vTaskDelay()`, que retrasa una tarea una cantidad determinada de
ticks desde que es llamada, esta función especifica un tiempo absoluto en el cual
la tarea decide desbloquearse.

Por ejemplo, `vTaskDelayUntil()` se puede implementar de la siguiente forma:

```c
#include "FreeRTOS.h"
#include "task.h"

void TareaPeriodica(void *pvParameters) {
    // 1. Declarar e inicializar la variable que guarda la última vez que la tarea se ejecutó
    TickType_t xUltimoTiempo;
    const TickType_t xPeriodo = pdMS_TO_TICKS(500); // Período de 500 milisegundos

    xUltimoTiempo = xTaskGetTickCount();

    for(;;) {
        // 2. Realizar el procesamiento periódico
        // TODO: Coloca aquí tu código (lectura de sensor, cálculo, etc.)

        // 3. Bloquear la tarea hasta el siguiente período exacto
        vTaskDelayUntil(&xUltimoTiempo, xPeriodo);
    }
}
```

### ¿Cuándo se ejecutará la Tarea IDLE y cómo se puede utilizar?

La tarea IDLE de FreeRTOS se ejecuta automáticamente cuando ninguna
otra tarea de la aplicación está lista para procesarse. Se utiliza
principalmente para realizar tareas de limpieza (liberar memoria de
tareas eliminadas) y ejecutar rutinas de bajo consumo, ya que siempre
tiene la prioridad más baja del sistema.

## Experimentos

### Tarea 3 - Modificar `task_led` para implementar un procesamiento periódico

Se agrega un `vTaskDelayUntil()` para que la tarea se procese cada 1 ms:

```c
void task_led(void *parameters)
{
+	TickType_t xUltimoTiempo;
+	const TickType_t periodo = pdMS_TO_TICKS(1);
	/*  Declare & Initialize Task Function variables */
	g_task_led_cnt = G_TASK_LED_CNT_INI;

	/* Print out: Task Initialized */
	LOGGER_INFO(" ");
	LOGGER_INFO("%s is running - Tick [mS] = %3d", pcTaskGetName(NULL), (int)xTaskGetTickCount());

	HAL_GPIO_WritePin(task_led_dta.gpio_port, task_led_dta.pin, LED_OFF);
+	xUltimoTiempo = xTaskGetTickCount();
	/* As per most tasks, this task is implemented in an infinite loop. */
	for (;;)
	{
		/* Update Task Counter */
		g_task_led_cnt++;

		/* Run Task Statechart */
    	task_led_statechart();
+    	vTaskDelayUntil(&xUltimoTiempo, periodo);
	}
}
```

En este caso, el sistema permanece funcional y responde cuando se pulsa el botón azul, con la salvedad de que
cuando se inspeccionan las tareas, se observa lo siguiente:

- IDLE -> READY
- Task BTN -> RUNNING (99% run time)
- Task LED -> DELAYED (1% run time)

### Tarea 4 - Modificar `task_btn` para implementar un procesamiento periodico

Se realiza la misma modificación pero en `task_btn`. En este caso, se observa
el siguiente comportamiento con las tareas:

- IDLE -> RUNNING (98%)
- Task BTN -> DELAYED
- Task LED -> DELAYED