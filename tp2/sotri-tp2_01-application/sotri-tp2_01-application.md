# Análisis del código fuente — sotri-tp2_01-application

**Microcontrolador:** STM32F446RETx (Cortex-M4, 84 MHz)  
**RTOS:** FreeRTOS v10.3.1 integrado con STM32CubeMX (CMSIS-RTOS v1)

> **Nota:** El enunciado menciona `startup_stm32f103rbtx.s` (STM32F103RBTx, Cortex-M3), pero el archivo presente en el proyecto es `startup_stm32f446retx.s`, correspondiente al STM32F446RETx (Cortex-M4). El análisis se realiza sobre los archivos reales del proyecto.

---

## 1. Descripción de cada archivo

### 1.1 `startup_stm32f446retx.s`

Es el punto de entrada real del programa. Define la tabla de vectores de interrupción (`g_pfnVectors`) y el `Reset_Handler`.

**Tabla de vectores de interrupción (`g_pfnVectors`)**

Se ubica en la sección `.isr_vector`, que el linker coloca en la dirección física `0x08000000` (inicio de la Flash). Las primeras dos entradas son obligatorias para todo Cortex-M:

| Posición | Contenido |
|----------|-----------|
| 0x00 | `_estack` — valor inicial del stack pointer (SP) |
| 0x04 | `Reset_Handler` — dirección del punto de entrada tras reset |
| 0x08..0x3C | Excepciones del núcleo Cortex-M4 (NMI, HardFault, etc.) |
| 0x40..último | IRQs de periféricos STM32F446 (WWDG, TIM1, TIM2, USART, etc.) |

Los manejadores no implementados se declaran como `weak` y apuntan al `Default_Handler`, que es un bucle infinito (`b Infinite_Loop`). Cualquier función con el mismo nombre en otro archivo C sobreescribe el alias débil.

**`Reset_Handler`**

Secuencia de ejecución tras un reset de hardware:

```
Reset_Handler:
  ldr   sp, =_estack        ; 1. Inicializa SP con el tope del stack
  bl    SystemInit           ; 2. Inicializa FPU y VTOR (no cambia el clock)
  ; Copia sección .data de Flash a SRAM
  ldr r0, =_sdata
  ldr r1, =_edata
  ldr r2, =_sidata
  [bucle CopyDataInit]       ; 3. Copia variables inicializadas
  ; Pone en cero la sección .bss
  ldr r2, =_sbss
  ldr r4, =_ebss
  [bucle FillZerobss]        ; 4. Inicializa variables no inicializadas en 0
  bl    __libc_init_array    ; 5. Constructores C++ / funciones _init
  bl    main                 ; 6. Llama a main()
  bx    lr                   ; (nunca debería llegar aquí)
```

### 1.2 `main.c`

Contiene el punto de entrada `main()` y la configuración de todos los periféricos.

**Secuencia de `main()`:**

```c
main()
  HAL_Init()                    // Inicializa HAL, TIM1 como timebase (1 ms)
  SystemClock_Config()          // Configura PLL → SYSCLK = 84 MHz
  MX_GPIO_Init()                // GPIO: LED (PA5), botón B1 (PC13, EXTI caída)
  MX_USART2_UART_Init()         // UART2: 115200 bps, 8N1
  MX_TIM2_Init()                // TIM2: temporizador de alta frecuencia
  HAL_TIM_Base_Start_IT(&htim2) // Arranca TIM2 con interrupción
  app_init()                    // Inicializa la aplicación FreeRTOS
  osKernelStart()               // Inicia el scheduler (no retorna)
  while (1) { }                 // ← nunca se alcanza
```

**`SystemClock_Config()`** — configuración del PLL a partir del oscilador interno HSI:

| Parámetro | Valor |
|-----------|-------|
| Fuente | HSI (16 MHz) |
| PLLM | 16 → VCO input = 1 MHz |
| PLLN | 336 → VCO output = 336 MHz |
| PLLP | 4 → SYSCLK = **84 MHz** |
| AHB prescaler | /1 → HCLK = **84 MHz** = `SystemCoreClock` |
| APB1 prescaler | /2 → PCLK1 = **42 MHz** |
| APB2 prescaler | /1 → PCLK2 = **84 MHz** |

**`MX_TIM2_Init()`** — temporizador de alta frecuencia para estadísticas de tiempo de ejecución de FreeRTOS:

| Parámetro | Valor | Cálculo |
|-----------|-------|---------|
| Clock fuente | APB1 × 2 | APB1 prescaler ≠ 1 → TIM2 clock = 42 MHz × 2 = **84 MHz** |
| Prescaler (PSC) | 2−1 = 1 | Clock contador = 84 MHz / 2 = **42 MHz** |
| Period (ARR) | 4200−1 = 4199 | Frecuencia de overflow = 42 MHz / 4200 = **10 kHz** |
| Periodo de interrupción | — | **100 µs** |

**`HAL_TIM_PeriodElapsedCallback()`** — callback llamado por `HAL_TIM_IRQHandler()` al completar el período de un timer:

- Si `htim->Instance == TIM1`: llama `HAL_IncTick()` → incrementa `uwTick` (base de tiempo de la HAL, 1 ms).
- Si `htim->Instance == TIM2`: incrementa `ulHighFrequencyTimerTicks` (contador de alta frecuencia para FreeRTOS run-time stats).

**`configureTimerForRunTimeStats()` / `getRunTimeCounterValue()`** — funciones requeridas por FreeRTOS cuando `configGENERATE_RUN_TIME_STATS == 1`:

- `configureTimerForRunTimeStats()`: reinicia `ulHighFrequencyTimerTicks = 0` al iniciar el scheduler.
- `getRunTimeCounterValue()`: retorna el valor actual de `ulHighFrequencyTimerTicks`.

### 1.3 `stm32f4xx_it.c`

Define los manejadores de interrupción del sistema (sobreescribe los aliases débiles del startup).

