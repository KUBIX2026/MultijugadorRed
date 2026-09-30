# Protocolo de comunicación multijugador (Arquitectura Maestro-Esclavo)

## 1. Objetivo

Establecer las características técnicas de la comunicación serial entre las dos FPGA (Tang Nano 20K) para el modo multijugador. 

A diferencia de una red de computadores convencional, implementaremos una arquitectura basada en consolas retro clásicas (inspirada estrictamente en el hardware del Cable Link de la Game Boy de Nintendo). Esto implica una topología **Maestro - Esclavo**, donde una consola (la Principal) genera la señal de reloj y administra el ritmo de la comunicación, mientras que la otra (la Invitada) responde a sus solicitudes. El objetivo es intercambiar los datos espaciales y de controles a 60 FPS sin *lag* perceptible.

## 2. Comunicación mediante SIO (Serial I/O)

### 2.1 Características generales del Hardware

Para este módulo desarrollaremos un protocolo SIO síncrono a medida. A diferencia de un bloque SPI comercial, nuestro diseño omite deliberadamente la señal de selección (Chip Select / CS), ya que es un enlace dedicado punto a punto. Esto simplifica la conexión física a solo cuatro cables entre los pines de expansión de las placas:

* **SC (Serial Clock):** Señal de reloj. Generada exclusivamente por la FPGA Maestra.
* **SO (Serial Out):** Pin de salida de datos.
* **SI (Serial In):** Pin de entrada de datos.
* **GND:** Tierra común (Crítica para evitar ruido y lecturas erróneas por diferencias de voltaje).

La conexión de datos es cruzada: el pin `SO` de la Maestra va al `SI` de la Esclava, y el `SO` de la Esclava va al `SI` de la Maestra. La señal `SC` viaja en un solo sentido (Maestra → Esclava).

**Figura 1. Conexión física entre las dos interfaces.**
![Conexión entre las consolas](imagenes/conexion_sio.png)

Como necesitamos que los botones y coordenadas de ambos jugadores viajen simultáneamente para evitar ventajas competitivas, la arquitectura física es **full-duplex síncrona**.

**Figura 2. Tipos de comunicación según la dirección de transmisión.**
![Tipos de comunicación](imagenes/tipos_comunicacion.png)

### 2.2 Funcionamiento: El Registro de Desplazamiento (Shift Register)

El corazón de este protocolo en Verilog no es un transmisor complejo, sino un **Registro de Desplazamiento de 8 bits** instanciado en ambas FPGAs. 

El flujo de datos funciona como un trueque simultáneo:
1. El procesador (FemtoRV32) de la consola Maestra da la orden de iniciar transmisión.
2. La máquina de estados de la Maestra enciende el reloj y genera exactamente 8 pulsos por el cable `SC`.
3. **Sincronización por flancos:** En el flanco de bajada de `SC`, ambas FPGAs empujan su bit más significativo hacia el cable `SO`. En el flanco de subida de `SC`, ambas FPGAs leen el valor del cable `SI` y lo guardan.
4. Al terminar los 8 pulsos, el reloj se detiene. Ambas consolas han intercambiado exactamente 1 byte al mismo tiempo.

**Datos en paralelo → Shift Register (Maestra) ↔ SC / SI / SO ↔ Shift Register (Esclava) → Datos en paralelo**

**Figura 3. Proceso de intercambio de datos simultáneo.**
![Proceso de transmisión SIO](imagenes/transmision_sio.png)

### 2.3 Justificación de la arquitectura

Adoptar este diseño nos da ventajas críticas a nivel de implementación RTL:
* **Sincronismo perfecto:** Al compartir el reloj, nos olvidamos de los *Framing Errors* (errores de trama) típicos del UART. La Esclava sabe exactamente cuándo leer el dato sin importar variaciones de temperatura o frecuencia.
* **Intercambio 1 a 1 (Trueque):** Garantiza que si la consola 1 envía su posición, forzosamente recibe los botones o posición de la consola 2 en el mismo ciclo.

