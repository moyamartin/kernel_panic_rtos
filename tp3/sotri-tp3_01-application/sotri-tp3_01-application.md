# Análisis del código fuente — sotri-tp3_01-application

**Trabajo Práctico N° 3 — Sincronización de Tareas de FreeRTOS**
**Microcontrolador:** STM32F446RETx (Cortex-M4, 84 MHz)
**RTOS:** FreeRTOS integrado con STM32CubeMX (CMSIS-RTOS)
**Patrón objetivo:** Productor-Consumidor (*Producer-Consumer*)
*(Fuente: The Little Book of Semaphores, Allen B. Downey — §4.1 Producer-consumer problem)*

> **Nota importante sobre el estado del código:** Los archivos analizados corresponden a la **plantilla inicial (scaffolding)** del ejercicio. Las tareas A y B y la inicialización de la aplicación contienen los *comentarios guía* para crear una **cola** (queue), un **semáforo** y/o un **mutex**, pero **estos objetos de sincronización todavía no están creados ni utilizados**. En el estado actual, las tareas A y B son simplemente dos hilos periódicos independientes que imprimen por consola y se duermen. El análisis describe tanto lo que el código hace hoy como la intención del patrón productor-consumidor que se debe completar.

---

## 1. Visión general de los archivos

| Archivo | Rol |
|---------|-----|
| `app.c` | Inicialización de la aplicación: crea las tareas, reserva el lugar para los objetos de sincronización, arranca interrupciones de la app y el contador de ciclos. |
| `app_it.c` | Inicialización de interrupciones de la aplicación (esqueleto de sección crítica). |
| `task_a.c` | Tarea A — futura **productora**. Hoy: bucle periódico cada 250 ms. |
| `task_b.c` | Tarea B — futura **consumidora**. Hoy: bucle periódico cada 2500 ms. |
| `freertos.c` (app/src) | Hooks de la aplicación: idle, tick y stack overflow. |

---

## 2. `app.c` — Inicialización de la aplicación

`app_init()` es el punto de entrada de la capa de aplicación, invocado desde `main()` **antes** de arrancar el scheduler (`osKernelStart()`).

### 2.1 Variables globales de contadores

Se declaran y se inicializan a cero una serie de contadores de instrumentación:

```c
g_app_cnt                = 0;   // contador general de la aplicación
g_app_task_cnt           = 0;   // contador de activaciones de tareas
g_app_tick_cnt           = 0;   // ticks del kernel (lo actualiza vApplicationTickHook)
g_task_idle_cnt          = 0;   // iteraciones de la tarea idle
g_app_stack_overflow_cnt = 0;   // desbordamientos de stack detectados
g_tasks_cnt              = 0;   // contador de tareas
```

Cada valor inicial proviene de un `#define` (`G_APP_CNT_INI`, etc.), todos iguales a `0ul`.

### 2.2 Mensajes de inicio

Mediante la macro `LOGGER_INFO` se imprime por UART el nombre de la función, el *tick* actual (`xTaskGetTickCount()`) y tres cadenas descriptivas de la aplicación (`p_app`, `p_app_`, `p_app__`), que identifican el ejercicio como un sistema *Event-Triggered* con patrón productor-consumidor.

### 2.3 Objetos de sincronización (pendientes de implementar)

El código contiene **únicamente comentarios guía** en este punto. Las declaraciones reales están comentadas:

```c
/* Declare a variable of type QueueHandle_t ... */          // ← cola: NO creada
/* Declare a variable of type SemaphoreHandle_t ... mutex */ // ← semáforo/mutex: NO creado
```

Solo se declaran efectivamente los *handles* de las tareas:

```c
TaskHandle_t h_task_a;
TaskHandle_t h_task_b;
```

> En la solución completa, aquí deberían aparecer `xQueueCreate(...)`, `xSemaphoreCreateBinary()` / `xSemaphoreCreateCounting(...)` o `xSemaphoreCreateMutex()`, junto con la verificación de que la creación fue exitosa y su registro con `vQueueAddToRegistry()`.

### 2.4 Creación de las tareas

```c
xTaskCreate(task_a, "Task A", configMINIMAL_STACK_SIZE, NULL,
            tskIDLE_PRIORITY + 1, &h_task_a);
configASSERT(pdPASS == ret);

xTaskCreate(task_b, "Task B", configMINIMAL_STACK_SIZE, NULL,
            tskIDLE_PRIORITY + 1, &h_task_b);
configASSERT(pdPASS == ret);
```

