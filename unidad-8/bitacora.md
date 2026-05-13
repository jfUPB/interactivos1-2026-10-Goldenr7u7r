# Unidad 8

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

# Bitácora Unidad 8 - Iteración ingenieril

## 1. Diagrama inicial de la arquitectura

La arquitectura de esta iteración integra tres fuentes de entrada: **micro:bit**, **Strudel** y **Open Stage Control**. Cada fuente entra al sistema por medio de su propio adapter, respetando la arquitectura trabajada durante el curso.

```text
micro:bit
   ↓
MicrobitASCIIAdapter2
   ↓
bridgeServer.js
   ↓
bridgeClient.js
   ↓
FSMTask
   ↓
updateLogic()
   ↓
drawRunning()

Strudel
   ↓
StrudelAdapter
   ↓
bridgeServer.js
   ↓
bridgeClient.js
   ↓
FSMTask
   ↓
updateLogic()
   ↓
drawRunning()

Open Stage Control
   ↓
OSCAdapter
   ↓
bridgeServer.js
   ↓
bridgeClient.js
   ↓
FSMTask
   ↓
updateControls() / updateLogic()
   ↓
drawRunning()
```

En esta estructura, `bridgeServer.js` funciona como una capa de transporte y distribución de mensajes. La interpretación visual no se realiza en el servidor, sino en el frontend, principalmente dentro de `FSMTask`, `updateLogic()` y `drawRunning()`.

---

## 2. Adapters utilizados

Para esta iteración se definió el modo de ejecución `microbitstrudelosc` en `bridgeServer.js`. Este modo instancia tres adapters independientes:

```js
new MicrobitASCIIAdapter2({
  path,
  baud: BAUD,
  verbose: VERBOSE
}),

new StrudelAdapter({
  port: STRUDEL_PORT,
  verbose: VERBOSE
}),

new OSCAdapter({
  port: OSC_PORT,
  verbose: VERBOSE
})
```

Los adapters usados son:

### MicrobitASCIIAdapter2

Se usa para recibir los datos seriales enviados por la micro:bit. Este adapter espera un mensaje con encabezado, campos definidos y checksum. Su responsabilidad es validar el frame, parsear los datos y entregar valores normalizados al bridge.

### StrudelAdapter

Se usa para recibir eventos musicales enviados desde Strudel. El adapter escucha mensajes en WebSocket y normaliza los sonidos en familias musicales como `kick`, `snare`, `clap`, `hat`, `bass`, `synth`, `fx`, entre otras.

### OSCAdapter

Se usa para recibir mensajes desde Open Stage Control. Estos mensajes modifican parámetros persistentes del sistema visual, como color, tamaño, rastro y color de relleno de la figura generada por la micro:bit.

---

## 3. Contrato de mensajes de cada fuente

### Contrato micro:bit

La micro:bit envía datos por serial a 115200 baudios. El formato usado para `MicrobitASCIIAdapter2` es:

```text
$T:<tiempo>|X:<valorX>|Y:<valorY>|A:<botonA>|B:<botonB>|CHK:<checksum>
```

Ejemplo:

```text
$T:12|X:-120|Y:300|A:1|B:0|CHK:421
```

El adapter valida que:

- El mensaje empiece con `$`.
- Contenga los campos `T`, `X`, `Y`, `A`, `B` y `CHK`.
- Los valores de acelerómetro estén dentro del rango esperado.
- Los botones estén representados como `0` o `1`.
- El checksum coincida con la suma definida.

Después de normalizar, el mensaje que llega al frontend tiene esta forma:

```js
{
  type: "microbit",
  x: -120,
  y: 300,
  btnA: true,
  btnB: false,
  t: 123456789
}
```

En el sketch, la micro:bit controla la figura principal:

- `x` modifica el radio del polígono.
- `y` modifica la resolución del polígono.
- `btnA` activa el dibujo de la figura.
- `btnB` activa el relleno de la figura.

