# **Codificación de Diagramas de Estado en C**

[https://gemini.google.com/app/3cdc8c0b4cff48a3](https://gemini.google.com/app/3cdc8c0b4cff48a3)

*User prompt: ¿Puedes ayudarme a realizar un Trabajo Práctico sobre Codificación en C de Diagramas de Estado?*

Response: Para codificar un diagrama de estados en C, el enfoque más clásico, robusto y fácil de entender para un Trabajo Práctico es usar una **máquina de estados finitos (FSM)** estructurada con switch-case.

Vamos a armar tu Trabajo Práctico utilizando el ejemplo más universal para entender este concepto: **un molinete de subte (o torniquete)**.

## **1\. Diseño del Diagrama de Estados (El Modelo)**

Antes de programar, debes tener claro qué componentes tiene tu sistema. En un molinete tenemos:

> * **Estados (Lo que el sistema "es"):**  
  * BLOQUEADO: El brazo no gira.  
  * DESBLOQUEADO: El brazo puede girar.  
> * **Eventos (Lo que "sucede" y provoca cambios):**  
  * MONEDA: Alguien inserta el pago.  
  * EMPUJE: Alguien empuja el brazo para pasar.

**La lógica (Transiciones):**

> * Si está BLOQUEADO y el evento es MONEDA → Pasa a DESBLOQUEADO.  
> * Si está BLOQUEADO y el evento es EMPUJE → Sigue BLOQUEADO.  
> * Si está DESBLOQUEADO y el evento es EMPUJE → Pasa a BLOQUEADO.  
> * Si está DESBLOQUEADO y el evento es MONEDA → Sigue DESBLOQUEADO (la persona perdió su moneda).

## **2\. Implementación en C (Paso a Paso)**

Para traducir esto a código C, la mejor práctica es utilizar enum (enumeraciones) para que el código sea legible y no usar "números mágicos" (como estado 0, estado 1).

### **Código Completo de Ejemplo**

Aquí tienes la plantilla completa que puedes usar, compilar y probar.

`#include <stdio.h>`

`/* =========================================`  
   `1. DEFINICIÓN DE ESTADOS Y EVENTOS`  
   `========================================= */`  
`typedef enum {`  
    `ESTADO_BLOQUEADO,`  
    `ESTADO_DESBLOQUEADO`  
`} Estado;`

`typedef enum {`  
    `EVENTO_MONEDA,`  
    `EVENTO_EMPUJE,`  
    `EVENTO_NINGUNO // Útil para ciclos donde no pasa nada`  
`} Evento;`

`/* =========================================`  
   `2. SIMULADOR DE HARDWARE / ENTRADAS`  
   `========================================= */`  
`// En un caso real, esta función leería un sensor o un botón.`  
`// Aquí usamos la consola para que el usuario ingrese el evento.`  
`Evento leer_evento() {`  
    `char opcion;`  
    `printf("\nIngrese evento (m = Moneda, e = Empuje, s = Salir): ");`  
    `scanf(" %c", &opcion);`

    `switch(opcion) {`  
        `case 'm': return EVENTO_MONEDA;`  
        `case 'e': return EVENTO_EMPUJE;`  
        `case 's': return EVENTO_NINGUNO;`  
        `default:  return EVENTO_NINGUNO;`  
    `}`  
`}`

`/* =========================================`  
   `3. FUNCIÓN PRINCIPAL (LA MÁQUINA DE ESTADOS)`  
   `========================================= */`  
`int main() {`  
    `// Inicializamos el sistema en su estado base`  
    `Estado estado_actual = ESTADO_BLOQUEADO;`  
    `Evento evento_actual;`  
    `int ejecutando = 1;`

    `printf("--- INICIANDO SISTEMA DE MOLINETE ---\n");`

    `// El loop infinito o "Super Loop" común en sistemas embebidos`  
    `while (ejecutando) {`  
        `evento_actual = leer_evento();`

        `if (evento_actual == EVENTO_NINGUNO) {`  
            `printf("Apagando sistema...\n");`  
            `ejecutando = 0;`  
            `break;`  
        `}`

        `// --- LÓGICA DE LA MÁQUINA DE ESTADOS ---`  
        `switch (estado_actual) {`  
              
            `case ESTADO_BLOQUEADO:`  
                `if (evento_actual == EVENTO_MONEDA) {`  
                    `printf("Accion: Liberando traba...\n");`  
                    `estado_actual = ESTADO_DESBLOQUEADO; // Transición`  
                    `printf("-> Nuevo Estado: DESBLOQUEADO\n");`  
                `}`   
                `else if (evento_actual == EVENTO_EMPUJE) {`  
                    `printf("Accion: Alarma! Inserte moneda primero.\n");`  
                    `// Se mantiene en ESTADO_BLOQUEADO`  
                `}`  
                `break;`

            `case ESTADO_DESBLOQUEADO:`  
                `if (evento_actual == EVENTO_EMPUJE) {`  
                    `printf("Accion: Persona paso. Cerrando traba...\n");`  
                    `estado_actual = ESTADO_BLOQUEADO; // Transición`  
                    `printf("-> Nuevo Estado: BLOQUEADO\n");`  
                `}`   
                `else if (evento_actual == EVENTO_MONEDA) {`  
                    `printf("Accion: Moneda rechazada. Ya esta abierto.\n");`  
                    `// Se mantiene en ESTADO_DESBLOQUEADO`  
                `}`  
                `break;`  
        `}`  
    `}`

    `return 0;`  
`}`

## **3\. Arquitecturas Alternativas (Para sumar nota en el TP)**

El método de switch-case es perfecto para máquinas de estado pequeñas (menos de 5 estados). Si en tu trabajo práctico te exigen algo más avanzado o tu diagrama tiene muchos estados (10+), puedes mencionar o utilizar estos dos métodos:

> 1. **Tabla de Transiciones (Matriz):** Creas una matriz bidimensional (Array) donde las filas son los estados, las columnas los eventos, y el contenido de la celda es el "Siguiente Estado". Es muy rápido y consume pocos recursos de CPU, pero gasta más memoria RAM.  
> 2. **Punteros a Funciones:** Cada estado es una función independiente en C (void estado\_bloqueado(), void estado\_desbloqueado()). Una variable puntero de función apunta al estado actual y se va reasignando. Es el método más limpio y escalable utilizado en ingeniería de software profesional.

---

