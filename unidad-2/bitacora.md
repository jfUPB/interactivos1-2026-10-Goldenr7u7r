# Unidad 2

## Bitácora de proceso de aprendizaje

#### Actividad 1

---

##### Análisis de una máquina de estados simple

En este programa se implementa una máquina de estados para controlar el parpadeo de dos pixeles del display del micro:bit a diferentes velocidades. Cada pixel es una instancia de la clase Pixel, y cada uno posee su propio temporizador (Timer). El uso de Timer permite manejar el tiempo sin utilizar sleep() bloqueante, lo que hace posible que ambos pixeles funcionen de manera concurrente.

**1. ¿Cuáles son los estados en el programa?**

En la primera versión del programa existe un único estado:

estado_waitTimeout

En este estado el pixel se enciende al entrar y, cuando ocurre el evento Timeout, alterna entre encendido y apagado.

En la segunda versión del programa el modelo se divide en dos estados más explícitos:

estado_waitInON → Representa el estado en el que el pixel está encendido.

estado_waitInOFF → Representa el estado en el que el pixel está apagado.

En esta versión, el evento Timeout provoca la transición entre ambos estados, haciendo que el pixel cambie entre encendido y apagado de manera periódica.

**2. ¿Cuáles son los eventos en el programa?**

Los eventos que maneja la máquina de estados son:

ENTRY: Evento interno que se ejecuta automáticamente al entrar a un estado.

EXIT: Evento interno que se ejecuta automáticamente al salir de un estado.

Timeout: Evento generado por el objeto Timer cuando se cumple el intervalo de tiempo programado.

El evento Timeout no es generado por el usuario, sino por el temporizador cuando detecta que ha transcurrido el tiempo configurado.

**3.¿Cuáles son las acciones en el programa?**

Las acciones son las operaciones que modifican el estado del sistema o generan un efecto visible. En este programa las acciones principales son:

Encender el pixel (establecer brillo 9).

Apagar el pixel (establecer brillo 0).

Actualizar el display con display.set_pixel(x, y, pixelState).

Iniciar el temporizador con myTimer.start().

Generar el evento Timeout mediante post_event("Timeout").

Realizar transiciones entre estados (en la segunda versión del modelo).

**Conclusión**

El programa demuestra el uso de una máquina de estados reactiva controlada por eventos y temporizadores. Cada pixel funciona como una máquina independiente con su propio temporizador, lo que permite que ambos parpadeen a distintas velocidades sin bloquear la ejecución del programa. Esta arquitectura facilita la concurrencia y mejora el control del flujo del sistema.

