# Protocolo de comunicación multijugador (Arquitectura Maestro-Esclavo)

## 1. Objetivo

Establecer las características técnicas de la comunicación serial entre las dos FPGA para el modo multijugador. 

A diferencia de una red convencional, implementaremos una arquitectura basada en consolas retro clásicas (inspirada estrictamente en el hardware del *Cable Link* de la Game Boy de Nintendo). Esto implica una topología **Maestro - Esclavo**, donde una consola (la Principal) genera la señal de reloj y administra el ritmo de la comunicación, mientras que la otra (la Invitada) responde a sus solicitudes. El objetivo es intercambiar los datos espaciales y de controles a 60 FPS sin *lag* perceptible.

## 2. Comunicación mediante SIO (Serial I/O)

### 2.1 Características generales del Hardware

Para este módulo desarrollaremos un protocolo SIO (Serial Input/Output) síncrono diseñado a medida. Este protocolo está optimizado para un enlace dedicado punto a punto, lo que nos permite simplificar la conexión física a solo cuatro cables, conectando los pines de expansión de ambas FPGAs:

* **SC (Serial Clock):** Señal de reloj. Generada exclusivamente por la FPGA Maestra.
* **SO (Serial Out):** Pin de salida de datos.
* **SI (Serial In):** Pin de entrada de datos.
* **GND:** Tierra común (Crítica para evitar ruido y lecturas erróneas por diferencias de voltaje).

La conexión de datos es cruzada: el pin `SO` de la Maestra va al `SI` de la Esclava, y el `SO` de la Esclava va al `SI` de la Maestra. La señal `SC` viaja en un solo sentido (Maestra → Esclava).

**Figura 1. Conexión física entre las dos interfaces.**
![Conexión entre las consolas](imagenes/conexion_sio.png)

Como necesitamos que los botones y coordenadas de ambos jugadores viajen simultáneamente para evitar ventajas competitivas, la arquitectura física es **full-duplex síncrona**.

### 2.2 Funcionamiento: El Registro de Desplazamiento (Shift Register)

La clave para entender y diseñar este protocolo en hardware (Verilog) es que no se trata de un transmisor complejo, sino de un **Registro de Desplazamiento de 8 bits** instanciado en ambas FPGAs, funcionando como un espejo.

El intercambio de información ocurre como un trueque simultáneo paso a paso:
1. **Preparación:** El procesador de cada FPGA carga el byte que quiere enviar en su respectivo Registro de Desplazamiento.
2. **Activación:** El procesador de la consola Maestra da la orden de iniciar la transmisión. Su hardware enciende el reloj y genera exactamente 8 pulsos por el cable `SC`.
3. **Sincronización por flancos:** 
   * En cada **flanco de bajada** de la señal `SC`, ambas FPGAs empujan su bit más significativo hacia el cable de salida `SO`.
   * En cada **flanco de subida** de la señal `SC`, ambas FPGAs leen el valor presente en su cable de entrada `SI` y lo guardan en su registro.
4. **Finalización:** Al terminar los 8 pulsos, el reloj `SC` se detiene por completo. En ese instante, ambas consolas han intercambiado exactamente 1 byte al mismo tiempo.

**Datos en paralelo → Shift Register (Maestra) ↔ SC / SI / SO ↔ Shift Register (Esclava) → Datos en paralelo**

### 2.3 Diferencias con el SPI tradicional (Justificación del diseño)

Aunque este protocolo SIO pertenece a la misma familia síncrona que el SPI, implementamos la filosofía de Nintendo, la cual tiene dos diferencias estructurales fundamentales en el hardware que justifican su uso para este proyecto:

* **Ausencia de pin "Chip Select" (CS):** El SPI comercial exige un cuarto cable de control (CS) que el maestro debe poner en bajo para "despertar" al esclavo. En nuestro diseño, al ser solo dos consolas, el esclavo siempre está escuchando. Su hardware reacciona de inmediato en el milisegundo en que detecta actividad en el pin `SC`.
* **Intercambio 1 a 1 estricto:** El SPI moderno permite ráfagas largas asimétricas. Nuestro hardware obliga a que el intercambio sea simétrico y bloqueante. Si la consola 1 envía un byte, forzosamente recibe un byte de la consola 2 en el mismo ciclo.