Ambas tareas se crean con:

- **Prioridad 1** (`tskIDLE_PRIORITY + 1`), un nivel por encima de la tarea *idle*.
- **Stack** de `configMINIMAL_STACK_SIZE` palabras (el mínimo configurado).
- **Sin parámetros** (`NULL`).
- *Handle* guardado en `h_task_a` / `h_task_b`.

El valor de retorno se valida con `configASSERT(pdPASS == ret)`: si la creación falla (típicamente por falta de heap), el sistema queda detenido para depuración.

### 2.5 Cierre de la inicialización

```c
ret = xPortGetFreeHeapSize();  // heap libre restante (heap_4)
app_it_init();                 // inicializa interrupciones de la app
cycle_counter_init();          // habilita el contador de ciclos DWT
```

- `xPortGetFreeHeapSize()`: consulta cuánta memoria queda sin asignar en el *heap* de FreeRTOS (útil para verificar que la creación de tareas/objetos no agotó la memoria). El valor se guarda en `ret` pero no se usa más allá de la inspección en depuración.
- `app_it_init()`: ver §3.
- `cycle_counter_init()`: habilita el *Data Watchpoint and Trace* (DWT) del Cortex-M4 para medir tiempos de ejecución por conteo de ciclos.

---

## 3. `app_it.c` — Interrupciones de la aplicación

```c
void app_it_init(void)
{
    /* Protect shared resource */
    __asm("CPSID i");   // deshabilita interrupciones (PRIMASK = 1)
    __asm("CPSIE i");   // habilita interrupciones    (PRIMASK = 0)
}
```

Actualmente es un **esqueleto**. La secuencia `CPSID i` / `CPSIE i` delimita una **sección crítica vacía**:

- `CPSID i` pone a 1 el bit `PRIMASK`, enmascarando todas las interrupciones configurables.
- `CPSIE i` pone a 0 `PRIMASK`, rehabilitándolas.

Entre ambas instrucciones se debería inicializar de forma atómica cualquier recurso compartido entre el contexto de tarea y el contexto de ISR (por ejemplo, estructuras vinculadas a una interrupción externa). Como hoy no hay nada entre ellas, la función no produce ningún efecto observable: deshabilita y rehabilita las interrupciones de inmediato.

---

## 4. `task_a.c` — Tarea A (futura productora)

### 4.1 Constantes

```c
#define G_TASK_A_CNT_INI   0ul
#define TASK_A_DEL_ZERO    (pdMS_TO_TICKS(0ul))     // 0 ms
#define TASK_A_DEL_MAX     (pdMS_TO_TICKS(250ul))   // 250 ms
```

`pdMS_TO_TICKS()` convierte milisegundos a *ticks* del kernel según `configTICK_RATE_HZ`.

### 4.2 Cuerpo de la tarea

```c
void task_a(void *parameters)
{
    g_task_a_cnt = G_TASK_A_CNT_INI;                       // contador a 0
    LOGGER_INFO("  %s is running ...", pcTaskGetName(NULL)); // mensaje de inicio

    for (;;)
    {
        g_task_a_cnt++;                       // cuenta cada iteración
        LOGGER_INFO(p_task_a_wait_250mS);     // "==> Task A - Wait: 250mS"
        vTaskDelay(TASK_A_DEL_MAX);           // se bloquea 250 ms
    }
}
```

Comportamiento actual:

1. Inicializa su contador `g_task_a_cnt` a cero.
2. Imprime una vez su nombre y el *tick* de arranque.
3. En bucle infinito: incrementa el contador, imprime un mensaje y se **bloquea 250 ms** con `vTaskDelay()`.

Usa `vTaskDelay()` (relativo): el período efectivo es ligeramente mayor a 250 ms porque el retardo empieza **después** de imprimir. Como la impresión es muy rápida, la desviación es despreciable.

> **Rol previsto (productor):** en la versión completa, esta tarea debería **producir** un dato y depositarlo en la cola (`xQueueSend`) o señalizar al consumidor mediante un semáforo (`xSemaphoreGive`) cada 250 ms.

---

## 5. `task_b.c` — Tarea B (futura consumidora)

### 5.1 Constantes

