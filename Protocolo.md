# Protocolo de comunicación

## 1. Objetivo

Establecer las características de la comunicación serial entre las dos FPGA para el modo multijugador. 

A diferencia de una red de computadores convencional, implementaremos una arquitectura basada en consolas retro clásicas (inspirada en el Cable Link de la Game Boy de Nintendo). Esto implica una topología **Maestro - Esclavo**, donde una consola (la Principal) administra el ritmo de la comunicación y la otra (la Invitada) responde a sus solicitudes para intercambiar los datos en tiempo real y sin *lag*.

## 2. Comunicación mediante SIO (Serial I/O)

### 2.1 Características generales

Para este módulo usaremos el protocolo SIO síncrono. A diferencia de un SPI comercial, este diseño no usa cable de selección (Chip Select), simplificando la conexión a solo cuatro cables físicos entre las placas: 

* **SC (Serial Clock):** La señal de reloj.
* **SO (Serial Out):** Salida de datos.
* **SI (Serial In):** Entrada de datos.
* **GND:** Tierra común para igualar voltajes.

La conexión de datos es cruzada: el pin SO de la FPGA Maestra va al pin SI de la FPGA Esclava, y el SO de la Esclava va al SI de la Maestra. El reloj (SC) solo viaja de la Maestra hacia la Esclava.

**Figura 1. Conexión física entre las dos interfaces (Estilo Nintendo).**

![Conexión entre las consolas](imagenes/conexion_sio.png)

Como necesitamos que los botones y físicas de ambos jugadores viajen al mismo tiempo, esta arquitectura es estrictamente **full-duplex síncrona**.

**Figura 2. Tipos de comunicación según la dirección de transmisión.**

![Tipos de comunicación](imagenes/tipos_comunicacion.png)

### 2.2 Funcionamiento y Sincronización

La magia de este protocolo está en que se basa en un **Registro de Desplazamiento (Shift Register) de 8 bits** en cada FPGA. 

El flujo de datos funciona como un trueque simultáneo:
1. El código en C de la FPGA Maestra ordena enviar un dato.
2. El hardware (Verilog) de la Maestra genera exactamente 8 pulsos por el cable `SC`.
3. Por cada pulso de reloj, un bit sale por `SO` y al mismo tiempo un bit entra por `SI`.
4. Al terminar los 8 pulsos, ambas consolas han intercambiado exactamente 1 byte al mismo tiempo.

**Datos en paralelo → Shift Register (Maestra) ↔ SC / SI / SO ↔ Shift Register (Esclava) → Datos en paralelo**

**Figura 3. Proceso de intercambio de datos simultáneo.**

![Proceso de transmisión SIO](imagenes/transmision_sio.png)

Al ser una comunicación síncrona, no necesitamos bits de inicio ni de parada. La Esclava sabe exactamente cuándo leer el dato porque obedece a los flancos de subida y bajada de la señal de reloj `SC` que le manda la Maestra.

**Figura 4. Reloj y datos síncronos.**

![Comunicación síncrona](imagenes/comunicacion_sincrona.png)

### 2.3 Justificación técnica de la arquitectura

Implementar este diseño SIO en lugar de UART nos da estas ventajas a nivel de hardware:

* **Hardware mucho más simple:** No hay que adivinar los tiempos de los bits ni hacer *oversampling* (sobremuestreo). El diseño en Verilog se reduce a un registro de desplazamiento puro.
* **Intercambio 1 a 1 (Trueque):** Garantiza que si la consola 1 envía su posición, forzosamente recibe la posición de la consola 2 en el mismo ciclo exacto de reloj.
* **Frecuencia estable:** Todo se mueve al ritmo exacto que dicte el divisor de reloj de la FPGA Maestra, eliminando errores de desfase.

## 3. Estructura de la Trama de Datos (Payload)

Como el hardware intercambia la información de a 1 byte (8 bits) a la vez, organizamos los datos del juego en paquetes fijos de **4 bytes**. El procesador Maestro mandará a ejecutar la transferencia 4 veces seguidas para completar un paquete. No se envían imágenes ni gráficos, solo posiciones y estados lógicos.

| Byte | Función | Descripción | Ejemplo (Hex) |
| :--- | :--- | :--- | :--- |
| **1** | Cabecera | Byte fijo para indicarle al software que aquí inicia un mensaje. | `0xAA` |
| **2** | Comando / ID | Indica qué tipo de dato o evento se está enviando. | `0x01` |
| **3** | Dato 1 | Primer parámetro (por ejemplo, coordenada X). | `0x2F` |
| **4** | Dato 2 | Segundo parámetro (por ejemplo, coordenada Y). | `0x78` |

*Nota: Si el firmware recibe un paquete cuyo primer byte no sea `0xAA`, se descarta para no mover los elementos de la pantalla a posiciones erróneas.*

## 4. Comandos

Usaremos el Byte 2 para definir qué tipo de mensaje se está mandando entre las FPGAs:

* **`0x01` - Posición Local:** Envía la posición del jugador.
* **`0x02` - Posición Remota:** Actualiza la posición del rival en la pantalla.
* **`0x03` - Elementos neutros:** Coordenadas de objetos compartidos (como la pelota).
* **`0xF0` - Estado del juego:** Eventos como inicio, pausa, punto o fin de partida.
* **`0xF1` - Sincronización:** Mensaje inicial (*handshake*) para verificar que el cable está conectado antes de empezar a renderizar.

## 5. Pendientes

* Revisar los juegos que vamos a implementar para definir los datos exactos que necesita mandar cada placa.
* Definir cuántos bits requerimos por cada variable (posiciones, puntaje, etc.).
* Diseñar el bloque RTL (Verilog) del Registro de Desplazamiento (Shift Register).
* Implementar el generador de reloj en la FPGA Maestra para la señal `SC` y hacer que la FPGA Esclava se sincronice solo con esa entrada externa.
* Escribir el código en C (firmware) de la FPGA Maestra para que haga un sondeo continuo (*polling*) de la Esclava y así evitar *lag* en los controles del Jugador 2.
* Hacer las simulaciones en GTKWave verificando que la señal de reloj haga exactamente los 8 ciclos requeridos por cada byte antes de probar en el hardware real.
