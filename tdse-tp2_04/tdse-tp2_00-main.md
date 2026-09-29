# **Análisis de Código STM32**

[https://gemini.google.com/app/cdf3e04aef1027c0](https://gemini.google.com/app/cdf3e04aef1027c0)

*User prompt: Analizar y explicar (en español), el funcionamiento del código fuente contenido en los archivos adjuntos: main.c, stm32f1xx\_it.c y startup\_stm32f103rbtx.s. Indicar la evolución de las variables SysTick y SystemCoreClock al ejecutar dicho código fuente desde su inicio (Reset\_Handler: de startup\_stm32f103rbtx.s) hasta el loop principal de la aplicación (while (1) de main.c).*

Response: El código fuente proporcionado gestiona el proceso de arranque, la configuración del sistema de reloj a 64 MHz, la inicialización de periféricos (GPIO y USART2) y la atención de interrupciones para un microcontrolador STM32F103 (Cortex-M3) mediante la biblioteca HAL de ST.

### **Funcionamiento del Código Fuente**

> * **startup\_stm32f103rbtx.s**:  
  * **Tabla de Vectores (g\_pfnVectors)**: Define la dirección inicial del puntero de pila (\_estack) y la lista de direcciones de los manejadores de excepciones e interrupciones (Reset\_Handler, SysTick\_Handler, EXTI15\_10\_IRQHandler, etc.).  
  * **Rutina Reset\_Handler**: Es el primer código ejecutado tras un reinicio del microcontrolador.  
    1. Llama a SystemInit para establecer la configuración básica e inicial del sistema de reloj.  
    2. Copia los valores iniciales de las variables globales de la memoria Flash a la SRAM (sección .data, de \_sdata a \_edata).  
    3. Llena de ceros el espacio reservado para variables no inicializadas en SRAM (sección .bss, de \_sbss a \_ebss).  
    4. Invoca a \_\_libc\_init\_array para ejecutar constructores estáticos de C/C++.  
    5. Salta a la función principal main() del programa en C.  
> * **main.c**:  
  * **main()**: Punto de entrada del programa de aplicación.  
  * **HAL\_Init()**: Inicializa la biblioteca HAL, ajusta la prioridad de los vectores de interrupción y configura la base de tiempo mediante el temporizador SysTick.  
  * **SystemClock\_Config()**: Habilita el oscilador interno HSI (8 MHz). Configura el PLL dividiendo HSI por 2 (4 MHz) y multiplicando por 16 para obtener una frecuencia de reloj de sistema (*SYSCLK*) de 64 MHz. Configura los divisores de bus (AHB a 64 MHz, APB1 a 32 MHz, APB2 a 64 MHz), define la latencia de la Flash a 2 ciclos y reajusta la base de tiempo del SysTick.  
  * **Periféricos**:  
    * MX\_GPIO\_Init(): Habilita los relojes de los puertos A, B, C y D. Configura el pin LD2 como salida digital y el botón B1 como interrupción externa (EXTI15\_10\_IRQn) por flanco ascendente.  
    * MX\_USART2\_UART\_Init(): Inicializa el módulo USART2 a 115200 baudios, 8 bits de datos, 1 bit de parada y sin paridad.  
  * **Ejecución**: Ejecuta app\_init() y entra al bucle infinito while (1) ejecutando continuamente app\_update().  
> * **stm32f1xx\_it.c**:  
  * **Manejadores de Excepción**: Incluye funciones como NMI\_Handler, HardFault\_Handler, MemManage\_Handler, BusFault\_Handler, etc., configuradas en bucles infinitos para detener la ejecución en caso de fallos graves del procesador.  
  * **SysTick\_Handler()**: Interrupción periódica del temporizador de sistema que invoca a HAL\_IncTick() para incrementar el contador de tiempo global (uwTick) y ejecuta HAL\_SYSTICK\_IRQHandler().  
  * **EXTI15\_10\_IRQHandler()**: Captura la interrupción por hardware del botón B1 (pines GPIO 10 a 15\) y la redirige a la HAL (HAL\_GPIO\_EXTI\_IRQHandler).

### **Evolución de SystemCoreClock y SysTick (uwTick)**

| Etapa de Ejecución | Frecuencia SystemCoreClock | Registro SysTick-\>LOAD | Estado de SysTick / Variable uwTick |
| :---- | :---- | :---- | :---- |
| **1\. Reset\_Handler (startup)** | 8000000 Hz (8 MHz) | Inactivo / No configurado | Hardware deshabilitado. uwTick pasa a valor 0 tras la inicialización del segmento .bss. |
| **2\. HAL\_Init() (en main.c)** | 8000000 Hz (8 MHz) | 7999 (10008000000​−1) | Se habilita el SysTick para generar interrupciones cada 1 ms. Se fija uwTick \= 0 y se incrementa en \+1 con cada interrupción. |
| **3\. SystemClock\_Config()** | Cambia de 8 MHz a **64000000 Hz (64 MHz)** | Reconfigurado a 63999 (100064000000​−1) | La HAL recalcula SysTick-\>LOAD para conservar el período de 1 ms a 64 MHz. uwTick continúa acumulando tiempo (*uwTick*\=*N*). |
| **4\. Bucle while (1)** | **64000000 Hz (64 MHz)** | 63999 | SysTick interrumpe exactamente cada 1 ms a 64 MHz. uwTick contiene el número de milisegundos transcurridos desde HAL\_Init(). |

### **Cálculos del Sistema de Reloj y Temporización**

> * **Frecuencia del CPU (SystemCoreClock)**:  
  * Oscilador base: HSI \= 8 MHz.  
  * Entrada del bloque PLL: 2HSI​\=28 MHz​\=4 MHz.  
  * Multiplicador del PLL: PLLMUL=16.  
  * Frecuencia del reloj de sistema:  
    *f*SYSCLK​\=4 MHz×16=64 MHz  
  * Tras ejecutar HAL\_RCC\_ClockConfig(), la variable global se actualiza a SystemCoreClock \= 64000000\.  
> * **Cálculo del Período de SysTick**:  
  * La fórmula utilizada por la HAL para lograr una interrupción cada 1 ms (1000 Hz) es:  
    LOAD=(1000SystemCoreClock​)−1  
  * Inicialmente a 8 MHz:  
    LOAD=10008000000​−1=7999  
  * Tras la reconfiguración a 64 MHz:  
    LOAD=100064000000​−1=63999  
  * Cada vez que el contador del temporizador llega a cero, se ejecuta la rutina de interrupción SysTick\_Handler() en stm32f1xx\_it.c, invocando a HAL\_IncTick() que realiza el incremento: uwTick \= uwTick \+ 1\.

---

