# Análisis del proyecto `sotri-tp1_01-application`

**Plataforma:** STM32F446RE (Cortex-M4F, 180 MHz máx.)
**Archivos analizados:** `startup_stm32f446retx.s`, `main.c`, `stm32f4xx_it.c`,
`FreeRTOSConfig.h`, `freertos.c` (Core), `stm32f4xx_hal_timebase_tim.c`, `system_stm32f4xx.c`,
`app.c`, `task_btn.c`, `task_led.c`, `task_led_interface.c`, `freertos.c` (app)

> **Nota:** El enunciado menciona nombres de archivo propios del STM32F103 (`startup_stm32f103rbtx.s`, `stm32f1xx_it.c`). El proyecto analizado usa un STM32F446RE; los archivos equivalentes son `startup_stm32f446retx.s` y `stm32f4xx_it.c`. El análisis se basa en el código fuente real encontrado.

---

## 1. Descripción general de cada archivo

### 1.1 `startup_stm32f446retx.s`

Archivo de arranque en ensamblador ARM Thumb-2. Define:

- La **tabla de vectores de interrupción** (`g_pfnVectors`), que ocupa el comienzo de la memoria Flash (0x0800\_0000). El primer word contiene el valor inicial del Stack Pointer (`_estack`); el segundo, la dirección del `Reset_Handler`. Los siguientes slots corresponden a las excepciones del núcleo (NMI, HardFault, MemManage, BusFault, UsageFault, SVC, DebugMon, PendSV, SysTick) y a las interrupciones de periféricos del STM32F446 (TIM1, TIM2, USART2, etc.).
- El **`Reset_Handler`**, que es el punto de entrada real tras un reset (ver sección 2).
- El **`Default_Handler`**, que es un bucle infinito asignado como alias débil (`weak`) a todos los manejadores de interrupción que la aplicación no defina explícitamente.

Todos los handlers de periférico están declarados con `.weak`, de modo que cualquier función C con el mismo nombre los sobreescribe automáticamente en el enlace.

### 1.2 `system_stm32f4xx.c`

Implementa dos elementos clave del estándar CMSIS:

- La variable global **`SystemCoreClock`**, inicializada en `16000000` (16 MHz = frecuencia del oscilador interno HSI en reset).
- La función **`SystemInit()`**, llamada por el `Reset_Handler` antes de cualquier código C. En este proyecto sólo activa la FPU (`CP10`/`CP11` en `CPACR`) si el compilador la usa, y opcionalmente reubica el vector table. **No modifica `SystemCoreClock`**.
- La función **`SystemCoreClockUpdate()`**, que lee los registros RCC y recalcula `SystemCoreClock` según la fuente de reloj activa.

### 1.3 `main.c`

Contiene el punto de entrada de la aplicación C (`main()`). Las responsabilidades son:

| Función | Propósito |
|---|---|
| `HAL_Init()` | Inicializa la HAL; `HAL_MspInit()` activa SYSCFG/PWR y fija PendSV en prioridad NVIC 15; luego `HAL_InitTick()` configura TIM1 como base de tiempo (reemplaza SysTick para la HAL) |
| `SystemClock_Config()` | Configura el PLL para obtener SYSCLK = 84 MHz |
| `MX_GPIO_Init()` | Configura LED2 (salida) y botón B1 (entrada con flanco descendente) |
| `MX_USART2_UART_Init()` | UART2 a 115200 bps, 8N1 |
| `MX_TIM2_Init()` | Configura TIM2 (prescaler=1, period=4199); vía `HAL_TIM_Base_MspInit()` activa TIM2_CLK y habilita TIM2_IRQn con prioridad NVIC = 5 |
| `HAL_TIM_Base_Start_IT(&htim2)` | **Arranca TIM2** con interrupción de actualización (UEV) habilitada — `ulHighFrequencyTimerTicks++` cada 100 µs |
| `app_init()` | Inicializa contadores globales, crea `task_btn` y `task_led` (pila 256 words, prioridad 1 cada una), inicializa DWT |
| `osKernelStart()` | Arranca el scheduler de FreeRTOS — **nunca retorna** |
| `StartDefaultTask()` | Guardada como fallback: bucle `osDelay(1)`. **No se crea en esta compilación** (`#ifdef _defaultTask_` no está definido) |
| `HAL_TIM_PeriodElapsedCallback()` | Callback de desbordamiento: TIM1 → `HAL_IncTick()`; TIM2 → `ulHighFrequencyTimerTicks++` |
| `configureTimerForRunTimeStats()` | Implementación real (no-`__weak`) en `main.c`: resetea `ulHighFrequencyTimerTicks = 0` al iniciar el scheduler |
| `getRunTimeCounterValue()` | Implementación real (no-`__weak`) en `main.c`: retorna `ulHighFrequencyTimerTicks` al kernel FreeRTOS |

### 1.4 `stm32f4xx_it.c`

Define los manejadores de interrupción activos en la aplicación:

- **Excepciones del núcleo** (`NMI_Handler`, `HardFault_Handler`, `MemManage_Handler`, `BusFault_Handler`, `UsageFault_Handler`, `DebugMon_Handler`): bucles infinitos, para retención del estado en depuración.
- **`TIM1_UP_TIM10_IRQHandler`**: delega en `HAL_TIM_IRQHandler(&htim1)`. Es la ISR de la base de tiempo HAL.
- **`TIM2_IRQHandler`**: delega en `HAL_TIM_IRQHandler(&htim2)`. Es la ISR del timer de aplicación.

Los manejadores de FreeRTOS (`SVC_Handler`, `PendSV_Handler`, `SysTick_Handler`) **no** aparecen aquí porque son resueltos mediante macros en `FreeRTOSConfig.h`:

```c
#define vPortSVCHandler    SVC_Handler
#define xPortPendSVHandler PendSV_Handler
#define xPortSysTickHandler SysTick_Handler
```

### 1.5 `FreeRTOSConfig.h`

Archivo de configuración del kernel FreeRTOS. Los parámetros más relevantes:

