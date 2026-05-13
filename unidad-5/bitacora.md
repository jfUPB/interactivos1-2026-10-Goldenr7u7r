# Unidad 5
## Bitácora de proceso de aprendizaje

#### Actividad 1

<img width="1862" height="940" alt="image" src="https://github.com/user-attachments/assets/f2e6212f-897d-4c6e-9555-40cf184ec4c0" />

<img width="1837" height="939" alt="image" src="https://github.com/user-attachments/assets/23729c3b-1aab-4684-8bb9-b2673008af8f" />

## Bitácora de aplicación 

### Microbit editor

``` python
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0, 0, 9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    checksum = sum(data) % 256
    packet = b'\xAA' + data + bytes([checksum])
    uart.write(packet)
    sleep(100)
```

### MicrobitBinaryAdapter.js

``` .js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }

    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

   _onChunk(chunk) {
  console.log("chunk recibido:", chunk);
  console.log("hex:", chunk.toString("hex"));

  this.buf = Buffer.concat([this.buf, chunk]);

  while (this.buf.length > 0) {
    const headerIndex = this.buf.indexOf(0xAA);

    if (headerIndex < 0) {
      console.warn("No se encontró header en buffer:", this.buf.toString("hex"));
      this.buf = Buffer.alloc(0);
      return;
    }

    if (headerIndex > 0) {
      console.warn("Descartando bytes antes del header:", this.buf.slice(0, headerIndex).toString("hex"));
      this.buf = this.buf.slice(headerIndex);
    }

    if (this.buf.length < 8) {
      console.log("Esperando más bytes. Buffer actual:", this.buf.toString("hex"));
      return;
    }

    const frame = this.buf.slice(0, 8);
    console.log("Frame candidato:", frame.toString("hex"));

    let calculatedChecksum = 0;
    for (let i = 1; i <= 6; i++) {
      calculatedChecksum = (calculatedChecksum + frame[i]) % 256;
    }

    const receivedChecksum = frame[7];
    console.log("Checksum calculado:", calculatedChecksum, "recibido:", receivedChecksum);

    if (calculatedChecksum !== receivedChecksum) {
      console.warn("Trama corrupta, descartando:", frame.toString("hex"));
      this.buf = this.buf.slice(1);
      continue;
    }

    const x = frame.readInt16BE(1);
    const y = frame.readInt16BE(3);
    const btnA = frame[5] === 1;
    const btnB = frame[6] === 1;

    console.log("Paquete válido:", { x, y, btnA, btnB });

    this.onData?.({ x, y, btnA, btnB });

    this.buf = this.buf.slice(8);
  }
}

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitBinaryAdapter;
```

### bridgeServer.js

