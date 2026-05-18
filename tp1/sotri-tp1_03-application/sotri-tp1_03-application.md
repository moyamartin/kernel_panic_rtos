### Como usar el parametro de Tarea?

Para usar el parametro de la tarea, primero se pasa el puntero al dato que se quiere pasar y se hace un cast a un puntero a void

```
    // Configuración global de los parámetros para cada instancia
    task_config_t config1 = { .task_id = 1, .delay_ms = 500 };

    xTaskCreate(ATask, "ATask_1", 1024, (void*)&config1, 1, NULL);
```

Despues, en el handler de la funcion, se tiene que volver a castear al tipo de datos original:

```c
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
```


### Como cambiar la prioridad de una Tarea ya creada?

Llamando a la funcion primitiva:

```c
vTaskPrioritySet( TaskHandle_t xTask,
                  UBaseType_t uxNewPriority);
```

donde:

- `xTask`: es el `handle` de la task cuya prioridad quiere ser modificada.
- `uxNewPriority`: La prioridad a la que se quiere modificar.

## Paso 3

Se modificaron las funciones `task_btn` y `task_btn_statechart` para que tomen como parametro
un puntero a una variable tipo `task_btn_data_t`. Donde `task_btn_data_t` contiene los datos
para manejar un GPIO distinto de la placa.

Asi mismo, se creo una tarea distinta para que maneje el boton G1 (externo) mapeado al gpio PC0:

- app.c:
```c
/* Task Btn configurations */
task_btn_dta_t task_btn_dta_b1 = {
		EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
		B1_GPIO_Port, B1_Pin, &task_led_dta_ld2
};
task_btn_dta_t task_btn_dta_g1 = {
		EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
		G1_GPIO_Port, G1_Pin, &task_led_dta_ldred
};

...

    /* Task BTN thread at priority 1 */
    ret = xTaskCreate(task_btn,							/* Pointer to the function thats implement the task. */
					  "Task BTN B1",					/* Text name for the task. This is to facilitate debugging only. */
					  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
					  (void *)&task_btn_dta_b1,			/* We are using the task parameter. */
					  (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
					  &h_task_btn);						/* We are using a variable as task handle. */

    /* Check the thread was created successfully. */
    configASSERT(pdPASS == ret);

    /* Task BTN thread at priority 1 */
    ret = xTaskCreate(task_btn,							/* Pointer to the function thats implement the task. */
            "Task BTN G1",					/* Text name for the task. This is to facilitate debugging only. */
            (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
					  (void *)&task_btn_dta_g1,			/* We are using the task parameter. */
					  (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
					  &h_task_btn);						/* We are using a variable as task handle. */

    /* Check the thread was created successfully. */
    configASSERT(pdPASS == ret);
```

En este caso, al pulsar cualquiera de los dos botones el led verde de la placa (LD2) se enciende y se apaga como corresponde.
Y en los logs se observa:

```
[info]  Task BTN B1 - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN B1 - BTN HOVER
[info]  Task LED - LED OFF
[info]  Task BTN G1 - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN G1 - BTN HOVER
[info]  Task LED - LED OFF
```

## Paso 4

Se modifica `task_led` de la misma forma que `task_btn` y se agrega un LED rojo externo sobre el GPIO `PC1`.
Adicionalmente se modifica `task_led_interface` para que acepte como parametro un puntero a una variable tipo
`task_led_dta_t` y se modifica `task_btn_dta_t` para que tenga un miembro que sea un puntero a una variable tipo
`task_led_dta_t`. De esta forma, ambos botones se pueden independizar para los dos LEDs que se agregan en este paso.