| Parámetro | Valor | Significado |
|---|---|---|
| `configUSE_PREEMPTION` | 1 | Scheduler preemptivo |
| `configCPU_CLOCK_HZ` | `SystemCoreClock` | 84 000 000 Hz tras `SystemClock_Config()` |
| `configTICK_RATE_HZ` | 1000 | Tick del kernel = 1 ms |
| `configMAX_PRIORITIES` | 7 | 7 niveles de prioridad de tarea |
| `configMINIMAL_STACK_SIZE` | 128 words | Tamaño de pila mínima (idle task) |
| `configTOTAL_HEAP_SIZE` | 15 360 bytes | Heap FreeRTOS |
| `configSUPPORT_STATIC_ALLOCATION` | 1 | Permite asignación estática de TCBs |
| `configGENERATE_RUN_TIME_STATS` | 1 | Activa estadísticas de CPU por tarea |
| `configUSE_IDLE_HOOK` | 1 | Activa hook de idle |
| `configUSE_TICK_HOOK` | 1 | Activa hook de tick |
| `configCHECK_FOR_STACK_OVERFLOW` | 1 | Detección de desbordamiento de pila |
| `configKERNEL_INTERRUPT_PRIORITY` | `15 << 4 = 0xF0` | Prioridad más baja (15) para SysTick y PendSV |
| `configMAX_SYSCALL_INTERRUPT_PRIORITY` | `5 << 4 = 0x50` | Prioridad máxima desde la cual se pueden llamar APIs FreeRTOS seguras |

El mapping de handlers:
```c
#define vPortSVCHandler    SVC_Handler      // inicio del primer task
#define xPortPendSVHandler PendSV_Handler   // context switch
#define xPortSysTickHandler SysTick_Handler // tick del kernel
```

El comentario del archivo indica que `xPortSysTickHandler` se *comenta* cuando la base de tiempo de la HAL es SysTick (para evitar conflicto). Aquí **no está comentado** porque la HAL usa TIM1, dejando SysTick libre para FreeRTOS.

Soporte para estadísticas de runtime:
```c
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS configureTimerForRunTimeStats
#define portGET_RUN_TIME_COUNTER_VALUE         getRunTimeCounterValue
```

### 1.6 `freertos.c`

`Core/Src/freertos.c` contiene los hooks y funciones auxiliares de FreeRTOS generados por STM32CubeIDE. Las marcadas como `__weak` son sobreescritas en el enlace por implementaciones reales ubicadas en `main.c` (runtime stats) y `app/src/freertos.c` (hooks de idle, tick y stack overflow):

| Función | Atributo | Comportamiento en `Core/Src/freertos.c` | Sobreescrita por |
|---|---|---|---|
| `configureTimerForRunTimeStats()` | `__weak` | Vacía (stub) | `main.c` → `ulHighFrequencyTimerTicks = 0` |
| `getRunTimeCounterValue()` | `__weak` | Retorna 0 (stub) | `main.c` → `return ulHighFrequencyTimerTicks` |
| `vApplicationIdleHook()` | `__weak` | Vacía (stub) | `app/src/freertos.c` → `g_task_idle_cnt++` |
| `vApplicationTickHook()` | `__weak` | Vacía (stub) | `app/src/freertos.c` → `g_app_tick_cnt++` |
| `vApplicationStackOverflowHook()` | `__weak` | Vacía (stub) | `app/src/freertos.c` → `configASSERT(0)` |
| `vApplicationGetIdleTaskMemory()` | — | Provee memoria estática para la idle task (`xIdleTaskTCBBuffer`, `xIdleStack[128]`) | No sobreescrita |

---

## 2. Comportamiento desde `Reset_Handler` hasta el `while(1)` de `main()`

### 2.1 Secuencia detallada paso a paso

