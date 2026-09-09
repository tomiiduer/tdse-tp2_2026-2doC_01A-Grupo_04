# Análisis y Explicación del Código Fuente: Sistema Controlado por Eventos (Task System)

**Asignatura:** Técnicas Digitales / Sistemas Embebidos (TDSE)  
**Trabajo Práctico:** TP2 - Arquitectura de Software Orientada a Eventos y Máquinas de Estado Finito (FSM)  
**Archivo Generado:** `tdse-tp2_00-system.md`  

---

## 1. Introducción y Arquitectura General del Sistema

El conjunto de archivos analizados (`task_system_attribute.h`, `task_actuator_attribute.h`, `task_system.c`, `task_system_interface.c` y `task_actuator_interface.c`) implementa una **arquitectura de software embebido no bloqueante basada en eventos y máquinas de estados finitos (FSM - Finite State Machine)**.

Esta arquitectura responde al patrón de diseño de sistemas de tiempo real multitarea cooperativo sin sistema operativo (Bare-Metal Scheduler / Super-Loop), donde los componentes principales son:

1. **Atributos de Datos (`task_*_attribute.h`):** Definen las estructuras de datos, enumeraciones de estados y eventos, e identificadores de las tareas.
2. **Interfaz de Intercomunicación (`task_*_interface.c`):** Proveen mecanismos de desacoplamiento para el intercambio de mensajes/eventos entre tareas mediante una **cola circular FIFO (First-In, First-Out)** o asignación directa de flags de disparo.
3. **Lógica de Control (`task_system.c`):** Implementa el ciclo de vida de la tarea de sistema (`init` y `update`), gestionando las transiciones de estado a través de la función `task_system_normal_statechart()`.

```
[ Productor de Eventos ] 
          │
          ▼  put_event_task_system()
┌─────────────────────────────────────────┐
│ event_task_system_queue (Cola FIFO 16)  │
└────────────────────┬────────────────────┘
                     │  get_event_task_system()
                     ▼
┌─────────────────────────────────────────┐
│ task_system (FSM Control)               │
│ - ST_SYS_IDLE ◄──► ST_SYS_ACTIVE        │
└────────────────────┬────────────────────┘
                     │  put_event_task_actuator()
                     ▼
┌─────────────────────────────────────────┐
│ task_actuator (LED Actuator)            │
│ - EV_LED_IDLE / EV_LED_ACTIVE           │
└─────────────────────────────────────────┘
```

---

## 2. Descripción Detallada de los Archivos

### 2.1. `task_system_attribute.h`
Define los tipos de datos fundamentales para la tarea del sistema:
- **`task_system_ev_t` (Enumeración de Eventos):**
  - `EV_SYS_IDLE` (0): Evento para pasar/permanecer en estado inactivo.
  - `EV_SYS_ACTIVE` (1): Evento para activar la lógica del sistema.
- **`task_system_st_t` (Enumeración de Estados):**
  - `ST_SYS_IDLE` (0): Estado de reposo / inactivo.
  - `ST_SYS_ACTIVE` (1): Estado activo / operacional.
- **Estructura `task_system_dta_t`:**
  - `uint32_t tick`: Temporizador o contador de ticks de la tarea.
  - `task_system_st_t state`: Estado actual de la FSM del sistema.
  - `task_system_ev_t event`: Último evento recibido para ser procesado por la FSM.
  - `bool flag`: Bandera booleana que indica la presencia de un nuevo evento no procesado (`true`) o consumido (`false`).
- Declara externamente el arreglo global `task_system_dta_list[]`.

