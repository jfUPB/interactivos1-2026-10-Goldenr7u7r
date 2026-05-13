# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 01

<img width="238" height="256" alt="image" src="https://github.com/user-attachments/assets/ac2f3f93-0eb6-4d29-b166-629e7fe36865" />

``` js

function setup()
{
createCanvas(400,400);
} 

```

1) Un sistema físico interactivo es aquel donde se une el diseño y la tecnología para desarrollar una experiencia donde el usuario puede interactuar con él.
2) Puedo desarrollar animaciones donde los usuarios puedan elegir el flujo de desarrolló de la misma


### Actividad 02

1) El diseño/arte generativo es un programa en el que el usuario puede generar diseños o arte con base a un programa previamente desarrollado
2) Puedo realizar un programa donde el usuario pueda generar un personaje en 3D con diferentes características

### Actividad 03

<img width="1026" height="814" alt="image" src="https://github.com/user-attachments/assets/58ec88c8-c20b-4ef7-aef6-8235521fb13c" />

``` js
// Importa todas las funciones y objetos del módulo microbit
// Permite usar botones, pantalla LED, acelerómetro, UART, etc.
from microbit import *

// Inicializa la comunicación serial (UART)
// 115200 es la velocidad de transmisión en bits por segundo
uart.init(baudrate=115200)

// Muestra la imagen de una mariposa en la pantalla LED del micro:bit
display.show(Image.BUTTERFLY)

// Bucle infinito: el programa se ejecuta continuamente
while True:

    // Verifica si el botón A está presionado
    if button_a.is_pressed():

        // Envía el carácter 'A' por UART
        uart.write('A')

        // Espera 500 milisegundos para evitar múltiples envíos
        sleep(500)

    // Verifica si el botón B está presionado
    if button_b.is_pressed():

        // Envía el carácter 'B' por UART
        uart.write('B')

        // Espera 500 milisegundos
        sleep(500)

    // Verifica si se detectó el gesto de agitar el micro:bit
    if accelerometer.was_gesture('shake'):

        // Envía el carácter 'C' por UART
        uart.write('C')

        // Espera 500 milisegundos
        sleep(500)

    // Verifica si hay datos disponibles en el buffer UART
    if uart.any():

        // Lee 1 byte recibido por UART
        data = uart.read(1)

        // Comprueba que se haya recibido un dato válido
        if data:

            // Compara el byte recibido con el carácter 'h'
            // ord('h') convierte el carácter a su valor ASCII
            if data[0] == ord('h'):

                // Muestra un corazón en la pantalla LED
                display.show(Image.HEART)

                // Mantiene la imagen durante 500 milisegundos
                sleep(500)

                // Cambia la imagen a una cara feliz
                display.show(Image.HAPPY)

```

<img width="1379" height="922" alt="image" src="https://github.com/user-attachments/assets/31f7e9b9-d601-4640-b313-0c5178c3c43e" />

<img width="1917" height="936" alt="image" src="https://github.com/user-attachments/assets/df8b2724-04de-44bc-8e9b-625549a98de8" />

<img width="1039" height="781" alt="image" src="https://github.com/user-attachments/assets/a4fd74c1-e123-4455-8acd-22f5268583c6" />

<img width="1919" height="784" alt="image" src="https://github.com/user-attachments/assets/94d9f358-9432-43dd-b121-809d227f620f" />