| Manejador | Función |
|-----------|---------|
| `NMI_Handler` | Bucle infinito (error no recuperable) |
| `HardFault_Handler` | Bucle infinito (error no recuperable) |
| `MemManage_Handler` | Bucle infinito |
| `BusFault_Handler` | Bucle infinito |
| `UsageFault_Handler` | Bucle infinito |
| `DebugMon_Handler` | No hace nada |
| `TIM1_UP_TIM10_IRQHandler` | Llama `HAL_TIM_IRQHandler(&htim1)` → `HAL_IncTick()` cada 1 ms |
| `TIM2_IRQHandler` | Llama `HAL_TIM_IRQHandler(&htim2)` → `ulHighFrequencyTimerTicks++` cada 100 µs |
| `EXTI15_10_IRQHandler` | Llama `HAL_GPIO_EXTI_IRQHandler(B1_Pin)` → maneja botón B1 (PC13) |

**Nota sobre SVC_Handler y PendSV_Handler:** estos manejadores no aparecen en `stm32f4xx_it.c` porque son redefinidos mediante macros en `FreeRTOSConfig.h` directamente hacia las implementaciones del kernel FreeRTOS.

### 1.4 `FreeRTOSConfig.h`

Archivo de configuración que adapta el kernel FreeRTOS al hardware y a la aplicación.

**Parámetros clave:**

| Define | Valor | Significado |
|--------|-------|-------------|
| `configUSE_PREEMPTION` | 1 | Scheduler apropiativo habilitado |
| `configCPU_CLOCK_HZ` | `SystemCoreClock` (84 MHz) | Frecuencia del núcleo para SysTick |
| `configTICK_RATE_HZ` | 1000 | Tick de FreeRTOS = 1 ms |
| `configMAX_PRIORITIES` | 7 | Máximo 7 niveles de prioridad |
| `configMINIMAL_STACK_SIZE` | 128 palabras | Stack mínimo para la tarea idle |
| `configTOTAL_HEAP_SIZE` | 15360 bytes | Heap total para FreeRTOS |
| `configGENERATE_RUN_TIME_STATS` | 1 | Habilita estadísticas de tiempo de ejecución por tarea |
| `configUSE_IDLE_HOOK` | 1 | Activa `vApplicationIdleHook()` |
| `configUSE_TICK_HOOK` | 1 | Activa `vApplicationTickHook()` |
| `configCHECK_FOR_STACK_OVERFLOW` | 1 | Detección de desbordamiento de stack |
| `configKERNEL_INTERRUPT_PRIORITY` | 15 << 4 = 0xF0 | Prioridad de SysTick y PendSV (la más baja) |
| `configMAX_SYSCALL_INTERRUPT_PRIORITY` | 5 << 4 = 0x50 | Prioridad máxima para ISRs que llaman API FreeRTOS |

**Mapeo de manejadores del kernel al NVIC de Cortex-M4:**

```c
#define vPortSVCHandler    SVC_Handler      // llamada de servicio (cambio de contexto inicial)
#define xPortPendSVHandler PendSV_Handler   // cambio de contexto apropiativo
#define xPortSysTickHandler SysTick_Handler // tick del kernel (1 ms)
```

Estos `#define` hacen que las funciones C del kernel FreeRTOS (`vPortSVCHandler`, etc.) sean compiladas con los nombres de los manejadores del vector de interrupciones del startup.

**Macros para run-time stats:**

```c
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS configureTimerForRunTimeStats
#define portGET_RUN_TIME_COUNTER_VALUE         getRunTimeCounterValue
```

Conectan el timer de alta frecuencia (TIM2) con el sistema de estadísticas de FreeRTOS.

### 1.5 `freertos.c` (dos archivos)

El proyecto tiene dos versiones:

#### `Core/Src/freertos.c` (generado por CubeMX)

Provee implementaciones débiles (`__weak`) de los hooks de FreeRTOS. Estas son las versiones por defecto que pueden ser sobreescritas:

- `configureTimerForRunTimeStats()` — implementación débil vacía (sobreescrita en `main.c`)
- `getRunTimeCounterValue()` — implementación débil que retorna 0 (sobreescrita en `main.c`)
- `vApplicationIdleHook()` — implementación débil vacía (sobreescrita en `app/src/freertos.c`)
- `vApplicationTickHook()` — implementación débil vacía (sobreescrita en `app/src/freertos.c`)
- `vApplicationStackOverflowHook()` — implementación débil vacía (sobreescrita en `app/src/freertos.c`)
- `vApplicationGetIdleTaskMemory()` — provee memoria estática para la tarea idle (`xIdleTaskTCBBuffer`, `xIdleStack[128]`)

#### `app/src/freertos.c` (implementación de la aplicación)

Sobreescribe los hooks con comportamiento real:

- **`vApplicationIdleHook()`**: se ejecuta en cada iteración de la tarea idle (cuando no hay otra tarea lista). Incrementa `g_task_idle_cnt` — permite medir cuánto tiempo el sistema está desocupado.

- **`vApplicationTickHook()`**: se ejecuta en cada tick del kernel (desde la ISR de SysTick). Incrementa `g_app_tick_cnt` — contador de ticks de la aplicación.

- **`vApplicationStackOverflowHook()`**: llamado si se detecta desbordamiento de stack. Llama a `configASSERT(0)` que deshabilita interrupciones y queda en un bucle infinito para depuración.

---

## 2. Evolución de `SystemCoreClock` y `SysTick`

### 2.1 `SystemCoreClock`

Esta variable global es declarada e inicializada en `system_stm32f4xx.c`:

```c
uint32_t SystemCoreClock = 16000000;
```

