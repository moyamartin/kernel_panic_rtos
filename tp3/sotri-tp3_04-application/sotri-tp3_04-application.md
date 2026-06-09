# Análisis del código fuente — `sotri-tp3_04-application`

**Aplicación:** *RTOS - Event-Triggered Systems (ETS)* — "Security airlock" (esclusa de seguridad).
**Fuente:** CESE - Sistemas Operativos de Tiempo Real (FIUBA / UTN-FRBA).

Este documento explica el funcionamiento de los archivos `app.c`, `app_it.c`, `freertos.c` y de las tareas de la aplicación. Notar que en este proyecto **no existen** `task_entry_a.c` ni `task_exit_a.c` (nombres genéricos de actividades anteriores); en su lugar hay cuatro tareas de compuerta (`task_gate_a.c`, `task_gate_b.c`, `task_gate_c.c`, `task_gate_d.c`) más una tarea de estímulo (`task_test.c`). En este documento se analizan esas tareas en lugar de los nombres genéricos `task_entry_a`/`task_exit_a`.

> **Estado del proyecto:** se trata del *esqueleto de arranque* de un TP. La estructura de tareas, prioridades, hooks e interrupciones está montada, pero la lógica de la esclusa de seguridad y los mecanismos de sincronización (colas, semáforos, mutex) están declarados como comentarios "to be done" y todavía no se implementaron. El análisis lo señala explícitamente donde corresponde.

---

## 1. `app.c` — Inicialización de la aplicación

Es el punto de entrada de la capa de aplicación. Su única función pública, `app_init()`, se ejecuta una vez antes de arrancar el scheduler de FreeRTOS y se encarga de crear todas las tareas.

### Datos globales
- **Cadenas de presentación** (`p_app`, `p_app_`, `p_app__`): describen la aplicación; se imprimen al inicio. Aquí `p_app_` identifica la app como `"sotri-tp3_04-application: Security airlock"`.
- **Contadores globales** (`g_app_cnt`, `g_app_task_cnt`, `g_app_tick_cnt`, `g_task_idle_cnt`, `g_app_stack_overflow_cnt`, `g_tasks_cnt`): variables de diagnóstico que se inicializan a 0 con las macros `*_INI`. Son las que actualizan los *hooks* de `freertos.c`.
- **Handles de tareas** (`h_task_gate_a`, `h_task_gate_b`, `h_task_gate_c`, `h_task_gate_d`, `h_task_test`): de tipo `TaskHandle_t`, sirven para referenciar cada tarea creada.
- Hay comentarios reservados para declarar futuros `QueueHandle_t` y `SemaphoreHandle_t`/mutex — **aún no implementados**.

### Flujo de `app_init()`
1. Inicializa los contadores globales a sus valores iniciales.
2. Imprime por el `LOGGER` el mensaje de "aplicación corriendo" junto con el tick actual (`xTaskGetTickCount()`) y las tres cadenas de presentación.
3. **Crea cinco tareas** con `xTaskCreate()`, todas con stack `configMINIMAL_STACK_SIZE` y sin parámetros (`NULL`). Tras cada creación verifica el éxito con `configASSERT(pdPASS == ret)`:

   | Tarea        | Función       | Prioridad              | Handle           |
   |--------------|---------------|------------------------|------------------|
   | Task Gate A  | `task_gate_a` | `tskIDLE_PRIORITY + 3` | `h_task_gate_a`  |
   | Task Gate B  | `task_gate_b` | `tskIDLE_PRIORITY + 2` | `h_task_gate_b`  |
   | Task Gate C  | `task_gate_c` | `tskIDLE_PRIORITY + 3` | `h_task_gate_c`  |
   | Task Gate D  | `task_gate_d` | `tskIDLE_PRIORITY + 2` | `h_task_gate_d`  |
   | Task Test    | `task_test`   | `tskIDLE_PRIORITY + 1` | `h_task_test`    |

   Las compuertas A y C comparten la prioridad más alta (+3), B y D la intermedia (+2), y la tarea de *test* es la de menor prioridad de las cinco (aunque luego se sube a sí misma, ver §5).

   > Detalle a notar: en la creación de **Task Gate C** el nombre de texto pasado a `xTaskCreate()` quedó como `"Task Gate B"` (parece un copy/paste). No afecta el funcionamiento —el nombre es solo para depuración— pero conviene corregirlo.
4. Consulta el heap libre con `xPortGetFreeHeapSize()` (heap_4: arreglo `ucHeap[configTOTAL_HEAP_SIZE]`) — valor de diagnóstico.
5. Llama a `app_it_init()` para inicializar las interrupciones de la aplicación.
6. Llama a `cycle_counter_init()` (módulo DWT) para habilitar el contador de ciclos usado en mediciones de tiempo.

---

## 2. `app_it.c` — Interrupciones de la aplicación

Maneja la inicialización de interrupciones y los callbacks de GPIO externos.