``` js
// Variable para manejar la comunicación serial
let port;

// Botón para conectar y desconectar el micro:bit
let connectBtn;

// Función setup: se ejecuta una sola vez al iniciar el programa
function setup() {

    // Crea un lienzo de 400x400 píxeles
    createCanvas(400, 400);

    // Pinta el fondo de color gris claro
    background(220);

    // Crea el objeto de comunicación serial
    port = createSerial();

    // Crea el botón para conectar con el micro:bit
    connectBtn = createButton('Connect to micro:bit');

    // Posiciona el botón de conexión
    connectBtn.position(80, 300);

    // Asigna la función al evento de clic del botón
    connectBtn.mousePressed(connectBtnClick);

    // Crea un botón para enviar un mensaje al micro:bit
    let sendBtn = createButton('Send Love');

    // Posiciona el botón de envío
    sendBtn.position(220, 300);

    // Asigna la función que se ejecuta al presionar el botón
    sendBtn.mousePressed(sendBtnClick);

    // Establece el color de relleno a blanco
    fill('white');

    // Dibuja un círculo en el centro del canvas
    ellipse(width / 2, height / 2, 100, 100);
}

// Función draw: se ejecuta continuamente
function draw() {

    // Verifica si hay datos disponibles en el puerto serial
    if (port.availableBytes() > 0) {

        // Lee 1 byte de datos del puerto serial
        let dataRx = port.read(1);

        // Si el dato recibido es 'A'
        if (dataRx == 'A') {

            // Cambia el color del círculo a rojo
            fill('red');
        }

        // Si el dato recibido es 'B'
        else if (dataRx == 'B') {

            // Cambia el color del círculo a amarillo
            fill('yellow');
        }

        // Si se recibe cualquier otro valor
        else {

            // Cambia el color del círculo a verde
            fill('green');
        }

        // Limpia el fondo
        background(220);

        // Dibuja el círculo en el centro del canvas
        ellipse(width / 2, height / 2, 100, 100);

        // Cambia el color del texto a negro
        fill('black');

        // Muestra el carácter recibido en el centro del canvas
        text(dataRx, width / 2, height / 2);
    }

    // Actualiza el texto del botón según el estado del puerto
    if (!port.opened()) {

        // Si no está conectado, muestra "Connect to micro:bit"
        connectBtn.html('Connect to micro:bit');
    }
    else {

        // Si está conectado, muestra "Disconnect"
        connectBtn.html('Disconnect');
    }
}

// Función que se ejecuta al presionar el botón de conexión
function connectBtnClick() {

    // Si el puerto no está abierto
    if (!port.opened()) {

        // Abre la conexión serial con el micro:bit
        port.open('MicroPython', 115200);
    } 
    else {

        // Si el puerto está abierto, lo cierra
        port.close();
    }
}

// Función que se ejecuta al presionar el botón "Send Love"
function sendBtnClick() {

    // Envía el carácter 'h' al micro:bit por el puerto serial
    port.write('h');
}

```

### Actividad 4

- Paso 1

Creación del programa del micro:bit

Vamos a crear un programa en el micro:bit que tenga un input (un botón) y un output (enviar un mensaje por serial) que indique que se ha presionado el botón.
Lo primero que debes hacer es abrir el editor de micro:bit. Luego, importa de la biblioteca microbit todas las funciones que necesitas para interactuar con el micro:bit.

<img width="2752" height="1808" alt="image" src="https://github.com/user-attachments/assets/a0713738-e301-493e-b675-05cc7dce3b05" />

- Paso 2

Lectura del botón

Ahora vamos a leer el estado del botón A del micro:bit. Para ello, utilizaremos un bucle que se ejecutará continuamente y verificará si el botón A ha sido presionado.