| Momento | Valor | Razón |
|---------|-------|-------|
| Antes del `Reset_Handler` (variables en Flash, pre-.data) | 16,000,000 Hz | Valor inicial en `.data` |
| Después de `SystemInit()` | **16,000,000 Hz** | `SystemInit()` solo habilita FPU y configura VTOR; no modifica el clock |
| Después de copiar `.data` y limpiar `.bss` | 16,000,000 Hz | Valor copiado desde Flash a SRAM |
| Al entrar a `main()`, antes de `SystemClock_Config()` | **16,000,000 Hz** | HSI sin PLL |
| Después de `HAL_Init()` | 16,000,000 Hz | `HAL_InitTick()` usa el valor actual para configurar TIM1 |
| Después de `SystemClock_Config()` | **84,000,000 Hz** | `HAL_RCC_ClockConfig()` actualiza `SystemCoreClock` y llama `HAL_InitTick()` nuevamente |
| Durante toda la ejecución del scheduler | **84,000,000 Hz** | No se modifica el clock en tiempo de ejecución |

### 2.2 `SysTick`

| Momento | Estado del SysTick |
|---------|-------------------|
| Tras reset (antes de `SystemInit`) | No configurado; registros en valores de reset |
| Después de `SystemInit()` | No configurado (no toca SysTick) |
| Después de `HAL_Init()` | **No configurado por HAL** — el timebase es TIM1, no SysTick. `HAL_InitTick()` configura TIM1, no SysTick |
| Después de `SystemClock_Config()` | No configurado por HAL (solo TIM1 es reconfigurado) |
| Antes de `osKernelStart()` | No activo |
| Dentro de `osKernelStart()` → `vTaskStartScheduler()` → `xPortStartScheduler()` | **Configurado por FreeRTOS**: LOAD = (`SystemCoreClock` / `configTICK_RATE_HZ`) − 1 = (84,000,000 / 1,000) − 1 = **83,999** |
| Durante la ejecución del scheduler | Genera interrupción cada **1 ms**, ejecuta `SysTick_Handler` = `xPortSysTickHandler` |

El `SysTick_Handler` de FreeRTOS (`xPortSysTickHandler`) realiza las siguientes acciones en cada tick:
1. Incrementa el contador de ticks interno del kernel.
2. Desbloquea tareas que cumplieron su tiempo de espera (`vTaskDelay`, `xQueueReceive` con timeout, etc.).
3. Si hay tarea de mayor prioridad lista, pone en pendiente `PendSV` para realizar el cambio de contexto.
4. Llama a `vApplicationTickHook()` (que incrementa `g_app_tick_cnt`).

---

## 3. Comportamiento del programa desde `Reset_Handler` hasta `while(1)`

### Paso 1: `Reset_Handler` (startup_stm32f446retx.s)

1. El hardware carga SP desde la posición 0 de la tabla de vectores (`_estack` ≈ 0x20020000, fin de la SRAM de 128 KB).
2. El hardware salta a `Reset_Handler` (posición 1 de la tabla de vectores).
3. `Reset_Handler` fija SP = `_estack` (refuerzo por software).
4. Llama a `SystemInit()`: habilita FPU (CP10/CP11), opcionalmente configura VTOR. `SystemCoreClock` permanece en 16 MHz.
5. Copia la sección `.data` (variables globales inicializadas) desde la Flash al inicio de la SRAM.
6. Pone en cero la sección `.bss` (variables globales sin inicializar, incluyendo `g_app_cnt`, `g_app_tick_cnt`, etc.).
7. Llama a `__libc_init_array()` (constructores estáticos de C++, si los hubiera).
8. Llama a `main()`.

### Paso 2: `main()` → `HAL_Init()`

- Llama `HAL_MspInit()`: sin operación en este proyecto.
- Llama `HAL_InitTick(HAL_TICK_FREQ_DEFAULT)` → implementado en `stm32f4xx_hal_timebase_tim.c`:
  - Habilita el clock de TIM1 (`__HAL_RCC_TIM1_CLK_ENABLE()`).
  - `uwTimclock = PCLK2 = 16 MHz` (clock aún sin PLL).
  - `PSC = (16,000,000 / 1,000,000) − 1 = 15`.
  - `ARR = (1,000,000 / 1,000) − 1 = 999`.
  - Configura e inicia TIM1 con interrupciones: genera interrupción cada 1 ms (16 MHz / 16 / 1000 = 1 kHz).
  - Habilita `TIM1_UP_TIM10_IRQn` en el NVIC.
- **SysTick: aún no configurado.**
- Desde este momento, `uwTick` se incrementa cada 1 ms vía `TIM1`.

### Paso 3: `main()` → `SystemClock_Config()`

- Configura el oscilador HSI y el PLL:
  - `HAL_RCC_OscConfig()`: enciende HSI, configura PLL (PLLM=16, PLLN=336, PLLP=4). VCO output = 336 MHz, SYSCLK = 84 MHz.
- Configura los buses:
  - `HAL_RCC_ClockConfig()`: AHB/1, APB1/2, APB2/1. FLASH_LATENCY_2.
  - Internamente llama `SystemCoreClockUpdate()` → `SystemCoreClock = 84,000,000`.
  - Llama `HAL_InitTick()` nuevamente con el nuevo clock:
    - `uwTimclock = PCLK2 = 84 MHz`.
    - `PSC = (84,000,000 / 1,000,000) − 1 = 83`.
    - `ARR = 999`.
    - TIM1 reconfigura la interrupción: sigue siendo cada 1 ms (84 MHz / 84 / 1000 = 1 kHz).

### Paso 4: `main()` → `MX_GPIO_Init()`

- Habilita clocks de GPIOC, GPIOH, GPIOA, GPIOB.
- Configura PC13 (botón B1) como entrada con EXTI por flanco de bajada.
- Configura PA5 (LED LD2) como salida push-pull.
- Habilita `EXTI15_10_IRQn` con prioridad 5 en el NVIC.

### Paso 5: `main()` → `MX_USART2_UART_Init()`

- Configura USART2: 115200 bps, 8N1, sin control de flujo.

### Paso 6: `main()` → `MX_TIM2_Init()`

- Configura TIM2 (clock = 84 MHz):
  - PSC = 1, ARR = 4199 → desbordamiento cada 100 µs (10 kHz).
