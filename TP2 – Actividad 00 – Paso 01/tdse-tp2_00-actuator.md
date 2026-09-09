# Análisis y Explicación del Código Fuente: Módulo Actuador (Task Actuator)

**Asignatura:** Técnicas de Diseño de Sistemas Embebidos (TDSE)  
**Trabajo Práctico:** TP2 - Actuador (`tdse-tp2_00-actuator`)  
**Autor del Código Analizado:** Juan Manuel Cruz (<jcruz@fi.uba.ar> / <jcruz@frba.utn.edu.ar>)  

---

## 1. Introducción y Arquitectura General del Módulo

El módulo **Task Actuator** está diseñado para la gestión y control no bloqueante de actuadores discretos (en este caso, un diodo emisor de luz o **LED**) en un sistema embebido basado en microcontrolador (arquitectura STM32 utilizando la biblioteca HAL).

El diseño implementa una **Máquina de Estados Finitos (FSM)** orientada a eventos (*Event-Driven Statechart*) y sigue el patrón de diseño de **módulos no bloqueantes actualizados por tiempo** (*Update by Time Code* / *Time-Triggered Non-Blocking Architecture*).

### Estructura de Archivos
El módulo se divide en tres archivos con responsabilidades claramente delimitadas:
1. **`task_actuator_attribute.h`**: Archivo de cabecera que define los tipos de datos, enumeraciones de estados y eventos, y las estructuras de configuración estática y datos dinámicos.
2. **`task_actuator.c`**: Implementación principal que contiene la tabla de configuración local, el arreglo de datos dinámicos en RAM, las funciones de inicialización (`task_actuator_init`), actualización periódica (`task_actuator_update`) y la lógica del Statechart (`task_actuator_statechart`).
3. **`task_actuator_interface.c`**: Proporciona la interfaz de comunicación entre tareas (`put_event_task_actuator`), permitiendo a otros módulos de la aplicación (por ejemplo, tareas de pulsador o supervisores) enviar eventos al actuador de forma segura y desacoplada.

---

## 2. Análisis Detallado de los Archivos de Código Fuente

### 2.1. `task_actuator_attribute.h`

Este archivo encapsula las definiciones abstractas de datos y tipos que estructuran la tarea:

* **Enumeración de Eventos (`task_actuator_ev_t`)**:
  ```c
  typedef enum task_actuator_ev {
      EV_LED_IDLE,    /* Evento para solicitar apagar / pasar a reposo */
      EV_LED_ACTIVE   /* Evento para solicitar encender / pasar a activo */
  } task_actuator_ev_t;
  ```
* **Enumeración de Estados (`task_actuator_st_t`)**:
  ```c
  typedef enum task_actuator_st {
      ST_LED_IDLE,    /* Estado de reposo (LED Apagado) */
      ST_LED_ACTIVE   /* Estado activo (LED Encendido) */
  } task_actuator_st_t;
  ```
* **Enumeración de Identificadores (`task_actuator_id_t`)**:
  ```c
  typedef enum task_actuator_id {
      ID_LED_A        /* Identificador único del actuador LED A */
  } task_actuator_id_t;
  ```
* **Estructura de Configuración Estática (`task_actuator_cfg_t`)**:
  Almacena los parámetros constantes fijados en tiempo de compilación (generalmente ubicados en memoria Flash):
  * `identifier`: Identificador del actuador (`ID_LED_A`).
  * `gpio_port`: Puntero al puerto GPIO del hardware (`GPIO_TypeDef *`).
  * `pin`: Número de pin del puerto GPIO (`uint16_t`).
  * `led_on`: Estado lógico para encender el LED (`GPIO_PIN_SET` o `GPIO_PIN_RESET`).
  * `led_off`: Estado lógico para apagar el LED.
  * `tick_max`: Tiempo máximo / retardo asociado (en milisegundos).
* **Estructura de Datos Dinámicos (`task_actuator_dta_t`)**:
  Almacena el estado en tiempo de ejecución de cada actuador (ubicado en memoria RAM):
  * `tick`: Contador de tiempo acumulado o ticks de temporización.
  * `state`: Estado actual de la máquina de estados (`ST_LED_IDLE` o `ST_LED_ACTIVE`).
  * `event`: Último evento recibido pendiente de ser procesado.
  * `flag`: Flag/Bandera booleana que indica la presencia de un evento no consumido (`true` = evento pendiente, `false` = sin evento).