``` .js

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device microbit2 
//     node bridgeServer.js --device microbitbinary

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
// const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const MicrobitASCIIAdapter2 = require("./adapters/MicrobitASCIIAdapter2");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

const log = 
{
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) 
{
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) 
{
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() 
{
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  // if (DEVICE === "microbit-bin") {
  //   const path = SERIAL_PATH ?? await findMicrobitPort();
  //   if (!path) {
  //     log.error("micro:bit not found. Use --serialPort to specify manually.");
  //     process.exit(1);
  //   }
  //   return new MicrobitBinaryAdapter({ path, baud: BAUD });
  // }
  if (DEVICE === "microbit2") {

    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }

    log.info(`micro:bit2 found at ${path}`);

    return new MicrobitASCIIAdapter2({
      path,
      baud: BAUD,
      verbose: VERBOSE
    });
  }

  if (DEVICE === "microbitbinary") {

    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }

    log.info(`micro:bitbinary found at ${path}`);

    return new MicrobitBinaryAdapter({
      path,
      baud: BAUD,
      verbose: VERBOSE
    });
  }
  return new SimAdapter({ hz: SIM_HZ });
}

async function main() { 
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Device Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Device Error: ${detail}`);
    status(wss, "error", detail);
  };

  adapter.onData = (d) => {
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state = adapter.connected ? "connected" : "ready";

    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);

        if (adapter.connected) {
          log.info(`[HW-POLICY] Adapter already open. Sending current status to incoming client.`);
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        
        try {
          await adapter.connect();
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
      
      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapater kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        
        try {
          await adapter.disconnect();
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        adapter.disconnect();
      }
    });
  });

  if (DEVICE === "sim") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
}); 
```
#### Actividad 02 (Adapter binario)

**Predicción**

Se esperaba que el nuevo firmware del micro:bit enviara datos en formato binario en lugar de texto ASCII, por lo que el adapter anterior no sería capaz de interpretarlos correctamente.

Se predijo que sería necesario:

- Acumular los datos en un buffer de bytes.
  
- Identificar el inicio de cada paquete mediante un byte de sincronización (`0xAA`).

- Esperar hasta tener la longitud completa del paquete (8 bytes).

- Validar la integridad de los datos usando un checksum.

- Convertir los valores binarios a variables utilizables (`x`, `y`, `btnA`, `btnB`).

**Ejecución**

Se creó un nuevo archivo llamado:

adapters/MicrobitBinaryAdapter.js

Este adapter:

- Hereda de `BaseAdapter`.

- Abre el puerto serial a 115200 baudios.

- Acumula los bytes entrantes en un buffer (`Buffer` de Node.js).

- Implementa lógica de framing usando el header `0xAA`.

- Verifica el checksum de cada paquete.

- Emite los datos con el mismo formato que el adapter ASCII:

this.onData?.({ x, y, btnA, btnB });

También se modificó `bridgeServer.js` para registrar el nuevo adapter en el caso  `"microbitbinary"`.

**Problemas encontrados**

1. **No se observaban datos en consola**

Inicialmente, el sistema mostraba que el puerto serial se abría correctamente, pero no se veían datos.

Causa:

No había evidencia de que el método `_onChunk` estuviera procesando los datos.

Solución:

Se agregaron logs dentro de `_onChunk` para verificar la llegada de datos desde el micro:bit.

2. **Datos incompletos**

Los datos del puerto serial no siempre llegan en paquetes completos.

Causa:

La comunicación serial envía fragmentos de datos.

Solución:

Se implementó un buffer acumulativo:

this.buf = Buffer.concat([this.buf, chunk]);

Y se procesan los datos solo cuando hay al menos 8 bytes disponibles.

3. **Desincronización de paquetes**

El buffer podía contener datos que no empezaban en el header correcto.

Causa:

Lectura parcial o ruido en la transmisión.

Solución:

Se buscó el byte de inicio `0xAA` y se descartaron los bytes anteriores para re-sincronizar el stream.

4. **Validación de checksum**

Era necesario garantizar que los datos no estuvieran corruptos.

Solución:

Se implementó la validación del checksum como:

(suma de bytes 1 a 6) % 256

Si el checksum no coincide, la trama se descarta y se registra una advertencia en consola.

**Observación**

Después de implementar el adapter, el sistema logró:

- Conectarse correctamente al micro:bit.

- Recibir datos binarios.

- Interpretar correctamente los valores del acelerómetro y los botones.

- Enviar los datos al sistema sin modificar otros componentes.

Ejemplo observado en consola:

[ADAPTER] Device Connected: serial open COM12 @115200
Paquete válido: { x: 120, y: -300, btnA: false, btnB: true }

También se observaron mensajes de error cuando el checksum no coincidía:

Bad checksum, discarding frame

Esto confirma que el sistema detecta y descarta correctamente datos corruptos.

**Resultado**

El sistema funciona correctamente con el nuevo protocolo binario sin necesidad de modificar la arquitectura existente.

El nuevo adapter:

- Mantiene el mismo contrato de salida.

- Es compatible con el resto del sistema sin cambios.

- Maneja correctamente errores y desincronización.

**Reflexión**

El cambio de protocolo de ASCII a binario no requirió modificar todo el sistema, sino únicamente crear un nuevo adapter siguiendo la misma interfaz.

Esto demuestra la importancia de una arquitectura basada en adaptadores, ya que permite extender el sistema para soportar nuevos formatos sin afectar los demás componentes.

**Evidencia**

<img width="901" height="526" alt="image" src="https://github.com/user-attachments/assets/82c66f38-8e20-4f09-be4a-b5acc9f6036a" />

<img width="782" height="476" alt="image" src="https://github.com/user-attachments/assets/67fcd32c-5813-4aea-9a9b-2aa6a0cdc0bc" />

<img width="1873" height="1032" alt="image" src="https://github.com/user-attachments/assets/1f256be2-9b24-428a-975c-43c696e17e87" />

<img width="1862" height="1027" alt="image" src="https://github.com/user-attachments/assets/86074e0a-a500-42c5-9882-e070f47bfd35" />

<img width="1862" height="1025" alt="image" src="https://github.com/user-attachments/assets/c1813686-bc06-4a2c-a4d7-6da5fc51c65d" />

## Bitácora de reflexión
