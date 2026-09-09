# Análisis y Explicación de Código Fuente: Módulo Sensor y Cola de Eventos del Sistema

Este documento contiene el análisis detallado y la explicación del funcionamiento del sistema de captura de entradas (sensores/botones) y comunicación por eventos mediante una cola FIFO circular, correspondiente a los archivos fuente:
* `task_sensor_attribute.h`
* `task_system_attribute.h` / `task_system_interface.h`
* `task_sensor.c`
* `task_system_interface.c`

---

## 1. Arquitectura y Estructura del Módulo 

El sistema implementa un modelo de diseño **no bloqueante orientado a eventos** impulsado por tiempo (*Update by Time Code*, con período típico de $1\text{ ms}$).

### Componentes Clave:
1. **Atributos de Configuración y Estado (`task_sensor_attribute.h`)**:
   - **`task_sensor_cfg_t`**: Define los parámetros estáticos de hardware de cada sensor/botón: puerto/pin GPIO, nivel lógico activo (`pressed`), tiempos límite (`tick_max`), y las señales/eventos del sistema que deben generarse al presionar (`signal_down`) o soltar (`signal_up`).
   - **`task_sensor_dta_t`**: Guarda el estado dinámico del sensor en tiempo de ejecución: `tick` (contador temporal), `state` (estado de la máquina de estados) y `event` (evento actual detectado).

2. **Interfaz de Eventos del Sistema (`task_system_interface.h` y `task_system_interface.c`)**:
   - Implementa una cola FIFO circular denominada `event_task_system_queue` de tamaño fijo (`QUEUE_LENGTH = 16`).
   - Permite encolar (`put_event_task_system`), desencolar (`get_event_task_system`), e inspeccionar la presencia de eventos (`any_event_task_system`).

3. **Lógica de Control de Sensores (`task_sensor.c`)**:
   - Inicializa los punteros y valores por defecto mediante `task_sensor_init()`.
   - Ejecuta periódicamente la función `task_sensor_update()`, la cual itera sobre todos los sensores registrados invocando `task_sensor_statechart(index)`.

---

## 2. Evolución de Variables del Sensor (`task_sensor_dta_list`)

### Unidad de Medida del Campo `tick`
* **Unidad de medida:** **Milisegundos ($	ext{ms}$)**.
* **Explicación:** Está asociado al tick del sistema (`HAL_GetTick()`) y al período de actualización de la tarea ($1\text{ ms}$). Se utiliza para conteos de temporización y desrebote (*debouncing*).

### Tabla de Evolución Temporal de Variables (`task_sensor_dta_list[index]`)
Para el sensor configurado en `index = 0` (`ID_BTN_A`):
* `signal_down` = `EV_SYS_IDLE`
* `signal_up` = `EV_SYS_ACTIVE`

| Instancia / Evento de Ejecución | `index` | `tick` (ms) | `state` | `event` | Descripción del Comportamiento / Transición |
| :--- | :---: | :---: | :--- | :--- | :--- |
| **Inicio (`task_sensor_init()`)** | 0 | 0 | `ST_BTN_IDLE` (0) | `EV_BTN_UP` (0) | Se inicializa la estructura de datos del sensor. Estado inicial en reposo, evento inicial en botón suelto. |
| **Iteración en Reposo (`task_sensor_update()`)** | 0 | 0 | `ST_BTN_IDLE` (0) | `EV_BTN_UP` (0) | `HAL_GPIO_ReadPin` detecta que el botón no está presionado. La FSM permanece en estado `ST_BTN_IDLE`. |
| **Transición por Presión (`task_sensor_update()`)** | 0 | 0 | `ST_BTN_ACTIVE` (1) | `EV_BTN_DOWN` (1) | `HAL_GPIO_ReadPin` detecta nivel de presión (`pressed`). La FSM emite `EV_SYS_IDLE` a la cola del sistema y pasa a `ST_BTN_ACTIVE`. |
| **Mantenimiento Presionado (`task_sensor_update()`)** | 0 | 0 | `ST_BTN_ACTIVE` (1) | `EV_BTN_DOWN` (1) | El botón continúa presionado. La FSM se mantiene en `ST_BTN_ACTIVE` sin emitir nuevos eventos. |
| **Transición por Liberación (`task_sensor_update()`)** | 0 | 0 | `ST_BTN_IDLE` (0) | `EV_BTN_UP` (0) | `HAL_GPIO_ReadPin` detecta que se soltó el botón. La FSM emite `EV_SYS_ACTIVE` a la cola del sistema y regresa a `ST_BTN_IDLE`. |
| **Caso `default` (Recuperación de Error)** | 0 | 0 (`DEL_BTN_MIN`) | `ST_BTN_IDLE` (0) | `EV_BTN_UP` (0) | Si la FSM entra en un estado no válido, restablece `tick = DEL_BTN_MIN` (0), `state = ST_BTN_IDLE` y `event = EV_BTN_UP`. |

---