### `app_it_init()`
Pensada para configurar recursos protegidos al inicio. Actualmente solo encierra una sección con primitivas de bajo nivel de Cortex-M:
- `__asm("CPSID i")` — deshabilita interrupciones globales.
- `__asm("CPSIE i")` — las vuelve a habilitar.

Entre ambas no hay código todavía ("Init to be done"): es el lugar previsto para inicializar de forma atómica un recurso compartido.

### `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`
Callback de la HAL de ST que se dispara ante una interrupción de línea EXTI (por ejemplo, al pulsar un botón). Comprueba si el pin que originó la interrupción es `BTN_A_PIN`; en ese caso debería ejecutar el trabajo asociado ("Work to be done"). **Cuerpo aún sin implementar:** aquí iría, típicamente, la notificación a una tarea desde la ISR (`...FromISR`) para señalizar un evento de la esclusa.

---

## 3. Tareas de compuerta — `task_gate_a.c`, `task_gate_b.c`, `task_gate_c.c`, `task_gate_d.c`

Las cuatro tareas son estructuralmente idénticas (cambian nombres, contadores y mensajes). Representan las cuatro compuertas de la esclusa de seguridad.

Patrón común de cada tarea, p. ej. `task_gate_a()`:
1. Inicializa su contador propio (`g_task_gate_a_cnt = 0`).
2. Imprime un mensaje de inicio con su nombre (`pcTaskGetName(NULL)`) y el tick actual.
3. Entra en un **bucle infinito** `for(;;)` que:
   - incrementa el contador de la tarea,
   - imprime un mensaje "Wait: 2500mS",
   - bloquea con `vTaskDelay(pdMS_TO_TICKS(2500))` durante 2,5 s.

Cada archivo define dos macros de retardo: `*_DEL_ZERO` (0 ms) y `*_DEL_MAX` (2500 ms); solo se usa la segunda.

En esta etapa las tareas son **periódicas y autónomas**: simplemente "laten" cada 2,5 s sin sincronizarse entre sí. El objetivo del TP es transformarlas en tareas *event-triggered*: deberían quedar bloqueadas esperando una señal/evento (semáforo, notificación o cola) producido por `task_test` o por la ISR —que represente un *pedido de apertura* (`OPEN_REQUEST_x`) o un *cierre de puerta* (`DOOR_CLOSED_x`)— en lugar de despertar solas por tiempo.

> Nota sobre prioridades: como Gate A y Gate C comparten la prioridad más alta (+3) y usan `vTaskDelay`, FreeRTOS las reparte por *round-robin* en cada quantum del tick; las compuertas B y D (+2) corren cuando las de mayor prioridad están bloqueadas.

---

## 4. `task_test.c` — Tarea de estímulo (generador de eventos)

Tarea cuyo propósito es **excitar periódicamente a las demás tareas** simulando la secuencia de eventos de la esclusa.

### Eventos
Define el enum `e_task_test_t` con los eventos del sistema:

```
Error,
OPEN_REQUEST_A, DOOR_CLOSED_A,
OPEN_REQUEST_B, DOOR_CLOSED_B,
OPEN_REQUEST_C, DOOR_CLOSED_C,
OPEN_REQUEST_D, DOOR_CLOSED_D
```

Cada compuerta tiene dos eventos asociados: un **pedido de apertura** (`OPEN_REQUEST_x`) y la **confirmación de puerta cerrada** (`DOOR_CLOSED_x`).

### Secuencia de eventos seleccionable en compilación
Mediante la macro `E_TASK_TEST_X` (fijada en `1`) y bloques `#if` se elige, en tiempo de compilación, qué arreglo `e_task_test_array[]` se usa. Cada variante define una secuencia distinta de estímulos:

| `E_TASK_TEST_X` | Secuencia (`e_task_test_array[]`)                                  |
|-----------------|-------------------------------------------------------------------|
| 0               | `{Error, …}` (variante de prueba de error)                        |
| **1 (activa)**  | `{OPEN_REQUEST_A, DOOR_CLOSED_A, OPEN_REQUEST_A, DOOR_CLOSED_A}`   |
| 2               | `{OPEN_REQUEST_A, DOOR_CLOSED_A, OPEN_REQUEST_B, DOOR_CLOSED_B}`   |
| 3               | `{OPEN_REQUEST_B, DOOR_CLOSED_B, OPEN_REQUEST_C, DOOR_CLOSED_C}`   |
| 4               | `{OPEN_REQUEST_C, DOOR_CLOSED_C, OPEN_REQUEST_D, DOOR_CLOSED_D}`   |
| 5               | `{OPEN_REQUEST_D, DOOR_CLOSED_D, OPEN_REQUEST_A, DOOR_CLOSED_A}`   |

Con la configuración actual (`X = 1`), la tarea estimula repetidamente la **compuerta A**: pide apertura, espera cierre, y repite.

