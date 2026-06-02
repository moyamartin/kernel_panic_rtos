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

## 9. Implementación del problema Readers-Writers

**Patrón:** Readers-Writers sin inanición del escritor
*(Fuente: The Little Book of Semaphores, Allen B. Downey — §4.2)*

### 9.1 Descripción del problema

El problema **Readers-Writers** modela el acceso concurrente a un recurso compartido bajo dos reglas fundamentales:

- **Lectores concurrentes:** múltiples lectores pueden acceder al recurso simultáneamente.
- **Escritor exclusivo:** un escritor requiere acceso exclusivo — ningún lector ni otro escritor puede estar activo mientras escribe.

Una solución naive que siempre da prioridad a los lectores puede causar **inanición del escritor**: si los lectores llegan continuamente, el escritor nunca obtiene acceso. Esta implementación usa un **torniquete** para evitarlo.

### 9.2 Objetos de sincronización

Se crean tres primitivas en `app_init()`:

```c
h_room_empty_binary_semaphore = xSemaphoreCreateBinary();
xSemaphoreGive(h_room_empty_binary_semaphore);   // inicia en 1 (sala vacía)

h_critical_section_mutex = xSemaphoreCreateMutex();

h_turnstile_mutex = xSemaphoreCreateMutex();
```

| Handle | Tipo | Valor inicial | Rol |
|--------|------|---------------|-----|
| `h_room_empty_binary_semaphore` | Semáforo binario | 1 | Exclusión escritor/lectores. El primer lector lo toma; el último lo devuelve. El escritor lo toma para garantizar la sala vacía. |
| `h_critical_section_mutex` | Mutex | libre | Protege el contador `g_reader_cnt` (lectores activos). |
| `h_turnstile_mutex` | Mutex | libre | Torniquete anti-inanición. El escritor lo retiene mientras espera y escribe; los lectores deben pasar por él antes de entrar. |

### 9.3 Tareas

```c
xTaskCreate(task_a, "Task Writer",   configMINIMAL_STACK_SIZE, NULL,
            tskIDLE_PRIORITY + 1, &h_writer);

xTaskCreate(task_b, "Task Reader A", configMINIMAL_STACK_SIZE, NULL,
            tskIDLE_PRIORITY + 1, &h_reader_a);

xTaskCreate(task_b, "Task Reader B", configMINIMAL_STACK_SIZE, NULL,
            tskIDLE_PRIORITY + 1, &h_reader_b);
```

Las tres tareas tienen la misma prioridad (`tskIDLE_PRIORITY + 1`). Los dos lectores ejecutan la misma función `task_b()` y son diferenciados por el nombre asignado al crearlos.

### 9.4 Lógica del escritor (`task_a.c`)

```c
for (;;)
{
    xSemaphoreTake(h_turnstile_mutex, portMAX_DELAY);        // (1)
    xSemaphoreTake(h_room_empty_binary_semaphore, portMAX_DELAY);  // (2)

    LOGGER_INFO("Writing -- critical section for writers");   // sección crítica

    xSemaphoreGive(h_turnstile_mutex);                       // (3)
    xSemaphoreGive(h_room_empty_binary_semaphore);           // (4)

    LOGGER_INFO(p_task_a_wait_250mS);
    vTaskDelay(TASK_A_DEL_MAX);   // 250 ms — fuera de cualquier lock
}
```

1. Toma el torniquete: bloquea a los lectores que lleguen después.
2. Espera que la sala quede vacía (último lector debe salir antes de que el escritor entre).
3. Libera el torniquete: los lectores acumulados pueden empezar a pasar.
4. Marca la sala como vacía para la próxima iteración.

El `vTaskDelay(250 ms)` ocurre **fuera de todos los semáforos**, lo que permite que otros hilos (lectores o el scheduler) corran libremente durante la pausa del escritor.

### 9.5 Lógica del lector (`task_b.c`)

