# Análisis del código fuente — `sotri-tp3_03-application`

**Aplicación:** *RTOS - Event-Triggered Systems (ETS)* — "Vehicular crossing" (cruce vehicular).
**Fuente:** CESE - Sistemas Operativos de Tiempo Real (FIUBA / UTN-FRBA).

Este documento explica el funcionamiento de los archivos `app.c`, `app_it.c`, `freertos.c` y de las tareas de la aplicación. Notar que en este proyecto **no existen** `task_a.c` ni `task_b.c`; en su lugar hay cuatro tareas de evento (`task_entry_a.c`, `task_exit_a.c`, `task_entry_b.c`, `task_exit_b.c`) más una tarea de estímulo (`task_test.c`). En este documento se analizan esas tareas en lugar de los nombres genéricos `task_a`/`task_b`.

> **Estado del proyecto:** se trata del *esqueleto de arranque* de un TP. La estructura de tareas, prioridades, hooks e interrupciones está montada, pero la lógica del cruce vehicular y los mecanismos de sincronización (colas, semáforos, mutex) están declarados como comentarios "to be done" y todavía no se implementaron. El análisis lo señala explícitamente donde corresponde.

---

## 1. `app.c` — Inicialización de la aplicación

Es el punto de entrada de la capa de aplicación. Su única función pública, `app_init()`, se ejecuta una vez antes de arrancar el scheduler de FreeRTOS y se encarga de crear todas las tareas.

### Datos globales
- **Cadenas de presentación** (`p_app`, `p_app_`, `p_app__`): describen la aplicación; se imprimen al inicio.
- **Contadores globales** (`g_app_cnt`, `g_app_task_cnt`, `g_app_tick_cnt`, `g_task_idle_cnt`, `g_app_stack_overflow_cnt`, `g_tasks_cnt`): variables de diagnóstico que se inicializan a 0 con las macros `*_INI`. Son las que actualizan los *hooks* de `freertos.c`.
- **Handles de tareas** (`h_task_entry_a`, `h_task_exit_a`, `h_task_entry_b`, `h_task_exit_b`, `h_task_test`): de tipo `TaskHandle_t`, sirven para referenciar cada tarea creada.
- Hay comentarios reservados para declarar futuros `QueueHandle_t` y `SemaphoreHandle_t`/mutex — **aún no implementados**.

### Flujo de `app_init()`
1. Inicializa los contadores globales a sus valores iniciales.
2. Imprime por el `LOGGER` el mensaje de "aplicación corriendo" junto con el tick actual (`xTaskGetTickCount()`) y las tres cadenas de presentación.
3. **Crea cinco tareas** con `xTaskCreate()`, todas con stack `configMINIMAL_STACK_SIZE` y sin parámetros (`NULL`). Tras cada creación verifica el éxito con `configASSERT(pdPASS == ret)`:

   | Tarea          | Función        | Prioridad             | Handle            |
   |----------------|----------------|-----------------------|-------------------|
   | Task Entry A   | `task_entry_a` | `tskIDLE_PRIORITY + 3`| `h_task_entry_a`  |
   | Task Exit A    | `task_exit_a`  | `tskIDLE_PRIORITY + 2`| `h_task_exit_a`   |
   | Task Entry B   | `task_entry_b` | `tskIDLE_PRIORITY + 3`| `h_task_entry_b`  |
   | Task Exit B    | `task_exit_b`  | `tskIDLE_PRIORITY + 2`| `h_task_exit_b`   |
   | Task Test      | `task_test`    | `tskIDLE_PRIORITY + 1`| `h_task_test`     |

   Las tareas de *entrada* (A y B) tienen mayor prioridad que las de *salida*, y la tarea de *test* es la de menor prioridad de las cinco (aunque luego se sube a sí misma, ver §5).
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
Callback de la HAL de ST que se dispara ante una interrupción de línea EXTI (por ejemplo, al pulsar un botón). Comprueba si el pin que originó la interrupción es `BTN_A_PIN`; en ese caso debería ejecutar el trabajo asociado ("Work to be done"). **Cuerpo aún sin implementar:** aquí iría, típicamente, la notificación a una tarea desde la ISR (`...FromISR`) para señalizar un evento del cruce.

---

## 3. Tareas de entrada/salida — `task_entry_a.c`, `task_exit_a.c`, `task_entry_b.c`, `task_exit_b.c`

