# **Análisis de variables de tarea actuator**

[https://gemini.google.com/app/a44e0653f8e0d95f](https://gemini.google.com/app/a44e0653f8e0d95f)

*User prompt: Indicar la evolución de las variables index, task\_actuator\_dta\_list\[index\].tick (indicar unidad de medida), task\_actuator\_dta\_list\[index\].state, task\_actuator\_dta\_list\[index\].event y task\_actuator\_dta\_list\[index\].flag al ejecutar dicho código fuente desde su inicio (task\_actuator\_init() de task\_actuator.c) y en sucesivas ejecuciones del loop principal de la aplicación ((task\_actuator\_update() de task\_actuator.c). Indicar el comportamiento de la función void task\_actuator\_statechart(uint32\_t index). Indicar la evolución de las variables identifier, task\_actuator\_dta\_list\[identifier\].event y task\_actuator\_dta\_list\[identifier\].flag al ejecutar dicho código fuente desde su inicio (task\_actuator\_init() de task\_actuator.c) y en sucesivas ejecuciones del loop principal de la aplicación ((task\_actuator\_update() de task\_actuator.c).*

Response: **Evolución de variables de control e instancia de la tarea**

| Variable / Atributo | Inicialización (task\_actuator\_init) | Ejecución en Loop (task\_actuator\_update) |
| :---- | :---- | :---- |
| **index** | Itera únicamente en 0 (dado que ACTUATOR\_DTA\_QTY \= 1). | Toma el valor 0 en cada iteración del ciclo. |
| **task\_actuator\_dta\_list\[index\].tick** | Se inicializa implícitamente en 0 por asignación global de memoria BSS. **Unidad de medida:** milisegundos (mS). | Mantiene su valor, excepto si se cae en el caso default de la FSM, donde se fuerza a DEL\_LED\_MIN (0). |
| **task\_actuator\_dta\_list\[index\].state** | Se inicializa en ST\_LED\_IDLE. | Transiciona a ST\_LED\_ACTIVE si flag \== true y event \== EV\_LED\_ACTIVE. Vuelve a ST\_LED\_IDLE si flag \== true y event \== EV\_LED\_IDLE. |
| **task\_actuator\_dta\_list\[index\].event** | Se inicializa en EV\_LED\_IDLE. | Retiene su valor actual hasta ser actualizado externamente por la función de interfaz put\_event\_task\_actuator(). |
| **task\_actuator\_dta\_list\[index\].flag** | Se inicializa en false. | Pasa a false inmediatamente después de que task\_actuator\_statechart() procesa y valida la transición de estado. |

**Comportamiento de la función task\_actuator\_statechart(uint32\_t index)**

La función implementar una Máquina de Estados Finitos (FSM) no bloqueante que controla el estado del actuador (LED):

> * **ST\_LED\_IDLE:** Verifica si hay una novedad marcada (flag \== true) y si el evento es EV\_LED\_ACTIVE. De cumplirse, limpia la bandera (flag \= false), enciende el LED mediante HAL\_GPIO\_WritePin (led\_on) y transiciona la FSM al estado ST\_LED\_ACTIVE.  
> * **ST\_LED\_ACTIVE:** Verifica si hay una novedad marcada (flag \== true) y si el evento es EV\_LED\_IDLE. De cumplirse, limpia la bandera (flag \= false), apaga el LED mediante HAL\_GPIO\_WritePin (led\_off) y transiciona la FSM al estado ST\_LED\_IDLE.  
> * **default:** Actúa como controlador de fallos ante estados no válidos. Resetea la estructura asignando tick \= DEL\_LED\_MIN (0), state \= ST\_LED\_IDLE, event \= EV\_LED\_IDLE y flag \= false.

**Evolución de variables de la interfaz de eventos**

La función put\_event\_task\_actuator(event, identifier) expone el punto de entrada para inyectar eventos desde otras tareas o interrupciones:

> * **identifier:** Recibe el parámetro con el índice del actuador a modificar (en este sistema, ID\_LED\_A equivalente a 0).  
> * **task\_actuator\_dta\_list\[identifier\].event:**  
  * **Inicio (init):** Comienza en EV\_LED\_IDLE.  
  * **Al invocar la interfaz:** Al llamarse put\_event\_task\_actuator(), almacena el nuevo evento provisto (EV\_LED\_ACTIVE o EV\_LED\_IDLE).  
  * **Ejecución en Loop (update):** Retiene dicho valor de evento de manera persistente hasta que la interfaz vuelva a modificarlo.  
> * **task\_actuator\_dta\_list\[identifier\].flag:**  
  * **Inicio (init):** Comienza en false.  
  * **Al invocar la interfaz:** put\_event\_task\_actuator() establece la variable en true para señalar un evento pendiente de procesamiento.  
  * **Ejecución en Loop (update):** Vuelve a false automáticamente una vez ejecutada la transición interna en task\_actuator\_statechart().

---

