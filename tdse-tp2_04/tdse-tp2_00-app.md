# **Análisis y Evolución de Variables de Aplicación Embebida**

[https://gemini.google.com/app/2ce06464c0941b9a](https://gemini.google.com/app/2ce06464c0941b9a)

*User prompt: Analizar y explicar (en español), el funcionamiento del código fuente contenido en los archivos adjuntos: app.c, app\_it.c, logger.c, logger.h, systick.c y dwt.h. Indicar la evolución de las variables g\_app\_tick\_cnt, g\_app\_runtime\_us, index, task\_dta\_list\[index\].NOE, task\_dta\_list\[index\].LET, task\_dta\_list\[index\].BCET y task\_dta\_list\[index\].WCET (indicar unidad de medida) al ejecutar dicho código fuente desde su inicio (app\_init() de app.c) y en sucesivas ejecuciones del loop principal de la aplicación ((app\_update() de app.c). Indicar el impacto de usar LOGGER\_INFO() en las variables g\_app\_runtime\_us y task\_dta\_list\[index\].WCET.*

Response: El código fuente proporcionado implementa un sistema embebido "Bare Metal" basado en eventos de tiempo (Event-Triggered System). Utiliza la interrupción del temporizador del sistema (SysTick) para llevar el ritmo de ejecución de un conjunto de tareas (sensor, system, actuator) y perfila el tiempo de ejecución de cada una utilizando el contador de ciclos por hardware (DWT) del microcontrolador ARM Cortex.

A continuación, se detalla la evolución de las variables consultadas durante el ciclo de vida de la aplicación:

> * **g\_app\_tick\_cnt (Unidad: cantidad de eventos/ticks):**  
  * **Inicio:** Se inicializa en 0 (ZERO) dentro de la función app\_it\_init(), la cual es llamada al final de app\_init().  
  * **Evolución:** Se incrementa en 1 cada vez que se dispara la interrupción de hardware del temporizador en HAL\_SYSTICK\_Callback(). Luego, dentro del bucle principal en app\_update(), se decrementa en 1 cada vez que el sistema detecta que hay ticks pendientes por procesar (ZERO \< g\_app\_tick\_cnt) para habilitar la ejecución de las tareas. Actúa como un semáforo contador de los ticks del sistema.  
> * **g\_app\_runtime\_us (Unidad: microsegundos):**  
  * **Inicio:** No se inicializa explícitamente en app\_init() con un valor inicial fijo, pero en app\_update() se reinicia a 0 al comenzar cada ciclo de ejecución de tareas.  
  * **Evolución:** Dentro del bucle for que recorre las tareas en app\_update(), acumula la suma de los tiempos de ejecución (LET) de todas las tareas ejecutadas en ese tick (g\_app\_runtime\_us \+= task\_dta\_list\[index\].LET;). Representa el tiempo total que tomó procesar todas las tareas en el ciclo actual.  
> * **index (Unidad: índice/posición del arreglo):**  
  * **Inicio/Evolución:** Es una variable local utilizada como contador de bucles tanto en app\_init() como en app\_update(). Comienza en 0 e incrementa de a 1 hasta que la condición TASK\_QTY \> index deja de cumplirse. Sirve para acceder secuencialmente a la configuración y a los datos de cada tarea.  
> * **task\_dta\_list\[index\].NOE (Unidad: cantidad de ejecuciones):**  
  * **Inicio:** En app\_init(), se inicializa con el valor de la macro TASK\_X\_NOE\_INI (0).  
  * **Evolución:** En app\_update(), se incrementa en 1 (task\_dta\_list\[index\].NOE++;) inmediatamente después de que la tarea correspondiente (task\_update) termina de ejecutarse.  
> * **task\_dta\_list\[index\].LET (Unidad: microsegundos):**  
  * **Inicio:** En app\_init(), se inicializa con el valor TASK\_X\_LET\_INI (0).  
  * **Evolución:** En app\_update(), tras finalizar la ejecución de la tarea, se actualiza llamando a cycle\_counter\_get\_time\_us(). Este valor refleja el tiempo exacto que la tarea demoró en ejecutarse en el ciclo actual basándose en los registros de hardware del DWT.  
> * **task\_dta\_list\[index\].BCET (Unidad: microsegundos):**  
  * **Inicio:** En app\_init(), se inicializa con el valor TASK\_X\_BCET\_INI (1000).  
  * **Evolución:** En app\_update(), se compara el último tiempo de ejecución (LET) con el BCET actual. Si LET es menor que BCET, BCET adopta el valor de LET, registrando así el mejor caso (el tiempo más corto) de ejecución histórico de la tarea.  
> * **task\_dta\_list\[index\].WCET (Unidad: microsegundos):**  
  * **Inicio:** En app\_init(), se inicializa con el valor TASK\_X\_WCET\_INI (0).  
  * **Evolución:** En app\_update(), se compara con el LET. Si LET es mayor que el WCET actual, WCET adopta el valor de LET, guardando así el peor caso (el tiempo más largo) de ejecución histórico.

### **Impacto de utilizar LOGGER\_INFO()**

El uso de la macro LOGGER\_INFO() dentro del código de una tarea tiene un impacto severo y directo sobre el rendimiento del sistema y las variables de perfilado:

> 1. **Aumento del tiempo de ejecución:** La macro LOGGER\_INFO() formatea cadenas de texto utilizando snprintf y luego invoca funciones de impresión que pueden utilizar semihosting (printf). Estas operaciones de entrada/salida (E/S) son extremadamente lentas y bloqueantes.  
> 2. **Impacto en task\_dta\_list\[index\].WCET:** Como el tiempo de la tarea se mide encendiendo el contador antes de ejecutarla y leyéndolo al finalizar (cycle\_counter\_get\_time\_us()), el retardo introducido por el formateo y la impresión hará que el LET de esa tarea se dispare. Esto actualizará inmediatamente la variable WCET a un valor altísimo, ensuciando la métrica real del tiempo de ejecución en el peor de los casos de la lógica pura de la tarea.  
> 3. **Impacto en g\_app\_runtime\_us:** Dado que esta variable es la sumatoria de todos los LET de las tareas del ciclo, reflejará un incremento proporcional al tiempo que haya demorado la impresión, indicando que el procesador estuvo ocupado mucho más tiempo en ese tick.  
> 4. **Bloqueo de interrupciones:** La macro LOGGER\_INFO() deshabilita globalmente las interrupciones mediante instrucción en ensamblador (\_\_asm("CPSID i");) antes de formatear e imprimir, y las vuelve a habilitar al terminar (\_\_asm("CPSIE i");). Esto significa que, mientras se está registrando la información, interrupciones críticas como HAL\_SYSTICK\_Callback pueden retrasarse o perderse, lo cual desestabilizaría el valor de g\_app\_tick\_cnt y la cadencia de tiempo real de toda la aplicación.

---

<img width="699" height="597" alt="image" src="https://github.com/user-attachments/assets/4eb48044-3011-41a2-9abc-6bdcf37a6fd4" />

Las unidades de tiempo de las variables están en microsegundos (uS)