### 2.2. `task_actuator_attribute.h`
Define las estructuras y tipos necesarios para la tarea encargada de controlar el actuador (en este caso, un LED):
- **`task_actuator_ev_t`:** Eventos para el actuador (`EV_LED_IDLE = 0`, `EV_LED_ACTIVE = 1`).
- **`task_actuator_st_t`:** Estados del actuador (`ST_LED_IDLE = 0`, `ST_LED_ACTIVE = 1`).
- **`task_actuator_id_t`:** Identificador del actuador (`ID_LED_A = 0`).
- **`task_actuator_cfg_t`:** Configuración de hardware (puerto GPIO, pin, estados activo/pasivo, temporizaciones).
- **`task_actuator_dta_t`:** Estructura de estado dinámica del actuador (`tick`, `state`, `event`, `flag`).

### 2.3. `task_system_interface.c`
Implementa la gestión de la cola circular FIFO de eventos para la tarea del sistema:
- **Estructura `event_task_system_queue_t`:**
  - `head`: Índice de inserción de eventos.
  - `tail`: Índice de extracción de eventos.
  - `count`: Cantidad actual de eventos en cola.
  - `queue[16]`: Arreglo de eventos con longitud fija `QUEUE_LENGTH = 16`. Valor nulo configurado como `EMPTY = 255` (`0xFF`).
- **Funciones de Interfaz:**
  - `init_event_task_system()`: Inicializa `head = 0`, `tail = 0`, `count = 0` y llena el buffer con `EMPTY`.
  - `put_event_task_system(event)`: Inserta un evento en la posición `head`, incrementa `count` y actualiza `head` de manera circular (`head % 16`).
  - `get_event_task_system()`: Extrae el evento en la posición `tail`, resetea la posición a `EMPTY`, decrementa `count` y actualiza `tail` de manera circular.
  - `any_event_task_system()`: Retorna `true` si `head != tail` (existen eventos pendientes).

### 2.4. `task_actuator_interface.c`
Proporciona la función de interfaz para enviar eventos directamente a la tarea del actuador:
- **`put_event_task_actuator(event, identifier)`:**
  - Obtiene el puntero a los datos del actuador indicado por `identifier`.
  - Asigna `p_task_actuator_dta->event = event`.
  - Establece `p_task_actuator_dta->flag = true` para indicarle a la FSM del actuador que tiene un evento pendiente de atender.

### 2.5. `task_system.c`
Contiene la implementación principal de la tarea del sistema:
- **`task_system_init(parameters)`:**
  - Inicializa la cola de eventos llamando a `init_event_task_system()`.
  - Configura el modo de trabajo mediante `task_system_set_mode(NORMAL)`.
  - Recorre el arreglo `task_system_dta_list[]` e inicializa cada instancia en `ST_SYS_IDLE`, evento `EV_SYS_IDLE` y `flag = false`.
- **`task_system_update(parameters)`:**
  - Ejecutado de forma periódica en el loop principal. Invoca la función del statechart correspondiente al modo activo (`NORMAL` -> `task_system_normal_statechart()`).
- **`task_system_normal_statechart()` / `task_system_statechart()`:**
  - Evalúa la cola de eventos. Si hay eventos disponibles, los extrae y actualiza `p_task_system_dta->event` y pone `flag = true`.
  - Aplica la lógica de transiciones de estados según el valor actual de `state`, `flag` y `event`.
  - Notifica al actuador invocando `put_event_task_actuator()`.

---

## 3. Evolución de las Variables de la Tarea de Sistema (`task_system_dta_list[index]`)

En la aplicación, la macro `SYSTEM_DTA_QTY` vale `1` (definida como `MODE_QTY`), por lo cual existe un único elemento en la lista, donde **`index = 0`** (correspondiente al modo `NORMAL`).

### 3.1. Unidad de Medida de `tick`
* **Unidad de Medida:** **Milisegundos [mS]** (o ticks del temporizador del sistema / Systick de la plataforma STM32).
* **Fundamento:** En los encabezados de `task_system.c` se especifica: `Update by Time Code, period = 1mS`. Además, los macros del sistema (`DEL_SYS_MIN = 0ul`, `DEL_SYS_MED = 250ul`, `DEL_SYS_MAX = 500ul`) definen retardos en milisegundos.

### 3.2. Tabla de Evolución Temporal de Variables de `task_system_dta_list[0]`