## 3. Comportamiento de la Función `task_sensor_statechart(uint32_t index)`

La función `task_sensor_statechart` ejecuta la lógica de la máquina de estados finitos (FSM) para la instancia especificada por `index`:

1. **Lectura de Entrada (Mapeo Hardware a Evento):**
   * Obtiene la configuración `p_task_sensor_cfg` y los datos de estado `p_task_sensor_dta` correspondientes al `index`.
   * Lee el estado del pin físico mediante `HAL_GPIO_ReadPin(p_task_sensor_cfg->gpio_port, p_task_sensor_cfg->pin)`.
   * Si la lectura coincide con `p_task_sensor_cfg->pressed`, se asigna el evento local `EV_BTN_DOWN`.
   * En caso contrario, se asigna el evento local `EV_BTN_UP`.

2. **Evaluación de la Máquina de Estados (`switch (p_task_sensor_dta->state)`):**
   * **Estado `ST_BTN_IDLE`:**
     - Si el evento evaluado es `EV_BTN_DOWN`, significa que se acaba de presionar el botón.
     - Llama a `put_event_task_system(p_task_sensor_cfg->signal_down)` para registrar la señal en la cola global del sistema.
     - Transiciona el estado a `ST_BTN_ACTIVE`.
   * **Estado `ST_BTN_ACTIVE`:**
     - Si el evento evaluado es `EV_BTN_UP`, significa que se acaba de liberar el botón.
     - Llama a `put_event_task_system(p_task_sensor_cfg->signal_up)` para enviar la señal correspondiente a la cola.
     - Transiciona el estado a `ST_BTN_IDLE`.
   * **Rama `default`:**
     - Proporciona tolerancia a fallos ante la corrupción de memoria o estados inválidos.
     - Resetea `p_task_sensor_dta->tick = DEL_BTN_MIN` (0).
     - Fuerza el estado a `ST_BTN_IDLE` y el evento a `EV_BTN_UP`.

---

## 4. Evolución de las Variables de la Cola Circular (`event_task_system_queue`)

La estructura `event_task_system_queue_t` gestiona un buffer circular para evitar desplazamientos de memoria. Parámetros:
* `QUEUE_LENGTH = 16`
* Valor de celda vacía: `EMPTY = 255`

### Evolución de los Punteros, Contador y Buffer de la Cola:

| Etapa de Ejecución / Función Invocada | `head` | `tail` | `count` | `queue[0]` | `queue[1]` | `queue[2]` | `queue[3..15]` |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Inicialización (`init_event_task_system`)** | 0 | 0 | 0 | 255 (`EMPTY`) | 255 (`EMPTY`) | 255 (`EMPTY`) | 255 (`EMPTY`) |
| **Botón presionado 1.ª vez (`put_event_task_system(EV_SYS_IDLE)`)** | 1 | 0 | 1 | `EV_SYS_IDLE` | 255 (`EMPTY`) | 255 (`EMPTY`) | 255 (`EMPTY`) |
| **Botón liberado 1.ª vez (`put_event_task_system(EV_SYS_ACTIVE)`)** | 2 | 0 | 2 | `EV_SYS_IDLE` | `EV_SYS_ACTIVE` | 255 (`EMPTY`) | 255 (`EMPTY`) |
| **Lectura de 1.er evento (`get_event_task_system()`)** | 2 | 1 | 1 | 255 (`EMPTY`) | `EV_SYS_ACTIVE` | 255 (`EMPTY`) | 255 (`EMPTY`) |
| **Lectura de 2.º evento (`get_event_task_system()`)** | 2 | 2 | 0 | 255 (`EMPTY`) | 255 (`EMPTY`) | 255 (`EMPTY`) | 255 (`EMPTY`) |
| **Uso continuo hasta la posición 15** | 15 | 15 | 0 | ... | ... | ... | 255 (`EMPTY`) |
| **Encolado en extremo superior (`head = 15 -> 0`)** | 0 | 15 | 1 | 255 (`EMPTY`) | ... | ... | Evento |

### Detalle Mecánico de las Funciones de la Cola:
1. **`init_event_task_system()`**:
   * Pone `head = 0`, `tail = 0`, `count = 0`.
   * Rellena todas las 16 posiciones de `queue` con `EMPTY` (`255`).
2. **`put_event_task_system(event)`**:
   * Incrementa `count`.
   * Guarda el evento en `queue[head]` e incrementa `head`.
   * Si `head == QUEUE_LENGTH` (16), reinicia `head = 0` (comportamiento circular).
3. **`get_event_task_system()`**:
   * Decrementa `count`.
   * Extrae el evento de `queue[tail]`.
   * Marca la posición extraída como `EMPTY` (`255`) e incrementa `tail`.
   * Si `tail == QUEUE_LENGTH` (16), reinicia `tail = 0` (comportamiento circular).
4. **`any_event_task_system()`**:
   * Retorna `true` si `head != tail` (hay eventos pendientes), o `false` si la cola está vacía (`head == tail`).
