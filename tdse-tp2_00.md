# Trabajo Práctico Nº 2: Codificación en C de Diagramas de Estado

**Materia:** Técnicas Digitales / Sistemas Embebidos (TDSE)  
**Tema:** Implementación de Máquinas de Estado Finito (FSM) en lenguaje C  

---

## 1. Objetivos

*   Comprender y aplicar técnicas de traducción sistemática de un Diagrama de Estados a código en C.
*   Familiarizarse con las distintas arquitecturas de software para la implementación de FSM: método de sentencias condicionales (`switch-case`), tablas de transición de estados y punteros a funciones.
*   Aplicar buenas prácticas de programación en sistemas embebidos (uso de `enum`, encapsulamiento, código no bloqueante).

---

## 2. Introducción Teórica

Una Máquina de Estados Finito (FSM) es un modelo matemático de computación utilizado para diseñar algoritmos lógicos en sistemas embebidos. En C, la codificación de una FSM suele requerir la definición de:

1.  **Estados:** Representados convencionalmente mediante una enumeración (`typedef enum`).
2.  **Eventos (o Entradas):** Señales que provocan las transiciones.
3.  **Transiciones:** La lógica que determina el próximo estado basándose en el estado actual y el evento ocurrido.
4.  **Acciones:** Tareas que se ejecutan al entrar a un estado, al salir de él, o durante una transición (modelo Moore o Mealy).

### Patrones de Diseño Comunes:
*   **Aproximación Anidada (Switch-Case):** Ideal para FSMs simples (menos de 10 estados). Es explícita pero puede volverse inmanejable si la máquina crece demasiado.
*   **Tabla de Estados (State Table):** Consiste en un arreglo bidimensional (Estado Actual vs. Evento) que devuelve el próximo estado y la acción a ejecutar. Excelente para máquinas grandes.
*   **Punteros a Funciones (State Pattern):** Cada estado se implementa como una función independiente. El despachador simplemente llama a la función apuntada por un puntero de estado.

---

## 3. Actividades Prácticas

### Ejercicio 1: Implementación Básica con `switch-case` (El Molinete de Metro)

Un molinete de metro tiene dos estados: `CERRADO` y `ABIERTO`.
*   Si está `CERRADO` y se inserta una moneda (evento `EV_MONEDA`), el molinete se desbloquea y pasa al estado `ABIERTO`.
*   Si está `CERRADO` y alguien empuja (evento `EV_EMPUJE`), el molinete sigue `CERRADO` y suena una alarma.
*   Si está `ABIERTO` y alguien empuja (evento `EV_EMPUJE`), la persona pasa, el molinete se bloquea y vuelve a `CERRADO`.
*   Si está `ABIERTO` y se inserta otra moneda (evento `EV_MONEDA`), el molinete sigue `ABIERTO` (la moneda es rechazada o tragada sin efecto extra).

**Consigna:**
1.  Definir los `enum` para los estados y los eventos.
2.  Escribir una función `Molinete_Init()` para inicializar la máquina.
3.  Escribir una función `Molinete_Update(Evento_t evento)` que implemente la máquina de estados utilizando estructuras `switch-case`.
4.  Incluir funciones *dummy* (simuladas con `printf`) para las acciones de hardware: `Bloquear_Molinete()`, `Desbloquear_Molinete()`, `Sonar_Alarma()`.

#### Código Base Sugerido:
```c
typedef enum {
    ESTADO_CERRADO,
    ESTADO_ABIERTO
} EstadoMolinete_t;

typedef enum {
    EV_MONEDA,
    EV_EMPUJE,
    EV_NINGUNO
} Evento_t;

EstadoMolinete_t estado_actual;

void Molinete_Init(void) {
    estado_actual = ESTADO_CERRADO;
    Bloquear_Molinete();
}
```

---

### Ejercicio 2: Semáforo Vehicular con Temporizador (Acciones de Entrada y Salida)

Diseñe en C la máquina de estados de un semáforo estándar (Rojo, Amarillo, Verde).
La transición entre estados no está dada por pulsadores, sino por el evento `EV_TIMEOUT` de un temporizador de hardware.

*   `ESTADO_ROJO`: Dura 5 segundos. Luego pasa a `ESTADO_VERDE`.
*   `ESTADO_VERDE`: Dura 4 segundos. Luego pasa a `ESTADO_AMARILLO`.
*   `ESTADO_AMARILLO`: Dura 1 segundo. Luego pasa a `ESTADO_ROJO`.

**Consigna:**
1. Implemente esta máquina prestando especial atención a las **Acciones de Entrada** (Entry Actions). Cada vez que se ingresa a un nuevo estado, se deben encender las luces correspondientes y reiniciar el contador/temporizador.
2. Diseñe el prototipo de la FSM para que sea no bloqueante (no usar `delay()` o `sleep()` dentro de los estados). Se asume que una rutina de interrupción (ISR) o un *scheduler* llamará a la FSM periódicamente.

---

### Ejercicio 3: Máquina Expendedora (Uso de Variables de Contexto y Guardas)

Diseñar una FSM para una máquina expendedora de café. El café cuesta $50.
*   **Eventos:** `EV_INGRESAR_10`, `EV_INGRESAR_20`, `EV_BOTON_CAFE`, `EV_BOTON_CANCELAR`.
*   **Estados propuestos:** `ESPERANDO_MONEDAS`, `SIRVIENDO_CAFE`, `DEVOLVIENDO_CAMBIO`.

**Consigna:**
1. En este ejercicio, la FSM necesita "memoria" adicional (contexto) además del estado puro. Defina una estructura `FSM_Expendedora_t` que contenga el estado actual y una variable entera `saldo_acumulado`.
2. Implementar condiciones de guarda. Por ejemplo, en el estado `ESPERANDO_MONEDAS`, si llega el evento `EV_BOTON_CAFE`, la transición a `SIRVIENDO_CAFE` **solo** debe ocurrir si `saldo_acumulado >= 50`.
3. Escribir la implementación utilizando la aproximación de **Punteros a Funciones** o **Tabla de Estados bidimensional** (opcional para alumnos avanzados).

---

## 4. Entregables

El alumno deberá entregar un archivo comprimido o un repositorio (ej. GitHub) que contenga:
1.  **Archivos `.c` y `.h`** para cada uno de los ejercicios. Se requiere una clara separación de la lógica de aplicación (la FSM) y la emulación del hardware (`main.c` con `printf`).
2.  **Diagramas de Estado** (en formato imagen, PDF o UML) correspondientes a cada código implementado.
3.  Un archivo `Makefile` o script de CMake para compilar todos los ejercicios de manera automatizada.

---

## 5. Referencias y Bibliografía
*   Miro Samek (2008). *Practical UML Statecharts in C/C++: Event-Driven Programming for Embedded Systems*. Newnes.
*   Documentación sobre patrones de diseño en C embebido: "State Pattern in C".
