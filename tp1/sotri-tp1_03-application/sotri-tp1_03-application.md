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
