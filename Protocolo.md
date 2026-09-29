# Protocolo de comunicación

## 1. Objetivo

Establecer las características técnicas y la estructura de datos para la comunicación bidireccional entre las dos FPGA del proyecto multijugador. 

La comunicación permitirá que las dos consolas (arquitectura *Peer-to-Peer*) intercambien la información necesaria en tiempo real para que los jugadores puedan participar en una misma partida sin latencia perceptible, sincronizando eventos de hardware, físicas y estados.

## 2. Comunicación mediante UART

### 2.1 Características generales

UART (Universal Asynchronous Receiver-Transmitter) es un método de comunicación serial asíncrona. Para la comunicación se utilizan principalmente tres líneas físicas de conexión entre las placas: **TX**, encargada de transmitir la información; **RX**, encargada de recibirla; y una línea común de **GND** para igualar las referencias de voltaje.

En la conexión entre las dos FPGA, la salida de transmisión de una UART se conecta con la entrada de recepción de la otra. De esta manera, la línea TX de la primera FPGA se conecta con RX de la segunda, y viceversa.

**Figura 1. Conexión entre las dos interfaces UART.**

![Conexión entre las dos UART](imagenes/conexion_uart.png)

UART permite realizar comunicación en diferentes sentidos: **simplex**, cuando la información viaja en una sola dirección; **half-duplex**, cuando ambos dispositivos pueden transmitir, pero no al mismo tiempo; y **full-duplex**, cuando ambos dispositivos pueden transmitir y recibir simultáneamente. Ya que se desea transmitir la información del juego tanto de la FPGA1 a la FPGA2 y viceversa al mismo tiempo para evitar ventajas de tiempo de reacción, el tipo de conexión empleada es estrictamente **full-duplex**.

**Figura 2. Tipos de comunicación según la dirección de transmisión.**

![Tipos de comunicación](imagenes/tipos_comunicacion.png)

### 2.2 Funcionamiento y Sincronización

UART permite transmitir información que se encuentra representada en paralelo (en el bus del SoC RISC-V) convirtiéndola en una secuencia de bits que puede viajar de forma serial. En el dispositivo receptor, esta secuencia se recibe y se vuelve a organizar para obtener nuevamente los datos.

En el proyecto, el proceso puede representarse de la siguiente manera:

**Datos en paralelo → UART 1 (RTL) → TX → RX → UART 2 (RTL) → Datos en paralelo**

**Figura 3. Proceso de transmisión de datos mediante UART.**

![Proceso de transmisión UART](imagenes/transmision_uart.png)

Al tratarse de una comunicación asíncrona, UART no necesita una señal de reloj (SCLK) compartida entre los dos dispositivos. Para garantizar la sincronización, ambas FPGA se configurarán con un *Baud Rate* idéntico de **115,200 bps**, calculado a partir de los divisores de reloj en Verilog.

Para identificar el comienzo y el final de cada unidad de información se utilizan bits de inicio y de parada.

**Figura 4. Comunicación asíncrona entre dos sistemas.**

![Comunicación asíncrona](imagenes/comunicacion_asincrona.png)

Una transmisión UART se organiza en una trama que contiene los elementos necesarios para que el receptor pueda identificar y recibir correctamente los datos. La trama comienza con un bit de inicio, continúa con los bits de datos y finaliza con uno o más bits de parada. Para optimizar la velocidad en este proyecto, se omitirá el bit de paridad.

**Figura 5. Formato de una trama UART.**

![Formato de trama UART](imagenes/Formato_UART.png)

### 2.3 Ventajas para el proyecto

Entre las características que hacen conveniente el uso de UART frente a alternativas como SPI o I2C para este proyecto se encuentran:

* **Arquitectura descentralizada:** No requiere una línea de reloj compartida entre las FPGA, evitando el modelo Maestro-Esclavo y permitiendo que ambas consolas sean 100% independientes.
* **Simplicidad de Hardware:** Utiliza una conexión muy sencilla basada únicamente en tres cables (TX, RX, GND).
* **Ausencia de colisiones:** Permite comunicación bidireccional continua y sin cuellos de botella mediante el modo full-duplex.

## 3. Estructura de la Trama de Datos (Payload)

Para evitar la desincronización por pérdida de bytes individuales, la información no se enviará de forma suelta. La transmisión se estructurará en paquetes cerrados de **4 bytes** que encapsulan el estado de la partida y las coordenadas espaciales (no se transmitirán datos gráficos).

| Byte | Función | Descripción | Ejemplo (Hex) |
| :--- | :--- | :--- | :--- |
| **1** | Cabecera (Start) | Identificador fijo de inicio de trama para alinear al receptor. | `0xAA` |
| **2** | Comando / ID | Indica qué tipo de información se está enviando. | `0x01` |
| **3** | Dato 1 (Ej. Coordenada X) | Primer parámetro del comando. | `0x2F` |
| **4** | Dato 2 (Ej. Coordenada Y) | Segundo parámetro del comando. | `0x78` |

*Nota sobre tolerancia a fallos: Si el receptor lee un primer byte distinto a la cabecera predefinida (`0xAA`), descartará el paquete completo para evitar corromper las físicas del juego.*

## 4. Diccionario de Comandos

Los comandos (enviados en el Byte 2 de la trama) serán utilizados para indicar qué tipo de información específica están procesando las FPGA. Se establecen los siguientes identificadores base:

* **`0x01` - Posición Local:** Envía las coordenadas del jugador dueño de la consola. (Bytes 3 y 4 representan X e Y).
* **`0x02` - Posición Remota:** Actualiza las coordenadas del contrincante.
* **`0x03` - Posición Pelota/Proyectil:** Sincroniza la ubicación de elementos móviles neutrales.
* **`0xF0` - Estado de Partida:** Control del flujo del juego. (El Byte 3 indica el evento: `0x01` Iniciar, `0x02` Pausar, `0x03` Game Over, `0x04` Reinicio).
* **`0xF1` - Sincronización de Red:** *Handshake* utilizado para establecer la conexión inicial entre las placas antes del renderizado.

## 5. Pendientes Técnicos (Plan de Acción)

* Diseñar la máquina de estados algorítmica (ASM) para los módulos Transmisor (TX) y Receptor (RX).
* Calcular e implementar el divisor de reloj en Verilog para alcanzar con exactitud los 115,200 bps.
* Programar las rutinas en lenguaje C dentro del firmware para el empaquetado y desempaquetado de los 4 bytes a través de los registros CSR del procesador FemtoRV32.
* Estructurar el *Testbench* para simular las formas de onda de transmisión y validar la recepción en GTKWave.