```
RESET DEL HARDWARE
│
├─ CPU en Thread mode, privilegiado, Stack = MSP
├─ Reloj: HSI = 16 MHz (por defecto hardware)
├─ SysTick: deshabilitado
└─ SystemCoreClock: indeterminado en RAM (sección .data no copiada aún)

RESET_HANDLER (startup_stm32f446retx.s)
│
├─ 1. ldr sp, =_estack
│     SP apunta al tope de la RAM interna (fin de SRAM)
│
├─ 2. bl SystemInit          (system_stm32f4xx.c)
│     · Activa FPU: SCB->CPACR |= CP10|CP11
│     · NO modifica SystemCoreClock
│     · Reloj sigue en HSI = 16 MHz
│     · SysTick: sigue deshabilitado
│
├─ 3. Copia sección .data (Flash → SRAM)
│     · Bucle: ldr r4,[r2,r3] / str r4,[r0,r3]
│     · Resultado: SystemCoreClock = 16 000 000 en RAM
│       (valor de inicialización estática copiado desde Flash)
│
├─ 4. Rellena sección .bss con ceros
│     · Bucle: str r3,[r2]
│     · Variables globales sin inicializador quedan en 0
│
├─ 5. bl __libc_init_array
│     · Ejecuta constructores estáticos de C++ (vacío en C puro)
│
└─ 6. bl main

MAIN()
│
├─ A. HAL_Init()
│     │
│     ├─ HAL_SetTickFreq(HAL_TICK_FREQ_DEFAULT)
│     ├─ HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4)
│     │    → 4 bits de prioridad de grupo, 0 de subgrupo
│     │
│     └─ HAL_InitTick(TICK_INT_PRIORITY)    ← stm32f4xx_hal_timebase_tim.c
│          · Habilita reloj de TIM1 (APB2)
│          · Consulta PCLK2 = 16 MHz (reloj aún sin configurar)
│          · Prescaler TIM1 = (16 000 000 / 1 000 000) − 1 = 15
│          · Period TIM1 = (1 000 000 / 1000) − 1 = 999
│          · Contador TIM1: 16 MHz / 16 = 1 MHz → desborda cada 1 ms ✓
│          · Habilita interrupción TIM1_UP (TIM1_UP_TIM10_IRQn)
│          · SysTick: NO se configura (TIM1 reemplaza a SysTick para la HAL)
│          · SystemCoreClock = 16 000 000
│
├─ B. SystemClock_Config()
│     │
│     ├─ Habilita reloj PWR, escala de voltaje 3
│     ├─ Configura PLL:
│     │    Fuente: HSI = 16 MHz
│     │    PLLM = 16  →  VCO entrada = 16/16 = 1 MHz
│     │    PLLN = 336 →  VCO salida  = 336 MHz
│     │    PLLP = 4   →  SYSCLK      = 336/4 = 84 MHz
│     │    PLLQ = 2   →  USB/SDIO clock (no usado aquí)
│     │
│     ├─ Configura buses:
│     │    AHB  prescaler = 1  → HCLK  = 84 MHz
│     │    APB1 prescaler = 2  → PCLK1 = 42 MHz  (TIM2, USART2, ...)
│     │    APB2 prescaler = 1  → PCLK2 = 84 MHz  (TIM1, ...)
│     │    Flash latency = 2 wait states
│     │
│     └─ HAL_RCC_ClockConfig() internamente llama HAL_InitTick() de nuevo:
│          · Ahora PCLK2 = 84 MHz
│          · Prescaler TIM1 = (84 000 000 / 1 000 000) − 1 = 83
│          · Period TIM1 = 999  → 84 MHz / 84 = 1 MHz → 1 ms ✓
│          · SystemCoreClock = 84 000 000  ← ACTUALIZADO
│          · SysTick: sigue deshabilitado
│
├─ C. MX_GPIO_Init()
│     · Habilita relojes GPIOA, B, C, H
│     · B1 (PC13): entrada, flanco descendente (EXTI)
│     · LD2 (PA5): salida push-pull, nivel inicial bajo
│
├─ D. MX_USART2_UART_Init()
│     · USART2: 115200 bps, 8N1, sin control de flujo, oversampling ×16
│
├─ E. MX_TIM2_Init() → HAL_TIM_Base_Init() → HAL_TIM_Base_MspInit()
│     · Instancia: TIM2 (bus APB1)
│     · Reloj TIM2: APB1 prescaler = 2 > 1 → TIM2 clock = 2 × PCLK1 = 84 MHz
│     · Prescaler = 2 − 1 = 1  → contador a 84 MHz / 2 = 42 MHz
│     · Period    = 4200 − 1   → desborda cada 4200 cuentas
│     · Frecuencia de desbordamiento = 42 MHz / 4200 = 10 000 Hz (100 µs)
│     · Modo: conteo ascendente, sin preload de ARR
│     · MspInit: activa TIM2_CLK, fija TIM2_IRQn en prioridad NVIC 5 / subprio 0
│     · TIM2 configurado pero aún no arrancado
│     · SysTick: sigue deshabilitado
│     · SystemCoreClock = 84 000 000
│
├─ E'. HAL_TIM_Base_Start_IT(&htim2)
│     · Habilita la interrupción de actualización (UEV) de TIM2 en el periférico
│     · TIM2 ARRANCA: genera TIM2_IRQHandler cada 100 µs
│          → HAL_TIM_IRQHandler → HAL_TIM_PeriodElapsedCallback → ulHighFrequencyTimerTicks++
│     · SysTick: sigue deshabilitado
│
├─ F. app_init()    (app.c — llamada desde main() antes del scheduler)
│     · Inicializa contadores globales a cero (g_app_cnt, g_app_tick_cnt, etc.)
│     · Log por UART (identificación del proyecto)
│     · Crea task_btn: pila 256 words, prioridad tskIDLE_PRIORITY + 1 = 1
│     · Crea task_led: pila 256 words, prioridad tskIDLE_PRIORITY + 1 = 1
│     · Inicializa DWT (contador de ciclos para medición en µs)
│     · defaultTask NO se crea (#ifdef _defaultTask_ no está definido)
│     · SysTick: sigue deshabilitado
│
├─ G. osKernelStart() → xTaskStartScheduler() → xPortStartScheduler()
│     │                                          (port.c, ARM_CM4F)
│     ├─ Fija PendSV en prioridad 0xF0 (nivel 15, la más baja)
│     ├─ Fija SysTick en prioridad 0xF0 (nivel 15, la más baja)
│     │
│     ├─ vPortSetupTimerInterrupt():
│     │    · Detiene y limpia SysTick (CTRL=0, VAL=0)
│     │    · LOAD = (84 000 000 / 1000) − 1 = 83 999
│     │    · CTRL = CLKSOURCE(1)|TICKINT(1)|ENABLE(1)
│     │    · SysTick ARRANCA, genera interrupción cada 1 ms ← PRIMER TICK
│     │    · SystemCoreClock = 84 000 000 (sin cambio)
│     │
│     ├─ vPortEnableVFP(): asegura FPU activa, activa lazy save (FPCCR)
│     │
│     └─ prvPortStartFirstTask():
│          · Carga MSP desde la posición 0 del vector table (= _estack)
│          · Limpia CONTROL (modo Thread privilegiado)
│          · cpsie i / cpsie f  → habilita interrupciones globales
│          · svc 0  → dispara SVC → vPortSVCHandler
│               · Restaura contexto de task_btn (primera tarea en el scheduler)
│               · PSP apunta a la pila de task_btn
│               · bx r14 (EXC_RETURN=0xFFFFFFFD) → retorna a Thread mode con PSP
│          · CPU comienza a ejecutar task_btn (o task_led, según el scheduler)
│
│  *** CONTROL NUNCA VUELVE A main() ***
│
└─ while(1) { }   ← CÓDIGO INALCANZABLE
```

---

## 3. Evolución de `SystemCoreClock` y SysTick

### 3.1 Variable `SystemCoreClock`

| Momento | Valor | Causa |
|---|---|---|
| Reset hardware | Indeterminado (RAM sin inicializar) | Sección `.data` aún no copiada |
| Tras `SystemInit()` (antes de copia `.data`) | Indeterminado en RAM | `SystemInit` no la modifica |
| Tras copia de `.data` (paso 3 del startup) | **16 000 000** | Valor de inicialización estática copiado desde Flash |
| Tras `HAL_Init()` | **16 000 000** | HAL usa el reloj por defecto (HSI) |
| Tras `SystemClock_Config()` → `HAL_RCC_ClockConfig()` | **84 000 000** | PLL activo; HAL llama `SystemCoreClockUpdate()` internamente |
| Durante `MX_TIM2_Init()`, `HAL_TIM_Base_Start_IT()`, `app_init()` | **84 000 000** | Sin cambio |
| Tras `osKernelStart()` y en adelante | **84 000 000** | Sin cambio |

### 3.2 SysTick

| Momento | Estado | LOAD / CTRL |
|---|---|---|
| Reset hardware | Deshabilitado | CTRL=0 |
| `SystemInit()`, copia `.data`, `__libc_init_array` | Deshabilitado | CTRL=0 |
| `HAL_Init()` | Deshabilitado | CTRL=0 — HAL usa TIM1, no SysTick |
| `SystemClock_Config()` | Deshabilitado | CTRL=0 |
| `MX_GPIO_Init()`, `MX_USART2_UART_Init()` | Deshabilitado | CTRL=0 |
| `MX_TIM2_Init()` | Deshabilitado | CTRL=0 |
| `HAL_TIM_Base_Start_IT(&htim2)` | Deshabilitado | CTRL=0 — TIM2 arranca (interrupción de periférico, independiente de SysTick) |
| `app_init()` | Deshabilitado | CTRL=0 |
| `osKernelStart()` → `vPortSetupTimerInterrupt()` | **Habilitado** | LOAD=83 999, CTRL=0x07 |
| En ejecución del scheduler | Genera IRQ cada **1 ms** | LOAD=83 999, reloj interno del core |