Las cuatro tareas son estructuralmente idénticas (cambian nombres, contadores y mensajes). Representan los cuatro eventos del cruce vehicular: entrada A, salida A, entrada B, salida B.

Patrón común de cada tarea, p. ej. `task_entry_a()`:
1. Inicializa su contador propio (`g_task_entry_a_cnt = 0`).
2. Imprime un mensaje de inicio con su nombre (`pcTaskGetName(NULL)`) y el tick actual.
3. Entra en un **bucle infinito** `for(;;)` que:
   - incrementa el contador de la tarea,
   - imprime un mensaje "Wait: 2500mS",
   - bloquea con `vTaskDelay(pdMS_TO_TICKS(2500))` durante 2,5 s.

Cada archivo define dos macros de retardo: `*_DEL_ZERO` (0 ms) y `*_DEL_MAX` (2500 ms); solo se usa la segunda.

En esta etapa las tareas son **periódicas y autónomas**: simplemente "laten" cada 2,5 s sin sincronizarse entre sí. El objetivo del TP es transformarlas en tareas *event-triggered*: deberían quedar bloqueadas esperando una señal/evento (semáforo, notificación o cola) producido por `task_test` o por la ISR, en lugar de despertar solas por tiempo.

> Nota sobre prioridades: como Entry A y Entry B comparten la prioridad más alta (+3) y usan `vTaskDelay`, FreeRTOS las reparte por *round-robin* en cada quantum del tick; las de salida (+2) corren cuando las de entrada están bloqueadas.

---

## 4. `task_test.c` — Tarea de estímulo (generador de eventos)

Tarea cuyo propósito es **excitar periódicamente a las demás tareas** simulando una secuencia de eventos del cruce.

### Tipos y datos
- `enum e_task_test_t { Error, Entry_A, Entry_B, Exit_A, Exit_B }`: enumera los posibles eventos.
- `e_task_test_array[]`: arreglo de eventos a reproducir. Su contenido se selecciona en **tiempo de compilación** con la macro `E_TASK_TEST_X` (bloques `#if`). Con el valor actual `E_TASK_TEST_X == 1`, el arreglo es `{Entry_A, Exit_A}`. Otros valores definen secuencias más largas o un caso de `Error`.
- Macro de retardo `TASK_TEST_TICK_DEL_MAX = 5000 ms`.

### Flujo de `task_test()`
1. Inicializa su contador y guarda `last_wake_time = xTaskGetTickCount()`.
2. Imprime el mensaje de inicio.
3. **Eleva su propia prioridad**: lee su prioridad actual con `uxTaskPriorityGet(NULL)`, le suma 2, y la aplica con `vTaskPrioritySet(NULL, ...)`. Como arrancó en `+1`, pasa a `+3`. El efecto buscado es que, una vez arrancadas las demás tareas, `task_test` tome el control para empezar a generar eventos.
4. Bucle infinito que recorre `e_task_test_array`:
   - por cada elemento incrementa `g_task_test_cnt` e imprime el índice,
   - un `switch` sobre el evento decide qué hacer: los casos `Entry_A`/`Exit_A` están **vacíos** (aquí debería señalizarse a la tarea correspondiente: dar un semáforo, enviar a una cola o notificar la tarea); el caso `Error`/`default` imprime un mensaje de error,
   - tras cada evento espera con `vTaskDelayUntil(&last_wake_time, 5000ms)`.

Uso de `vTaskDelayUntil` (en contraste con el `vTaskDelay` de las otras tareas): garantiza un **período fijo y absoluto** de 5000 ms entre activaciones (frecuencia constante, sin acumular *drift* por el tiempo de ejecución), apropiado para una tarea periódica de estímulo.

---

## 5. `freertos.c` (carpeta `app/src`) — Hook functions

Define las funciones *callback* (hooks) que el kernel de FreeRTOS invoca automáticamente, siempre que estén habilitadas por configuración. Aquí se usan para mantener los contadores de diagnóstico declarados en `app.c`.

