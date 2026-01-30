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

<img width="472" height="568" alt="image" src="https://github.com/user-attachments/assets/4445222e-1078-4c74-af38-eaed72d4346e" />

<img width="1919" height="942" alt="image" src="https://github.com/user-attachments/assets/eda6876b-d548-4562-89ce-3901b5f0707c" />

<img width="1379" height="922" alt="image" src="https://github.com/user-attachments/assets/31f7e9b9-d601-4640-b313-0c5178c3c43e" />

<img width="1917" height="936" alt="image" src="https://github.com/user-attachments/assets/df8b2724-04de-44bc-8e9b-625549a98de8" />

<img width="1039" height="781" alt="image" src="https://github.com/user-attachments/assets/a4fd74c1-e123-4455-8acd-22f5268583c6" />

<img width="1919" height="784" alt="image" src="https://github.com/user-attachments/assets/94d9f358-9432-43dd-b121-809d227f620f" />

## Actividad 4

- Paso 1


## Actividad 5

<img width="1392" height="944" alt="image" src="https://github.com/user-attachments/assets/d13758b5-01d0-4a37-bc70-a0bd48dd2e1d" />

<img width="1030" height="819" alt="image" src="https://github.com/user-attachments/assets/dc1fc950-d18d-4cb7-9ef1-0d34d8d2327b" />

```
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
```
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
## Bitácora de aplicación 



## Bitácora de reflexión