---

### Contrato Strudel

El `StrudelAdapter` espera mensajes con dirección:

```text
/dirt/play
```

El formato esperado es similar a:

```js
{
  address: "/dirt/play",
  timestamp: Date.now(),
  args: [
    "s", "bd",
    "delta", 0.25,
    "cycle", 0,
    "cps", 0.5
  ]
}
```

El adapter convierte los argumentos en un objeto y normaliza el sonido. El mensaje que llega al frontend tiene esta forma:

```js
{
  type: "strudel",
  timestamp: 123456789,
  payload: {
    sound: "kick",
    cleanSound: "bd",
    source: "bd",
    delta: 0.25,
    cycle: 0,
    cps: 0.5,
    bank: ""
  }
}
```

La propiedad `sound` clasifica el sonido en una familia visual. Algunas categorías usadas son:

```text
kick, snare, clap, hat, cymbal, tom, rim, perc, bass, pluck, synth, pad, vocal, fx, sample
```

En el sketch, estas categorías se asocian con figuras distintas, por ejemplo:

```text
kick → círculo grande
clap/snare → barra horizontal
hat → cuadrado pequeño
cymbal → aro
perc → triángulo
rim → cruz
bass → franja grave
synth → onda
fx → rayo
```

---

### Contrato Open Stage Control

Open Stage Control envía mensajes OSC que llegan al frontend con esta estructura general:

```js
{
  type: "osc",
  payload: {
    address: "/rgb_1",
    args: [255, 120, 30]
  }
}
```

Los controles usados en esta iteración son:

```text
/rgb_1
```

Modifica el color de las figuras generadas por los eventos de Strudel.

```text
/visual_size
```

Modifica el tamaño de las figuras generadas por Strudel.

```text
/trail
```

Activa o desactiva el rastro visual del fondo.

```text
/microbit_fill_rgb
```

Modifica el color de relleno de la figura generada por la micro:bit.

Esto permite que Open Stage Control afecte tanto los eventos de Strudel como la visual principal de la micro:bit.

---

## 4. Pruebas técnicas básicas de integración

### Prueba 1: conexión del bridge

Se ejecutó el bridge con el modo `microbitstrudelosc`:

```bash
node bridgeServer.js --device microbitstrudelosc --serialPort COM14 --baud 115200 --strudelPort 8080 --oscPort 9000 --wsPort 8081
```

Resultado esperado:

```text
WS listening on ws://127.0.0.1:8081 device=microbitstrudelosc
micro:bit2 found at COM14
Strudel adapter will listen on ws://127.0.0.1:8080
OSC adapter will listen on udp://127.0.0.1:9000
Device Connected: serial open COM14 @115200
Device Connected: ws server open 8080
Device Connected: osc udp server open 9000
```

Con esta prueba se verificó que los tres adapters se instancian correctamente desde `bridgeServer.js`.

---

### Prueba 2: recepción de datos de micro:bit

Se probó la micro:bit usando el formato requerido por `MicrobitASCIIAdapter2`.

Código usado en micro:bit:

```python
from microbit import *

uart.init(115200)
display.set_pixel(0, 0, 9)

t = 0

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()

    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0

    chk = abs(xValue) + abs(yValue) + aState + bState

    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(
        t,
        xValue,
        yValue,
        aState,
        bState,
        chk
    )

    uart.write(data)

    t = t + 1
    sleep(100)
```

Resultado: el sistema recibió valores de `x`, `y`, `btnA` y `btnB`, y la figura del micro:bit pudo modificarse en el sketch.

---

### Prueba 3: recepción de mensajes OSC

Se probó Open Stage Control con un nuevo archivo JSON para la Unidad 8. Este incluye controles para:

```text
/rgb_1
/microbit_fill_rgb
/visual_size
/trail
```

Resultado: los controles OSC modificaron parámetros persistentes del sistema visual. En particular, se agregó un control nuevo para modificar el color de relleno de la figura generada por la micro:bit.