---

### 2.2. `task_actuator.c`

Contiene la lógica de ejecución del actuador y la máquina de estados.

* **Constantes y Macros**:
  * `DEL_LED_MIN` ($0	ext{ ms}$), `DEL_LED_MED` ($250	ext{ ms}$), `DEL_LED_MAX` ($500	ext{ ms}$).
  * `ACTUATOR_CFG_QTY`: Cantidad total de actuadores configurados (calculado mediante `sizeof`).
  * `task_actuator_cfg_list[]`: Tabla constante con la configuración de hardware para `ID_LED_A`.
  * `task_actuator_dta_list[]`: Arreglo en RAM que guarda el estado de cada actuador.

* **`task_actuator_init(void *parameters)`**:
  * Es la función de inicialización llamada al arranque del sistema.
  * Recorre el arreglo de actuadores (`index` de `0` a `ACTUATOR_DTA_QTY - 1`).
  * Para cada actuador:
    1. Asigna el estado inicial: `p_task_actuator_dta->state = ST_LED_IDLE`.
    2. Asigna el evento inicial: `p_task_actuator_dta->event = EV_LED_IDLE`.
    3. Asigna la bandera de evento: `p_task_actuator_dta->flag = false`.
    4. Garantiza el estado físico seguro aparente escribiendo en el GPIO: `HAL_GPIO_WritePin(..., led_off)`.
    5. Imprime logs informativos mediante la macro `LOGGER_INFO`.

* **`task_actuator_update(void *parameters)`**:
  * Es la función ejecutada periódicamente en el loop principal (período típico $1	ext{ ms}$).
  * Recorre todos los actuadores e invoca a `task_actuator_statechart(index)` para ejecutar la transición de estados correspondiente.

---

### 2.3. `task_actuator_interface.c`

Define la API pública para inyectar eventos a los actuadores desde otras capas de la aplicación:

* **`put_event_task_actuator(task_actuator_ev_t event, task_actuator_id_t identifier)`**:
  * Recibe como parámetros el evento deseado (`event`) y el identificador del actuador objetivo (`identifier`).
  * Indexa el arreglo dinámico `task_actuator_dta_list[identifier]`.
  * Almacena el evento: `p_task_actuator_dta->event = event`.
  * Activa la bandera de evento: `p_task_actuator_dta->flag = true`.

---

## 3. Comportamiento de la Función `task_actuator_statechart(uint32_t index)`

La función `task_actuator_statechart(uint32_t index)` evalúa y ejecuta la máquina de estados finitos no bloqueante para el actuador especificado por `index`.

### Diagrama de Transición de Estados

```
              +-------------------------------------------------------+
              |                                                       |
              |             +---------------------------+             |
              |             |        ST_LED_IDLE        |             |
              |             |  (LED Físico: APAGADO)    |             |
              |             +-------------+-------------+             |
              |                           |                           |
              |                           | flag == true &&           |
              |                           | event == EV_LED_ACTIVE    |
              |                           |                           |
              |                           v                           |
              |             +---------------------------+             |
              |             |       ST_LED_ACTIVE       |             |
              |             |  (LED Físico: ENCENDIDO)  |             |
              |             +-------------+-------------+             |
              |                           |                           |
              |                           | flag == true &&           |
              |                           | event == EV_LED_IDLE      |
              |                           |                           |
              |                           +---------------------------+
              |                                                       |
              +-------------------------------------------------------+
                                  (Estado default)
```

### Lógica Interna del `switch-case`

1. **Estado `ST_LED_IDLE`**:
   * **Condición de Transición**: Se evalúa `(true == p_task_actuator_dta->flag) && (EV_LED_ACTIVE == p_task_actuator_dta->event)`.
   * **Acciones al cumplir la condición**:
     1. Desactiva la bandera: `p_task_actuator_dta->flag = false` (consume el evento).
     2. Enciende el hardware del LED: `HAL_GPIO_WritePin(..., led_on)`.
     3. Cambia al estado activo: `p_task_actuator_dta->state = ST_LED_ACTIVE`.
   * **Si la condición no se cumple**: Permanece en `ST_LED_IDLE` sin realizar cambios en el hardware ni en la bandera.