```c
#define G_TASK_B_CNT_INI   0ul
#define TASK_B_DEL_ZERO    (pdMS_TO_TICKS(0ul))      // 0 ms
#define TASK_B_DEL_MAX     (pdMS_TO_TICKS(2500ul))   // 2500 ms
```

### 5.2 Cuerpo de la tarea

```c
void task_b(void *parameters)
{
    g_task_b_cnt = G_TASK_B_CNT_INI;
    LOGGER_INFO("  %s is running ...", pcTaskGetName(NULL));

    for (;;)
    {
        g_task_b_cnt++;
        LOGGER_INFO(p_task_b_wait_250mS);   // (la cadena dice "250mS" pero el delay es 2500 ms)
        vTaskDelay(TASK_B_DEL_MAX);         // se bloquea 2500 ms
    }
}
```

Estructura idéntica a la tarea A, pero con un período de **2500 ms** (10× más lento que A).

> **Observación:** la cadena de texto `p_task_b_wait_250mS` indica `"Wait: 250mS"`, mientras que el retardo real es `TASK_B_DEL_MAX = 2500 ms`. Es una inconsistencia heredada de la plantilla (la constante de texto se copió de la tarea A). No afecta la ejecución, solo el mensaje impreso.

> **Rol previsto (consumidor):** en la versión completa, esta tarea debería **consumir** los datos producidos por la tarea A, bloqueándose en la cola (`xQueueReceive`) o esperando el semáforo (`xSemaphoreTake`) en lugar de usar un `vTaskDelay` fijo.

### 5.3 Relación entre A y B en el patrón productor-consumidor

Hoy ambas tareas corren **desacopladas**: no comparten ningún recurso ni se sincronizan. El patrón productor-consumidor que se debe implementar requiere:

| Elemento | Productor (A) | Consumidor (B) |
|----------|---------------|----------------|
| Cola (buffer) | `xQueueSend()` | `xQueueReceive()` |
| Sincronización | `xSemaphoreGive()` | `xSemaphoreTake()` |
| Exclusión mutua | `xSemaphoreTake(mutex)` / `Give` | `xSemaphoreTake(mutex)` / `Give` |

La asimetría de períodos actual (A produce cada 250 ms, B consume cada 2500 ms) ilustra justamente el problema clásico: si A produce más rápido de lo que B consume, **una cola finita se llena** y debe gestionarse el bloqueo del productor (objetivo didáctico del TP).

---

## 6. `freertos.c` (app/src) — Hooks de la aplicación

Este archivo sobreescribe los *hooks* débiles (`__weak`) que provee CubeMX, dándoles comportamiento de instrumentación.

### 6.1 `vApplicationIdleHook()`

```c
void vApplicationIdleHook(void)
{
    g_task_idle_cnt++;
}
```

Se ejecuta repetidamente dentro de la **tarea idle** (prioridad 0), cuando ninguna tarea de mayor prioridad está lista. Requiere `configUSE_IDLE_HOOK == 1`.

- **Restricción clave:** no debe bloquear ni llamar a APIs que puedan bloquear (`vTaskDelay`, esperas en cola/semáforo).
- **Utilidad:** `g_task_idle_cnt` permite estimar la **carga de CPU**: cuanto más alto crece, más tiempo está ocioso el procesador. En esta aplicación, como A y B pasan la mayor parte del tiempo bloqueadas, la idle se ejecuta muchísimo y este contador domina.

### 6.2 `vApplicationTickHook()`

```c
void vApplicationTickHook(void)
{
    g_app_tick_cnt++;
}
```

Se ejecuta **desde la ISR del SysTick**, una vez por *tick* del kernel (cada 1 ms con `configTICK_RATE_HZ = 1000`). Requiere `configUSE_TICK_HOOK == 1`.

- **Restricción clave:** corre en contexto de interrupción → solo puede usar APIs terminadas en `FromISR`. Debe ser muy corto y consumir poco stack.
- **Utilidad:** `g_app_tick_cnt` es una medida de tiempo real transcurrido en *ticks*.

### 6.3 `vApplicationStackOverflowHook()`

```c
void vApplicationStackOverflowHook(xTaskHandle xTask, signed char *pcTaskName)
{
    taskENTER_CRITICAL();
    configASSERT(0);          // deshabilita interrupciones + bucle infinito
    taskEXIT_CRITICAL();
    g_app_stack_overflow_cnt++;
}
```