---

## 4. Interacción de SysTick y TIM1 con FreeRTOS

### 4.1 SysTick — El tick del kernel

FreeRTOS en Cortex-M4 usa el **SysTick** como fuente de tick del scheduler. La cadena de eventos en cada periodo de 1 ms es:

```
SysTick desborda (cada 1 ms, LOAD=83 999 con SYSCLK=84 MHz)
    │
    └─ SysTick_Handler()   ← resuelto como xPortSysTickHandler() por el #define en FreeRTOSConfig.h
           │
           └─ portDISABLE_INTERRUPTS()   (eleva BASEPRI a configMAX_SYSCALL_INTERRUPT_PRIORITY)
                  │
                  └─ xTaskIncrementTick()
                         · Incrementa el contador de ticks del kernel
                         · Desbloquea tareas cuyo delay expiró
                         · Si hay tarea de mayor prioridad lista → retorna pdTRUE
                         │
                         └─ (si pdTRUE) portNVIC_INT_CTRL_REG |= portNVIC_PENDSVSET_BIT
                                 │
                                 └─ PendSV_Handler()  ← xPortPendSVHandler()
                                        · Guarda contexto de la tarea actual (R4-R11, LR, FPU si aplica)
                                        · Llama vTaskSwitchContext() → selecciona nueva tarea
                                        · Restaura contexto de nueva tarea
                                        · bx r14 → retorna a nueva tarea
```

**Para qué:** SysTick proporciona la cadencia temporal del kernel. Permite que `vTaskDelay()`, `osDelay()`, timeouts de semáforos/colas y el scheduler preemptivo funcionen. Sin SysTick, el kernel no puede avanzar su contador de tiempo ni realizar cambios de contexto automáticos.

**Prioridades de las excepciones FreeRTOS:**
- SysTick y PendSV → prioridad NVIC = 0xF0 (nivel 15, la más baja posible con 4 bits).
- SVC → misma prioridad.
- Las ISRs de periféricos que llamen APIs FreeRTOS deben tener prioridad ≥ 5 (numéricamente ≥ 0x50 = `configMAX_SYSCALL_INTERRUPT_PRIORITY`).
- TIM1 y TIM2 pueden configurarse con cualquier prioridad ≥ 5.

### 4.2 TIM1 — La base de tiempo de la HAL (interacción indirecta con FreeRTOS)

En este proyecto la HAL **no usa SysTick** como base de tiempo: usa **TIM1**. Esto es intencional para evitar el conflicto con FreeRTOS, que sí ocupa SysTick.

```
TIM1 desborda (cada 1 ms, prescaler=83, period=999, PCLK2=84 MHz)
    │
    └─ TIM1_UP_TIM10_IRQHandler()   (stm32f4xx_it.c)
           │
           └─ HAL_TIM_IRQHandler(&htim1)   (stm32f4xx_hal_tim.c)
                  │
                  └─ HAL_TIM_PeriodElapsedCallback(&htim1)   (main.c)
                         │
                         └─ HAL_IncTick()
                                · Incrementa uwTick (contador global HAL en ms)
                                · uwTick es usado por HAL_Delay(), HAL_GetTick(),
                                  y por timeouts internos de la HAL (SPI, I2C, UART, etc.)
```

**Para qué:** `uwTick` es el reloj de pared de la HAL. La función `HAL_InitTick()` (definida en `stm32f4xx_hal_timebase_tim.c`) configura TIM1 de forma que su interrupción de actualización genere un tick HAL de 1 ms, equivalente al que normalmente proveería el SysTick. Al separar ambas fuentes de tiempo (TIM1 para HAL, SysTick para FreeRTOS), se evita que los dos sistemas intenten manejar la misma excepción.

**Relación con FreeRTOS:** Durante la inicialización previa al scheduler (pasos A–F del arranque), `HAL_Delay()` usa `uwTick` incrementado por TIM1. Una vez iniciado el scheduler, las tareas FreeRTOS deben usar `osDelay()` o `vTaskDelay()` en lugar de `HAL_Delay()`, ya que `HAL_Delay()` realiza polling activo sobre `uwTick` y bloquea el CPU.

---

## 5. Interacción de TIM2 con la HAL

### 5.1 Inicialización vía HAL

TIM2 es inicializado en `main.c` usando la API de la HAL:

```c
// En MX_TIM2_Init():
htim2.Instance = TIM2;
htim2.Init.Prescaler          = 2 - 1;        // divide por 2
htim2.Init.CounterMode        = TIM_COUNTERMODE_UP;
htim2.Init.Period             = 4200 - 1;      // recarga en 4200
htim2.Init.ClockDivision      = TIM_CLOCKDIVISION_DIV1;
htim2.Init.AutoReloadPreload  = TIM_AUTORELOAD_PRELOAD_DISABLE;
HAL_TIM_Base_Init(&htim2);                     // configura registros TIM2
HAL_TIM_ConfigClockSource(&htim2, ...);        // fuente interna
HAL_TIMEx_MasterConfigSynchronization(&htim2, ...); // sin sincronismo maestro
```

`HAL_TIM_Base_Init()` internamente invoca `HAL_TIM_Base_MspInit()` (definida en `stm32f4xx_hal_msp.c`), que activa el reloj del periférico y configura los pines si corresponde.

### 5.2 Manejo de la interrupción vía HAL

La ISR de TIM2 está registrada en `stm32f4xx_it.c`:

```c
void TIM2_IRQHandler(void)
{
    HAL_TIM_IRQHandler(&htim2);
}
```