### Flujo de `task_test()`
1. Inicializa su contador (`g_task_test_cnt = 0`) y registra `last_wake_time = xTaskGetTickCount()` para el control de periodo.
2. Imprime el mensaje de inicio.
3. **Se auto-eleva la prioridad:** lee su prioridad actual con `uxTaskPriorityGet(NULL)`, le suma 2 y la aplica con `vTaskPrioritySet(NULL, …)`. Así pasa de ser la de menor prioridad a la mayor de las cinco, lo que la hace arrancar de inmediato y dominar la generación de eventos.
4. En el **bucle infinito** recorre `e_task_test_array[]`:
   - incrementa `g_task_test_cnt`,
   - imprime el índice del evento,
   - en un `switch` clasifica el evento. Los `case` de los eventos reales (`OPEN_REQUEST_*`, `DOOR_CLOSED_*`) están **vacíos** ("to be done"); solo el `case Error`/`default` imprime el mensaje de error. Aquí es donde, en la implementación final, se enviaría la señal a la compuerta correspondiente (semáforo / notificación / cola).
   - espera con `vTaskDelayUntil(&last_wake_time, pdMS_TO_TICKS(5000))`, garantizando un periodo **exacto de 5000 ms** entre estímulos (a diferencia de `vTaskDelay`, que mide desde el final de la ejecución).

> Notar que `OPEN_REQUEST_B` y `DOOR_CLOSED_B` no tienen `case` propio en el `switch`: caerían en `default` (Error). Con la secuencia activa (`X = 1`) esto no se ejerce, pero es un detalle a completar si se cambia de variante.

---

## 5. `freertos.c` (capa de aplicación, `app/src/freertos.c`) — Hooks de FreeRTOS

Implementa los *hooks* (callbacks) que el kernel invoca automáticamente, según la configuración de `FreeRTOSConfig.h`:

- **`vApplicationIdleHook()`** — se llama repetidamente desde la tarea *idle* (la de menor prioridad) cuando no hay ninguna otra tarea lista para ejecutarse. Aquí solo incrementa `g_task_idle_cnt`. Es el lugar típico para poner el micro en bajo consumo. *No debe bloquear.* Requiere `configUSE_IDLE_HOOK = 1`.
- **`vApplicationTickHook()`** — se ejecuta dentro de la ISR del tick del sistema, en cada interrupción de tick. Solo incrementa `g_app_tick_cnt`. Debe ser muy breve y usar únicamente API `...FromISR`. Requiere `configUSE_TICK_HOOK = 1`.
- **`vApplicationStackOverflowHook(xTask, pcTaskName)`** — se invoca si se detecta desbordamiento de stack de una tarea (con `configCHECK_FOR_STACK_OVERFLOW` en 1 o 2). Entra en sección crítica y hace `configASSERT(0)` para **detener la ejecución y facilitar la depuración**; luego incrementa `g_app_stack_overflow_cnt`.

---

## 6. `freertos.c` (capa CubeMX, `Core/Src/freertos.c`) — Soporte del kernel

Archivo **generado por STM32CubeMX**. No contiene lógica de aplicación; provee el andamiaje que el kernel necesita:

- **Prototipos** de los hooks (`vApplicationIdleHook`, `vApplicationTickHook`, `vApplicationStackOverflowHook`) y de las funciones de estadísticas de tiempo de ejecución.
- **Implementaciones `__weak`** por defecto de esos hooks y de `configureTimerForRunTimeStats()` / `getRunTimeCounterValue()`. Al estar marcadas `__weak`, son **sobreescritas** por las versiones reales definidas en `app/src/freertos.c` (§5) cuando estas existen.
- **`vApplicationGetIdleTaskMemory()`** — requerida cuando se usa **asignación estática** (`configSUPPORT_STATIC_ALLOCATION`). Entrega al kernel los buffers estáticos para la tarea *idle*: el TCB (`xIdleTaskTCBBuffer`) y su stack (`xIdleStack[configMINIMAL_STACK_SIZE]`).
- Bloques `USER CODE BEGIN/END` reservados para código de usuario, aquí vacíos.

---

## Resumen general

| Archivo                       | Rol                                                                 |
|-------------------------------|---------------------------------------------------------------------|
| `app.c`                       | Crea las 5 tareas y arranca interrupciones y contador de ciclos.    |
| `app_it.c`                    | Init de interrupciones y callback EXTI de botón (sin lógica aún).   |
| `task_gate_a/b/c/d.c`         | Cuatro compuertas; hoy solo laten cada 2,5 s (periódicas).          |
| `task_test.c`                 | Genera eventos de la esclusa cada 5 s; se auto-eleva la prioridad.  |
| `app/src/freertos.c`          | Hooks de aplicación (idle, tick, stack overflow) → contadores.      |
| `Core/Src/freertos.c`         | Andamiaje CubeMX: hooks `__weak` y memoria estática de la idle task.|

El proyecto es la **base estructural** de la esclusa de seguridad *event-triggered*: las tareas y prioridades existen, pero la sincronización entre la tarea de estímulo, las compuertas y la ISR (mediante semáforos, notificaciones o colas) está pendiente de implementar, tal como lo indican los comentarios "to be done".