- **No inicia TIM2 aún** (solo configura los registros).

### Paso 7: `main()` → `HAL_TIM_Base_Start_IT(&htim2)`

- Habilita el contador de TIM2 y su interrupción de desbordamiento.
- Habilita `TIM2_IRQn` en el NVIC.
- A partir de aquí, `TIM2_IRQHandler` → `HAL_TIM_IRQHandler()` → `HAL_TIM_PeriodElapsedCallback()` → `ulHighFrequencyTimerTicks++` cada 100 µs.

### Paso 8: `main()` → `app_init()`

- Inicializa contadores de la aplicación a cero.
- Imprime mensajes de inicio por UART (vía `LOGGER_INFO`).
- Crea la cola `h_btn_led_q` (capacidad 5 elementos de tipo `task_led_ev_t`).
- Crea el semáforo binario `h_btn_led_bin_sem`.
- Crea la tarea `task_btn` (prioridad `tskIDLE_PRIORITY + 1`, stack 256 palabras).
- Crea la tarea `task_led` (prioridad `tskIDLE_PRIORITY + 1`, stack 256 palabras).
- Llama a `app_it_init()`: deshabilita y vuelve a habilitar interrupciones globales (esqueleto de inicialización de interrupciones de la aplicación).
- Llama a `cycle_counter_init()`: inicializa el contador de ciclos DWT para medición de tiempo.

### Paso 9: `main()` → `osKernelStart()`

- Llama a `vTaskStartScheduler()`:
  1. Crea la tarea idle (usa memoria estática provista por `vApplicationGetIdleTaskMemory()`).
  2. Llama a `portCONFIGURE_TIMER_FOR_RUN_TIME_STATS` = `configureTimerForRunTimeStats()`: reinicia `ulHighFrequencyTimerTicks = 0`.
  3. Configura SysTick: LOAD = 83,999, habilita interrupción SysTick con prioridad `configKERNEL_INTERRUPT_PRIORITY` (0xF0 = prioridad 15, la más baja).
  4. Configura PendSV con prioridad 0xF0 (la más baja).
  5. Llama `SVC` (instrucción de software) para arrancar la primera tarea mediante `vPortSVCHandler` (`SVC_Handler`).
  6. **No retorna jamás.**

### Resultado: el `while(1)` en `main()` nunca se ejecuta

El scheduler toma el control. Las tareas activas son:
- **Task BTN** (prioridad 1): maneja eventos del botón.
- **Task LED** (prioridad 1): controla el LED.
- **Idle Task** (prioridad 0): ejecuta `vApplicationIdleHook()` cuando no hay tareas listas.

---

## 4. Interacción de SysTick y TIM1 con FreeRTOS

### 4.1 SysTick y FreeRTOS

El SysTick es el **tick del kernel FreeRTOS**. La conexión se establece mediante la macro en `FreeRTOSConfig.h`:

```c
#define xPortSysTickHandler SysTick_Handler
```

Esto hace que la función `xPortSysTickHandler` del port de FreeRTOS sea compilada con el nombre `SysTick_Handler`, que es exactamente el símbolo en la tabla de vectores del startup. No hay código de glue adicional: el NVIC llama directamente a la implementación del kernel.

**¿Cómo interactúa SysTick con el scheduler?**

Cada 1 ms (período = 83,999 conteos a 84 MHz), el SysTick genera una interrupción con la **prioridad más baja del sistema** (`configKERNEL_INTERRUPT_PRIORITY = 0xF0`). La ISR de FreeRTOS (`xPortSysTickHandler`):

1. **Incrementa el tick count** del kernel (`xTickCount++`).
2. **Desbloquea tareas** que estaban esperando con timeout: cualquier tarea cuyo delay expiró pasa al estado "Ready".
3. **Evalúa si hay preempción**: si existe una tarea de mayor prioridad lista para ejecutar, activa la excepción **PendSV** (poniendo en 1 el bit PENDSVSET del registro ICSR).
4. **Llama a `vApplicationTickHook()`** (si `configUSE_TICK_HOOK == 1`): en esta aplicación, incrementa `g_app_tick_cnt`.

**¿Por qué SysTick tiene la prioridad más baja?**

Porque FreeRTOS requiere que SysTick y PendSV tengan la prioridad NVIC más baja. Esto garantiza que cualquier ISR de periférico (TIM1, TIM2, EXTI, etc.) pueda preemptar al kernel, pero el kernel solo se ejecuta cuando no hay ISR de mayor prioridad activa. La regla es: **las ISRs que llaman funciones FreeRTOS terminadas en `FromISR` deben tener prioridad ≥ `configMAX_SYSCALL_INTERRUPT_PRIORITY` (0x50 = prioridad 5)**.

### 4.2 TIM1 y FreeRTOS

TIM1 es la **base de tiempo de la HAL** (alternativa al SysTick). Su rol es proveer `uwTick`, el contador de milisegundos que usa la HAL para `HAL_Delay()`, `HAL_GetTick()`, y los timeouts internos de los periféricos.

La cadena de llamadas es:

```
TIM1 overflow (cada 1 ms)
  → TIM1_UP_TIM10_IRQHandler()      [stm32f4xx_it.c]
    → HAL_TIM_IRQHandler(&htim1)    [HAL]
      → HAL_TIM_PeriodElapsedCallback(&htim1)  [main.c]
        → HAL_IncTick()             [HAL] → uwTick++
```

**¿Por qué TIM1 en lugar de SysTick para la HAL?**

Cuando FreeRTOS está presente, el SysTick es propiedad del kernel. Si también la HAL usara SysTick (comportamiento por defecto sin RTOS), habría conflicto: dos entidades intentando configurar y usar el mismo temporizador. La solución de STM32CubeMX es redirigir el timebase de la HAL a TIM1, dejando SysTick exclusivamente para FreeRTOS.