`HAL_TIM_IRQHandler()` (en `stm32f4xx_hal_tim.c`) lee los flags de TIM2, los limpia y llama al callback correspondiente. Para el evento de desbordamiento (Update Event) llama a `HAL_TIM_PeriodElapsedCallback()`. En `main.c` ese callback sólo procesa TIM1:

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM1)
    {
        HAL_IncTick();
    }
    // TIM2: ninguna acción definida
}
```

### 5.3 Propósito de TIM2 en este proyecto

TIM2 es el **contador de alta resolución para las estadísticas de runtime de FreeRTOS** (`configGENERATE_RUN_TIME_STATS = 1`). La infraestructura definida en `FreeRTOSConfig.h`:

```c
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS  configureTimerForRunTimeStats
#define portGET_RUN_TIME_COUNTER_VALUE          getRunTimeCounterValue
```

requiere que `configureTimerForRunTimeStats()` inicialice el contador de tiempo y que `getRunTimeCounterValue()` lo lea. `Core/Src/freertos.c` provee stubs `__weak` (vacíos), pero `main.c` define implementaciones reales (no-`__weak`) que las sobreescriben en el enlace:

```c
// En main.c — implementaciones reales, no weak:
void configureTimerForRunTimeStats(void)  { ulHighFrequencyTimerTicks = 0; }
unsigned long getRunTimeCounterValue(void){ return ulHighFrequencyTimerTicks; }
```

TIM2 **sí se arranca** en `main.c` mediante `HAL_TIM_Base_Start_IT(&htim2)` antes de llamar a `osKernelStart()`. Cada vez que TIM2 desborda (cada 100 µs), su ISR incrementa la variable global `ulHighFrequencyTimerTicks`. Esto provee a FreeRTOS una base de tiempo de alta resolución para contabilizar el tiempo de CPU consumido por cada tarea.

FreeRTOS llama a `configureTimerForRunTimeStats()` al iniciar el scheduler (resetea el contador) y a `getRunTimeCounterValue()` en cada cambio de contexto y al generar los reportes de `vTaskGetRunTimeStats()`.

Con prescaler=1 y reloj TIM2 = 84 MHz / 2 = 42 MHz, cada overflow ocurre en exactamente 100 µs (resolución del contador ≈ 23.8 ns), ofreciendo una granularidad 10 veces superior al tick del kernel (1 ms).

### 5.4 Resumen de la interacción TIM2–HAL

```
MX_TIM2_Init()
    └─ HAL_TIM_Base_Init(&htim2)           ← configura registros TIM2 vía HAL
         └─ HAL_TIM_Base_MspInit()         ← activa TIM2_CLK, NVIC TIM2_IRQn prio=5

HAL_TIM_Base_Start_IT(&htim2)              ← TIM2 ARRANCA con UEV interrupt habilitada

TIM2 desborda (cada 100 µs)
    └─ TIM2_IRQHandler()                   (stm32f4xx_it.c)
           └─ HAL_TIM_IRQHandler(&htim2)   (stm32f4xx_hal_tim.c)
                  └─ HAL_TIM_PeriodElapsedCallback()  (main.c)
                         └─ ulHighFrequencyTimerTicks++

FreeRTOS runtime stats:
    portCONFIGURE_TIMER_FOR_RUN_TIME_STATS → configureTimerForRunTimeStats()
        └─ ulHighFrequencyTimerTicks = 0   (al iniciar scheduler)
    portGET_RUN_TIME_COUNTER_VALUE → getRunTimeCounterValue()
        └─ return ulHighFrequencyTimerTicks (en cada context switch / vTaskGetRunTimeStats)
```

---

## 6. Diagrama de flujo del arranque completo

```
                     RESET
                       │
              ┌────────▼────────┐
              │  Reset_Handler  │  SP = _estack
              │  (ensamblador)  │  SystemInit() → FPU ON
              │                 │  .data copiado → SystemCoreClock=16 MHz
              │                 │  .bss → 0
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │    HAL_Init()   │  TIM1 arranca (1 ms, 16 MHz base)
              │                 │  SystemCoreClock = 16 000 000
              │                 │  SysTick: APAGADO
              └────────┬────────┘
                       │
              ┌────────▼────────────────┐
              │  SystemClock_Config()   │  PLL: 84 MHz SYSCLK
              │                         │  TIM1 reconfigurado (84 MHz base)
              │                         │  SystemCoreClock = 84 000 000
              │                         │  SysTick: APAGADO
              └────────┬────────────────┘
                       │
              ┌────────────────────────────┐
              │ MX_GPIO_Init()             │
              │ MX_UART2_Init()            │  Periféricos listos
              │ MX_TIM2_Init()             │  TIM2 configurado, TIM2_IRQn prio=5
              │ HAL_TIM_Base_Start_IT()    │  TIM2 arranca → ulHighFreqTimerTicks++
              │ app_init()                 │  task_btn + task_led creadas (prio=1)
              └────────┬───────────────────┘
                       │
              ┌────────▼────────┐
              │ osKernelStart() │  SysTick LOAD=83999, CTRL=0x07
              │                 │  PendSV prio=0xF0, SysTick prio=0xF0
              │                 │  SVC 0 → arranca primera tarea (task_btn / task_led)
              └────────┬────────┘
                       │
     ┌─────────────────┼────────────────────┐
     │                 │                    │
 ┌───▼──────────┐  ┌───▼──────────┐  ┌─────▼──────────────┐
 │  task_btn    │  │  task_led    │  │    Idle Task        │
 │  prio=1      │  │  prio=1      │  │  vApplicationIdleHook│
 │  polling GPIO│  │  control LED │  │    g_task_idle_cnt++│
 └──────────────┘  └──────────────┘  └────────────────────┘
          │
          │  cada 1 ms:   SysTick ISR → xTaskIncrementTick → PendSV → context switch
          │  cada 1 ms:   TIM1 ISR   → HAL_IncTick()  (uwTick++)
          │  cada 100 µs: TIM2 ISR   → ulHighFrequencyTimerTicks++
