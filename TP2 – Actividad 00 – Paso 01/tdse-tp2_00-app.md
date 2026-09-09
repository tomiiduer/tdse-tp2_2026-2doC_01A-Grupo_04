# Análisis y Explicación del Código Fuente: Sistema Embarcado Base
**Asignatura:** Técnicas de Diseño de Sistemas Embebidos (TDSE)  
**Documento:** TP2_00 - Análisis de Aplicación Bare-Metal Event/Time-Triggered  
**Archivo de salida:** `tdse-tp2_00-app.md`

---

## 1. Introducción y Descripción General

El conjunto de archivos analizados (`app.c`, `app_it.c`, `logger.c`, `logger.h`, `systick.c` y `dwt.h`) implementa una arquitectura de software embarcado para microcontroladores ARM Cortex-M. Se trata de un sistema **Bare-Metal** basado en un esquema **Time-Triggered / Event-Triggered System (ETS)** con un planificador cooperativo derivado por ticks periódicos de tiempo (interrupción de `SysTick` configurada a 1 ms).

El sistema cuenta además con un mecanismo de monitoreo y perfilado de rendimiento en tiempo real (*profiling*) mediante el bloque de depuración **DWT (Data Watchpoint and Trace)** del núcleo ARM Cortex-M, permitiendo registrar métricas clave como:
* **NOE (*Number of Executions*):** Contador de ejecuciones de cada tarea.
* **LET (*Last Execution Time*):** Tiempo de ejecución de la última iteración (en microsegundos, $\mu\text{s}$).
* **BCET (*Best-Case Execution Time*):** Mejor tiempo de ejecución registrado (mínimo, en $\mu\text{s}$).
* **WCET (*Worst-Case Execution Time*):** Peor tiempo de ejecución registrado (máximo, en $\mu\text{s}$).
* **`g_app_runtime_us`:** Tiempo total de ejecución del superbloque de tareas en un ciclo de tick (en $\mu\text{s}$).

---

## 2. Análisis Detallado Archivo por Archivo

### 2.1. `dwt.h`
Este archivo de cabecera implementa funciones `inline` para manipular el periférico **DWT (Data Watchpoint and Trace)** presente en arquitecturas ARM Cortex-M (como Cortex-M3/M4/M7). 

* **`cycle_counter_init()`**:
  * Habilita el bloque TRACE/DEBUG del núcleo modificando el registro `CoreDebug->DEMCR` (bit `TRCENA`).
  * Reinicia el contador de ciclos del DWT (`DWT->CYCCNT = 0`).
  * Inicia la cuenta de ciclos habilitando el bit `CYCCNTENA` en `DWT->CTRL`.
* **`cycle_counter_reset()`**: Pone a cero el registro `DWT->CYCCNT` para iniciar una medición puntual.
* **`cycle_counter_enable()` / `cycle_counter_disable()`**: Activa o congela la cuenta de ciclos del DWT.
* **`cycle_counter_get()`**: Retorna el valor actual del registro `DWT->CYCCNT` (ciclos de reloj del procesador transcurridos).
* **`cycle_counter_get_time_us()`**: Convierte los ciclos transcurridos a microsegundos mediante la fórmula:
  $$\text{Tiempo (}\mu\text{s)} = \frac{\text{DWT->CYCCNT}}{\text{SystemCoreClock} / 1\,000\,000}$$
  Esta conversión provee una precisión sub-microsegundo según la frecuencia del sistema (`SystemCoreClock`).

---

### 2.2. `systick.c`
Proporciona la función `systick_delay_us(uint32_t delay_us)`, la cual implementa un retardo bloqueante de alta precisión en microsegundos utilizando el temporizador del sistema **SysTick**.

* **Funcionamiento interno:**
  1. Captura el valor inicial del contador decreciente `SysTick->VAL`.
  2. Calcula la cantidad de cuentas (*ticks* de reloj) requeridas: $\text{target} = \text{delay\_us} \times (\text{SystemCoreClock} / 1\,000\,000)$.
  3. Ejecuta un bucle `while(1)` calculando la diferencia entre la lectura actual (`SysTick->VAL`) y la lectura inicial.
  4. Maneja adecuadamente el desbordamiento (*underflow / wrap-around*) del contador cuando `current > start`, sumando el valor del registro de recarga `SysTick->LOAD`.

