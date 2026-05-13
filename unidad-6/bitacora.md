## Bitácora de aplicación 

## Bitácora de reflexión

1. Cómo configuré Strudel para emitir eventos

    Para esta actividad utilicé Strudel como una segunda aplicación web encargada de generar eventos musicales en tiempo real. Strudel no se usó para renderizar nada directamente, sino únicamente como fuente de datos.

    Se configuró para que, al generarse eventos musicales, estos se enviaran a través del `bridgeServer.js` hacia el cliente visual. De esta manera, Strudel actuó como emisor de eventos y el sistema principal como receptor e intérprete.

2. Qué estructura final de mensaje decidí usar

    La estructura de los mensajes fue sencilla, enfocada en permitir la comunicación sin complicar la arquitectura existente.

    Se manejaron mensajes con información básica del evento, identificando que provenían de Strudel y conteniendo los datos necesarios para el comportamiento visual. No se implementó una estructura compleja, ya que el objetivo era lograr la integración funcional respetando el sistema ya construido.

    Esto permitió que `bridgeClient.js` pudiera interpretar fácilmente los datos y pasarlos al flujo del programa.

3. Cómo conecté `bridgeClient.js`, `FSMTask`, `updateLogic` y `drawRunning`

    La conexión se realizó respetando la arquitectura existente:

    - `bridgeClient.js` se encargó de recibir los mensajes desde el servidor.

    - Al recibir datos de Strudel, estos se procesaban y se integraban al flujo del programa.

    - La FSM (`FSMTask`) no fue modificada, tal como lo indicaba la actividad.

    - En `updateLogic` se actualizaron las variables necesarias con base en los datos recibidos.

    - En `drawRunning` se usaron esas variables para generar la representación visual en el canvas.

    De esta forma, cada componente mantuvo su responsabilidad sin romper la arquitectura original.

4. Cómo separé recepción, cola temporal y renderizado

    La separación se manejó de forma sencilla:

    - **Recepción:** ocurrió en `bridgeClient.js`, donde se escuchaban los eventos provenientes de Strudel.

    - **Procesamiento:** los datos recibidos se pasaban a variables dentro del flujo del programa en `updateLogic`.

    - **Renderizado:** en `drawRunning` se utilizaban esas variables para dibujar en pantalla.

    No se implementó una cola temporal compleja, pero sí se mantuvo una separación clara entre recibir los datos y utilizarlos para dibujar, evitando mezclar responsabilidades.

5. Qué pruebas hice para verificar la sincronización

    Se realizaron varias pruebas durante el desarrollo:

    - Verificar que Strudel estuviera enviando eventos correctamente.

    - Confirmar que `bridgeClient.js` recibiera los datos.

    - Revisar en consola que los mensajes llegaran con el formato esperado.

    - Validar que los eventos realmente afectaran el comportamiento visual.

    - Ejecutar ambas aplicaciones al mismo tiempo para comprobar que la comunicación fuera correcta.

    Estas pruebas permitieron asegurar que el flujo completo funcionara desde la generación del evento hasta su visualización.

6. Qué problemas encontré y cómo los solucioné

    Durante el proceso se presentaron varios problemas:

    - **Problema 1: No se estaba dibujando nada en el canvas**

    Aunque Strudel estaba funcionando, inicialmente no se reflejaba en el sketch.

    **Solución:** se revisó la conexión entre el bridge y el cliente, asegurando que los eventos fueran correctamente recibidos e interpretados.

    - **Problema 2: Confusión entre archivos (`index.html` vs `indexStrudel.html`)**

    Había duda sobre dónde debía integrarse el nuevo sketch sin afectar el funcionamiento anterior.

    **Solución:** se mantuvo la estructura original y se separó la implementación de Strudel sin romper el flujo existente.

    - **Problema 3: Integración sin modificar la FSM**

    Era importante no alterar la máquina de estados.

    **Solución:** se adaptó únicamente la recepción y uso de datos, manteniendo intacta la lógica de estados.

# ¿Qué se modificó/añadió?

## `bridgeServer`

1. **Import del nuevo adapter**

``` .js
const StrudelAdapter = require("./adapters/StrudelAdapter");
```

2. **Nuevo argumento para el puerto de Strudel**

``` .js
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);
```

3. **Nuevo caso en createAdapter()**

``` .js
if (DEVICE === "strudel") {
  log.info(`Strudel adapter will listen on ws://127.0.0.1:${STRUDEL_PORT}`);

  return new StrudelAdapter({
    port: STRUDEL_PORT,
    verbose: VERBOSE
  });
}
```

4. **Cambio en adapter.onData**

**Antes:**

``` .js
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
```

**Después:**

``` .js
adapter.onData = (d) => {
  if (DEVICE === "strudel") {
    broadcast(wss, d);
    return;
  }

  broadcast(wss, {
    type: "microbit",
    x: d.x,
    y: d.y,
    btnA: !!d.btnA,
    btnB: !!d.btnB,
    t: nowMs(),
  });
};
```

5. **Auto-conexión del adapter en modo Strudel**

**Antes:**

``` .js
if (DEVICE === "sim") {
  await adapter.connect();
}
```

**Después:**

``` .js
if (DEVICE === "sim" || DEVICE === "strudel") {
  await adapter.connect();
}
```

## `StrudelAdapter.js`

**Código nuevo:**

``` .js
const { WebSocketServer } = require("ws");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error {}

function argsToObject(args) {
  if (!Array.isArray(args)) {
    throw new ParseError("args must be an array");
  }

  const obj = {};
  for (let i = 0; i < args.length; i += 2) {
    const key = args[i];
    const value = args[i + 1];
    obj[key] = value;
  }

  return obj;
}