**Prioridad de TIM1:** se configura con `TickPriority` pasado desde `HAL_Init()`. Debe ser mayor o igual a `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` (prioridad ≤ 5 en valor lógico) para poder llamar funciones HAL que internamente podrían necesitar el tick. En práctica, la prioridad de TIM1 es la más baja que HAL asigna por defecto, y es compatible con las restricciones de FreeRTOS.

**Interacción indirecta con FreeRTOS:** cuando una tarea FreeRTOS llama `vTaskDelay(1)`, se bloquea hasta que el SysTick genere un tick. Internamente, el HAL usa `HAL_Delay()` que espera sobre `uwTick` (incrementado por TIM1). Estas dos fuentes de tiempo son **independientes pero síncronas** (ambas a 1 ms), lo que asegura coherencia temporal.

---

## 5. Interacción de TIM2 con la HAL del proyecto

TIM2 es el **temporizador de alta frecuencia para las estadísticas de tiempo de ejecución de FreeRTOS** (`configGENERATE_RUN_TIME_STATS`).

### 5.1 Configuración

TIM2 se inicializa en `MX_TIM2_Init()` (generado por CubeMX) e iniciado en `main()`:

```c
HAL_TIM_Base_Start_IT(&htim2);
```

`HAL_TIM_Base_Start_IT()` usa la HAL para:
1. Llamar a `HAL_TIM_Base_MspInit()` (configura el clock del periférico y el NVIC via MSP).
2. Cargar los registros de TIM2 (PSC, ARR, CR1).
3. Habilitar la interrupción de actualización (bit UIE en DIER).
4. Arrancar el contador (bit CEN en CR1).

### 5.2 Cadena de interrupción

Cada 100 µs TIM2 desborda y genera una interrupción:

```
TIM2 overflow (cada 100 µs)
  → TIM2_IRQHandler()                   [stm32f4xx_it.c]
    → HAL_TIM_IRQHandler(&htim2)        [HAL]
      → HAL_TIM_PeriodElapsedCallback() [main.c]
        → ulHighFrequencyTimerTicks++
```

La HAL (`HAL_TIM_IRQHandler`) verifica el flag de interrupción (UIF), lo limpia y llama al callback. Toda la gestión del flag se realiza dentro de la HAL; el código de la aplicación solo ve el callback.

### 5.3 Uso por FreeRTOS (run-time stats)

FreeRTOS, cuando `configGENERATE_RUN_TIME_STATS == 1`, mide el tiempo de CPU consumido por cada tarea. Para ello usa una fuente de tiempo más rápida que su propio tick (para tener resolución sub-milisegundo):

```c
// FreeRTOSConfig.h
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS configureTimerForRunTimeStats
#define portGET_RUN_TIME_COUNTER_VALUE         getRunTimeCounterValue
```

- **`portCONFIGURE_TIMER_FOR_RUN_TIME_STATS`**: llamado por `vTaskStartScheduler()`. Reinicia `ulHighFrequencyTimerTicks = 0` para que las mediciones partan desde cero cuando el scheduler inicia.

- **`portGET_RUN_TIME_COUNTER_VALUE`**: llamado por el kernel cada vez que necesita una marca de tiempo (en cada cambio de contexto y en cada tick). Retorna el valor actual de `ulHighFrequencyTimerTicks`.

Con una resolución de 100 µs (10 kHz) y un tick de 1 ms (1 kHz), el contador de TIM2 es **10 veces más rápido** que el tick del kernel, lo que proporciona resolución suficiente para las estadísticas.

### 5.4 Resumen de roles de los temporizadores

| Temporizador | Frecuencia | Propósito | Interacción |
|-------------|-----------|-----------|-------------|
| **SysTick** | 1 kHz (1 ms) | Tick del kernel FreeRTOS | Propiedad exclusiva del kernel; dirige el scheduler, desbloquea tareas, llama tick hook |
| **TIM1** | 1 kHz (1 ms) | Timebase de la HAL (`uwTick`) | Incrementa `uwTick` para `HAL_Delay()` y timeouts de periféricos; libera SysTick para FreeRTOS |
| **TIM2** | 10 kHz (100 µs) | Contador de alta frecuencia | Provee `ulHighFrequencyTimerTicks` a FreeRTOS para `vTaskGetRunTimeStats()` |

---

## 6. Análisis de la capa de aplicación

### 6.1 `app.c` — Inicialización de la aplicación

`app_init()` es el punto de entrada de la aplicación, llamado desde `main()` antes de `osKernelStart()`. Su secuencia es:

**a) Inicialización de contadores globales**

```c
g_app_cnt             = 0;   // contador general
g_app_task_cnt        = 0;   // contador de activaciones de tareas
g_app_tick_cnt        = 0;   // tick count de la aplicación (actualizado por vApplicationTickHook)
g_task_idle_cnt       = 0;   // iteraciones de la tarea idle
g_app_stack_overflow_cnt = 0; // desbordamientos de stack detectados
```

**b) Creación de la cola de comunicación**

```c
h_btn_led_q = xQueueCreate(5, sizeof(task_led_ev_t));
```

Cola de capacidad 5 elementos del tipo `task_led_ev_t`. Se registra en el registro de queues de FreeRTOS (`vQueueAddToRegistry`) para facilitar la inspección con RTOS-aware debuggers.

**c) Creación del semáforo binario**

```c
h_btn_led_bin_sem = xSemaphoreCreateBinary();
```

Se crea en estado "vacío" (no disponible). Se añade al registro con el nombre `"BTN to LED Binary Semaphore Handle"`.

> **Nota:** La cola `h_btn_led_q` y el semáforo `h_btn_led_bin_sem` son creados pero **no utilizados** en la implementación actual de las tareas. La comunicación real entre `task_btn` y `task_led` se realiza a través del mecanismo de interfaz de memoria compartida descrito en §6.5. Ambos objetos representan un andamiaje (*scaffolding*) dispuesto para que en ejercicios subsiguientes se implemente comunicación basada en cola o semáforo.

**d) Creación de las tareas**