---

### 2.3. `logger.h` y `logger.c`
Implementan un módulo de registro/loggin de mensajes para depuración.

* **Configuración por macros (`logger.h`):**
  * `LOGGER_CONFIG_ENABLE = 1`: Activa el sistema de logs.
  * `LOGGER_CONFIG_MAXLEN = 64`: Tamaño del buffer estático de caracteres `logger_msg_buffer_`.
  * `LOGGER_CONFIG_USE_SEMIHOSTING = 1`: Redirige la salida a la consola de depuración del Host mediante **Semihosting** (`printf` / `fflush(stdout)`).
* **Macros `LOGGER_LOG(...)` y `LOGGER_INFO(...)`:**
  * Implementan secciones críticas deshabilitando interrupciones globales (`__asm("CPSID i")`) antes de formatear/imprimir el mensaje, y las vuelven a habilitar (`__asm("CPSIE i")`) al finalizar.
  * `LOGGER_INFO` antepone el prefijo `"[info] "` y agrega un carácter salto de línea `"
"`.
  * Hacen uso de `snprintf()` para formatear la cadena en `logger_msg`.
* **Función `logger_log_print_` (`logger.c`):**
  * Si `LOGGER_CONFIG_USE_SEMIHOSTING == 1`, invoca `printf(msg)` y fuerza el vaciado del buffer con `fflush(stdout)`.

---

### 2.4. `app_it.c`
Maneja la integración con la capa de interrupciones del sistema (`HAL`) y la gestión de tiempo del planificador.

* **Variable clave:** `volatile uint32_t g_app_tick_cnt;`
  * Declarada `volatile` porque se modifica dentro del manejador de interrupción ISR y se lee en el hilo principal (`app_update`).
* **`app_it_init()`**: Inicializa `g_app_tick_cnt = 0` protegiendo el acceso en una sección crítica (`CPSID i` / `CPSIE i`).
* **`HAL_SYSTICK_Callback()`**: Callback ejecutado periódicamente por la ISR de SysTick (típicamente cada 1 ms). Incrementa el contador de ticks pendientes: `g_app_tick_cnt++`.
* **`HAL_GPIO_EXTI_Callback()`**: Estructura para manejo de interrupciones externas por pulsadores (ej. `BTN_A_PIN`).

---

### 2.5. `app.c`
Es el núcleo del sistema de tareas. Define la tabla de tareas, sus variables de perfilado, el bucle de inicialización (`app_init`) y el planificador principal (`app_update`).

* **Estructuras de Datos:**
  * `task_cfg_t`: Contiene punteros a funciones `task_init`, `task_update` y sus parámetros.
  * `task_dta_t`: Almacena métricas por tarea: `NOE`, `LET`, `BCET`, `WCET`.
* **Tabla de Tareas `task_cfg_list[]`:**
  Define 3 tareas en el sistema (`TASK_QTY = 3`):
  1. `index = 0`: Sensor (`task_sensor_init`, `task_sensor_update`)
  2. `index = 1`: Sistema/Lógica (`task_system_init`, `task_system_update`)
  3. `index = 2`: Actuador (`task_actuator_init`, `task_actuator_update`)
* **`app_init()`**:
  1. Emite mensajes informativos de inicio vía `LOGGER_INFO`.
  2. Inicializa `g_app_cnt = 0`.
  3. Inicializa el contador DWT (`cycle_counter_init()`).
  4. Recorre el arreglo de tareas invocando las funciones de inicialización e inicializando la estructura de datos: `NOE = 0`, `LET = 0`, `BCET = 1000` $\mu\text{s}$, `WCET = 0` $\mu\text{s}$.
  5. Inicializa interrupciones (`app_it_init()`).