- **`vApplicationIdleHook()`** — se ejecuta repetidamente desde la *idle task* (la de menor prioridad), solo cuando ninguna otra tarea puede correr. Requiere `configUSE_IDLE_HOOK == 1`. No debe llamar APIs bloqueantes. Aquí incrementa `g_task_idle_cnt`. Es el lugar típico para poner el micro en bajo consumo.
- **`vApplicationTickHook()`** — se ejecuta dentro de la ISR del tick del sistema. Requiere `configUSE_TICK_HOOK == 1`. Al correr en contexto de interrupción debe ser muy breve y solo usar APIs `...FromISR`. Aquí incrementa `g_app_tick_cnt`.
- **`vApplicationStackOverflowHook(xTask, pcTaskName)`** — se invoca si se detecta *desbordamiento de pila* (requiere `configCHECK_FOR_STACK_OVERFLOW` en 1 o 2). Entra en sección crítica (`taskENTER_CRITICAL`), cuelga la ejecución con `configASSERT(0)` para permitir depuración y luego incrementa `g_app_stack_overflow_cnt`. Recibe el handle y el nombre de la tarea culpable.

> Nota: en `Core/Src/` existe otro `freertos.c` generado por STM32CubeMX. El analizado aquí es el de la capa de aplicación (`app/src/freertos.c`), que contiene los hooks de usuario.

---

## 6. Visión de conjunto

```
                 app_init()  (app.c)
                     │  crea 5 tareas + init IT + cycle counter
                     ▼
 ┌─────────────┬─────────────┬─────────────┬─────────────┬───────────────┐
 │ task_entry_a│ task_exit_a │ task_entry_b│ task_exit_b │   task_test   │
 │   (prio +3) │  (prio +2)  │   (prio +3) │  (prio +2)  │ (+1 → sube +3)│
 │  delay 2.5s │  delay 2.5s │  delay 2.5s │  delay 2.5s │ período 5s    │
 └─────────────┴─────────────┴─────────────┴─────────────┴───────────────┘
        ▲                                                       │
        └──── (a implementar: semáforo/cola/notificación) ──────┘
                          eventos de cruce

 app_it.c   : EXTI / botón → (a implementar) notificación desde ISR
 freertos.c : hooks idle / tick / stack-overflow → contadores de diagnóstico
```

**Resumen:** `app.c` arma la aplicación creando cinco tareas con un esquema de prioridades pensado para un sistema de cruce vehicular dirigido por eventos. Las cuatro tareas de entrada/salida hoy solo son periódicas (laten cada 2,5 s); `task_test` actúa como generador de estímulos cada 5 s y se autopromueve en prioridad para tomar el control. `app_it.c` prepara el manejo de interrupciones externas (botón) y `freertos.c` provee los hooks de diagnóstico del kernel. La parte de **sincronización entre tareas** (colas/semáforos/mutex) y la **lógica concreta del cruce** están planificadas en el código mediante comentarios pero todavía no implementadas: ese es el trabajo a completar.

---

# 7. Implementación de la sincronización del cruce (trabajo realizado)

> Las secciones 1–6 describen el esqueleto original. Esta sección documenta la solución de sincronización efectivamente implementada en `task_entry_a/b` y `task_exit_a/b`.

## 7.1. Modelo del cruce

El cruce es de **un solo sentido a la vez** (A→A o B→B), **sin turnos**:

- Mientras circula al menos un vehículo por un sentido, el **semáforo del sentido opuesto está en Rojo** (exclusión de dirección).
- No hay alternancia forzada: **pasa el que llega primero**; siguen entrando los del mismo sentido **hasta colmar la capacidad** (`G_TASKS_CNT_MAX = 3`); el resto no entra.
- Cuando el cruce **se vacía** (`g_tasks_cnt == 0`), queda libre y vuelve a pasar el primero que llegue, de cualquier sentido.

## 7.2. Recursos compartidos y primitivas

| Recurso | Tipo | Rol |
|---|---|---|
| `g_tasks_cnt` | `uint32_t` | Vehículos circulando en el cruce (capacidad). |
| `h_mutex_mut_sem` | mutex | Protege el acceso a `g_tasks_cnt`, `semaforo_a` y `semaforo_b`. |
| `semaforo_a`, `semaforo_b` | `uint8_t` (1 = verde, 0 = rojo) | Luces de cada sentido. Inicializan **ambas en verde** (cruce libre). |
| `h_entry_a/b_bin_sem`, `h_exit_a/b_bin_sem` | semáforos binarios | Eventos de llegada/salida, disparados por `task_test`. |

> Se descartó el **semáforo contador** (`xSemaphoreCreateCounting`) en favor de **contador + mutex**, tal como pide el enunciado: el mutex vuelve atómica la decisión "cambiar ocupación + actualizar luces".

## 7.3. Lógica de ingreso (`task_entry_a`, simétrica en B)

Dentro de la sección crítica (`h_mutex_mut_sem`):

