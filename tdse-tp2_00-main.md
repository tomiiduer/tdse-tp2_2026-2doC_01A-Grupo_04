# Análisis de Código Fuente STM32

## Análisis de los Archivos Fuente

*   **startup_stm32f103rbtx.s**: Es el archivo de arranque (startup) escrito en lenguaje ensamblador para la arquitectura Cortex-M3. Define la tabla de vectores de interrupción (`g_pfnVectors`), situando la dirección base de la pila (`_estack`) y los punteros a los manejadores de excepciones, como `Reset_Handler` o `SysTick_Handler`. Su función principal `Reset_Handler` se ejecuta inmediatamente tras encender o reiniciar el microcontrolador. Se encarga de llamar a `SystemInit` para la configuración inicial mínima, copiar los valores de las variables inicializadas desde la memoria Flash a la RAM (`.data`), rellenar con ceros las variables no inicializadas (`.bss`) y finalmente transferir el control a la función `main()` del programa en C.
*   **main.c**: Contiene el punto de entrada de la aplicación en C, la función `main()`. Orquesta la puesta en marcha del sistema ejecutando primero `HAL_Init()` para resetear periféricos y arrancar el SysTick, seguido de `SystemClock_Config()` para establecer las frecuencias de operación deseadas. Luego, inicializa los pines GPIO y el periférico USART2, llama a `app_init()` para preparar la lógica del usuario, y atrapa la ejecución en un bucle infinito `while (1)` donde llama continuamente a `app_update()`.
*   **stm32f1xx_it.c**: Agrupa las Rutinas de Servicio de Interrupción (ISR). Las funciones aquí atrapan eventos del hardware y excepciones de la CPU. Contiene bucles infinitos para manejar fallos críticos como `HardFault_Handler`. Incluye rutinas vitales como `SysTick_Handler()`, que incrementa la base de tiempo global mediante `HAL_IncTick()`, y rutinas para eventos de usuario, como `EXTI15_10_IRQHandler()`, la cual captura los cambios de estado de un botón (`B1_Pin`).

## Evolución de SystemCoreClock y SysTick desde el Reinicio

1.  **Fase de Arranque (`Reset_Handler`)**
    Cuando el microcontrolador arranca, el hardware ejecuta `Reset_Handler`. Al llamar a la instrucción `bl SystemInit`, se establece la configuración inicial base de CMSIS. En este punto, `SystemCoreClock` adopta el valor de frecuencia por defecto del oscilador interno del microcontrolador (típicamente el HSI a 8 MHz para la familia STM32F1). El temporizador SysTick se encuentra desactivado y la cuenta de *ticks* es cero.

2.  **Inicialización de la capa HAL (`HAL_Init` en `main.c`)**
    Al entrar a `main()` y llamar a `HAL_Init()`, la librería de hardware de ST configura e inicia el periférico SysTick por primera vez. Utilizando el valor actual de `SystemCoreClock` (8 MHz), programa el temporizador para que desborde exactamente cada 1 milisegundo. A partir de este momento, se dispara un evento periódico que invoca a `SysTick_Handler()` en `stm32f1xx_it.c`, la cual llama a `HAL_IncTick()`. Esto comienza a incrementar una variable global oculta en la capa HAL (usualmente `uwTick`) a un ritmo de +1 por milisegundo.

3.  **Configuración del Reloj (`SystemClock_Config` en `main.c`)**
    Esta función redefine las frecuencias de operación activando el multiplicador de reloj (PLL). Toma el oscilador interno, lo divide por 2 y lo multiplica por 16 (`RCC_PLLSOURCE_HSI_DIV2` y `RCC_PLL_MUL16`). Al aplicar estos cambios con `HAL_RCC_ClockConfig()`, la variable del sistema `SystemCoreClock` se actualiza al nuevo valor resultante (típicamente 64 MHz para estas configuraciones en un STM32F1). Como la frecuencia del procesador cambió, la misma función re-calibra el temporizador SysTick automáticamente usando la nueva `SystemCoreClock`, asegurando que siga interrumpiendo exactamente cada 1 milisegundo sin perder sincronía.

4.  **Bucle Principal (`while (1)` en `main.c`)**
    Para cuando el programa alcanza las inicializaciones de la aplicación y entra al bucle `while (1)`, la variable `SystemCoreClock` permanece constante en su valor máximo configurado. El temporizador SysTick continúa operando ininterrumpidamente en segundo plano, ejecutando `SysTick_Handler()` a 1 kHz, por lo que la variable global de los ticks acumula dinámicamente todos los milisegundos transcurridos desde que se ejecutó `HAL_Init()` al principio del sistema.