```
/* Task LED configurations */
task_led_dta_t task_led_dta_ld2 = {
		false, EV_LED_OFF, ST_LED_OFF, DEL_LED_MIN,
		LD2_GPIO_Port, LD2_Pin
};
task_led_dta_t task_led_dta_ldred = {
		false, EV_LED_OFF, ST_LED_OFF, DEL_LED_MIN,
		LDRED_GPIO_Port, LDRED_Pin
};

/* Task Btn configurations */
task_btn_dta_t task_btn_dta_b1 = {
		EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
		B1_GPIO_Port, B1_Pin, &task_led_dta_ld2
};
task_btn_dta_t task_btn_dta_g1 = {
		EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
		G1_GPIO_Port, G1_Pin, &task_led_dta_ldred
};


...


    /* Task BTN thread at priority 1 */
    ret = xTaskCreate(task_btn,                         /* Pointer to the function thats implement the task. */
                      "Task BTN B1",                    /* Text name for the task. This is to facilitate debugging only. */
                      (2 * configMINIMAL_STACK_SIZE),   /* Stack depth in words. */
                      (void *)&task_btn_dta_b1,         /* We are using the task parameter. */
                      (tskIDLE_PRIORITY + 1ul),         /* This task will run at priority 1. */
                      &h_task_btn);                     /* We are using a variable as task handle. */

    /* Check the thread was created successfully. */
    configASSERT(pdPASS == ret);

    /* Task BTN thread at priority 1 */
    ret = xTaskCreate(task_btn,                         /* Pointer to the function thats implement the task. */
                      "Task BTN G1",                    /* Text name for the task. This is to facilitate debugging only. */
                      (2 * configMINIMAL_STACK_SIZE),   /* Stack depth in words. */
                      (void *)&task_btn_dta_g1,         /* We are using the task parameter. */
                      (tskIDLE_PRIORITY + 1ul),         /* This task will run at priority 1. */
                      &h_task_btn);                     /* We are using a variable as task handle. */

    /* Check the thread was created successfully. */
    configASSERT(pdPASS == ret);


    /* Task LED thread at priority 1 */
    ret = xTaskCreate(task_led,                         /* Pointer to the function thats implement the task. */
                      "Task LED LD2",                   /* Text name for the task. This is to facilitate debugging only. */
                      (2 * configMINIMAL_STACK_SIZE),   /* Stack depth in words. */
                      (void *)&task_led_dta_ld2,        /* We are not using the task parameter. */
                      (tskIDLE_PRIORITY + 2ul),         /* This task will run at priority 1. */
                      &h_task_led);                     /* We are using a variable as task handle. */

    /* Check the thread was created successfully. */
    configASSERT(pdPASS == ret);

    /* Task LED thread at priority 1 */
    ret = xTaskCreate(task_led,                         /* Pointer to the function thats implement the task. */
                      "Task LED LDRED",                 /* Text name for the task. This is to facilitate debugging only. */
                      (2 * configMINIMAL_STACK_SIZE),   /* Stack depth in words. */
                      (void *)&task_led_dta_ldred,      /* We are not using the task parameter. */
                      (tskIDLE_PRIORITY + 2ul),         /* This task will run at priority 1. */
                      &h_task_led);                     /* We are using a variable as task handle. */

    /* Check the thread was created successfully. */
    configASSERT(pdPASS == ret);
```

Finalmente, se incrementa la prioridad de las tareas de los LEDs y dentro del `super-loop` de `task_led` se reduce
la prioridad de dichas tareas a la misma que de las botones:

- `task_led.c`
```c
	for (;;)
	{
		/* Update Task Counter */
		g_task_led_cnt++;

		/* Run Task Statechart */
    	task_led_statechart(task_led_dta);
    	vTaskPrioritySet(NULL, tskIDLE_PRIORITY + 1ul);
	}
```

De esta forma podemos observar lo siguiente:

```
[info] Task LED LDRED is running - Tick [mS] =   0
[info]  
[info] Task LED LD2 is running - Tick [mS] =   0
[info]  
[info] Task BTN B1 is running - Tick [mS] =   0
[info]  
[info] Task BTN G1 is running - Tick [mS] =   1
```

las tareas de los LEDs se inician al principio y despues la tarea de los botones.
De esta forma nos aseguramos que los LEDs esten inicializados antes de que el sistema
pueda responder a los pulsadores.

Finalmente, podemos observar que el sistema responde a los pulsadores como es
esperado:

```
[info]  Task BTN B1 - BTN PRESSED
[info]  Task LED LD2 - LED BLINK
[info]  Task BTN B1 - BTN HOVER
[info]  Task LED LD2 - LED OFF
[info]  Task BTN G1 - BTN PRESSED
[info]  Task LED LDRED - LED BLINK
[info]  Task BTN G1 - BTN HOVER
[info]  Task LED LDRED - LED OFF
```