## 3. Estructura de la Trama de Datos (Payload)

Como el hardware intercambia de a 1 byte (8 bits) a la vez por cada ráfaga de reloj, el software del juego agrupará la información en paquetes fijos de **4 bytes**. El código en C de la Maestra ordenará al hardware ejecutar la transferencia 4 veces seguidas por cada actualización de pantalla. Para no saturar el bus, nunca enviaremos gráficos, solo variables de estado.

| Byte | Función | Descripción | Ejemplo (Hex) |
| :--- | :--- | :--- | :--- |
| **1** | Cabecera (Start) | Byte fijo de alineación. Le avisa al firmware que aquí inicia un paquete válido. | `0xAA` |
| **2** | Comando / ID | Indica qué entidad del juego se está actualizando. | `0x01` |
| **3** | Dato 1 | Primer parámetro (ej. Coordenada X, o botón presionado). | `0x2F` |
| **4** | Dato 2 | Segundo parámetro (ej. Coordenada Y, o estado). | `0x78` |

**Tolerancia a fallos:** Si un paquete llega y su primer byte no es la cabecera esperada (`0xAA`), el código en C asume que hubo un desface en los bits y descarta el paquete entero, evitando *glitches* en las posiciones del juego.

## 4. Comandos y el problema del "Polling"

La principal desventaja arquitectónica de un sistema Maestro-Esclavo frente a uno *Peer-to-Peer*, es que la consola del Jugador 2 (Esclava) **no puede iniciar una transmisión por sí sola** cuando el jugador presiona un botón, ya que no controla la señal de reloj.

Para solucionar esto sin generar latencia, el firmware de la Maestra implementará un sondeo constante (*Polling*). Sesenta veces por segundo, antes de renderizar cada *frame*, la Maestra le enviará un comando a la Esclava para forzarla a que devuelva el estado de sus botones y coordenadas. Usaremos los siguientes identificadores para el Byte 2 de nuestra trama:

* **`0x01` - Posición Local:** Envía la posición del jugador.
* **`0x02` - Posición Remota:** Actualiza la posición del rival en la pantalla.
* **`0x03` - Elementos neutros:** Coordenadas de objetos compartidos (como una pelota).
* **`0xF0` - Estado del juego:** Eventos como inicio, pausa, punto o fin de partida.
* **`0xF1` - Sincronización:** Mensaje inicial (*handshake*) para verificar que el cable está conectado.

## 5. Plan de Acción y Pendientes

El desarrollo de este módulo de comunicación se dividirá en tres grandes fases de ingeniería:

**Fase 1: Diseño de Hardware (RTL en Verilog)**
* Instanciar el divisor de reloj en la FPGA Maestra (partiendo del oscilador base de la tarjeta) para generar una frecuencia estable para la señal `SC` (ej. 100 kHz).
* Diseñar la lógica del *Shift Register* de 8 bits operando estrictamente con los flancos de subida y bajada.
* Crear las dos variantes del hardware: El módulo Maestro (que controla el reloj y cuenta los 8 pulsos) y el módulo Esclavo (que solo obedece a los flancos del reloj externo).

**Fase 2: Firmware y Control (Lenguaje C)**
* Mapear las direcciones de memoria de los registros para que el procesador pueda escribir el byte a enviar y leer el byte recibido desde el hardware SIO.
* Programar la rutina de *polling* a 60 FPS en el ciclo principal (`while(1)`) de la FPGA Maestra para leer al Jugador 2.
* Crear la lógica de validación de la cabecera `0xAA`.

**Fase 3: Validación y Pruebas**
* Analizar los requerimientos específicos de los minijuegos para ajustar el tamaño definitivo de las variables (si requerimos más de 8 bits para una coordenada de la pantalla, adaptar la trama).
* Crear el banco de pruebas (*Testbench*) integrando el módulo Maestro y Esclavo en GTKWave. Validar que el intercambio de bits ocurra exactamente durante los 8 ciclos de `SC` y que la línea quede estable al terminar.