```c
xTaskCreate(task_btn, "Task BTN", 2*configMINIMAL_STACK_SIZE, NULL,
            tskIDLE_PRIORITY + 1, &h_task_btn);

xTaskCreate(task_led, "Task LED", 2*configMINIMAL_STACK_SIZE, NULL,
            tskIDLE_PRIORITY + 1, &h_task_led);
```

Ambas tareas se crean con:
- Prioridad 1 (`tskIDLE_PRIORITY + 1`), un nivel por encima de la tarea idle.
- Stack de 256 palabras (2 × 128 = 256 words = 1024 bytes).
- Sin parámetros (`NULL`).
- Handles almacenados para referencia futura.

**e) Inicializaciones finales**

- `app_it_init()`: inicializa las interrupciones de la aplicación (ver §6.2).
- `cycle_counter_init()`: habilita el contador de ciclos DWT del Cortex-M4 para medición de tiempo de ejecución de código.

---

### 6.2 `app_it.c` — Interrupciones de la aplicación

**`app_it_init()`**

```c
__asm("CPSID i");   // deshabilita interrupciones (PRIMASK = 1)
__asm("CPSIE i");   // habilita interrupciones  (PRIMASK = 0)
```

Actualmente es un esqueleto: deshabilita y re-habilita interrupciones de inmediato. La secuencia CPSID/CPSIE delimita una sección crítica vacía pensada para inicializar recursos compartidos entre el contexto de tarea y el contexto de ISR.

**`HAL_GPIO_EXTI_Callback()`**

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == BTN_A_PIN)
    {
        /* Work to be done. */
    }
}
```

Callback débilmente ligado que sobreescribe el `__weak` de la HAL. Es invocado desde `EXTI15_10_IRQHandler` → `HAL_GPIO_EXTI_IRQHandler()` cuando el botón B1 (PC13) genera un flanco. El cuerpo está vacío en esta implementación; la lógica de botón se realiza íntegramente por *polling* en `task_btn`. El callback es el punto de extensión para una futura implementación basada en interrupciones (p. ej., dar un semáforo desde ISR con `xSemaphoreGiveFromISR()`).

---

### 6.3 `task_btn.c` — Tarea de lectura y debounce del botón

#### Tipos de datos (`task_btn_attribute.h`)

```c
typedef enum { EV_BTN_UP, EV_BTN_DOWN }          task_btn_ev_t;
typedef enum { ST_BTN_UP, ST_BTN_FALLING,
               ST_BTN_DOWN, ST_BTN_RISING }       task_btn_st_t;

typedef struct {
    task_btn_ev_t  event;      // evento actual leído del GPIO
    task_btn_st_t  state;      // estado actual de la máquina
    TickType_t     tick;       // marca de tiempo para debounce
    GPIO_TypeDef  *gpio_port;  // puerto GPIO del botón
    uint16_t       pin;        // pin GPIO del botón
} task_btn_dta_t;
```

Estado inicial:

```c
task_btn_dta_t task_btn_dta = {
    EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
    B1_GPIO_Port, B1_Pin
};
```

#### Cuerpo de la tarea `task_btn()`

```
Inicializa g_task_btn_cnt = 0
Imprime mensaje de inicio (tick actual)
loop infinito:
  g_task_btn_cnt++
  task_btn_statechart()        ← ejecuta la máquina de estados
  vTaskDelay(50 ticks = 50 ms) ← duerme 50 ms al final del cuerpo
```

Usa `vTaskDelay` (no `vTaskDelayUntil`): el período efectivo es ligeramente mayor que 50 ms porque el retardo empieza **después** de que la función retorna. Dado que `task_btn_statechart()` es muy rápida, la variación es despreciable.

#### Máquina de estados `task_btn_statechart()`

Antes de evaluar los estados, la función lee el GPIO:

```c
if (BTN_PRESSED == HAL_GPIO_ReadPin(port, pin))
    task_btn_dta.event = EV_BTN_DOWN;
else
    task_btn_dta.event = EV_BTN_UP;
```

`BTN_PRESSED = GPIO_PIN_RESET` (botón activo bajo en la Nucleo F446RE).

**Diagrama de estados:**

```
           BTN_DOWN detectado
┌─────────┐  ──────────────────►  ┌──────────────┐
│ ST_BTN  │    guarda tick         │  ST_BTN      │
│   UP    │                        │  FALLING     │
└─────────┘  ◄──────────────────  └──────────────┘
           BTN_UP (rebote)              │
                                        │ elapsed≥50ms
                                        │ && BTN_DOWN
                                        ▼
                              put_event_task_led(EV_LED_BLINK)
           BTN_UP detectado             │
┌─────────┐  ◄──────────────────  ┌──────────────┐
│ ST_BTN  │    guarda tick         │  ST_BTN      │
│  DOWN   │                        │  RISING      │
└─────────┘  ──────────────────►  └──────────────┘
           BTN_DOWN (rebote)            │
                                        │ elapsed≥50ms
                                        │ && BTN_UP
                                        ▼
                              put_event_task_led(EV_LED_OFF)