## 3. Estructura de la Trama de Datos (Payload)

Como el hardware intercambia de a 1 byte (8 bits) a la vez, el software agrupará la información del juego en paquetes fijos de **4 bytes**. El código en C de la Maestra ejecutará la transferencia 4 veces seguidas por cada actualización de pantalla. Para no saturar el bus, nunca enviaremos gráficos, solo variables de estado.

| Byte | Función | Descripción | Ejemplo (Hex) |
| :--- | :--- | :--- | :--- |
| **1** | Cabecera (Start) | Byte fijo de alineación. Le avisa al firmware que aquí inicia un paquete válido. | `0xAA` |
| **2** | Comando / ID | Indica qué entidad del juego se está actualizando. | `0x01` |
| **3** | Dato 1 | Primer parámetro (ej. Coordenada X, o botón presionado). | `0x2F` |
| **4** | Dato 2 | Segundo parámetro (ej. Coordenada Y, o estado). | `0x78` |

**Ejemplo de uso (Juego tipo Pong):** 
Si el Jugador 1 mueve su paleta a la posición Y=120, su FPGA enviará: `[0xAA] [0x01] [0x00] [0x78]`. Al mismo tiempo, estará recibiendo la trama equivalente de la FPGA del Jugador 2. Si un paquete llega y su primer byte no es `0xAA`, el código en C lo descarta para evitar *glitches* en las posiciones.

## 4. Comandos y el problema del "Polling"

Al ser una arquitectura Maestro-Esclavo, la consola del Jugador 2 (Esclava) **no puede iniciar una transmisión por sí sola** cuando el jugador presiona un botón. 

Para solucionar esto, el firmware de la Maestra implementará un sondeo constante (*Polling*). 60 veces por segundo, antes de renderizar cada *frame*, la Maestra le enviará un comando vacío a la Esclava solo para forzarla a que devuelva el estado de sus botones o posición. 

* **`0x01` - Posición Local:** Envía la posición del jugador.
* **`0x02` - Posición Remota:** Actualiza la posición del rival en la pantalla.
* **`0x03` - Elementos neutros:** Coordenadas de objetos compartidos (como la pelota).
* **`0xF0` - Estado del juego:** Eventos como inicio, pausa, punto o fin de partida.
* **`0xF1` - Sincronización:** Mensaje inicial (*handshake*) para verificar que el cable está conectado antes de iniciar la ejecución de las físicas.

## 5. Plan de Acción y Pendientes

El desarrollo de este módulo se dividirá en tres grandes fases:

**Fase 1: Diseño de Hardware (RTL en Verilog)**
* Instanciar el divisor de reloj en la FPGA Maestra (partiendo de los 27 MHz de la Tang Nano) para generar una frecuencia estable para la señal `SC` (ej. 100 kHz).
* Diseñar la lógica del *Shift Register* de 8 bits operando con los flancos de subida y bajada.
* Crear las dos variantes del hardware: El módulo Maestro (que controla el reloj y la FSM) y el módulo Esclavo (que solo obedece al reloj externo).

**Fase 2: Firmware y Control (Lenguaje C)**
* Mapear las direcciones de memoria de los registros CSR para que el FemtoRV32 pueda escribir y leer del hardware SIO.
* Programar la rutina de *polling* a 60 FPS en el ciclo principal (`while(1)`) de la FPGA Maestra.
* Crear la lógica de validación de la cabecera `0xAA` para descartar paquetes corruptos.

**Fase 3: Validación y Pruebas**
* Analizar los requerimientos específicos de los minijuegos a implementar para ajustar el tamaño definitivo de las variables (si requerimos más de 8 bits para una coordenada espacial de la pantalla, adaptar la trama).
* Crear el *Testbench* integrando ambos módulos (Maestro y Esclavo) en GTKWave, validando que el intercambio de bits ocurra exactamente durante los 8 ciclos de `SC` y que la línea quede en reposo al terminar.