Lo llama el kernel cuando detecta un **desbordamiento de stack** (requiere `configCHECK_FOR_STACK_OVERFLOW ≥ 1`). Recibe el *handle* y el nombre de la tarea culpable.

- Entra en sección crítica y ejecuta `configASSERT(0)`, que detiene el sistema en un bucle infinito con interrupciones deshabilitadas, para poder inspeccionar el estado con un depurador (JTAG/SWD).
- El incremento de `g_app_stack_overflow_cnt` es **inalcanzable** tras el `configASSERT(0)`; queda como documentación de la intención.

---

## 7. Secuencia de ejecución y flujo temporal

### 7.1 Arranque

```
main()
  └─ app_init()                        [app.c]
       ├─ inicializa contadores
       ├─ imprime mensajes de inicio
       ├─ (cola / semáforo / mutex: PENDIENTES)
       ├─ xTaskCreate(task_a, prio 1)
       ├─ xTaskCreate(task_b, prio 1)
       ├─ xPortGetFreeHeapSize()
       ├─ app_it_init()                [app_it.c]  (sección crítica vacía)
       └─ cycle_counter_init()         (DWT)
  └─ osKernelStart()                   (arranca el scheduler; no retorna)
```

### 7.2 Línea de tiempo en régimen permanente (estado actual)

```
t=0 ms:    Task A corre → cuenta, imprime → vTaskDelay(250 ms)
t=0 ms:    Task B corre → cuenta, imprime → vTaskDelay(2500 ms)
t=0+ ms:   Idle corre   → g_task_idle_cnt++ (continuamente)
...
t=250 ms:  Task A despierta → cuenta, imprime → vTaskDelay(250 ms)
t=500 ms:  Task A despierta → ...
t=750 ms:  Task A despierta → ...
...
t=2500 ms: Task B despierta → cuenta, imprime → vTaskDelay(2500 ms)
           (mientras tanto Task A se activó ~10 veces)
```

- **Task A** se activa ~10 veces por cada activación de **Task B**.
- Entre activaciones, ambas están bloqueadas (estado *Blocked*), por lo que la **tarea idle** acumula la mayor parte del tiempo de CPU.
- Como ambas tareas tienen la misma prioridad y solo se ejecutan brevemente, no hay competencia real por la CPU en esta etapa.

### 7.3 Arquitectura en tiempo de ejecución

```
┌──────────────────────────────────────────────┐
│                Interrupciones                 │
│  SysTick (1 ms) → scheduler + tick hook       │
│                   (g_app_tick_cnt++)          │
└───────────────┬───────────────────────────────┘
                │ planifica
                ▼
┌──────────────────────────────────────────────┐
│               Tareas FreeRTOS                 │
│                                               │
│  ┌────────────┐        ┌────────────┐         │
│  │  task_a    │        │  task_b    │         │
│  │  prio = 1  │  (sin  │  prio = 1  │         │
│  │  250 ms    │  enlace│  2500 ms   │         │
│  │  vTaskDelay│   aún) │  vTaskDelay│         │
│  └────────────┘        └────────────┘         │
│       productor             consumidor        │
│       (futuro)              (futuro)          │
│                                               │
│  ┌────────────┐                               │
│  │ Idle Task  │  prio = 0                     │
│  │ idle_cnt++ │  (domina el tiempo de CPU)    │
│  └────────────┘                               │
└──────────────────────────────────────────────┘
```

---

## 8. Resumen

- **`app.c`**: crea las dos tareas (A y B) con prioridad 1 y stack mínimo, valida la creación con `configASSERT`, e inicializa contadores, interrupciones de la app y el contador de ciclos DWT. **Deja preparados (en comentarios) los objetos de sincronización del patrón productor-consumidor, que aún deben crearse.**
- **`app_it.c`**: esqueleto de inicialización de interrupciones; sección crítica vacía con `CPSID`/`CPSIE`.
- **`task_a.c`**: futura productora; hoy un bucle periódico cada 250 ms que imprime e incrementa un contador.
- **`task_b.c`**: futura consumidora; hoy un bucle periódico cada 2500 ms (con un texto de mensaje heredado que dice "250mS").
- **`freertos.c`**: instrumenta el sistema mediante los hooks de *idle* (carga de CPU), *tick* (tiempo real) y *stack overflow* (detención segura ante error).