2. **Estado `ST_LED_ACTIVE`**:
   * **Condición de Transición**: Se evalúa `(true == p_task_actuator_dta->flag) && (EV_LED_IDLE == p_task_actuator_dta->event)`.
   * **Acciones al cumplir la condición**:
     1. Desactiva la bandera: `p_task_actuator_dta->flag = false` (consume el evento).
     2. Apaga el hardware del LED: `HAL_GPIO_WritePin(..., led_off)`.
     3. Cambia al estado de reposo: `p_task_actuator_dta->state = ST_LED_IDLE`.
   * **Si la condición no se cumple**: Permanece en `ST_LED_ACTIVE` sin modificar nada.

3. **Caso `default` (Manejo de Errores / Estados Inválidos)**:
   * Si por algún motivo de falla de memoria la variable `state` toma un valor fuera de la enumeración definida:
     1. Reinicia el contador de tiempo: `p_task_actuator_dta->tick = DEL_LED_MIN` ($0$).
     2. Restablece el estado: `p_task_actuator_dta->state = ST_LED_IDLE`.
     3. Restablece el evento: `p_task_actuator_dta->event = EV_LED_IDLE`.
     4. Limpia la bandera: `p_task_actuator_dta->flag = false`.

---

## 4. Evolución de Variables Internas durante la Ejecución

### Variables Analizadas:
* `index`: Índice numérico del actuador en la lista (para `ID_LED_A`, `index = 0`).
* `task_actuator_dta_list[index].tick`: Contador de tiempo acumulado.  
  * **Unidad de medida**: **Milisegundos ($	ext{mS}$)**, correspondiente a la periodicidad del tick de temporización del sistema (`HAL_GetTick()` / $1	ext{ ms}$).
* `task_actuator_dta_list[index].state`: Estado actual del actuador (`ST_LED_IDLE` / `ST_LED_ACTIVE`).
* `task_actuator_dta_list[index].event`: Evento registrado (`EV_LED_IDLE` / `EV_LED_ACTIVE`).
* `task_actuator_dta_list[index].flag`: Estado de evento pendiente (`false` / `true`).

---

### Tabla de Evolución Temporal Paso a Paso