![image](https://github.com/user-attachments/assets/19bc162a-146d-4d0c-9d6a-82229a313781)

Observa que utilizamos button_a.was_pressed() para detectar si el botón ha sido presionado. También podrías usar button_a.is_pressed() si quieres saber si el botón está presionado en ese momento, pero was_pressed() es más adecuado para detectar eventos únicos como un clic. Si usas is_pressed(), el programa podría enviar múltiples mensajes si el botón se mantiene presionado.

- Paso 3

Lectura del botón y envío del mensaje

Ahora que sabemos cuándo se presiona el botón, vamos a enviar un mensaje por el puerto serial del micro:bit. Esto nos permitirá recibir el mensaje en p5.js.
Nota que debes inicializar la comunicación serial con uart.init(baudrate=115200) antes de enviar mensajes. El baudrate es la velocidad de transmisión de datos, y 115200 es una velocidad comúnmente utilizada para la comunicación serial. Finalmente, utilizamos uart.write('A') para enviar el mensaje ‘A’ cuando se presiona el botón A.

<img width="2752" height="1806" alt="image" src="https://github.com/user-attachments/assets/d866867f-1d62-4b54-9db2-5f7194aceb11" />

- Paso 4

Aplicación en p5.js - Biblioteca de conexión serial

Lo primero que debes hacer es añadir la biblioteca que te permite conectar el micro:bit con p5.js. Recuerda que esto lo haces en el archivo index.html de tu proyecto p5.js.

<img width="2752" height="1808" alt="image" src="https://github.com/user-attachments/assets/124c0ad7-aff7-4524-a833-0806ddaae039" />

- Paso 5

Aplicación en p5.js - Configuración inicial

Vas a crear un par de variables globales (estas las creas por fuera de cualquier función). Al ser globales, podrás acceder a ellas desde cualquier parte del código. Estas variables te servirán para almacenar una referencia al objeto que te permitirá manipular el puerto serial y la otra variable te servirá para guardar una referencia al objeto que representa el botón con el cual podrías conectar y desconectar el micro:bit de la aplicación.
Observa la función connectBtnClick(). Esta función se ejecuta cuando el usuario hace click en el botón de conexión. A esto se le llama un “event handler” o manejador de eventos.

``` js
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
}

function draw() {
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}
```

<img width="2752" height="1810" alt="image" src="https://github.com/user-attachments/assets/0d208a6f-ce41-4e14-b137-c0153cbb6832" />

- Paso 6

Aplicación en p5.js - Dibujar en cada frame

En draw() vas a dibujar un cuadrado en la pantalla todo el tiempo, pero tendrás que decidir en cada frame qué color debe tener. Entonces, antes de dibujarlo vas a leer el puerto serial y ver si hay algún mensaje disponible. Si hay un mensaje, lo vas a leer (debes siempre consumir el mensaje para que no se acumule) y cambiarás el color a rojo. Si no hay mensaje, el cuadrado será de color verde.
Antes de terminar el frame vas a actualizar el estado del botón para que refleje el estado de conexión del micro:bit.

<img width="2752" height="1806" alt="image" src="https://github.com/user-attachments/assets/6a0c03ad-a2d2-4559-9c15-a98a812af5f3" />

- Paso 7

Aplicación en p5.js - Dibujar en cada frame

Cuando se presiona el botón A del micro:bit, se envía un mensaje por el puerto serial, este mensaje se lee en un frame. En ese frame se le cambia el color al cuadrado. Pero, al siguiente frame, se lee el puerto serial y no hay mensajes disponibles, por lo que el cuadrado vuelve a ser verde.

``` js
from microbit import *

uart.init(baudrate=115200)

while True:

    if button_a.is_pressed():
        uart.write('A')
    else:
        uart.write('N')

    sleep(100)
```
<img width="1919" height="943" alt="image" src="https://github.com/user-attachments/assets/3aece9ba-b3cb-4006-bd35-a9f47211d122" />

``` js
   let port;
  let connectBtn;
  let connectionInitialized = false;

  function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton("Connect to micro:bit");
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
  }

  function draw() {
    background(220);

    if (port.opened() && !connectionInitialized) {
      port.clear();
      connectionInitialized = true;
    }

    if (port.availableBytes() > 0) {
      let dataRx = port.read(1);
      if (dataRx == "A") {
        fill("red");
      } else if (dataRx == "N") {
        fill("green");
      }
    }

    rectMode(CENTER);
    rect(width / 2, height / 2, 50, 50);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");
    } else {
      connectBtn.html("Disconnect");
    }
  }

  function connectBtnClick() {
    if (!port.opened()) {
      port.open("MicroPython", 115200);
      connectionInitialized = false;
    } else {
      port.close();
    }
  }
```
<img width="2752" height="1825" alt="image" src="https://github.com/user-attachments/assets/086e3730-8781-404f-a762-8a79a01fca34" />

### Actividad 5

<img width="1392" height="944" alt="image" src="https://github.com/user-attachments/assets/d13758b5-01d0-4a37-bc70-a0bd48dd2e1d" />

<img width="1030" height="819" alt="image" src="https://github.com/user-attachments/assets/dc1fc950-d18d-4cb7-9ef1-0d34d8d2327b" />

``` js
let port; // Variable que maneja la comunicación serial
let connectBtn; // Botón para conectar y desconectar la micro:bit
let x; // Posición horizontal del círculo

function setup() 
{
    createCanvas(800, 800); // Crear el canvas de 800x800 píxeles
  
    port = createSerial(); 
    // Inicializar el puerto serial
  
    x= width / 2;  // Posición inicial del círculo (centro de la pantalla)
  
    // Crear botón para conectar/desconectar la micro:bit
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(350, 200);
    connectBtn.mousePressed(connectBtnClick);   
}

function draw() 
{
  
    background(220);  // Limpiar el fondo
  
    // Dibujar el círculo rosado
    fill('pink');
    ellipse(x, height/2, 150, 150);
  
    if(port.availableBytes() > 0) // Dibujar el círculo rosado
    {
        let dataRx = port.read(1);  // Leer un byte del puerto serial
      
        if(dataRx == 'A')  // Interpretar el dato recibido
        {
            x -= 10; //mover a la izquierda
        }
        else if(dataRx == 'B')
        {
            x +=10; //mover a la derecha
        }

        x = constrain(x,0,width);  // Evitar que el círculo salga del canvas
              
     // Actualizar el texto del botón según el estado del puerto  
    if (!port.opened()) 
    {
        connectBtn.html('Connect to micro:bit');
    }
    else 
    {
        connectBtn.html('Disconnect');
    }      
    }    
}

function connectBtnClick() 
{
    if (!port.opened()) 
    {
        port.open('MicroPython', 115200); // Abrir conexión con la micro:bit
    } 
    else 
    {
        port.close();  // Cerrar la conexión serial
    } 
}
```
``` js
from microbit import *

uart.init(baudrate=115200)
display.show(Image.HAPPY)

while True:
    if button_a.was_pressed():
        uart.write('A')
        sleep(100)

    if button_b.was_pressed():
        uart.write('B')
        sleep(100)
```

### Actividad 6

``` js
// Importa todas las funciones y objetos del módulo microbit
// Permite usar botones, UART, sleep, etc.
from microbit import *

// Inicializa la comunicación UART (serial)
// 115200 es la velocidad de transmisión en bits por segundo
uart.init(baudrate=115200)

// Bucle infinito: el programa se ejecuta continuamente
while True:

    // Verifica si el botón A está presionado
    if button_a.is_pressed():
        
        // Si el botón A está presionado, envía el carácter 'A' por UART
        uart.write('A')
    
    else:
        // Si el botón A NO está presionado, envía el carácter 'N' por UART
        uart.write('N')

    // Detiene la ejecución durante 100 milisegundos
    // Evita enviar datos demasiado rápido
    sleep(100)

```

``` js
// Variable para manejar el puerto serial
let port;

// Botón para conectar o desconectar el micro:bit
let connectBtn;

// Indica si la conexión ya fue inicializada
let connectionInitialized = false;

// Función setup: se ejecuta una sola vez al iniciar el programa
function setup() {

  // Crea un lienzo de 400x400 píxeles
  createCanvas(400, 400);

  // Pinta el fondo de color gris claro
  background(220);

  // Crea el objeto de comunicación serial
  port = createSerial();

  // Crea un botón para conectarse al micro:bit
  connectBtn = createButton("Connect to micro:bit");

  // Posiciona el botón en la pantalla
  connectBtn.position(80, 300);

  // Asigna la función que se ejecutará al presionar el botón
  connectBtn.mousePressed(connectBtnClick);
}

// Función draw: se ejecuta continuamente (loop)
function draw() {

  // Limpia el fondo en cada frame
  background(220);

  // Si el puerto está abierto y aún no se ha inicializado la conexión
  if (port.opened() && !connectionInitialized) {

    // Limpia el buffer del puerto serial
    port.clear();

    // Marca la conexión como inicializada
    connectionInitialized = true;
  }

  // Verifica si hay datos disponibles en el puerto serial
  if (port.availableBytes() > 0) {

    // Lee 1 byte de datos del puerto serial
    let dataRx = port.read(1);

    // Si el dato recibido es "A"
    if (dataRx == "A") {

      // Cambia el color de relleno a rojo
      fill("red");

    // Si el dato recibido es "N"
    } else if (dataRx == "N") {

      // Cambia el color de relleno a verde
      fill("green");
    }
  }

  // Establece el modo de dibujo del rectángulo desde el centro
  rectMode(CENTER);

  // Dibuja un rectángulo centrado en el canvas
  rect(width / 2, height / 2, 50, 50);

  // Cambia el texto del botón según el estado del puerto
  if (!port.opened()) {

    // Si no está conectado, muestra "Connect to micro:bit"
    connectBtn.html("Connect to micro:bit");

  } else {

    // Si está conectado, muestra "Disconnect"
    connectBtn.html("Disconnect");
  }
}

// Función que se ejecuta al presionar el botón
function connectBtnClick() {

  // Si el puerto NO está abierto
  if (!port.opened()) {

    // Abre el puerto serial con el micro:bit usando baudrate 115200
    port.open("MicroPython", 115200);

    // Reinicia la inicialización de la conexión
    connectionInitialized = false;

  } else {

    // Si el puerto está abierto, lo cierra
    port.close();
  }
}

```


## Bitácora de aplicación 



## Bitácora de reflexión