El esqueleto está listo para completar el **problema productor-consumidor**: falta crear la cola y/o el semáforo en `app_init()` y reemplazar los `vTaskDelay` de las tareas por las operaciones de envío (`xQueueSend`/`xSemaphoreGive`) y recepción (`xQueueReceive`/`xSemaphoreTake`) que las sincronicen.

---

## 9. Implementación del patrón productor-consumidor

### 9.1 Cambios realizados sobre el scaffolding

**`app.h` — macro de tamaño de buffer y variables compartidas**

Se movió `G_BUFFER_SIZE` al header y se declararon las tres variables del buffer circular:

```c
#define G_BUFFER_SIZE  10ul

extern uint8_t  g_shared_buffer[G_BUFFER_SIZE];
extern uint32_t g_buffer_head;   // índice de escritura (productor)
extern uint32_t g_buffer_tail;   // índice de lectura  (consumidor)
```

**`app.c` — definición e inicialización**

```c
uint8_t  g_shared_buffer[G_BUFFER_SIZE] = {0};
uint32_t g_buffer_head = 0ul;
uint32_t g_buffer_tail = 0ul;
```

Se crearon los tres objetos de sincronización:

```c
h_spaces_counting_semaphore = xSemaphoreCreateCounting(G_BUFFER_SIZE, G_BUFFER_SIZE);
h_items_counting_semaphore  = xSemaphoreCreateCounting(G_BUFFER_SIZE, 0ul);
h_sync_mutex                = xSemaphoreCreateMutex();
```

| Handle | Tipo | Máximo | Valor inicial | Rol |
|--------|------|--------|---------------|-----|
| `h_spaces_counting_semaphore` | Semáforo contador | 10 | **10** | Espacios libres en el buffer |
| `h_items_counting_semaphore` | Semáforo contador | 10 | **0** | Ítems disponibles para consumir |
| `h_sync_mutex` | Mutex | — | disponible | Exclusión mutua sobre el buffer |

> **Por qué `items` es semáforo contador y no binario:** el productor (más rápido) acumula varios ítems en el buffer antes de que el consumidor procese el primero. Con semáforo binario, los `xSemaphoreGive` adicionales se descartan silenciosamente y el consumidor perdería ítems. El semáforo contador acumula hasta `G_BUFFER_SIZE` señales pendientes, una por ítem depositado.

**`task_a.c` — lógica de producción**

Dentro de la sección crítica se escribe `(uint8_t)g_task_a_cnt` en la posición `g_buffer_head` y se avanza el índice con aritmética modular:

```c
vTaskDelay(TASK_A_DEL_MAX);                               // 250 ms entre producciones

xSemaphoreTake(h_spaces_counting_semaphore, portMAX_DELAY); // bloquea si buffer lleno
xSemaphoreTake(h_sync_mutex, portMAX_DELAY);
{
    g_shared_buffer[g_buffer_head] = (uint8_t)g_task_a_cnt;
    LOGGER_INFO("Add element to shared buffer[%lu] = %d (cnt: %lu)",
                g_buffer_head, g_shared_buffer[g_buffer_head], g_task_a_cnt);
    g_buffer_head = (g_buffer_head + 1ul) % G_BUFFER_SIZE;
}
xSemaphoreGive(h_sync_mutex);
xSemaphoreGive(h_items_counting_semaphore);               // señaliza ítem disponible
```

El dato almacenado es `g_task_a_cnt` truncado a 8 bits: con `G_BUFFER_SIZE = 10` y el contador incrementando en 1 por iteración, los primeros 255 ciclos no pierden información; a partir del ítem 256 los valores vuelven a cero.

**`task_b.c` — lógica de consumo**

Dentro de la sección crítica se lee `g_shared_buffer[g_buffer_tail]` y se avanza el índice:

```c
xSemaphoreTake(h_items_counting_semaphore, portMAX_DELAY);  // bloquea si buffer vacío
xSemaphoreTake(h_sync_mutex, portMAX_DELAY);
{
    uint8_t value = g_shared_buffer[g_buffer_tail];
    LOGGER_INFO("Get element from shared buffer[%lu] = %d", g_buffer_tail, value);
    g_buffer_tail = (g_buffer_tail + 1ul) % G_BUFFER_SIZE;
}
xSemaphoreGive(h_sync_mutex);
xSemaphoreGive(h_spaces_counting_semaphore);                // libera un espacio
vTaskDelay(TASK_B_DEL_MAX);                                 // 2500 ms de procesamiento
```