```c
if (g_tasks_cnt < G_TASKS_CNT_MAX && semaforo_a == 1) {  /* mi luz en verde y hay lugar */
    g_tasks_cnt++;
    if (g_tasks_cnt == G_TASKS_CNT_MAX) {
        semaforo_a = 0;                 /* capacidad colmada: mi luz a rojo */
    }
    semaforo_b = 0;                     /* circula A: opuesto a rojo */
}
/* si la condición es falsa, el vehículo no ingresa */
```

Clave: se chequea la **luz propia** (`semaforo_a == 1`), que ya implica "el opuesto no está circulando y no está lleno".

## 7.4. Lógica de salida (`task_exit_a`, simétrica en B)

```c
if (g_tasks_cnt > 0) {                  /* guarda anti-underflow */
    g_tasks_cnt--;
}
if (semaforo_a == 0) {                  /* estaba lleno: reabro mi sentido */
    semaforo_a = 1;
}
if (g_tasks_cnt == 0) {                 /* cruce vacío: libero el opuesto */
    semaforo_b = 1;
}
```

La identidad de la tarea aporta el sentido activo: `exit_a` solo corre cuando salen vehículos de A, por eso no hace falta una variable extra de "sentido activo".

## 7.5. Invariante resultante

- `g_tasks_cnt > 0` por A ⟹ `semaforo_b == 0` (y viceversa).
- `g_tasks_cnt == 0` ⟹ ambas luces en verde.
- `g_tasks_cnt == G_TASKS_CNT_MAX` ⟹ ambas luces en rojo.

## 7.6. Limitaciones conocidas

- **El evento rechazado se descarta**: si la luz está en rojo o el cruce está lleno, el `h_entry_*_bin_sem` ya fue consumido y el vehículo no se re-encola (no "espera" en una cola, simplemente no entra).
- Una vez vacío el cruce, cada `Exit` espurio adicional vuelve a loguear "Transito liberado" (cosmético; la guarda anti-underflow evita corromper `g_tasks_cnt`).

---

# 8. Observaciones de los tests

Las secuencias de estímulo se seleccionan con `E_TASK_TEST_X` en `task_test.c`. **Todos los tests ejercitan solo el sentido A**, por lo que la exclusión de dirección se observa indirectamente (la luz B pasa a rojo), sin un vehículo B intentando entrar. Estado inicial en todos: `g_tasks_cnt = 0`, `semaforo_a = semaforo_b = 1` (verde).

| Test | `e_task_test_array` | Qué demuestra |
|---|---|---|
| 1 | `{Entry_A, Exit_A}` | Ingreso/egreso básico de 1 vehículo y liberación del cruce. |
| 2 | `{Entry_A×2, Exit_A×2}` | Dos vehículos simultáneos (< capacidad); B en rojo mientras hay tránsito. |
| 3 | `{Entry_A×3, Exit_A×3}` | Se alcanza la capacidad (3): ambas luces en rojo, y reapertura al salir. |
| 4 | `{Entry_A×4, Exit_A×4}` | **Rechazo por capacidad** y **guarda anti-underflow**. |
| 5 | `{Entry_A×5, Exit_A×5}` | Igual que 4, con 2 rechazos y 2 egresos espurios. |

## 8.1. Lectura de las trazas

**Test 1** — `Entry_A` → `g_tasks_cnt=1`, *"Vehiculo Circulando por A"* + *"Semaforo B en rojo"*; `Exit_A` → `g_tasks_cnt=0`, *"Transito liberado. Semaforo B en verde"*.

**Test 2** — dos `Entry_A` llevan `g_tasks_cnt` a 2 (cada uno reafirma *"Semaforo B en rojo"*, A sigue verde porque hay lugar). El segundo `Exit_A` vacía el cruce y libera B.

**Test 3** — el **tercer** `Entry_A` lleva `g_tasks_cnt` a 3: aparece *"Semaforo A en rojo"* **y** *"Semaforo B en rojo"* (cruce lleno). El primer `Exit_A` reabre A (*"Semaforo A en verde"*) sin liberar B (todavía hay tránsito); el último `Exit_A` libera B.

**Test 4** — confirma dos comportamientos:
- En el **índice 3** (4º `Entry_A`) **no** aparece *"Vehiculo Circulando por A"*: la condición `g_tasks_cnt < 3` es falsa → **vehículo rechazado por capacidad**.
- En el **índice 7** (4º `Exit_A`) **no** aparece *"Vehiculo saliendo de A"* pero sí *"Transito liberado…"*: como `g_tasks_cnt` ya era 0, la **guarda `if (g_tasks_cnt > 0)`** evita el decremento (no hay underflow); solo se reafirma la liberación.