| Instante / Evento de Ejecución | `index` | `.tick` [mS] | `.state` | `.event` | `.flag` | Explicación / Descripción del Cambio |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Paso 0: Antes de Iniciar** | 0 | Uninitialized (0*) | Uninitialized | Uninitialized | Uninitialized | Memoria global sin inicializar (se limpia a 0 en BSS). |
| **Paso 1: `task_system_init()`** | 0 | 0 | `ST_SYS_IDLE` (0) | `EV_SYS_IDLE` (0) | `false` (0) | Bucle de inicialización en `task_system_init()`. |
| **Paso 2: Loop `task_system_update()` (Sin eventos en cola)** | 0 | 0 | `ST_SYS_IDLE` (0) | `EV_SYS_IDLE` (0) | `false` (0) | `any_event_task_system()` retorna `false`. FSM evalúa `ST_SYS_IDLE` pero `flag` es `false`, no hay cambio de estado. |
| **Paso 3: Recepción de `EV_SYS_ACTIVE`** | 0 | 0 | `ST_SYS_IDLE` (0) | `EV_SYS_ACTIVE` (1) | `true` (1) | `task_system_normal_statechart()` detecta evento en cola, llama a `get_event_task_system()`, asigna `event = EV_SYS_ACTIVE` y `flag = true`. |
| **Paso 4: Procesamiento de `EV_SYS_ACTIVE` (mismo `update`)** | 0 | 0 | **`ST_SYS_ACTIVE`** (1) | `EV_SYS_ACTIVE` (1) | **`false`** (0) | FSM en `ST_SYS_IDLE` detecta `flag == true` y `event == EV_SYS_ACTIVE`. Consume flag (`false`), envía `EV_LED_ACTIVE` a actuador y conmuta estado a `ST_SYS_ACTIVE`. |
| **Paso 5: Loop `task_system_update()` (Estado Activo Estable)** | 0 | 0 | `ST_SYS_ACTIVE` (1) | `EV_SYS_ACTIVE` (1) | `false` (0) | No hay eventos en cola. Permanece en `ST_SYS_ACTIVE`. |
| **Paso 6: Recepción de `EV_SYS_IDLE`** | 0 | 0 | `ST_SYS_ACTIVE` (1) | `EV_SYS_IDLE` (0) | `true` (1) | Detecta evento en cola, lee `EV_SYS_IDLE` y setea `flag = true`. |
| **Paso 7: Procesamiento de `EV_SYS_IDLE` (mismo `update`)** | 0 | 0 | **`ST_SYS_IDLE`** (0) | `EV_SYS_IDLE` (0) | **`false`** (0) | FSM en `ST_SYS_ACTIVE` detecta `flag == true` y `event == EV_SYS_IDLE`. Consume flag (`false`), envía `EV_LED_IDLE` a actuador y vuelve a `ST_SYS_IDLE`. |
| **Paso Anómalo: Estado inválido (`default`)** | 0 | **`DEL_SYS_MIN` (0)** | `ST_SYS_IDLE` (0) | `EV_SYS_IDLE` (0) | `false` (0) | Si el estado no corresponde a `ST_SYS_IDLE` ni `ST_SYS_ACTIVE`, entra en `default` forzando reinicio de variables. |

---

## 4. Comportamiento de la Función `task_system_statechart()` / `task_system_normal_statechart()`

En el código analizado, la función encargada de ejecutar la máquina de estados es `task_system_normal_statechart(void)`, la cual opera internamente sobre el elemento `index = NORMAL` (`0`) del arreglo `task_system_dta_list`. Si conceptualmente la función se parametriza como `void task_system_statechart(uint32_t index)`, su comportamiento es el siguiente:

### 4.1. Algoritmo Paso a Paso
1. **Extracción y Recepción de Eventos:**
   - La función consulta a la cola FIFO invocando `any_event_task_system()`.
   - Si existen eventos pendientes (`true`), extrae el evento más antiguo mediante `get_event_task_system()`.
   - Asigna dicho evento al atributo `.event` del puntero de datos de la tarea y activa la bandera `.flag = true`.