### 9.2 Comportamiento observado

#### Logs de ejecución

```
[info] app_init is running - Tick [mS] = 0
[info]  app is a RTOS - Event-Triggered Systems (ETS)
[info]  app is a sotri-tp3_01-application: Producer-Consumer
[info]  app is a (Source => CESE - Sistemas Operativos de Tiempo Real)
[info]
[info]   Task B is running - Tick [mS] = 0
[info]
[info]   Task A is running - Tick [mS] = 0
[info]    ==> Task    A - Wait:   250mS
[info] Add element to shared buffer[0] = 1 (cnt: 1)
[info]    ==> Task    A - Wait:   250mS
[info] Get element from shared buffer[0] = 1
[info]    ==> Task    B - Wait:   2500mS
[info] Add element to shared buffer[1] = 2 (cnt: 2)
[info]    ==> Task    A - Wait:   250mS
[info] Add element to shared buffer[2] = 3 (cnt: 3)
[info]    ==> Task    A - Wait:   250mS
[info] Add element to shared buffer[3] = 4 (cnt: 4)
...
[info] Add element to shared buffer[9] = 10 (cnt: 10)
[info]    ==> Task    A - Wait:   250mS
[info] Get element from shared buffer[1] = 2
[info]    ==> Task    B - Wait:   2500mS
[info] Add element to shared buffer[0] = 11 (cnt: 11)
[info]    ==> Task    A - Wait:   250mS
...
```

#### Análisis de fases

**Fase 1 — arranque (t = 0):**
Task B arranca primero y bloquea de inmediato en `xSemaphoreTake(h_items_counting_semaphore)` porque `items = 0`. Task A arranca y entra a su loop.

**Fase 2 — primer ciclo (t = 250 ms):**
Task A produce `buffer[0] = 1` → `items = 1`. Task B despierta, consume `buffer[0] = 1`, libera espacio (`spaces = 10`) y se duerme 2500 ms.

**Fase 3 — llenado del buffer (t = 500 ms … 2750 ms):**
Task B está durmiendo. Task A produce un ítem cada 250 ms sin bloqueos: llena posiciones `[1]` a `[9]` con valores 2 a 10. Al intentar depositar el ítem 11 (`cnt = 11`) encuentra `spaces = 0` y **bloquea**.

**Fase 4 — régimen permanente (t ≥ 2750 ms):**
Cada vez que Task B despierta (cada 2500 ms), consume un ítem y libera un espacio. Task A se desbloquea, deposita su ítem pendiente, y vuelve a bloquear al intentar el siguiente (el buffer vuelve a llenarse de inmediato).

```
t=   0 ms: spaces=10, items=0. B bloquea.
t= 250 ms: A → buffer[0]=1,  spaces=9,  items=1.  B consume buffer[0]=1 → spaces=10, items=0. B duerme.
t= 500 ms: A → buffer[1]=2,  spaces=9,  items=1.
t= 750 ms: A → buffer[2]=3,  spaces=8,  items=2.
...
t=2500 ms: A → buffer[9]=10, spaces=0,  items=10. ← A BLOQUEA
t=2750 ms: B despierta → consume buffer[1]=2 → spaces=1, items=9.
           A desbloquea → buffer[0]=11, spaces=0, items=10. A vuelve a bloquear.
t=5250 ms: B despierta → consume buffer[2]=3 → spaces=1, items=9.
           A desbloquea → buffer[1]=12, spaces=0, items=10. (ciclo estable)
```

**El buffer opera como una cola FIFO circular.** `g_buffer_head` y `g_buffer_tail` avanzan por separado, cada uno módulo `G_BUFFER_SIZE`. El consumidor siempre lee el dato más antiguo (`tail`); el productor escribe en la próxima posición libre (`head`).

En régimen permanente:
- El buffer se mantiene **lleno** (`spaces == 0`).
- El productor queda **bloqueado** en `xSemaphoreTake(h_spaces_counting_semaphore)` hasta que el consumidor libera un espacio.
- El consumidor **nunca bloquea** en `items` una vez que el buffer está lleno.
- La cadencia efectiva del productor pasa de 250 ms a 2500 ms: la velocidad del sistema queda limitada por el eslabón más lento.