**Test 5** — misma evidencia que el test 4, amplificada: los índices 3 y 4 son **dos ingresos rechazados** (sin *"Vehiculo Circulando"*), y los índices 8 y 9 son **dos egresos espurios** (sin *"Vehiculo saliendo"*, solo *"Transito liberado…"* repetido). Ningún `g_tasks_cnt` se corrompe.

## 8.2. Conclusión

Las trazas validan el modelo de §7: capacidad limitada a 3, exclusión del sentido opuesto (B en rojo mientras circula A), liberación del cruce al vaciarse, **rechazo de ingresos cuando está lleno** y **protección ante egresos sin vehículo** (anti-underflow). Queda pendiente, si la cátedra lo pide, ejercitar el sentido B y/o implementar la **espera** real del vehículo rechazado (re-encolado) en lugar de su descarte.

---

# 9. Mecanismo de sincronización entre `task_test` y las tareas de evento

> Las secciones 7 y 8 describen *qué* hace la lógica del cruce. Esta sección documenta *cómo* se sincronizan las tareas: el flujo de señalización productor/consumidor que conecta `task_test` con `task_entry_a`, `task_exit_a`, `task_entry_b` y `task_exit_b`.

## 9.1. Primitivas involucradas

El mecanismo se apoya en **dos clases de primitivas**, con roles distintos:

| Primitiva | Tipo | Rol en la sincronización |
|---|---|---|
| `h_entry_a_bin_sem`, `h_exit_a_bin_sem`, `h_entry_b_bin_sem`, `h_exit_b_bin_sem` | semáforo binario | **Señalización de eventos** (event-triggering). Acoplan 1 a 1 cada evento de `task_test` con su tarea consumidora. |
| `h_mutex_mut_sem` | mutex | **Exclusión mutua** sobre el estado compartido (`g_tasks_cnt`, `semaforo_a`, `semaforo_b`). |

Todas se crean en `app_init()` (`app.c`) **antes** de arrancar el scheduler:
- los cuatro binarios con `xSemaphoreCreateBinary()` — nacen **vacíos** (tomar bloquea hasta que alguien dé);
- el mutex con `xSemaphoreCreateMutex()` — nace **disponible**.

Cada uno se registra con `vQueueAddToRegistry()` para facilitar la depuración (visible por nombre en el visor de RTOS). Nótese que `h_capacity_count_sem` está declarado pero **no se crea ni se usa**: es un vestigio de la idea descartada del semáforo contador (ver §7.2).

## 9.2. Patrón productor/consumidor

La sincronización es un patrón **productor único / consumidores dedicados** sobre semáforos binarios usados como *señales de evento* (no como locks):

- **Productor:** `task_test`. En cada iteración recorre `e_task_test_array` y, según el evento, hace **`xSemaphoreGive(...)`** sobre el binario correspondiente. *Da* la señal y sigue; no espera respuesta.
- **Consumidores:** las cuatro tareas de evento. Cada una está bloqueada en **`xSemaphoreTake(<su binario>, portMAX_DELAY)`** y solo se despierta cuando `task_test` da *ese* semáforo.

Correspondencia evento → semáforo → tarea (en el `switch` de `task_test.c`):

| Evento (`e_task_test_t`) | `task_test` ejecuta | Despierta a |
|---|---|---|
| `Entry_A` | `xSemaphoreGive(h_entry_a_bin_sem)` | `task_entry_a` |
| `Exit_A`  | `xSemaphoreGive(h_exit_a_bin_sem)`  | `task_exit_a`  |
| `Entry_B` | `xSemaphoreGive(h_entry_b_bin_sem)` | `task_entry_b` |
| `Exit_B`  | `xSemaphoreGive(h_exit_b_bin_sem)`  | `task_exit_b`  |
| `Error` / `default` | — (solo loguea error) | ninguna |

Esto convierte a las tareas en **event-triggered**: ya no despiertan solas por tiempo (como en el esqueleto de §3), sino que permanecen bloqueadas en su `Take` hasta recibir la señal. El `vTaskDelay(2500ms)` que conservan al final del lazo es un *retardo de servicio* (simula la duración de la maniobra), no el disparador del trabajo.

## 9.3. Anatomía de una tarea consumidora