* **`app_update()`**:
  1. Entra en sección crítica para verificar si existen ticks pendientes (`g_app_tick_cnt > 0`). Si hay, decrementa `g_app_tick_cnt` y fija `b_time_update_required = true`.
  2. Mientras exista trabajo pendiente (`while (b_time_update_required)`):
     * Incrementa `g_app_cnt++`.
     * Reinicia `g_app_runtime_us = 0`.
     * Recorre iterativamente las tareas (`index = 0, 1, 2`):
       * Llama a `cycle_counter_reset()`.
       * Ejecuta la tarea (`(*task_cfg_list[index].task_update)(...)`).
       * Incrementa `NOE++`.
       * Lee el tiempo transcurrido en microsegundos y actualiza `LET`.
       * Actualiza los extremos de tiempo:
         * Si `LET < BCET` $\rightarrow$ `BCET = LET`.
         * Si `LET > WCET` $\rightarrow$ `WCET = LET`.
       * Acumula el tiempo en la variable global: `g_app_runtime_us += LET`.
     * Revisa al final si llegaron nuevos ticks durante la ejecución de las tareas. Si `g_app_tick_cnt > 0`, decrementa el contador y repite el ciclo; de lo contrario, finaliza el bucle.

---

## 3. Evolución de las Variables del Sistema

A continuación se describe la evolución detallada de las variables desde el encendido/inicio (`app_init()`) y a lo largo de las sucesivas iteraciones del bucle principal (`app_update()`).

### 3.1. Resumen de Unidades de Medida y Significado

| Variable | Tipo / Estructura | Unidad de Medida | Significado / Descripción |
| :--- | :--- | :--- | :--- |
| `g_app_tick_cnt` | `volatile uint32_t` | Ticks (ms) | Cantidad de interrupciones de SysTick pendientes por procesar. |
| `g_app_runtime_us` | `uint32_t` | Microsegundos ($\mu\text{s}$) | Tiempo total acumulado de ejecución de todas las tareas en el ciclo actual. |
| `index` | `uint32_t` | Adimensional (Índice) | Índice que identifica la tarea en ejecución ($0 \le \text{index} < 3$). |
| `task_dta_list[index].NOE` | `uint32_t` | Ejecuciones (CANT) | Número de veces que se ha ejecutado la función `update` de la tarea `index`. |
| `task_dta_list[index].LET` | `uint32_t` | Microsegundos ($\mu\text{s}$) | Duración de la última ejecución (*Last Execution Time*) de la tarea `index`. |
| `task_dta_list[index].BCET` | `uint32_t` | Microsegundos ($\mu\text{s}$) | Mejor tiempo de ejecución (*Best-Case Execution Time*) observado para la tarea `index`. |
| `task_dta_list[index].WCET` | `uint32_t` | Microsegundos ($\mu\text{s}$) | Peor tiempo de ejecución (*Worst-Case Execution Time*) observado para la tarea `index`. |

---

### 3.2. Línea Temporal de Evolución de Variables

#### A. Durante la ejecución de `app_init()`
1. **Antes del bucle de inicialización:**
   * `g_app_cnt = 0`
   * `g_app_runtime_us` = No inicializada (indeterminada o 0).
2. **Durante el bucle `for (index = 0; index < 3; index++)`:**
   * Para cada `index` ($0, 1, 2$):
     * `task_dta_list[index].NOE = 0`
     * `task_dta_list[index].LET = 0` $\mu\text{s}$
     * `task_dta_list[index].BCET = 1000` $\mu\text{s}$ (valor semilla alto de inicialización)
     * `task_dta_list[index].WCET = 0` $\mu\text{s}$ (valor semilla bajo de inicialización)
3. **Al finalizar `app_init()` (vía `app_it_init()`):**
   * `g_app_tick_cnt = 0`

---

#### B. Durante la 1.ª ejecución del ciclo principal (`app_update()`) — Primer Tick (1 ms)
Supongamos que transcurre 1 ms y la interrupción de SysTick incrementa `g_app_tick_cnt` a 1.

1. **Entrada a `app_update()`:**
   * Sección crítica: `g_app_tick_cnt` pasa de 1 a 0. `b_time_update_required = true`.
   * Entra al `while`: `g_app_cnt` pasa de 0 a 1.
   * `g_app_runtime_us = 0`.

2. **Iteración `index = 0` (`task_sensor`):**
   * `cycle_counter_reset()` reinicia el DWT.
   * Ejecuta `task_sensor_update()`. Asumamos una duración real de $t_0$ $\mu\text{s}$.
   * `task_dta_list[0].NOE` pasa de 0 a **1**.
   * `task_dta_list[0].LET` = **$t_0$ $\mu\text{s}$**.
   * Como $t_0 < 1000$, `task_dta_list[0].BCET` pasa de $1000$ a **$t_0$ $\mu\text{s}$**.
   * Como $t_0 > 0$, `task_dta_list[0].WCET` pasa de $0$ a **$t_0$ $\mu\text{s}$**.
   * `g_app_runtime_us` pasa de 0 a **$t_0$ $\mu\text{s}$**.

