# Unidad 4

## Bitácora de proceso de aprendizaje

## Bitácora de aplicación 

### Actividad 2

<img width="1859" height="942" alt="Captura de pantalla 2026-03-12 185838" src="https://github.com/user-attachments/assets/edf221f9-1f2b-4e7c-8a50-6acb4bba5011" />

**Código micro:bit Python Editor**

``` .js
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:

    t = running_time()

    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()

    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0

    chk = abs(xValue) + abs(yValue) + aState + bState

    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(
        t, xValue, yValue, aState, bState, chk
    )

    uart.write(data)

    sleep(100)
```

**Código ``MicrobitASCIIAdapter2.js``**

``` .js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error {}

function parseAsciiFrame(line) {
  const raw = line.trim();

  if (!raw.startsWith("$")) {
    throw new ParseError("Frame must start with '$'");
  }

  const body = raw.slice(1);
  const parts = body.split("|");

  if (parts.length !== 6) {
    throw new ParseError(`Expected 6 fields, got ${parts.length}`);
  }

  const data = {};

  for (const part of parts) {
    const [key, value] = part.split(":");

    if (value === undefined) {
      throw new ParseError(`Invalid field format: ${part}`);
    }

    data[key] = value;
  }

  const requiredKeys = ["T", "X", "Y", "A", "B", "CHK"];
  for (const key of requiredKeys) {
    if (!(key in data)) {
      throw new ParseError(`Missing field: ${key}`);
    }
  }

  const t = Number(data.T);
  const x = Number(data.X);
  const y = Number(data.Y);
  const a = Number(data.A);
  const b = Number(data.B);
  const chk = Number(data.CHK);

  if (
    !Number.isInteger(t) ||
    !Number.isInteger(x) ||
    !Number.isInteger(y) ||
    !Number.isInteger(a) ||
    !Number.isInteger(b) ||
    !Number.isInteger(chk)
  ) {
    throw new ParseError("Invalid numeric data");
  }

  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) {
    throw new ParseError("Out of expected accelerometer range");
  }

  if (![0, 1].includes(a) || ![0, 1].includes(b)) {
    throw new ParseError("Invalid button data");
  }

  const calculatedChk = Math.abs(x) + Math.abs(y) + a + b;

  if (calculatedChk !== chk) {
    throw new ParseError(
      `Checksum mismatch: expected ${calculatedChk}, got ${chk}`
    );
  }

  return {
    x: x | 0,
    y: y | 0,
    btnA: a === 1,
    btnB: b === 1,
  };
}

class MicrobitASCIIAdapter2 extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
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
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    this.buf += chunk.toString("utf8");

    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {
        const parsed = parseAsciiFrame(line);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          console.warn("Corrupt or invalid frame discarded:", e.message, "raw:", line);
        } else {
          this._fail(e);
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
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

module.exports = MicrobitASCIIAdapter2;
```

**Código ``sketch2.js``**

``` .js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            strokeWeight(0.75);
            background(255);
            console.log("Microbit ready to draw");
            this.rxData = {
                x: 0,
                y: 0,
                btnA: false,
                btnB: false,
                prevA: false,
                prevB: false,
                ready: false
            };
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
    this.rxData.ready = true;

    // Guardamos datos crudos del microbit
    this.rxData.x = data.x;
    this.rxData.y = data.y;
    this.rxData.btnA = data.btnA;
    this.rxData.btnB = data.btnB;

    // Mapear eje Y -> resolución del polígono
    this.circleResolution = int(map(data.y, -2048, 2047, 2, 10));

    // Mapear eje X -> radio
    this.radius = map(data.x, -2048, 2047, -width/2, width/2);

    // Botón B -> relleno
    this.fillShape = data.btnB;
}
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(255);
    painter = new PainterTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {
        painter.postEvent({
            type: EVENTS.DATA, payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {

    let mb = painter.rxData;

    if (!mb.ready) return;

    // Solo dibuja cuando el botón A está presionado
    if (mb.btnA) {

        push();
        translate(width / 2, height / 2);

        let circleResolution = painter.circleResolution;
        let radius = painter.radius;
        let angle = TAU / circleResolution;

        if (painter.fillShape) {
            fill(34, 45, 122, 50);
        } else {
            noFill();
        }

        beginShape();

        for (let i = 0; i <= circleResolution; i++) {

            let x = cos(angle * i) * radius;
            let y = sin(angle * i) * radius;

            vertex(x, y);
        }

        endShape();

        pop();
    }
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

**Código ``index2.html``**

``` .html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Painter – Web Serial API</title>
  <link rel="stylesheet" href="style.css" />
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  <script src="fsm.js"></script>
  <script src="bridgeClient.js"></script>
  <script src="sketch2.js"></script>
</head>
<body>
</body>
</html>
``` 

**Código ``bridgeServer.js`` actualizado**

``` .js

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device microbit2 --wsPort 8081 --serialPort COM5 --baud 115200


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
## Bitácora de reflexión