Las cuatro consumidoras comparten la misma estructura de dos niveles de bloqueo anidados (ej. `task_entry_a`):

```c
for (;;) {
    g_task_entry_a_cnt++;

    xSemaphoreTake(h_entry_a_bin_sem, portMAX_DELAY);   /* (1) espera el EVENTO */
    {
        xSemaphoreTake(h_mutex_mut_sem, portMAX_DELAY); /* (2) toma el LOCK     */
        {
            /* sección crítica: lee/modifica g_tasks_cnt, semaforo_a, semaforo_b */
        }
        xSemaphoreGive(h_mutex_mut_sem);                /* libera el LOCK       */
    }
    LOGGER_INFO(...);
    vTaskDelay(TASK_ENTRY_A_DEL_MAX);                   /* retardo de servicio  */
}
```

Los dos `Take` cumplen funciones **completamente distintas**:
1. **`h_entry_a_bin_sem` (señal de evento):** sincroniza *temporalmente* la tarea con `task_test`. Mientras no haya evento, la tarea no consume CPU (bloqueada). Cada `Give` habilita exactamente **una** vuelta del lazo (el binario satura en 1: dos `Give` seguidos sin `Take` intermedio se colapsan en un solo evento — relevante para entender por qué los eventos deben espaciarse 5 s).
2. **`h_mutex_mut_sem` (lock):** garantiza que la actualización de `g_tasks_cnt` + luces sea **atómica** frente a las otras tres tareas de evento. Sin él, dos tareas que despertaran "a la vez" podrían intercalar lectura/escritura y romper el invariante de §7.5.

`task_exit_a/b` siguen el mismo patrón pero sin la condición de capacidad: tras tomar su binario y el mutex, decrementan y reabren luces (§7.4).

## 9.4. Secuencia de arranque e inicialización de los semáforos

Hay un detalle de inicialización en `task_entry_a` (ver commit *"inicializa semaforos en task_entry_a"*) que normaliza el estado antes del régimen permanente:

```c
xSemaphoreTake(h_entry_a_bin_sem, 0);   /* deja el binario en 0 (vacío) */
xSemaphoreTake(h_entry_b_bin_sem, 0);   /* idem para B */
```

Se ejecutan **una sola vez**, antes del `for(;;)`, con timeout `0` (no bloqueante): aseguran que ambos semáforos de entrada arranquen **vacíos** aunque algún evento espurio se hubiera señalizado durante el arranque, de modo que la primera vez que una tarea de entrada corre lo haga *solo* por un `Give` real de `task_test`. Tras eso, `task_entry_a` reajusta su prioridad a la de `task_entry_b` para que ambas vías queden en igualdad de condiciones.

> Nota: como los binarios ya nacen vacíos de `xSemaphoreCreateBinary()`, estos `Take` iniciales son **defensivos/idempotentes** (patrón heredado del esqueleto FreeRTOS). No cambian el resultado en el flujo normal, pero documentan explícitamente la precondición "semáforos vacíos al entrar al lazo".

## 9.5. Diagrama del flujo de señalización

```
        task_test  (productor, período 5 s, prioridad +3)
            │  recorre e_task_test_array
            │  switch(evento) → xSemaphoreGive(...)
            ├───────────────┬───────────────┬───────────────┐
            ▼               ▼               ▼               ▼
   h_entry_a_bin_sem  h_exit_a_bin_sem  h_entry_b_bin_sem  h_exit_b_bin_sem
            │               │               │               │   (semáforos binarios = señales)
        Take▼           Take▼           Take▼           Take▼
     task_entry_a    task_exit_a     task_entry_b    task_exit_b   (consumidores, bloqueados en Take)
            │               │               │               │
            └───────────────┴───────┬───────┴───────────────┘
                                     ▼
                          Take ──► h_mutex_mut_sem ──► Give     (exclusión mutua)
                                     │
                                     ▼
                   sección crítica: g_tasks_cnt, semaforo_a, semaforo_b
```

**Resumen del mecanismo:** `task_test` actúa como *fuente de eventos* y desacopla la generación del estímulo de su procesamiento mediante cuatro semáforos binarios (uno por evento), cada uno con una única tarea consumidora dedicada que espera bloqueada en él. Una vez despierta, la consumidora serializa su acceso al estado compartido del cruce con un único mutex (`h_mutex_mut_sem`), garantizando atomicidad e integridad del invariante (§7.5). Es una combinación clásica de **señalización binaria para sincronización temporal** + **mutex para sincronización de datos**.