```

---

## 7. Resumen ejecutivo

| Timer / Excepción | Rol | Frecuencia | Beneficiario |
|---|---|---|---|
| **SysTick** | Tick del kernel RTOS | 1 ms (LOAD=83 999 @ 84 MHz) | FreeRTOS scheduler |
| **PendSV** | Context switch | A demanda (pendido por SysTick ISR) | FreeRTOS scheduler |
| **SVC** | Inicio del primer task | Una vez al arrancar | FreeRTOS scheduler |
| **TIM1** | Base de tiempo HAL (`uwTick`) | 1 ms (prescaler=83, period=999 @ 84 MHz) | HAL (HAL_Delay, timeouts) |
| **TIM2** | Contador runtime stats de FreeRTOS | 10 kHz (100 µs) | FreeRTOS stats (`ulHighFrequencyTimerTicks`) |

La decisión de diseño clave de este proyecto es **mover la base de tiempo de la HAL de SysTick a TIM1**, liberando así SysTick para uso exclusivo de FreeRTOS. Esto elimina el conflicto que se produciría si ambos sistemas intentaran registrar su propio `SysTick_Handler`.

---

## 8. Capa de Aplicación: `app.c`, `task_btn.c`, `task_led.c`, `task_led_interface.c`, `freertos.c` (app)

### 8.1 Arquitectura general de la capa de aplicación

El proyecto implementa un sistema **orientado a eventos** (*Event-Triggered System*, ETS) usando dos tareas FreeRTOS que se comunican mediante una variable compartida. La arquitectura es la siguiente:

```
┌─────────────────────────────────────────────────────────────────┐
│                        main.c                                   │
│  HAL_Init → SystemClock_Config → GPIO/UART/TIM2 init            │
│  HAL_TIM_Base_Start_IT(&htim2)   ← arranca TIM2                 │
│  app_init()  ← crea task_btn y task_led                         │
│  osKernelStart()                                                │
└─────────────────────────────────────────────────────────────────┘
          │                         │
  ┌───────▼───────┐         ┌───────▼───────┐
  │  task_btn     │         │  task_led     │
  │  Prio: 1      │─────────▶  Prio: 1      │
  │  Statechart   │ evento  │  Statechart   │
  │  (4 estados)  │         │  (2 estados)  │
  └───────────────┘         └───────────────┘
         │                          │
  lee GPIO B1 (PC13)         escribe LED LD2 (PA5)
```

### 8.2 `board.h` — Abstracción de hardware

`board.h` centraliza las definiciones de pines dependientes de la placa mediante macros condicionales controladas por la constante `BOARD`:

```c
#define BOARD (NUCLEO_F103RC)   // valor seleccionado
```

El valor `NUCLEO_F103RC` activa el mismo bloque `#if` que `NUCLEO_F401RE` y `NUCLEO_F446RE`, ya que las tres placas Nucleo-64 comparten la misma asignación de pines para botón y LED:

| Macro | Valor para NUCLEO-F446RE | Significado |
|---|---|---|
| `BTN_PRESSED` | `GPIO_PIN_RESET` | Botón B1 activo en bajo (PC13 a GND al pulsar) |
| `BTN_HOVER` | `GPIO_PIN_SET` | Botón sin pulsar (pull-up interno o externo) |
| `LED_ON` | `GPIO_PIN_SET` | LED LD2 encendido (PA5 en alto) |
| `LED_OFF` | `GPIO_PIN_RESET` | LED LD2 apagado (PA5 en bajo) |

Esta abstracción permite portar la aplicación a otra placa Nucleo cambiando únicamente la línea `#define BOARD`.

### 8.3 `app.c` — Inicialización de la aplicación

`app_init()` es llamada desde `main.c` **dentro del contexto del primer task FreeRTOS** (`StartDefaultTask`) antes de entrar en su bucle. Realiza las siguientes acciones en orden:

```c
void app_init(void)
{
    // 1. Inicializa contadores globales a cero
    g_app_cnt = g_app_task_cnt = g_app_tick_cnt = 0;
    g_task_idle_cnt = g_app_stack_overflow_cnt = 0;

    // 2. Log por UART/semihosting (identificación del proyecto)
    LOGGER_INFO(...);

    // 3. Crea task_btn — prioridad 1 (tskIDLE_PRIORITY + 1), pila 256 words
    xTaskCreate(task_btn, "Task BTN", 2*configMINIMAL_STACK_SIZE,
                NULL, tskIDLE_PRIORITY + 1, &h_task_btn);

    // 4. Crea task_led — prioridad 1 (tskIDLE_PRIORITY + 1), pila 256 words
    xTaskCreate(task_led, "Task LED", 2*configMINIMAL_STACK_SIZE,
                NULL, tskIDLE_PRIORITY + 1, &h_task_led);

    // 5. Consulta heap libre (diagnóstico)
    xPortGetFreeHeapSize();

    // 6. Inicializa el contador de ciclos DWT (para medición de tiempo en µs)
    cycle_counter_init();
}
```

**Nota:** `main.c` incluye una tarea `defaultTask` (generada por STM32CubeIDE) que sólo está compilada si se define el símbolo `_defaultTask_`. En el proyecto actual **ese símbolo no está definido**, por lo que `defaultTask` no se crea. Las únicas tareas de aplicación que corren son `task_btn` y `task_led`.

**Variables globales exportadas:**

| Variable | Modificada por | Propósito |
|---|---|---|
| `g_app_cnt` | — | Reservada (no incrementada en el código visible) |
| `g_app_task_cnt` | — | Reservada |
| `g_app_tick_cnt` | `vApplicationTickHook` (cada tick) | Cuenta los ticks del kernel transcurridos |
| `g_task_idle_cnt` | `vApplicationIdleHook` (idle task) | Cuenta iteraciones de la idle task |
| `g_app_stack_overflow_cnt` | `vApplicationStackOverflowHook` | Cuenta desbordamientos de pila detectados |
| `h_task_btn` / `h_task_led` | `app_init` | Handles para referenciar las tareas |

### 8.4 `task_btn.c` — Tarea de botón con statechart de debounce

#### Estructura de datos

```c
typedef struct {
    task_btn_ev_t  event;      // evento actual: EV_BTN_UP o EV_BTN_DOWN
    task_btn_st_t  state;      // estado actual del statechart
    TickType_t     tick;       // timestamp del último cambio de estado
    GPIO_TypeDef * gpio_port;  // puerto GPIO del botón
    uint16_t       pin;        // pin del botón
} task_btn_dta_t;

// Inicialización:
task_btn_dta_t task_btn_dta = {EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
                                B1_GPIO_Port, B1_Pin};
```

#### Diagrama de estados

```
               HAL_GPIO = RESET (pulsado)
     ┌─────────────────────────────────────────────────────┐
     │  guarda tick                                        │
     ▼                                                     │
 ┌──────────┐   tick >= 50ms && aún RESET    ┌──────────────────┐
 │ ST_BTN_UP│──────────────────────────────▶│ ST_BTN_FALLING   │
 └──────────┘                               └──────────────────┘
     ▲                                               │
     │                                  tick >= 50ms && aún RESET
     │                          → put_event_task_led(EV_LED_BLINK)
     │                                               │
     │                                               ▼
     │  tick >= 50ms && aún SET       ┌────────────────────────┐
     │  → put_event_task_led(EV_LED_OFF)│    ST_BTN_DOWN       │
     │                                └────────────────────────┘
     │                                               │
     │                              HAL_GPIO = SET (suelto)
     │                                  guarda tick │
     │                                               ▼
     │                               ┌──────────────────────────┐
     └───────────────────────────────│    ST_BTN_RISING         │
                                     └──────────────────────────┘
```