| Evento / Instante de Ejecución | `index` | `.tick` [mS] | `.state` | `.event` | `.flag` | Salida Hardware (LED A) | Descripción de la Acción / Operación |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **Valor Inicial (Sección BSS)** | `0` | `0` | `0` (`ST_LED_IDLE`) | `0` (`EV_LED_IDLE`) | `false` | Indefinido | Memoria RAM limpia antes de iniciar código C. |
| **`task_actuator_init()`** | `0` | `0` | `ST_LED_IDLE` | `EV_LED_IDLE` | `false` | **OFF** (`led_off`) | Inicialización explícita y apague preventivo del LED. |
| **`task_actuator_update()` (Loop #1)** | `0` | `0` | `ST_LED_IDLE` | `EV_LED_IDLE` | `false` | **OFF** | `flag` es `false`. No hay transición de estado. |
| **`task_actuator_update()` (Loop #2)** | `0` | `0` | `ST_LED_IDLE` | `EV_LED_IDLE` | `false` | **OFF** | Reposo continuo, aguardando evento. |
| **Llamada a `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)`** | `0` | `0` | `ST_LED_IDLE` | **`EV_LED_ACTIVE`** | **`true`** | **OFF** | Interfaz asíncrona registra el evento deseado y activa la bandera. |
| **`task_actuator_update()` (Loop #3)** | `0` | `0` | **`ST_LED_ACTIVE`** | `EV_LED_ACTIVE` | **`false`** | **ON** (`led_on`) | `statechart()` detecta `flag == true` y `EV_LED_ACTIVE`, enciende el LED, borra `flag` y pasa a `ST_LED_ACTIVE`. |
| **`task_actuator_update()` (Loop #4)** | `0` | `0` | `ST_LED_ACTIVE` | `EV_LED_ACTIVE` | `false` | **ON** | Permanece en `ST_LED_ACTIVE` ya que `flag` es `false`. |
| **Llamada a `put_event_task_actuator(EV_LED_IDLE, ID_LED_A)`** | `0` | `0` | `ST_LED_ACTIVE` | **`EV_LED_IDLE`** | **`true`** | **ON** | Interfaz asíncrona registra evento para apagar LED y activa la bandera. |
| **`task_actuator_update()` (Loop #5)** | `0` | `0` | **`ST_LED_IDLE`** | `EV_LED_IDLE` | **`false`** | **OFF** (`led_off`) | `statechart()` detecta `flag == true` y `EV_LED_IDLE`, apaga el LED, borra `flag` y retorna a `ST_LED_IDLE`. |
| **`task_actuator_update()` (Loop #6)** | `0` | `0` | `ST_LED_IDLE` | `EV_LED_IDLE` | `false` | **OFF** | Retorno al estado de reposo continuo. |

---

## 5. Evolución de Variables de Interfaz (`put_event_task_actuator`)

### Variables Analizadas:
* `identifier`: Identificador del actuador objetivo pasado a la función (`ID_LED_A = 0`).
* `task_actuator_dta_list[identifier].event`: Evento inyectado en el buzón/estructura del actuador.
* `task_actuator_dta_list[identifier].flag`: Indicador de evento disponible para ser consumido.

### Ciclo de Vida del Evento mediante la Interfaz

```
[ Productor de Eventos ]  ---- put_event_task_actuator(EV, ID) ---->  [ task_actuator_dta_list[ID] ]
(Ej: Tarea Pulsador)                                                  .event = EV, .flag = true
                                                                                   |
                                                                                   v
[ Consumidor ] <---- task_actuator_update() / statechart() <-----------------------+
                     Evalúa (.flag == true), ejecuta acción física y limpia (.flag = false)
```

### Tabla de Evolución Temporal de la Interfaz

| Etapa del Sistema | Función Ejecutada / Invocador | `identifier` | `.event` | `.flag` | Estado en la Estructura RAM |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **1. Repositorio Inicial** | Inicialización del sistema | - | `EV_LED_IDLE` | `false` | Estado estable sin eventos pendientes. |
| **2. Excitación Externa** | `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)` | `0` (`ID_LED_A`) | `EV_LED_ACTIVE` | **`true`** | Evento registrado de encendido, bandera alta notificando nuevo evento. |
| **3. Procesamiento** | `task_actuator_update()` $
ightarrow$ `task_actuator_statechart(0)` | `0` | `EV_LED_ACTIVE` | **`false`** | Evento consumido en la FSM; la bandera vuelve a `false`. |
| **4. Reposo Activo** | Loop principal sin eventos | - | `EV_LED_ACTIVE` | `false` | Conserva la última constante de evento pero con `flag = false`. |
| **5. Excitación Externa** | `put_event_task_actuator(EV_LED_IDLE, ID_LED_A)` | `0` (`ID_LED_A`) | `EV_LED_IDLE` | **`true`** | Evento registrado de apagado, bandera alta notificando nuevo evento. |
| **6. Procesamiento** | `task_actuator_update()` $
ightarrow$ `task_actuator_statechart(0)` | `0` | `EV_LED_IDLE` | **`false`** | Evento consumido en la FSM; la bandera vuelve a `false`. |

---

## 6. Conclusiones sobre la Arquitectura

1. **Desacoplamiento y Modularidad**: La separación en `attribute`, `interface` e implementación principal asegura que otras tareas de la aplicación solo interactúen mediante `put_event_task_actuator()`, sin acceso directo a los pines o a la lógica de control.
2. **Ejecución No Bloqueante**: La función `task_actuator_statechart()` realiza comparaciones lógicas instantáneas y retorna inmediatamente, garantizando que el bucle principal de la aplicación no sufra demoras ni bloqueos (*polling* reactivo por banderas).
3. **Robustez y Tolerancia a Fallos**: El bloque `default` dentro del `switch` de la máquina de estados asegura que el sistema se recupere automáticamente ante cualquier anomalía o corrupción de memoria, retornando a un estado seguro (`ST_LED_IDLE` con LED apagado).