---

### Prueba 4: recepción de eventos Strudel

Se verificó que el `StrudelAdapter` queda escuchando en:

```text
ws://127.0.0.1:8080
```

También se identificó que el hecho de que Strudel produzca sonido no implica automáticamente que esté enviando eventos visuales al adapter. Para que el sketch pinte eventos de Strudel, los mensajes deben llegar al `StrudelAdapter` con dirección `/dirt/play`.

---

## 5. Errores encontrados y soluciones

### Error 1: confusión entre MicrobitASCIIAdapter y MicrobitASCIIAdapter2

Al principio se intentó usar `MicrobitASCIIAdapter2` con un código de micro:bit que enviaba datos en formato CSV simple:

```text
x,y,False,False
```

Esto produjo errores como:

```text
Corrupt or invalid frame discarded: Frame must start with '$'
```

La causa fue que `MicrobitASCIIAdapter2` no espera CSV simple, sino frames con encabezado `$`, campos separados por `|` y checksum.

Solución:

Se cambió el código de la micro:bit para que enviara mensajes con este formato:

```text
$T:<t>|X:<x>|Y:<y>|A:<a>|B:<b>|CHK:<chk>
```

---

### Error 2: nombres incorrectos en bridgeServer.js

Durante la creación del modo `microbitstrudelosc`, aparecieron errores por nombres de variables incorrectos, por ejemplo:

```text
ReferenceError: MicrobitASCIIAdapter is not defined
ReferenceError: SERIAL_PORT is not defined
```

La causa fue usar nombres que no existían en el archivo. En el bridge se usaban variables como:

```js
SERIAL_PATH
BAUD
```

pero se habían escrito accidentalmente nombres como:

```js
SERIAL_PORT
BAUD_RATE
```

Solución:

Se corrigió el bloque `microbitstrudelosc` para usar los mismos nombres usados por los modos ya existentes:

```js
const path = SERIAL_PATH ?? await findMicrobitPort();

new MicrobitASCIIAdapter2({
  path,
  baud: BAUD,
  verbose: VERBOSE
})
```

---

### Error 3: falta de separación entre sonido y visual

Al inicio, varios sonidos de Strudel llegaban al frontend como `other`, porque `StrudelAdapter` solo reconocía `bd`, `sd`, `cp` y `hh`.

Solución:

Se amplió la función `normalizeSound()` para clasificar los sonidos en familias más generales. Esto permite que el diseño sonoro de Strudel sea más variado y que cada familia tenga una traducción visual distinta.

---

### Error 4: texto visual no deseado en las figuras

En el sketch aparecían palabras o nombres de eventos girando junto con las figuras. Esto ocurría porque en la función `dibujarDefault()` se estaba dibujando texto después de aplicar rotación.

Solución:

Se eliminó el texto y se reemplazó por una figura geométrica tipo estrella. Esto hizo que la visual se viera más limpia y más adecuada para una performance.

---

## 6. Estado actual de la iteración

Al cierre de esta iteración ingenieril, el sistema ya cuenta con:

- Un modo de ejecución `microbitstrudelosc` en `bridgeServer.js`.
- Tres adapters independientes para micro:bit, Strudel y Open Stage Control.
- Un sketch fusionado llamado `sketchMicrobitStrudelOSC.js`.
- Una interfaz nueva de Open Stage Control para la Unidad 8.
- Recepción funcional de micro:bit.
- Recepción funcional de OSC.
- Estructura preparada para recibir eventos de Strudel.

El sistema mantiene la arquitectura del curso porque cada fuente entra por su adapter correspondiente, `bridgeServer.js` funciona como transporte, `bridgeClient.js` recibe los mensajes normalizados, `FSMTask` organiza los eventos, `updateLogic()` actualiza el estado y `drawRunning()` dibuja usando ese estado.
## Bitácora de reflexión