| Estado | Condición de transición | Acción |
|---|---|---|
| `ST_BTN_UP` | GPIO = RESET | Guarda tick, va a `ST_BTN_FALLING` |
| `ST_BTN_FALLING` | ≥ 50ms y GPIO = RESET | Envía `EV_LED_BLINK`, va a `ST_BTN_DOWN` |
| `ST_BTN_FALLING` | ≥ 50ms y GPIO = SET | Rebote descartado, vuelve a `ST_BTN_UP` |
| `ST_BTN_DOWN` | GPIO = SET | Guarda tick, va a `ST_BTN_RISING` |
| `ST_BTN_RISING` | ≥ 50ms y GPIO = SET | Envía `EV_LED_OFF`, va a `ST_BTN_UP` |
| `ST_BTN_RISING` | ≥ 50ms y GPIO = RESET | Rebote descartado, vuelve a `ST_BTN_DOWN` |

**Propósito del debounce (50 ms):** Los botones mecánicos generan múltiples transiciones espurias al pulsarse/soltarse (rebotes). El statechart valida que la señal permanezca estable durante 50 ms antes de considerar la transición como definitiva.

#### Comportamiento temporal del bucle

`task_btn` **nunca se bloquea ni llama a ninguna API de delay**. Su bucle `for(;;)` ejecuta `task_btn_statechart()` en cada iteración. El timing lo obtiene comparando `xTaskGetTickCount()` (ticks del kernel) contra el valor guardado en `task_btn_dta.tick`. La tarea consume CPU continuamente y es desalojada por el SysTick cada 1 ms para ceder tiempo a `task_led`.

### 8.5 `task_led.c` — Tarea de LED con statechart de parpadeo

#### Estructura de datos

```c
typedef struct {
    bool           flag;       // indicador: hay un evento pendiente
    task_led_ev_t  event;      // evento pendiente: EV_LED_OFF o EV_LED_BLINK
    task_led_st_t  state;      // estado: ST_LED_OFF o ST_LED_BLINK
    TickType_t     tick;       // timestamp del último toggle
    GPIO_TypeDef * gpio_port;  // puerto GPIO del LED
    uint16_t       pin;        // pin del LED
} task_led_dta_t;

// Inicialización:
task_led_dta_t task_led_dta = {false, EV_LED_OFF, ST_LED_OFF,
                                DEL_LED_MIN, LD2_GPIO_Port, LD2_Pin};
```

#### Diagrama de estados

```
                flag=true && EV_LED_BLINK
     ┌────────────────────────────────────────────┐
     │  flag=false, guarda tick                   │
     │  HAL_GPIO_WritePin(LED_ON)                 │
     ▼                                            │
 ┌──────────┐                          ┌──────────────────┐
 │ST_LED_OFF│                          │  ST_LED_BLINK    │
 └──────────┘                          └──────────────────┘
     ▲                                       │  │
     │  flag=true && EV_LED_OFF              │  │ tick >= 500ms
     │  flag=false                           │  │ HAL_GPIO_TogglePin
     │  HAL_GPIO_WritePin(LED_OFF)           │  └──(auto-transición)
     └───────────────────────────────────────┘
```

| Estado | Condición | Acción |
|---|---|---|
| `ST_LED_OFF` | `flag=true` y `event=EV_LED_BLINK` | Enciende LED, va a `ST_LED_BLINK` |
| `ST_LED_BLINK` | `flag=true` y `event=EV_LED_OFF` | Apaga LED, va a `ST_LED_OFF` |
| `ST_LED_BLINK` | Transcurrieron ≥ 500 ms | Invierte LED (`HAL_GPIO_TogglePin`), actualiza tick |

**Parpadeo:** En `ST_LED_BLINK`, la tarea invierte el estado del LED cada 500 ms. El resultado visible es un parpadeo a 1 Hz (500 ms encendido, 500 ms apagado).

**Patrón flag + event:** `task_led` usa dos campos para recibir eventos: `flag` indica que hay un evento nuevo, y `event` indica cuál es. Este patrón evita que `task_led` procese el mismo evento dos veces: consume el evento limpiando `flag = false` inmediatamente al procesarlo.

#### Comportamiento temporal

Al igual que `task_btn`, `task_led` **no se bloquea**. Su bucle `for(;;)` ejecuta `task_led_statechart()` continuamente, determinando el tiempo mediante comparaciones con `xTaskGetTickCount()`.

### 8.6 `task_led_interface.c` — Interfaz de comunicación entre tareas

```c
void put_event_task_led(task_led_ev_t event)
{
    task_led_dta.event = event;   // escribe el tipo de evento
    task_led_dta.flag  = true;    // señaliza que hay evento pendiente
}
```

Esta función es el único mecanismo de comunicación entre `task_btn` y `task_led`. Opera directamente sobre la variable compartida `task_led_dta` **sin ningún mecanismo de sincronización** (sin mutex, sin sección crítica, sin cola).

**¿Por qué funciona sin sincronización en este contexto?**
- Ambas tareas tienen la **misma prioridad** (1), por lo que el scheduler alterna entre ellas únicamente en los ticks del kernel (1 ms).
- Las dos asignaciones (`event` y `flag`) son operaciones de un registro de 32 bits / 8 bits, atómicas en Cortex-M4 para accesos alineados.
- `task_led` primero verifica `flag == true` y luego lee `event`. El orden de escritura en `put_event_task_led` (primero `event`, luego `flag`) asegura que cuando `task_led` ve `flag == true`, el valor de `event` ya es coherente.
- **Limitación:** Este diseño es válido para una única tarea productora y una única tarea consumidora. En escenarios más complejos (múltiples productores o acceso desde ISR) se requeriría una cola FreeRTOS (`xQueueSend`/`xQueueReceive`).

### 8.7 `freertos.c` (app) — Hooks reales de FreeRTOS

Este archivo, ubicado en `app/src/freertos.c`, **sobreescribe los stubs `__weak`** de `Core/Src/freertos.c` con implementaciones reales:

#### `vApplicationIdleHook()`

```c
void vApplicationIdleHook(void)
{
    g_task_idle_cnt++;   // incrementa en cada iteración de la idle task
}
```

Se ejecuta **continuamente** cuando no hay ninguna tarea de usuario en estado Ready. `g_task_idle_cnt` permite estimar la carga del sistema: cuanto más alto, más tiempo libre tiene la CPU. En este proyecto, dado que `task_btn` y `task_led` nunca se bloquean, `g_task_idle_cnt` crecerá muy lentamente o permanecerá en 0 (la idle task prácticamente no recibe CPU).

#### `vApplicationTickHook()`

```c
void vApplicationTickHook(void)
{
    g_app_tick_cnt++;    // se ejecuta en cada interrupción SysTick
}
```

Llamada **desde la ISR de SysTick** (dentro de `xPortSysTickHandler`). Debe ser muy breve y no puede usar APIs FreeRTOS sin el sufijo `FromISR`. `g_app_tick_cnt` es equivalente a `xTaskGetTickCount()` pero accesible como variable global sin llamada de función.

#### `vApplicationStackOverflowHook()`

```c
void vApplicationStackOverflowHook(xTaskHandle xTask, signed char *pcTaskName)
{
    taskENTER_CRITICAL();
    configASSERT(0);     // detiene la ejecución (bucle infinito con IRQs deshabilitadas)
    taskEXIT_CRITICAL();
    g_app_stack_overflow_cnt++;
}
```

Invocada cuando FreeRTOS detecta desbordamiento de pila de una tarea (método 1: verifica el patrón de llenado de pila al cambiar de contexto). La implementación entra en sección crítica y ejecuta `configASSERT(0)`, que desactiva interrupciones y cuelga el sistema. Útil para detectar el problema en tiempo de depuración.

### 8.8 TIM2 como contador de runtime stats (implementación completa)

En el `main.c` definitivo del proyecto, TIM2 **sí se arranca** y las funciones de runtime stats **sí están implementadas**:

```c
// En main() — después de MX_TIM2_Init():
HAL_TIM_Base_Start_IT(&htim2);   // arranca TIM2 con interrupción de desbordamiento

// En HAL_TIM_PeriodElapsedCallback():
if (htim->Instance == TIM2)
{
    ulHighFrequencyTimerTicks++;  // ~10 000 veces por segundo
}

// Implementaciones no-weak en main.c (anulan los stubs de freertos.c/Core):
void configureTimerForRunTimeStats(void) { ulHighFrequencyTimerTicks = 0; }
unsigned long getRunTimeCounterValue(void) { return ulHighFrequencyTimerTicks; }
```

**Cadena completa de runtime stats:**

```
TIM2 desborda cada 100 µs
    │
    └─ TIM2_IRQHandler → HAL_TIM_IRQHandler → HAL_TIM_PeriodElapsedCallback
           │
           └─ ulHighFrequencyTimerTicks++

FreeRTOS kernel (en cada context switch y en vTaskGetRunTimeStats):
    └─ portGET_RUN_TIME_COUNTER_VALUE → getRunTimeCounterValue()
           └─ return ulHighFrequencyTimerTicks
```

Con esto FreeRTOS puede reportar cuántas unidades de 100 µs consume cada tarea, con una resolución 10 veces superior al tick del kernel (1 ms).

### 8.9 Flujo de ejecución completo de la aplicación

```
osKernelStart() arranca el scheduler
│
├─ StartDefaultTask() inicia (prioridad Normal = 24)
│       └─ app_init()
│              ├─ Crea task_btn (prioridad 1)
│              ├─ Crea task_led (prioridad 1)
│              └─ cycle_counter_init()
│       └─ for(;;) { osDelay(1); }   ← queda bloqueada 1ms por iteración
│                                        (prioridad Normal > prioridad 1,
│                                         pero bloqueada la mayor parte del tiempo)
│
├─ task_btn (prioridad 1) — nunca se bloquea
│    for(;;)
│    {
│        g_task_btn_cnt++;
│        task_btn_statechart();   ← lee GPIO, actualiza estado
│    }
│
└─ task_led (prioridad 1) — nunca se bloquea
     for(;;)
     {
         g_task_led_cnt++;
         task_led_statechart();  ← evalúa flag/event, controla LED
     }
```

**Distribución del CPU:**

| Tarea | Prioridad | Estado habitual | Acción |
|---|---|---|---|
| `StartDefaultTask` | Normal (24) | Bloqueada 999ms/1000ms en `osDelay(1)` | Inactiva, cede CPU |
| `task_btn` | 1 | Always Ready | Polling GPIO + statechart |
| `task_led` | 1 | Always Ready | Statechart + control LED |
| Idle task | 0 | Ready cuando las demás bloquean | Incrementa `g_task_idle_cnt` |

`task_btn` y `task_led` tienen la misma prioridad y ninguna bloquea, por lo que se turnan cada tick de 1 ms (time slicing). La `StartDefaultTask` interrumpe su bucle de delay cada 1 ms durante un breve instante para volver a bloquearse, dado que tiene mayor prioridad.

### 8.10 Secuencia de interacción completa ante pulsación del botón

```
Usuario pulsa B1 (PC13 → GND)
│
├─ task_btn detecta GPIO_PIN_RESET → EV_BTN_DOWN → ST_BTN_FALLING
│
├─ (50 ms después, botón aún presionado)
│    task_btn → ST_BTN_DOWN
│    put_event_task_led(EV_LED_BLINK)
│         └─ task_led_dta.event = EV_LED_BLINK
│            task_led_dta.flag  = true
│
└─ task_led detecta flag=true, event=EV_LED_BLINK → ST_LED_BLINK
        └─ LED ON (GPIO_PIN_SET en PA5)
        └─ cada 500 ms: HAL_GPIO_TogglePin → LED parpadea a 1 Hz

Usuario suelta B1 (PC13 → VCC via pull-up)
│
├─ task_btn detecta GPIO_PIN_SET → EV_BTN_UP → ST_BTN_RISING
│
├─ (50 ms después, botón aún suelto)
│    task_btn → ST_BTN_UP
│    put_event_task_led(EV_LED_OFF)
│         └─ task_led_dta.event = EV_LED_OFF
│            task_led_dta.flag  = true
│
└─ task_led detecta flag=true, event=EV_LED_OFF → ST_LED_OFF
        └─ LED OFF (GPIO_PIN_RESET en PA5)
```