2. **Evaluación de la Máquina de Estados (Switch Case):**
   - **Caso `ST_SYS_IDLE` (0):**
     - Evalúa la guarda: `(flag == true) && (event == EV_SYS_ACTIVE)`.
     - Si se cumple:
       1. Limpia la bandera: `p_task_system_dta->flag = false`.
       2. Emite la orden al actuador: `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)`.
       3. Transiciona al nuevo estado: `p_task_system_dta->state = ST_SYS_ACTIVE`.
   - **Caso `ST_SYS_ACTIVE` (1):**
     - Evalúa la guarda: `(flag == true) && (event == EV_SYS_IDLE)`.
     - Si se cumple:
       1. Limpia la bandera: `p_task_system_dta->flag = false`.
       2. Emite la orden al actuador: `put_event_task_actuator(EV_LED_IDLE, ID_LED_A)`.
       3. Transiciona al nuevo estado: `p_task_system_dta->state = ST_SYS_IDLE`.
   - **Caso `default` (Tratamiento de Fallas / Robustez):**
     - Se ejecuta ante corrupción de memoria o estados no contemplados.
     - Restablece los atributos del sistema a su condición segura por defecto:
       - `tick = DEL_SYS_MIN` (`0`)
       - `state = ST_SYS_IDLE`
       - `event = EV_SYS_IDLE`
       - `flag = false`

---

## 5. Evolución de las Variables de la Cola Circular (`event_task_system_queue`)

La cola de eventos utiliza una estructura de buffer circular indexada por `head` (escritura) y `tail` (lectura), con capacidad para `QUEUE_LENGTH = 16` elementos.

### 5.1. Definición de Variables de la Cola
* `i`: Variable local empleada en bucles de iteración (p. ej. inicialización de 0 a 15).
* `event_task_system_queue.head`: Apunta a la siguiente posición libre donde se insertará un evento (`put_event_task_system`).
* `event_task_system_queue.tail`: Apunta a la posición del evento más antiguo a ser extraído (`get_event_task_system`).
* `event_task_system_queue.count`: Indica la cantidad total de elementos almacenados en la cola en un instante dado ($0 \le 	ext{count} \le 16$).
* `event_task_system_queue.queue[i]`: Arreglo de 16 posiciones que contiene los eventos almacenados. El valor `255` (`EMPTY`) indica celda vacía.

### 5.2. Tabla de Evolución en Inicialización, Encolado y Descolado

| Etapa / Acción | `i` | `.head` | `.tail` | `.count` | `queue[0]` | `queue[1]` | `queue[2..15]` | Explicación Detallada |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **Inicio: `init_event_task_system()`** | 0..15 | **0** | **0** | **0** | **255** | **255** | **255** | Bucle limpia el vector poniendo todas las celdas en `EMPTY` (`255`). Índices y contador a cero. |
| **Evento 1: Llega `EV_SYS_ACTIVE` (`put`)** | N/A | **1** | **0** | **1** | **`EV_SYS_ACTIVE` (1)** | 255 | 255 | Se incrementa `.count`. Se escribe en `queue[0]`. `head` avanza a 1. |
| **Ejecución `task_system_update()` (`get`)** | N/A | **1** | **1** | **0** | **255 (EMPTY)** | 255 | 255 | `any_event...()` retorna `true`. Se extrae de `queue[0]`, se pisa con `EMPTY`, decrementa `.count` a 0 y `tail` avanza a 1. |
| **Evento 2: Llega `EV_SYS_IDLE` (`put`)** | N/A | **2** | **1** | **1** | 255 | **`EV_SYS_IDLE` (0)** | 255 | Se escribe en `queue[1]`. `.count` sube a 1. `head` avanza a 2. |
| **Ejecución `task_system_update()` (`get`)** | N/A | **2** | **2** | **0** | 255 | **255 (EMPTY)** | 255 | Se lee `queue[1]`, se resetea a `EMPTY`, `.count` baja a 0 y `tail` avanza a 2. |
| **Encolado Masivo Hasta Llenar Cola (16 ops)** | N/A | **0** | **0** | **16** | $EV_x$ | $EV_y$ | $EV_z$ | Si no se descola, `head` incrementa hasta 15 y luego vuelve a 0 (wrap-around). `.count` alcanza 16. |