```c
for (;;)
{
    xSemaphoreTake(h_turnstile_mutex, portMAX_DELAY);  // (1) pasar torniquete
    xSemaphoreGive(h_turnstile_mutex);

    xSemaphoreTake(h_critical_section_mutex, portMAX_DELAY);
    g_reader_cnt++;
    if (g_reader_cnt == 1)
        xSemaphoreTake(h_room_empty_binary_semaphore, portMAX_DELAY); // (2) primer lector bloquea sala
    xSemaphoreGive(h_critical_section_mutex);

    LOGGER_INFO("%s Reading -- critical section for readers", pcTaskGetName(NULL)); // sección crítica

    xSemaphoreTake(h_critical_section_mutex, portMAX_DELAY);
    g_reader_cnt--;
    if (g_reader_cnt == 0)
        xSemaphoreGive(h_room_empty_binary_semaphore);  // (3) último lector libera sala
    xSemaphoreGive(h_critical_section_mutex);

    LOGGER_INFO("   ==> Task %s - Wait:   %lumS", pcTaskGetName(NULL), TASK_B_DEL_MAX);
    vTaskDelay(TASK_B_DEL_MAX);   // 2500 ms — fuera de cualquier lock
}
```

1. El torniquete serializa la entrada: si el escritor lo tiene tomado, el lector espera aquí. En cuanto lo obtiene, lo libera de inmediato para que otros lectores también puedan entrar.
2. Solo el **primer lector** toma `h_room_empty_binary_semaphore`, impidiendo que el escritor entre mientras haya lectores activos.
3. Solo el **último lector** devuelve `h_room_empty_binary_semaphore`, señalizando que la sala quedó vacía.

### 9.6 Mecanismo anti-inanición del escritor

Sin torniquete, si los lectores llegan continuamente, `g_reader_cnt` nunca baja a cero y el escritor nunca puede tomar `h_room_empty_binary_semaphore`. Con el torniquete:

1. El escritor toma `h_turnstile_mutex` → los lectores que lleguen después se bloquean en el paso (1).
2. El escritor espera que salgan los lectores que ya estaban dentro.
3. El escritor escribe con exclusividad.
4. El escritor libera el torniquete → los lectores acumulados entran, pero en la próxima vuelta el escritor volverá a tomar el torniquete.

Esto garantiza que el escritor obtenga acceso en un tiempo finito, acotado por el número de lectores que estaban en la sala cuando él llegó.

### 9.7 Comportamiento temporal

Con escritor en 250 ms y lectores en 2500 ms:

```
┌─ Ventana de lectura ─────────────────────────────────────────────┐
│  Task Reader B → lee (instantáneo) → vTaskDelay(2500 ms)         │
│  Task Reader A → lee (instantáneo) → vTaskDelay(2500 ms)         │
└──────────────────────────────────────────────────────────────────┘
┌─ Ventana de escritura (10 × 250 ms = 2500 ms) ───────────────────┐
│  Task Writer → escribe → vTaskDelay(250 ms)  ×10                 │
└──────────────────────────────────────────────────────────────────┘
                    ↑ se repite indefinidamente
```

- Ambos lectores leen de forma **concurrente** en cada ventana (ambos tienen `g_reader_cnt` activo al mismo tiempo).
- El escritor no puede entrar mientras algún lector está en la sala.
- Los lectores no pueden entrar mientras el escritor retiene el torniquete.

### 9.8 Traza del log

```
[info] Task Reader B Reading -- critical section for readers
[info]    ==> Task Task Reader B - Wait:   2500mS
[info] Task Reader A Reading -- critical section for readers
[info]    ==> Task Task Reader A - Wait:   2500mS
[info] Writing -- critical section for writers
[info]    ==> Task    A - Wait:   250mS
         ... (×10 en total) ...
[info] Writing -- critical section for writers
[info]    ==> Task    A - Wait:   250mS
[info] Task Reader B Reading -- critical section for readers
[info]    ==> Task Task Reader B - Wait:   2500mS
[info] Task Reader A Reading -- critical section for readers
[info]    ==> Task Task Reader A - Wait:   2500mS
```

La traza muestra:

- Ambos lectores (`Task Reader A` y `Task Reader B`) aparecen en cada ventana de lectura, confirmando la **lectura concurrente**.
- El escritor nunca aparece durante la lectura, verificando la **exclusión mutua escritor/lectores**.
- Exactamente **10 escrituras** entre cada par de lecturas, consistente con la relación de períodos 2500 ms / 250 ms = 10.

---