function normalizeSound(source) {
  const s = String(source || "").toLowerCase();

  if (s.includes("bd")) return "bd";
  if (s.includes("sd")) return "sd";
  if (s.includes("cp")) return "cp";
  if (s.includes("hh")) return "hh";

  return "other";
}

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080, verbose = false } = {}) {
    super();
    this.portNumber = port;
    this.verbose = verbose;
    this.wss = null;
  }

  async connect() {
    if (this.connected) return;

    this.wss = new WebSocketServer({ port: this.portNumber });

    this.wss.on("connection", (ws) => {
      if (this.verbose) {
        console.log(`Strudel connected on ws://127.0.0.1:${this.portNumber}`);
      }

      ws.on("message", (raw) => {
        this._handleMessage(raw);
      });

      ws.on("error", (err) => {
        this._fail(err);
      });

      ws.on("close", () => {
        if (this.verbose) {
          console.log("Strudel websocket client disconnected");
        }
        this.onDisconnected?.("strudel client disconnected");
      });
    });

    this.wss.on("error", (err) => {
      this._fail(err);
    });

    this.connected = true;
    this.onConnected?.(`ws server open ${this.portNumber}`);
  }

  async disconnect() {
    if (!this.connected) return;

    this.connected = false;

    if (this.wss) {
      await new Promise((resolve, reject) => {
        this.wss.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }

    this.wss = null;
    this.onDisconnected?.("ws server closed");
  }

  getConnectionDetail() {
    return `ws server ${this.portNumber}`;
  }

  _handleMessage(raw) {
    let msg;

    try {
      msg = JSON.parse(raw.toString("utf8"));
    } catch (e) {
      if (this.verbose) {
        console.log("Bad JSON from Strudel:", raw.toString("utf8"));
      }
      return;
    }

    try {
      const normalized = this._normalize(msg);
      if (normalized) {
        this.onData?.(normalized);
      }
    } catch (e) {
      if (e instanceof ParseError) {
        if (this.verbose) {
          console.log("Bad Strudel data:", e.message, "raw:", msg);
        }
      } else {
        this._fail(e);
      }
    }
  }

  _normalize(msg) {
    if (!msg || msg.address !== "/dirt/play") {
      return null;
    }

    const params = argsToObject(msg.args || []);
    const source = params.s || "unknown";
    const sound = normalizeSound(source);

    return {
      type: "strudel",
      timestamp: Number(msg.timestamp || Date.now()),
      payload: {
        sound,
        source,
        delta: Number(params.delta || 0.25),
        cycle: Number(params.cycle || 0),
        cps: Number(params.cps || 0),
        bank: String(params.bank || "")
      }
    };
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
  }

  async handleCommand(cmd) {
    return;
  }
}

module.exports = StrudelAdapter;
```

## `bridgeClient.js`

**Modificación para que también acepte mensajes de tipo `strudel`**

**Antes:**

``` .js
if (msg.type === "microbit") {
  this._onData?.(msg);
  return;
}
```

**Después:**

``` .js
if (msg.type === "microbit" || msg.type === "strudel") {
  this._onData?.(msg);
  return;
}
```

## `sketchStrudel.js`

1. **Nuevo evento FSM**
   
``` .js
STRUDEL_EVENT
```

2. **Cola de eventos**
   
``` .js
this.eventQueue = [];
```

3. **Lista de animaciones activas**
   
``` .js
this.activeAnimations = [];
```

4. `enqueueEvent(data)`
   
``` .js
this.eventQueue.push({
  timestamp: data.timestamp,
  sound: data.payload.sound,
  source: data.payload.source,
  delta: data.payload.delta || 0.25
});
```

5.  `updateLogic()`
   
``` .js
  updateLogic() {
        if (!this.ready) return;

        const now = Date.now() + this.latencyCorrection;

        while (this.eventQueue.length > 0 && now >= this.eventQueue[0].timestamp) {
            const ev = this.eventQueue.shift();

            this.activeAnimations.push({
                startTime: ev.timestamp,
                duration: ev.delta * 1000,
                type: ev.sound,
                source: ev.source,
                x: random(width * 0.2, width * 0.8),
                y: random(height * 0.2, height * 0.8),
                color: this.getColorForSound(ev.sound)
            });
        }

        for (let i = this.activeAnimations.length - 1; i >= 0; i--) {
            const anim = this.activeAnimations[i];
            const elapsed = now - anim.startTime;
            const progress = elapsed / anim.duration;

            if (progress > 1.0) {
                this.activeAnimations.splice(i, 1);
            } else {
                anim.progress = progress;
            }
        }
    }
```

6.  `bridge.onData(...)`
   
**Antes:**

``` .js
type: EVENTS.DATA,
payload: {
  x, y, btnA, btnB
}
```

**Después:**

``` .js
bridge.onData((data) => {
    if (data.type === "strudel") {
        painter.postEvent({
            type: EVENTS.STRUDEL_EVENT,
            payload: data
        });
    }
});
```

7. **`drawRunning()`**
   
``` .js
function drawRunning() {
    background(0, 30);

    for (const anim of painter.activeAnimations) {
        dibujarElemento(anim, anim.progress ?? 0);
    }
}
```

## `indexStrudel.html`

``` .js
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Painter – Strudel Visuals</title>
  <link rel="stylesheet" href="style.css" />
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  <script src="fsm.js"></script>
  <script src="bridgeClient.js"></script>
  <script src="sketchStrudel.js"></script>
</head>
<body>
</body>
</html>
```

## Bitácora de reflexión