```

**Lógica de debounce (50 ms de ventana):**

| Estado | Condición de transición | Acción | Próximo estado |
|--------|------------------------|--------|----------------|
| `ST_BTN_UP` | `EV_BTN_DOWN` | Guarda tick | `ST_BTN_FALLING` |
| `ST_BTN_FALLING` | elapsed ≥ 50 ms **y** `EV_BTN_DOWN` | `put_event_task_led(EV_LED_BLINK)` | `ST_BTN_DOWN` |
| `ST_BTN_FALLING` | elapsed ≥ 50 ms **y** `EV_BTN_UP` | (rebote, ignora) | `ST_BTN_UP` |
| `ST_BTN_DOWN` | `EV_BTN_UP` | Guarda tick | `ST_BTN_RISING` |
| `ST_BTN_RISING` | elapsed ≥ 50 ms **y** `EV_BTN_UP` | `put_event_task_led(EV_LED_OFF)` | `ST_BTN_UP` |
| `ST_BTN_RISING` | elapsed ≥ 50 ms **y** `EV_BTN_DOWN` | (rebote, ignora) | `ST_BTN_DOWN` |
| `default` | cualquier estado inválido | reinicia máquina | `ST_BTN_UP` |

**Mecanismo de debounce:**
Como la tarea se ejecuta cada 50 ms y la ventana de debounce es también 50 ms (`DEL_BTN_MAX = 50`), la verificación de estabilidad ocurre **exactamente en el siguiente ciclo** de la tarea tras detectar el cambio. Si la señal sigue en el mismo nivel → evento confirmado. Si volvió al nivel anterior → fue un rebote y se descarta.

**Salidas:** al confirmar pulsación envía `EV_LED_BLINK`; al confirmar liberación envía `EV_LED_OFF`. Ambos se envían mediante `put_event_task_led()` (ver §6.5).

---

### 6.4 `task_led.c` — Tarea de control del LED

#### Tipos de datos (`task_led_attribute.h`)

```c
typedef enum { EV_LED_OFF, EV_LED_BLINK }   task_led_ev_t;
typedef enum { ST_LED_OFF, ST_LED_BLINK }    task_led_st_t;

typedef struct {
    bool           flag;       // indica que hay un evento nuevo pendiente
    task_led_ev_t  event;      // último evento recibido
    task_led_st_t  state;      // estado actual de la máquina
    TickType_t     tick;       // marca de tiempo para el toggle del LED
    GPIO_TypeDef  *gpio_port;  // puerto GPIO del LED
    uint16_t       pin;        // pin GPIO del LED
} task_led_dta_t;
```

Estado inicial:

```c
task_led_dta_t task_led_dta = {
    false, EV_LED_OFF, ST_LED_OFF, DEL_LED_MIN,
    LD2_GPIO_Port, LD2_Pin
};
```

#### Cuerpo de la tarea `task_led()`

```
Inicializa g_task_led_cnt = 0
last_wake_time = xTaskGetTickCount()
Imprime mensaje de inicio
HAL_GPIO_WritePin(LED_OFF)     ← garantiza que el LED arranca apagado
loop infinito:
  g_task_led_cnt++
  task_led_statechart()
  vTaskDelayUntil(&last_wake_time, 50 ticks = 50 ms)
```

Usa `vTaskDelayUntil`: la tarea se activa exactamente cada 50 ms, medido desde `last_wake_time`. La función descuenta automáticamente el tiempo de ejecución del cuerpo para mantener una cadencia estrictamente periódica. Esto garantiza que la temporización del LED (toggle cada 500 ms = 10 ciclos de 50 ms) sea precisa, independientemente del tiempo de ejecución de `task_led_statechart()`.

#### Máquina de estados `task_led_statechart()`

**Diagrama de estados:**

```
             flag=true && EV_LED_BLINK
┌──────────┐  ─────────────────────────►  ┌──────────────┐
│ ST_LED   │   flag=false, guarda tick     │  ST_LED      │
│   OFF    │   LED_ON                      │  BLINK       │──┐
└──────────┘  ◄─────────────────────────  └──────────────┘  │
             flag=true && EV_LED_OFF        │              elapsed≥500ms
             LED_OFF, flag=false            └──────────────┘
                                             toggle LED
                                             guarda tick