![pulgar-encima-del-emoticon-47854113](https://github.com/user-attachments/assets/04295376-7576-4bd4-bca3-f446fa068bd6)

#### Actividad 2

---

#### Actividad 3

---

#### Actividad 4

---

<img width="1861" height="840" alt="image" src="https://github.com/user-attachments/assets/12910087-8e44-485f-b394-19788122ff06" />

``` js

from microbit import *               // Importa display, botones, acelerómetro y utilidades del micro:bit
import utime                         // Permite manejar tiempo en milisegundos (ticks_ms)
import music                         // Permite controlar el speaker

def make_fill_images(on='9', off='0'):      // Función que crea imágenes de llenado del display
    imgs = []                               // Lista donde se guardarán las 26 imágenes (0..25)
    for n in range(26):                     // Recorre valores de 0 a 25 (cantidad de pixeles encendidos)
        rows = []                           // Lista temporal para construir cada fila del display
        k = 0                               // Contador global de pixeles (0 a 24)
        for y in range(5):                  // Recorre las 5 filas del display
            row = []                        // Lista para construir una fila de 5 pixeles
            for x in range(5):              // Recorre las 5 columnas
                row.append(on if k < n else off)  // Enciende pixel si k<n, si no lo apaga
                k += 1                      // Avanza al siguiente pixel
            rows.append(''.join(row))       // Convierte la fila en string y la guarda
        imgs.append(Image(':'.join(rows)))  // Une filas y crea imagen tipo Image
    return imgs                             // Devuelve lista completa de imágenes

FILL = make_fill_images()                   // Genera las imágenes de 0 a 25 pixeles encendidos

SKULL = Image.SKULL                         // Imagen de calavera para el estado final

class Timer:                                // Clase que gestiona tiempos sin usar sleep()

    def __init__(self, owner, event_to_post, duration): // Constructor del timer
        self.owner = owner                  // Objeto que recibirá el evento cuando termine el tiempo
        self.event = event_to_post          // Nombre del evento a enviar (ej: "Timeout")
        self.duration = duration            // Duración en milisegundos
        self.start_time = 0                 // Tiempo inicial cuando se activa
        self.active = False                 // Indica si el timer está activo

    def start(self, new_duration=None):     // Método para iniciar el timer
        if new_duration is not None:        // Si se pasa nueva duración
            self.duration = new_duration    // Actualiza duración
        self.start_time = utime.ticks_ms()  // Guarda tiempo actual
        self.active = True                  // Activa el timer

    def stop(self):                         // Método para detener el timer
        self.active = False                 // Lo desactiva

    def update(self):                       // Método que verifica si el tiempo ya pasó
        if self.active:                     // Solo actúa si está activo
            elapsed = utime.ticks_diff(utime.ticks_ms(), self.start_time) // Calcula tiempo transcurrido
            if elapsed >= self.duration:    // Si pasó el tiempo programado
                self.active = False         // Desactiva el timer
                self.owner.post_event(self.event) // Envía el evento al dueño

class Task:                                 // Clase que implementa la máquina de estados

    def __init__(self):                     // Constructor principal
        self.event_queue = []               // Cola de eventos (FIFO)
        self.timers = []                    // Lista de timers internos
        self.tick = self.createTimer("Timeout", 1000) // Timer que genera evento cada 1 segundo
        self.n = 20                         // Valor inicial del temporizador (20 pixeles)
        self.estado_actual = None           // Estado actual
        self.transicion_a(self.estado_config) // Inicia en estado CONFIG

    def createTimer(self, event, duration): // Crea y registra un timer
        t = Timer(self, event, duration)    // Instancia el timer
        self.timers.append(t)               // Lo guarda en la lista
        return t                            // Lo retorna

    def post_event(self, ev):               // Agrega evento a la cola
        self.event_queue.append(ev)         // Inserta evento al final

    def update(self):                       // Método que se llama en el loop principal
        for t in self.timers:               // Actualiza todos los timers
            t.update()                      // Verifica si alguno venció
        while len(self.event_queue) > 0:    // Mientras haya eventos pendientes
            ev = self.event_queue.pop(0)    // Extrae el primero
            if self.estado_actual:          // Si hay estado activo
                self.estado_actual(ev)      // Envía evento al estado

    def transicion_a(self, nuevo_estado):   // Cambia de estado
        if self.estado_actual:              // Si ya había un estado
            self.estado_actual("EXIT")      // Ejecuta lógica de salida
        self.estado_actual = nuevo_estado   // Cambia referencia
        self.estado_actual("ENTRY")         // Ejecuta lógica de entrada

    def estado_config(self, ev):            // Estado CONFIG (modo configuración)

        if ev == "ENTRY":                   // Al entrar
            self.tick.stop()                // Asegura que el timer esté apagado
            music.stop()                    // Detiene cualquier sonido
            self.n = 20                     // Reinicia a 20 pixeles
            display.show(FILL[self.n])      // Muestra valor actual

        elif ev == "A":                     // Botón A presionado
            if self.n < 25:                 // Si no supera máximo
                self.n += 1                 // Incrementa valor
                display.show(FILL[self.n])  // Actualiza display

        elif ev == "B":                     // Botón B presionado
            if self.n > 15:                 // Si no baja del mínimo
                self.n -= 1                 // Decrementa valor
                display.show(FILL[self.n])  // Actualiza display

        elif ev == "S":                     // Evento shake
            self.transicion_a(self.estado_armed) // Pasa a cuenta regresiva

    def estado_armed(self, ev):             // Estado ARMED (cuenta regresiva)

        if ev == "ENTRY":                   // Al entrar
            display.show(FILL[self.n])      // Muestra valor actual
            self.tick.start(1000)           // Inicia timer de 1 segundo

        elif ev == "Timeout":               // Cada segundo
            if self.n > 0:                  // Si aún quedan pixeles
                self.n -= 1                 // Reduce uno
                display.show(FILL[self.n])  // Actualiza pantalla
                self.tick.start(1000)       // Reinicia timer
            else:                           // Si llegó a cero
                self.transicion_a(self.estado_alarm) // Va a alarma

        elif ev == "EXIT":                  // Al salir del estado
            self.tick.stop()                // Detiene timer

    def estado_alarm(self, ev):             // Estado ALARM (fin)

        if ev == "ENTRY":                   // Al entrar
            display.show(SKULL)             // Muestra calavera
            music.play(music.BA_DING, wait=False, loop=True) // Activa sonido continuo

        elif ev == "A":                     // Botón A reinicia sistema
            self.transicion_a(self.estado_config) // Regresa a configuración

        elif ev == "EXIT":                  // Al salir
            music.stop()                    // Apaga el sonido

task = Task()                               // Crea instancia de la máquina

while True:                                 // Loop infinito del sistema
    if button_a.was_pressed():              // Si se presiona botón A
        task.post_event("A")                // Envía evento A
    if button_b.was_pressed():              // Si se presiona botón B
        task.post_event("B")                // Envía evento B
    if accelerometer.was_gesture("shake"):  // Si detecta shake
        task.post_event("S")                // Envía evento S
    task.update()                           // Procesa eventos y timers
    utime.sleep_ms(20)                      // Pequeña pausa de estabilidad


```
![gng idk where the og source is but my friend posted this i lwk forgot about it and now it has 295+ saves __3](https://github.com/user-attachments/assets/e11e1e0b-bdf2-42d2-9140-98d74c579ac9)

## Bitácora de aplicación 



## Bitácora de reflexión