3. **Iteración `index = 1` (`task_system`):**
   * `cycle_counter_reset()` reinicia el DWT.
   * Ejecuta `task_system_update()`. Asumamos una duración real de $t_1$ $\mu\text{s}$.
   * `task_dta_list[1].NOE` pasa de 0 a **1**.
   * `task_dta_list[1].LET` = **$t_1$ $\mu\text{s}$**.
   * `task_dta_list[1].BCET` pasa de $1000$ a **$t_1$ $\mu\text{s}$**.
   * `task_dta_list[1].WCET` pasa de $0$ a **$t_1$ $\mu\text{s}$**.
   * `g_app_runtime_us` pasa a **$(t_0 + t_1)$ $\mu\text{s}$**.

4. **Iteración `index = 2` (`task_actuator`):**
   * `cycle_counter_reset()` reinicia el DWT.
   * Ejecuta `task_actuator_update()`. Asumamos una duración real de $t_2$ $\mu\text{s}$.
   * `task_dta_list[2].NOE` pasa de 0 a **1**.
   * `task_dta_list[2].LET` = **$t_2$ $\mu\text{s}$**.
   * `task_dta_list[2].BCET` pasa de $1000$ a **$t_2$ $\mu\text{s}$**.
   * `task_dta_list[2].WCET` pasa de $0$ a **$t_2$ $\mu\text{s}$**.
   * `g_app_runtime_us` alcanza el total del ciclo: **$(t_0 + t_1 + t_2)$ $\mu\text{s}$**.

5. **Evaluación final de la iteración:**
   * Revisa `g_app_tick_cnt`. Si no hubo desbordamientos o retrasos masivos, `g_app_tick_cnt == 0`.
   * `b_time_update_required = false`. Finaliza `app_update()`.

---

#### C. Estado Estacionario en N-sucesivas ejecuciones ($N$-ésimo Tick)

En cada nuevo tick ($N = 2, 3, 4, \dots$):
* **`g_app_tick_cnt`**: Típicamente oscila entre **0 y 1**. Se incrementa a 1 en la ISR y vuelve a 0 al ser consumido al inicio de `app_update()`.
* **`index`**: Toma ordenadamente los valores **0, 1 y 2** dentro del bucle interno `for`.
* **`task_dta_list[index].NOE`**: Incrementa en 1 en cada ciclo. Después de $N$ ciclos de ejecución, $\text{NOE} = N$.
* **`task_dta_list[index].LET`**: Actualiza su valor al tiempo exacto que tardó la función `update` en la iteración $N$ ($t_{\text{index}, N}$ $\mu\text{s}$).
* **`task_dta_list[index].BCET`**: Conserva el valor mínimo histórico:
  $$\text{BCET}_{N} = \min(\text{BCET}_{N-1},\, \text{LET}_N)$$
* **`task_dta_list[index].WCET`**: Conserva el valor máximo histórico:
  $$\text{WCET}_{N} = \max(\text{WCET}_{N-1},\, \text{LET}_N)$$
* **`g_app_runtime_us`**: En cada ciclo $N$, adopta el valor de la suma de tiempos de las 3 tareas en ese instante:
  $$\text{g\_app\_runtime\_us}_N = \sum_{i=0}^{2} \text{task\_dta\_list}[i].\text{LET}_N$$

---

## 4. Impacto del Uso de `LOGGER_INFO()` en las Métricas de Tiempo

El uso de la macro `LOGGER_INFO(...)` dentro de las funciones de actualización de las tareas o en el flujo del sistema tiene un impacto crítico e intrusivo sobre el comportamiento temporal del microcontrolador.