```

| Estado | Condición | Acción | Próximo estado |
|--------|-----------|--------|----------------|
| `ST_LED_OFF` | `flag=true` **y** `EV_LED_BLINK` | `flag=false`, guarda tick, enciende LED | `ST_LED_BLINK` |
| `ST_LED_BLINK` | `flag=true` **y** `EV_LED_OFF` | `flag=false`, apaga LED | `ST_LED_OFF` |
| `ST_LED_BLINK` | elapsed ≥ 500 ms | Guarda tick, `HAL_GPIO_TogglePin()` | `ST_LED_BLINK` |
| `default` | estado inválido | reinicia, LED_OFF | `ST_LED_OFF` |

**Rol del campo `flag`:**

El campo `flag` actúa como un indicador de "evento nuevo disponible". La máquina de estados solo toma una transición cuando `flag == true`, es decir, cuando `task_btn` depositó un evento nuevo mediante `put_event_task_led()`. Tras procesar el evento, `flag` se pone a `false` para no volver a procesarlo. Esto evita que un evento quede "activo" indefinidamente y cause transiciones repetidas.

**Temporización del blink:**

Con `DEL_LED_MAX = 500` ticks (500 ms a 1 kHz) y una tarea que corre cada 50 ms, el LED cambia de estado cada 10 iteraciones de la tarea. El ciclo completo de parpadeo (ON→OFF→ON) es de 1 segundo.

---

### 6.5 `task_led_interface.c` — Interfaz entre tareas

```c
void put_event_task_led(task_led_ev_t event)
{
    task_led_dta.event = event;
    task_led_dta.flag  = true;
}
```

Esta función es la única interfaz pública que `task_btn` usa para comunicarse con `task_led`. Actualiza directamente el campo `event` y activa el `flag` en la estructura `task_led_dta`.

**Mecanismo de comunicación: memoria compartida con flag**

En lugar de usar la cola `h_btn_led_q` o el semáforo binario `h_btn_led_bin_sem` (ambos creados en `app_init()` pero no utilizados), la comunicación se realiza mediante escritura directa en la estructura de datos de `task_led`.

**¿Por qué es seguro este acceso sin protección explícita?**

Ambas tareas tienen la misma prioridad (1). El scheduler de FreeRTOS con `configUSE_PREEMPTION = 1` sigue siendo **cooperativo entre tareas de igual prioridad** en cuanto al tiempo de quantum: el cambio de contexto entre `task_btn` y `task_led` sólo ocurre en los puntos de bloqueo (`vTaskDelay`, `vTaskDelayUntil`). Por lo tanto:

- `task_btn` llama `put_event_task_led()` dentro de su cuerpo (antes del `vTaskDelay`), sin que `task_led` pueda interrumpirla en ese momento.
- `task_led` lee y limpia `flag` dentro de su cuerpo (antes del `vTaskDelayUntil`), sin que `task_btn` pueda interrumpirla.
- No hay acceso concurrente real a `task_led_dta` entre las dos tareas.

El campo `flag` y el campo `event` son de tipo nativo de la plataforma (escalar de 32 bits), por lo que la escritura es atómica en Cortex-M4.

**Diferencia con el patrón de cola**

| Característica | Memoria compartida + flag | Cola FreeRTOS |
|----------------|--------------------------|---------------|
| Pérdida de eventos | Si `task_btn` llama dos veces antes que `task_led` lea, el primer evento se pierde | La cola preserva hasta `N` eventos |
| Desacoplamiento | Bajo (acceso directo a datos internos) | Alto (interfaz abstracta) |
| Bloqueo | No bloquea | Puede bloquear si la cola está llena |
| Overhead | Mínimo | Requiere mecanismos de sincronización del kernel |

En esta aplicación la pérdida de un evento de botón es aceptable porque el período de `task_btn` (50 ms) es mayor que la reacción esperada del usuario, y el flag actúa como latch de un solo nivel.

---

### 6.6 `freertos.c` (app/src/) — Hooks de la aplicación

#### `vApplicationIdleHook()`

```c
void vApplicationIdleHook(void)
{
    g_task_idle_cnt++;
}
```

Llamado en cada iteración del bucle de la tarea idle (prioridad 0), cuando ninguna de las tareas de prioridad 1 está en estado Ready. Requisitos del hook: **nunca debe bloquear** (sin `vTaskDelay`, sin espera en semáforos/queues). El valor de `g_task_idle_cnt` permite estimar la carga de CPU: a mayor frecuencia de idle, menor utilización del procesador por las tareas de aplicación.

#### `vApplicationTickHook()`

```c
void vApplicationTickHook(void)
{
    g_app_tick_cnt++;
}
```

Llamado desde la ISR del SysTick (contexto de interrupción), una vez por tick del kernel (cada 1 ms). Restricciones: no puede llamar funciones FreeRTOS que no terminen en `FromISR`. El valor de `g_app_tick_cnt` es una medida de tiempo real de la aplicación en ticks.

#### `vApplicationStackOverflowHook()`

```c
void vApplicationStackOverflowHook(xTaskHandle xTask, signed char *pcTaskName)
{
    taskENTER_CRITICAL();
    configASSERT(0);      // deshabilita interrupciones y bucle infinito
    taskEXIT_CRITICAL();
    g_app_stack_overflow_cnt++;
}
```

Llamado cuando FreeRTOS detecta desbordamiento de stack (requiere `configCHECK_FOR_STACK_OVERFLOW ≥ 1`). La implementación entra en sección crítica y ejecuta el assert fatal: `configASSERT(0)` deshabilita interrupciones (`taskDISABLE_INTERRUPTS()`) y entra en bucle infinito para detener el sistema y permitir depuración con JTAG/SWD. El incremento de `g_app_stack_overflow_cnt` es inalcanzable tras el assert, pero documenta la intención.

---

### 6.7 Arquitectura general del sistema en tiempo de ejecución

```
┌─────────────────────────────────────────────────────┐
│                    Interrupciones                    │
│  SysTick(1ms)  TIM1(1ms)  TIM2(100µs)  EXTI(botón) │
│      │             │           │              │       │
│      │             │           │              │       │
│  Scheduler    HAL_IncTick  ulHighFreq    (vacío)     │
│   FreeRTOS      uwTick     TimerTicks               │
└──────┬──────────────────────────────────────────────┘
       │  planifica
       ▼
┌────────────────────────────────────────────────────────┐
│                   Tareas FreeRTOS                      │
│                                                        │
│  ┌─────────────┐          ┌──────────────┐            │
│  │  task_btn   │          │  task_led    │            │
│  │  prio = 1   │          │  prio = 1    │            │
│  │  50 ms      │          │  50 ms       │            │
│  │  vTaskDelay │          │  vTaskDelay  │            │
│  │             │          │  Until       │            │
│  │  Lee GPIO   │          │  LEE flag    │            │
│  │  Máq.estados│──────►   │  Máq.estados │            │
│  │  Debounce   │  put_ev  │  500ms blink │            │
│  └─────────────┘  t_led  └──────────────┘            │
│                                                        │
│  ┌─────────────┐                                      │
│  │  Idle Task  │  prio = 0                            │
│  │  g_task_    │  (cuando btn y led están bloqueadas) │
│  │  idle_cnt++ │                                      │
│  └─────────────┘                                      │
└────────────────────────────────────────────────────────┘
```

**Flujo temporal típico:**

```
t=0ms:  task_btn corre → Lee GPIO, ejecuta statechart → vTaskDelay(50ms)
t=0ms:  task_led corre → Lee flag, ejecuta statechart → vTaskDelayUntil(50ms)
t=0ms:  idle task corre → g_task_idle_cnt++ (repetidamente)
...
t=50ms: task_btn despierta → detecta BTN_DOWN → guarda tick → ST_BTN_FALLING
t=50ms: task_led despierta → flag=false, no hay evento nuevo → toggle check
...
t=100ms: task_btn despierta → 50ms elapsed desde BTN_DOWN → confirma → put_event_task_led(EV_LED_BLINK)
t=100ms: task_led despierta → flag=true, EV_LED_BLINK → enciende LED, ST_LED_BLINK
...
t=600ms: task_led → 500ms elapsed → toggle LED (apaga)
t=650ms: task_led → 500ms elapsed → toggle LED (enciende)
...  (parpadeo continuo mientras botón sostenido)
...
t=N ms:  task_btn confirma liberación → put_event_task_led(EV_LED_OFF)
t=N ms:  task_led → flag=true, EV_LED_OFF → apaga LED, ST_LED_OFF
```