---

## 6. Evolución de las Variables de la Tarea Actuador (`task_actuator_dta_list[identifier]`)

La tarea de sistema interactúa directamente con el actuador a través del archivo `task_actuator_interface.c` mediante la función `put_event_task_actuator()`. El identificador utilizado en el sistema es `identifier = ID_LED_A` (con valor numérico `0`).

### 6.1. Definición de Variables del Actuador
* `identifier`: Constante de enumeración `ID_LED_A` (0) que indexa el elemento dentro de `task_actuator_dta_list[]`.
* `task_actuator_dta_list[ID_LED_A].event`: Almacena el último evento transmitido por la tarea de sistema (`EV_LED_IDLE = 0` o `EV_LED_ACTIVE = 1`).
* `task_actuator_dta_list[ID_LED_A].flag`: Indica a la FSM del actuador que existe una actualización de evento pendiente de procesamiento (`true`).

### 6.2. Tabla de Evolución Temporal de Variables del Actuador

| Instante / Disparo de Evento | `identifier` | `task_actuator_dta_list[0].event` | `task_actuator_dta_list[0].flag` | Explicación / Causa de la Modificación |
| :--- | :---: | :---: | :---: | :--- |
| **Estado Inicial (Power-On / Init)** | `ID_LED_A` (0) | `EV_LED_IDLE` (0) | `false` (0) | Estado inicial por defecto de la estructura de datos del actuador. |
| **Transición Sistema: IDLE ➔ ACTIVE** | `ID_LED_A` (0) | **`EV_LED_ACTIVE` (1)** | **`true` (1)** | La FSM del sistema ejecuta `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)`. Asigna el evento y levanta el flag. |
| **Atención en `task_actuator_update()`** | `ID_LED_A` (0) | `EV_LED_ACTIVE` (1) | **`false` (0)** | La FSM del actuador procesa la orden, realiza el encendido del LED vía GPIO y consume el flag (`false`). |
| **Transición Sistema: ACTIVE ➔ IDLE** | `ID_LED_A` (0) | **`EV_LED_IDLE` (0)** | **`true` (1)** | La FSM del sistema ejecuta `put_event_task_actuator(EV_LED_IDLE, ID_LED_A)`. Actualiza evento a IDLE y vuelve a setear el flag a `true`. |
| **Atención en `task_actuator_update()`** | `ID_LED_A` (0) | `EV_LED_IDLE` (0) | **`false` (0)** | La FSM del actuador procesa la orden, apaga el LED y limpia el flag a `false`. |

---

## 7. Conclusión y Consideraciones de Diseño

1. **Desacoplamiento Efectivo:** La arquitectura implementada logra separar completamente la detección y generación de eventos de la lógica de procesamiento (FSM) y de los drivers de salida (actuadores).
2. **Determinismo y Código No Bloqueante:** Ninguna función contiene bucles de espera pasiva (`while(1)` o retardos bloqueantes tipo `delay()`). Esto garantiza que el loop principal se ejecute periódicamente con bajo jitter y alta capacidad de respuesta.
3. **Manejo Seguro de Colas Circular:** La cola `event_task_system_queue` previene sobreescrituras indebidas mediante el índice `count` y la operación de wrap-around en `QUEUE_LENGTH` (`16`), garantizando una comunicación FIFO confiable.
4. **Resiliencia:** El uso del estado `default` en la FSM resguarda al microntrolador ante eventuales corrupciones de memoria o transiciones no permitidas, forzando la reinicialización segura del módulo.