### 4.1. Mecanismo de Impacto Interno
Cuando se invoca `LOGGER_INFO()`:
1. **Deshabilitación de Interrupciones:** Inicia con la instrucción ensamblador `CPSID i`, enmascarando todas las interrupciones configurables.
2. **Formateo de Cadenas:** La función `snprintf()` debe parsear la cadena de formato, convertir variables numéricas a texto e iterar sobre buffers de memoria. Esto consume miles de ciclos de reloj.
3. **Mecanismo de Semihosting:** Al estar habilitado `LOGGER_CONFIG_USE_SEMIHOSTING (1)`, la llamada a `printf` activa una instrucción de interrupción por software (como `BKPT 0xAB` en ARM). Esto congela parcialmente la ejecución de la CPU para transferir los caracteres hacia el depurador (*debugger* J-Link / ST-Link) conectado a la PC mediante la interfaz SWD/JTAG.
4. **Habilitación de Interrupciones:** Finaliza con `CPSIE i`.

---

### 4.2. Impacto Específico en `task_dta_list[index].WCET`

1. **Inclusión de Overhead en la Medición:**  
   El temporizador DWT se reinicia justo antes de llamar a `task_update` y se lee inmediatamente después:
   ```c
   cycle_counter_reset();
   (*task_cfg_list[index].task_update)(...);
   task_dta_list[index].LET = cycle_counter_get_time_us();
   ```
   Si la función `task_update` contiene llamadas a `LOGGER_INFO()`, el tiempo medido por el DWT (`LET`) **no reflejará únicamente el tiempo de ejecución del algoritmo propio de la tarea**, sino que incluirá el enorme tiempo extra (*overhead*) que toma formatear e transmitir el mensaje por semihosting.

2. **Distorsión Severa del WCET:**  
   Dado que una sola transmisión vía semihosting o `printf` puede tomar desde cientos de microsegundos hasta **varios milisegundos** (dependiendo del ancho de banda del debugger y de la PC Host), el valor de `LET` registrado será artificialmente gigante. Como `WCET = max(WCET, LET)`, el sistema registrará un **WCET completamente inservible y falseado**, ocultando el verdadero comportamiento del código algorítmico de la tarea.

---

### 4.3. Impacto Específico en `g_app_runtime_us`

1. **Inflación del Tiempo Total del Ciclo:**  
   La variable `g_app_runtime_us` es la suma acumulada de las variables `LET` de cada tarea en la iteración actual:
   $$\text{g\_app\_runtime\_us} = \text{LET}_{sensor} + \text{LET}_{system} + \text{LET}_{actuator}$$
   Si una o más tareas imprimen información mediante `LOGGER_INFO()`, `g_app_runtime_us` aumentará drásticamente en esa iteración, pudiendo superar con facilidad los **1000 $\mu\text{s}$ (1 ms)** del periodo del tick.

2. **Pérdida de Ticks de SysTick (*Tick Accumulation / Jitter*):**  
   Dado que `LOGGER_INFO` deshabilita interrupciones (`CPSID i`) durante su ejecución, si la llamada a `LOGGER_INFO` toma más de 1 ms:
   * La interrupción de SysTick no se atenderá a tiempo.
   * La variable `g_app_tick_cnt` dejará de reflejar un tiempo real preciso y acumulará retrasos.
   * Al finalizar la iteración, el bucle `while (b_time_update_required)` detectará `g_app_tick_cnt > 0` y ejecutará inmediatamente un nuevo ciclo de tareas sin esperar la siguiente ventana de tiempo real, provocando un fallo grave en la determinidad del sistema (*jerkiness* o *timing violation*).

---

## 5. Conclusiones y Recomendaciones

1. **Determinismo y Perfilado:** El esquema DWT + `task_dta_t` proporciona una herramienta de perfilado de bajísimo impacto (pocos ciclos de reloj) perfecta para evaluar el cumplimiento de requerimientos de tiempo real duro o firme (*Hard/Firm Real-Time*).
2. **Efecto Sonda (*Probe Effect*):** La introducción de `LOGGER_INFO()` mediante Semihosting representa un claro ejemplo de *Probe Effect*: alterar el comportamiento temporal del sistema por el hecho mismo de intentar medirlo o depurarlo.
3. **Buenas Prácticas de Depuración:**
   * Desactivar el logger (`LOGGER_CONFIG_ENABLE = 0`) al realizar las mediciones finales de `BCET`, `WCET` y `g_app_runtime_us`.
   * En caso de requerir logs en producción o medición, utilizar técnicas de **Buffered Logging / ITM (Instrumentation Trace Macrocell)** o comunicación por UART vía DMA sin bloqueos ni deshabilitación prolongada de interrupciones.
