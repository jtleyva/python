# Interview Questions

## Node.js

### Q1: What do you mean by Asynchronous API? ☆☆

**Respuesta:**
Todas las APIs de la librería de Node.js son asíncronas, lo que significa que no son bloqueantes (non-blocking). Esencialmente, esto significa que un servidor basado en Node.js nunca espera a que una API devuelva datos. El servidor pasa a la siguiente API después de llamarla, y un mecanismo de notificación de Eventos de Node.js ayuda al servidor a obtener la respuesta de la llamada a la API anterior.

**Fuente:** _tutorialspoint.com_

### Q2: What are the benefits of using Node.js? ☆☆

**Respuesta:**
Los siguientes son los principales beneficios de usar Node.js:

- **Asíncrono y Orientado a Eventos** - Todas las APIs de la librería de Node.js son asíncronas (no bloqueantes). El servidor pasa a la siguiente llamada a la API sin esperar la respuesta de la anterior, usando el mecanismo de eventos para obtener los resultados.
- **Muy Rápido** - Al estar construido sobre el motor de JavaScript V8 de Google Chrome, la ejecución de código en Node.js es muy rápida.
- **Un Solo Hilo pero Altamente Escalable** - Node.js utiliza un modelo de un solo hilo (single-threaded) con un bucle de eventos (event loop). Este mecanismo permite al servidor responder de forma no bloqueante, haciéndolo altamente escalable en comparación con servidores tradicionales que crean un número limitado de hilos para manejar las solicitudes. Un solo programa en Node.js puede atender un número mucho mayor de solicitudes que servidores tradicionales como Apache.
- **Sin Búfer** - Las aplicaciones Node.js nunca almacenan datos en un búfer. Simplemente emiten los datos en fragmentos (chunks).

**Fuente:** _tutorialspoint.com_

### Q3: Is Node a single threaded application? ☆☆

**Respuesta:**
¡Sí! Node usa un modelo de un solo hilo con un bucle de eventos.

**Fuente:** _tutorialspoint.com_

### Q4: What is global installation of dependencies? ☆☆

**Respuesta:**
Los paquetes/dependencias instalados globalmente se almacenan en el directorio **<directorio-usuario>**/npm. Dichas dependencias pueden usarse en la interfaz de línea de comandos (CLI) de Node.js, pero no pueden importarse directamente usando `require()` en una aplicación Node. Para instalar un paquete globalmente se usa la bandera `-g`.

**Fuente:** _tutorialspoint.com_

### Q5: What is an error-first callback? ☆☆

**Respuesta:**
Los _callbacks con error primero_ (error-first callbacks) se usan para pasar errores y datos. El primer argumento siempre es un objeto de error que el programador debe verificar si algo salió mal. Los argumentos adicionales se usan para pasar datos.

```js
fs.readFile(filePath, function (err, data) {
  if (err) {
    // manejar el error
  }
  // usar el objeto de datos
});
```

**Fuente:** _tutorialspoint.com_

### Q6: What's the difference between operational and programmer errors? ☆☆

**Respuesta:**
Los errores operacionales no son bugs, sino problemas con el sistema, como _tiempo de espera agotado (timeout)_ o _falla de hardware_. Por otro lado, los errores del programador son bugs reales en el código.

**Fuente:** _blog.risingstack.com_

### Q7: What is the difference between Nodejs, AJAX, and jQuery? ☆☆

**Respuesta:**
La característica común entre Node.js, AJAX y jQuery es que todos son implementaciones avanzadas de JavaScript. Sin embargo, sirven para propósitos completamente diferentes:

- **Node.js** – Es una plataforma del lado del servidor (server-side) para desarrollar aplicaciones cliente-servidor. Funciona en un servidor (similar a Apache, Django), no en un navegador.
- **AJAX** (Asynchronous Javascript and XML) – Es una técnica de scripting del lado del cliente (client-side), diseñada principalmente para actualizar el contenido de una página sin tener que recargarla entera.
- **jQuery** – Es una famosa librería de JavaScript que facilita AJAX, la navegación por el DOM, bucles, etc. Ayuda a gestionar la compatibilidad entre navegadores y facilita el desarrollo del lado del cliente.

**Fuente:** _techbeamers.com_

### Q8: How to make Post request in Node.js? ☆☆

**Respuesta:**
El siguiente fragmento de código puede usarse para hacer una petición POST en Node.js usando el módulo `request`.

```js
var request = require("request");
request.post(
  "http://www.example.com/action",
  {
    form: {
      key: "value",
    },
  },
  function (error, response, body) {
    if (!error && response.statusCode == 200) {
      console.log(body);
    }
  }
);
```

**Fuente:** _techbeamers.com_

### Q9: What are the key features of Node.js? ☆☆

**Respuesta:**
Las características clave de Node.js son:

- **I/O Asíncrono y orientado a eventos:** Facilita el manejo concurrente de peticiones. Las operaciones de entrada/salida se ejecutan en segundo plano sin bloquear otras solicitudes.
- **Ejecución de código rápida:** Usa el motor V8 de Google Chrome.
- **Un solo hilo pero altamente escalable:** Su modelo de bucle de eventos permite procesar miles de solicitudes concurrentes sin crear un hilo por cada una.
- **Uso de JavaScript:** Unifica el lenguaje entre el frontend y el backend.
- **Comunidad activa:** Mantiene el ecosistema de paquetes (NPM) grande y actualizado.
- **Sin Buffering:** Envía datos en fragmentos (streams).

**Fuente:** _techbeamers.com_

### Q10: What is control flow function? ☆☆

**Respuesta:**
Es un bloque de código genérico que se ejecuta entre varias llamadas a funciones asíncronas para gestionar el orden y el flujo de su ejecución.

**Fuente:** _lazyquestion.com_

### Q11: What are Event Listeners? ☆☆

**Respuesta:**
Los **Event Listeners** (oyentes de eventos) son similares a las funciones callback, pero están asociados a un evento específico. Por ejemplo, cuando un servidor escucha peticiones HTTP, se genera un evento y se invoca el event listener correspondiente. Node.js tiene eventos incorporados y permite crear eventos personalizados.

**Fuente:** _lazyquestion.com_

### Q12: If Node.js is single threaded then how it handles concurrency? ☆☆

**Respuesta:**
Node proporciona un solo hilo (single thread) al programador para facilitar la escritura de código. Sin embargo, internamente Node usa múltiples hilos POSIX (libuv) para varias operaciones de I/O, como archivos, DNS o red. 
Cuando Node recibe una petición de I/O, la delega a un hilo interno de trabajo. Una vez que la operación termina, empuja el resultado a la cola de eventos (event queue). El bucle de eventos (event loop) verifica la cola y ejecuta los callbacks en el hilo principal. Así es como maneja la concurrencia.

**Fuente:** _codeforgeek.com_

### Q13: What is Callback Hell? ☆☆

**Respuesta:**
Ocurre cuando múltiples funciones asíncronas que requieren callbacks se encadenan o anidan profundamente unas dentro de otras, lo que hace que el código sea difícil de leer y mantener (conocido también como la Pirámide de la Muerte).

**Fuente:** _codeforgeek.com_

### Q14: Could we run an external process with Node.js? ☆☆

**Respuesta:**
Sí. El módulo _Child process_ (proceso hijo) nos permite acceder a funcionalidades del sistema operativo o ejecutar otras aplicaciones. Puedes usarlo para ejecutar comandos del sistema sin bloquear el bucle de eventos.
Tiene tres formas principales de crear procesos hijos:
- `spawn`: Lanza un nuevo proceso con un comando dado.
- `exec`: Ejecuta un comando en una shell/consola y almacena la salida en un búfer.
- `fork`: Es un caso especial de spawn para crear procesos hijos de Node.js.

**Fuente:** _codeforgeek.com_

### Q15: List out the differences between AngularJS and NodeJS? ☆☆

**Respuesta:**
AngularJS es un framework para el desarrollo de aplicaciones web del lado del cliente (en el navegador). NodeJS es un entorno de ejecución (runtime) utilizado para construir aplicaciones del lado del servidor.

**Fuente:** _a4academics.com_

### Q16: How you can monitor a file for modifications in Node.js ? ☆☆

**Respuesta:**
Podemos aprovechar la función `watch()` del módulo File System (`fs`) para observar los cambios de un archivo.

**Fuente:** _codingdefined.com_

### Q17: What are the core modules of Node,js? ☆☆

**Respuesta:**
Algunos módulos centrales (core) son:
- EventEmitter
- Stream
- FS (File System)
- Net
- Global Objects

**Fuente:** _github.com/jimuyouyou_

### Q18: What is V8? ☆☆

**Respuesta:**
El motor V8 proporciona a Node.js un entorno de ejecución de JavaScript. Convierte el código JavaScript en código máquina (lenguaje de bajo nivel) que los microprocesadores pueden entender. V8 es mantenido por Google y usado originalmente en Chrome. Está escrito en C++.

**Fuente:** _nodejs.org_

### Q19: What is libuv? ☆☆

**Respuesta:**
**libuv** es una librería en C que se usa para abstraer las operaciones de entrada/salida (I/O) no bloqueantes en una interfaz consistente en todas las plataformas soportadas. Proporciona mecanismos para manejar el sistema de archivos, DNS, red, procesos hijos, etc. También incluye un **thread pool** (grupo de hilos) para delegar trabajo pesado que no se puede hacer asíncronamente a nivel del sistema operativo.

**Fuente:** _nodejs.org_


### Q21: What is REPL in context of Node? ☆☆☆

**Respuesta:**
**REPL** significa Read Eval Print Loop (Bucle de Leer, Evaluar, Imprimir) y representa un entorno informático como una consola de Windows o shell de Unix/Linux donde se introduce un comando y el sistema responde con una salida. Node.js viene empaquetado con un entorno REPL que realiza las siguientes tareas:
- **Read (Leer):** Lee la entrada del usuario, la analiza en una estructura de datos de JavaScript y la almacena en memoria.
- **Eval (Evaluar):** Toma y evalúa la estructura de datos.
- **Print (Imprimir):** Imprime el resultado.
- **Loop (Bucle):** Repite el proceso hasta que el usuario presiona `ctrl-c` dos veces.

**Fuente:** _tutorialspoint.com_

### Q22: What is Callback? ☆☆☆

**Respuesta:**
Un **Callback** es el equivalente asíncrono de una función. Una función callback se llama cuando finaliza una tarea determinada. Node hace un uso intensivo de los callbacks; todas las APIs de Node están escritas de forma que los soporten.
Por ejemplo, una función para leer un archivo puede iniciar la lectura y devolver el control al entorno de ejecución inmediatamente para que se pueda ejecutar la siguiente instrucción. Una vez que se completa la entrada/salida (I/O) del archivo, llamará a la función callback, pasándole el contenido del archivo como parámetro. Así no hay bloqueo ni espera por la I/O del archivo.

**Fuente:** _tutorialspoint.com_

### Q23: What is a blocking code? ☆☆☆

**Respuesta:**
Si la aplicación tiene que esperar por alguna operación de I/O para poder completar su ejecución posterior, entonces el código responsable de esa espera se conoce como código bloqueante (blocking code).

**Fuente:** _tutorialspoint.com_

### Q24: How Node prevents blocking code? ☆☆☆

**Respuesta:**
Proporcionando funciones callback. La función callback se llama cada vez que se desencadena el evento correspondiente, permitiendo que el hilo principal continúe ejecutando otro código mientras espera.

**Fuente:** _tutorialspoint.com_

### Q25: What is Event Loop? ☆☆☆

**Respuesta:**
Node.js es una aplicación de un solo hilo, pero soporta concurrencia mediante el concepto de eventos y callbacks. Como toda API de Node.js es asíncrona y se ejecuta en un solo hilo, usa llamadas a funciones asíncronas para mantener la concurrencia utilizando el patrón Observer (Observador). El hilo de Node mantiene un **bucle de eventos (event loop)** y siempre que se completa cualquier tarea, dispara el evento correspondiente que señala a la función event listener para que se ejecute.

**Fuente:** _tutorialspoint.com_

### Q26: What is Event Emmitter? ☆☆☆

**Respuesta:**
Todos los objetos que emiten eventos son miembros de la clase `EventEmitter`. Estos objetos exponen una función `eventEmitter.on()` que permite adjuntar una o más funciones a eventos con nombre emitidos por el objeto.
Cuando el objeto EventEmitter emite un evento, todas las funciones adjuntas a ese evento específico se llaman de forma síncrona.

```js
const EventEmitter = require("events");

class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();
myEmitter.on("event", () => {
  console.log("¡ha ocurrido un evento!");
});
myEmitter.emit("event");
```

**Fuente:** _tutorialspoint.com_

### Q27: What is purpose of Buffer class in Node? ☆☆☆

**Respuesta:**
La clase **Buffer** es una clase global y se puede acceder en la aplicación sin importar el módulo `buffer`. Un Buffer es una especie de array de enteros y corresponde a una asignación de memoria sin procesar fuera del heap (montículo) de V8. Un Buffer no puede cambiar de tamaño. Sirve para manejar flujos de datos binarios directamente.

**Fuente:** _tutorialspoint.com_

### Q28: What is difference between synchronous and asynchronous method of fs module? ☆☆☆

**Respuesta:**
Cada método en el módulo `fs` (File System) tiene tanto su forma síncrona como asíncrona. Los métodos asíncronos toman un último parámetro como callback de finalización y el primer parámetro de la función callback es el error (error-first callback). Es preferible usar el método asíncrono en lugar del síncrono, ya que el primero nunca bloquea la ejecución del programa, mientras que el segundo sí lo hace.

**Fuente:** _tutorialspoint.com_

### Q29: What are streams? ☆☆☆

**Respuesta:**
Los **Streams** (flujos) son objetos que te permiten leer datos de una fuente o escribir datos a un destino de forma continua. En Node.js, hay cuatro tipos de streams:
- **Readable (Legible):** Stream que se usa para operaciones de lectura.
- **Writable (Escribible):** Stream que se usa para operaciones de escritura.
- **Duplex:** Stream que se puede usar tanto para operaciones de lectura como de escritura.
- **Transform (De transformación):** Un tipo de stream duplex donde la salida se calcula basándose en la entrada.

**Fuente:** _tutorialspoint.com_

### Q30: What is Chaining in Node? ☆☆☆

**Respuesta:**
El **Chaining** (encadenamiento) es un mecanismo para conectar la salida de un stream a la entrada de otro stream y crear una cadena de múltiples operaciones de stream. Normalmente se usa con operaciones de canalización (piping).

**Fuente:** _tutorialspoint.com_

### Q31: What is the purpose of setTimeout function? ☆☆☆

**Respuesta:**
La función global `setTimeout(cb, ms)` se usa para ejecutar el callback `cb` después de **al menos** `ms` milisegundos. El retraso real depende de factores externos como la granularidad del temporizador del sistema operativo y la carga del sistema. Un temporizador no puede abarcar más de 24.8 días.

**Fuente:** _tutorialspoint.com_

### Q32: How can you avoid callback hells? ☆☆☆

**Respuesta:**
Para evitarlo, tienes varias opciones:
- **Modularización:** dividir los callbacks en funciones independientes y con nombre.
- Usar **Promises (Promesas)**.
- Usar **`async/await`** (la forma más moderna y recomendada).
- Usar `yield` con Generadores (un enfoque más antiguo).

**Fuente:** _tutorialspoint.com_

### Q33: What's the event loop? ☆☆☆

**Respuesta:**
El **Event Loop (bucle de eventos)** es lo que permite a Node.js realizar operaciones de I/O no bloqueantes (a pesar de que JavaScript es de un solo hilo) delegando operaciones al núcleo (kernel) del sistema operativo siempre que sea posible.
Cada I/O requiere un callback; una vez que terminan, son empujados al bucle de eventos para su ejecución. Dado que la mayoría de los kernels modernos son multi-hilo, pueden manejar múltiples operaciones ejecutándose en segundo plano. Cuando una de estas operaciones se completa, el kernel le dice a Node.js para que el callback apropiado pueda ser añadido a la cola de "poll" (sondeo) para ser ejecutado eventualmente.

**Fuente:** _blog.risingstack.com_

### Q34: How to avoid callback hell in Node.js? ☆☆☆

**Respuesta:**
Hay cuatro soluciones principales que pueden abordar el problema del "callback hell":
1. **Hacer tu programa modular:** Dividir la lógica en módulos más pequeños y unirlos desde el módulo principal.
2. **Usar mecanismos de Promesas:** Las promesas devuelven el resultado de la ejecución o el error/excepción encadenando `.then()` y `.catch()`.
3. **Usar generadores:** Rutinas ligeras que hacen que una función espere y se reanude mediante la palabra clave `yield`.
4. **Usar `async/await` (la más común hoy en día):** Permite escribir código asíncrono que se lee como si fuera código síncrono.

**Fuente:** _techbeamers.com_


### Q16: When should we use Node.js? ☆☆☆

**Respuesta:**
**Node.js** es adecuado para aplicaciones que tienen muchas conexiones concurrentes y cada _solicitud solo necesita muy pocos ciclos de CPU_, porque el bucle de eventos (con todos los demás clientes) se bloquea durante la ejecución de una función. Node.js es más adecuado para aplicaciones en tiempo real: juegos online, herramientas de colaboración, salas de chat, o cualquier cosa donde lo que hace un usuario deba ser visto por otros usuarios inmediatamente, sin recargar la página.

**Fuente:** _techbeamers.com_

### Q17: How does Node.js handle child threads? ☆☆☆

**Respuesta:**
Node.js, en su esencia, es un proceso de un solo hilo. No expone hilos hijos ni métodos de gestión de hilos al desarrollador. Técnicamente, Node.js sí genera hilos hijos para ciertas tareas como I/O asíncrono, pero estos se ejecutan entre bastidores y no ejecutan ningún código JavaScript de la aplicación, ni bloquean el bucle de eventos principal.
Si se desea soporte para hilos en una aplicación Node.js, hay herramientas disponibles como `worker_threads` o el módulo `ChildProcess`.

**Fuente:** _lazyquestion.com_

### Q18: What is the preferred method of resolving unhandled exceptions in Node.js? ☆☆☆

**Respuesta:**
Las excepciones no controladas en Node.js pueden atraparse a nivel del `Process` adjuntando un manejador para el evento `uncaughtException`.
```js
process.on("uncaughtException", function (err) {
  console.log("Excepción atrapada: " + err);
});
```
Sin embargo, `uncaughtException` es un mecanismo muy rudimentario. Una excepción que ha burbujeado hasta el nivel del `Process` significa que tu aplicación podría estar en un estado indefinido, y el único enfoque sensato sería reiniciar todo. En servidores web normales, el mejor enfoque es enviar una respuesta de error, dejar que las otras peticiones terminen y detener la escucha de nuevas peticiones en ese worker.

**Fuente:** _lazyquestion.com_

### Q19: What is stream and what are types of streams available in Node.js? ☆☆☆

**Respuesta:**
Los **Streams** son una colección de datos que podrían no estar disponibles todos a la vez y no tienen que caber en la memoria de una vez. Los streams proporcionan fragmentos (chunks) de datos de manera continua. Es útil para leer un gran conjunto de datos y procesarlo.
Hay cuatro tipos fundamentales de streams:
- Readable (Legible).
- Writeable (Escribible).
- Duplex (Doble).
- Transform (Transformación).

**Fuente:** _codeforgeek.com_

### Q20: What are the global objects of Node.js? ☆☆☆

**Respuesta:**
Estos objetos están disponibles en todos los módulos:
- **process** - Proporciona información y control sobre el proceso actual de Node.js.
- **console** - Usado para imprimir en stdout y stderr.
- **buffer** - Usado para manejar datos binarios.
- **__dirname** y **__filename** - Aunque técnicamente no son globales puros (son locales a cada módulo), se usan globalmente para obtener rutas.

**Fuente:** _github.com/jimuyouyou_

### Q1: What is Piping in Node? ☆☆☆☆

**Respuesta:**
El **Piping** (canalización) es un mecanismo para conectar la salida de un stream a otro stream. Normalmente se usa para obtener datos de un stream y pasar esa salida a otro stream automáticamente.
```js
readableStream.pipe(writableStream);
```

**Fuente:** _tutorialspoint.com_

### Q2: Name some of the events fired by streams. ☆☆☆☆

**Respuesta:**
Cada tipo de Stream es una instancia de **EventEmitter** y lanza varios eventos. Algunos comúnmente usados son:
- **data** - Disparado cuando hay datos disponibles para leer.
- **end** - Disparado cuando no hay más datos para leer.
- **error** - Disparado cuando hay un error recibiendo o escribiendo datos.
- **finish** - Disparado cuando todos los datos se han volcado al sistema subyacente (en streams de escritura).

**Fuente:** _tutorialspoint.com_

### Q3: What is the purpose of \_\_filename variable? ☆☆☆☆

**Respuesta:**
`__filename` representa el nombre de archivo del código que se está ejecutando. Esta es la ruta absoluta resuelta de este archivo de código. El valor dentro de un módulo es la ruta a ese archivo de módulo en específico.

**Fuente:** _tutorialspoint.com_

### Q4: How can you listen on port 80 with Node? ☆☆☆☆

**Respuesta:**
La mejor práctica es ejecutar la aplicación Node en cualquier puerto superior a 1024 (ej. 3000), y luego poner un proxy inverso como **Nginx** o **HAProxy** delante de él que escuche en el puerto 80 y redirija el tráfico a Node. Escuchar directamente en el puerto 80 con Node requiere privilegios de root (sudo), lo cual es un riesgo de seguridad.

**Fuente:** _blog.risingstack.com_

### Q5: What tools can be used to assure consistent code style? ☆☆☆☆

**Respuesta:**
La herramienta moderna y estándar de facto es **ESLint**, junto con formateadores de código como **Prettier**. (Herramientas más antiguas incluyen JSLint, JSHint o JSCS, pero ya no se usan habitualmente).
Estas herramientas son realmente útiles cuando se desarrolla código en equipo, para aplicar una guía de estilo determinada y detectar errores comunes mediante análisis estático.

**Fuente:** _blog.risingstack.com_

### Q6: What's a stub? Name a use case. ☆☆☆☆

**Respuesta:**
Los **Stubs** son funciones/programas que simulan los comportamientos de componentes o módulos durante el testing. Los stubs proporcionan respuestas preestablecidas (canned answers) a llamadas a funciones realizadas durante los casos de prueba.
Un caso de uso sería simular la lectura de un archivo para no tener que leer un archivo real en una prueba unitaria.

**Fuente:** _blog.risingstack.com_

### Q7: Does Node.js support multi-core platforms? And is it capable of utilizing all the cores? ☆☆☆☆

**Respuesta:**
Sí, Node.js funciona en un sistema multi-núcleo. Pero al ser por defecto una aplicación de un solo hilo, no puede utilizar completamente todos los núcleos del sistema por sí solo.
Sin embargo, Node.js incluye el módulo **Cluster** (o gestores de procesos como PM2) que es capaz de iniciar múltiples procesos trabajadores (workers) de Node.js que compartirán el mismo puerto y utilizarán todos los núcleos disponibles.

**Fuente:** _techbeamers.com_

### Q8: Is Node.js entirely based on a single-thread? ☆☆☆☆

**Respuesta:**
Sí, es cierto que Node.js procesa todas las solicitudes en un solo hilo principal (Event Loop). Pero eso es solo el código de usuario. Más allá del mecanismo de un solo hilo, hace uso de la biblioteca en C++ **libuv**, la cual toma a su cargo las operaciones no secuenciales de I/O (sistema de archivos, criptografía) utilizando un grupo de hilos en segundo plano (thread pool de trabajadores).

**Fuente:** _techbeamers.com_

### Q10: When to not use Node.js? ☆☆☆☆

**Respuesta:**
No deberíamos usar Node.js para casos donde la aplicación requiera un largo tiempo de procesamiento de CPU (Cálculos matemáticos pesados, procesamiento masivo de imágenes o video en el hilo principal). Si el servidor está haciendo un cálculo pesado, bloqueará el hilo principal y no podrá procesar ninguna otra solicitud (a menos que se delegue a `worker_threads` o microservicios en otros lenguajes).

**Fuente:** _techbeamers.com_

### Q11: Why to use Buffers instead of binary strings to handle binary data ? ☆☆☆☆

**Respuesta:**
El JavaScript puro históricamente no ha sido bueno manejando datos binarios puros. Las cadenas de texto (strings) están diseñadas para caracteres, no para bytes. Trabajar con strings binarios es muy lento y tiende a corromper los datos. Por eso siempre es aconsejable usar Buffers para manejar flujos TCP y lectura de archivos.

**Fuente:** _codingdefined.com_

### Q12: How to gracefully Shutdown Node.js Server? ☆☆☆☆

**Respuesta:**
Podemos apagar el servidor con elegancia (graceful shutdown) interceptando señales genéricas del SO como `SIGTERM` o `SIGINT`. Al atraparlas, debemos detener la aceptación de nuevas conexiones en el servidor HTTP (`server.close()`), terminar las conexiones activas a bases de datos, y luego terminar el proceso limpiamente con `process.exit(0)`.

**Fuente:** _codingdefined.com_

### Q13: What are the timing features of Node.js? ☆☆☆☆

**Respuesta:**
- **setTimeout/clearTimeout** - programa la ejecución de código después de N milisegundos.
- **setInterval/clearInterval** - ejecuta un bloque de código repetidamente.
- **setImmediate/clearImmediate** - ejecutará el código al final del ciclo actual del bucle de eventos (en la fase de check).
- **process.nextTick** - se usa para programar un callback que se invocará inmediatamente después de la operación actual, *antes* de que el bucle de eventos continúe.

**Fuente:** _github.com/jimuyouyou_

### Q14: Explain usage of NODE_ENV ☆☆☆☆

**Respuesta:**
Node fomenta la convención de usar una variable de entorno llamada `NODE_ENV` para marcar en qué entorno estamos (ej. `development` o `production`). Establecer `NODE_ENV` en `production` hace que frameworks como Express sean hasta 3 veces más rápidos, ya que habilitan la caché de vistas y desactivan mensajes de error detallados.

**Fuente:** _github.com/i0natan/nodebestpractices_

### Q15: What is LTS releases of Node.js why should you care? ☆☆☆☆

**Respuesta:**
Una versión **LTS (Soporte a Largo Plazo)** de Node.js recibe correcciones de errores críticos, actualizaciones de seguridad y mejoras de rendimiento durante al menos 18-30 meses. Las versiones LTS se indican con números pares (ej. 18, 20, 22). Son las mejores para producción debido a su estabilidad, mientras que las versiones "Current" (números impares) tienen una vida útil más corta y actualizaciones más experimentales.

**Fuente:** _github.com/i0natan/nodebestpractices_

### Q17: How would you handle errors for async code in Node.js? ☆☆☆☆

**Respuesta:**
El manejo de errores con callbacks (enfoque error-first) puede llevar a código anidado e ilegible. Es mejor usar **`async/await`** junto con bloques `try...catch` para atrapar errores de una manera compacta y síncrona visualmente.

```js
async function check(req, res) {
  try {
    const a = await someFunction();
    res.send("resultado");
  } catch (error) {
    res.status(500).send(error.message);
  }
}
```

**Fuente:** _github.com/i0natan/nodebestpractices_

### Q18: What's the difference between dependencies, devDependencies and peerDependencies in npm package.json file? ☆☆☆☆

**Respuesta:**
- **dependencies**: Dependencias que tu proyecto necesita para ejecutarse en producción.
- **devDependencies**: Dependencias necesarias solo para el desarrollo local o pruebas (ej. Jest, TypeScript, ESLint). No se instalan en entornos de producción.
- **peerDependencies**: Indica que tu paquete espera que el proyecto anfitrión (el que instala tu paquete) instale y proporcione esa dependencia. Muy usado al crear plugins (ej. un plugin de React pedirá React como peerDependency).

**Fuente:** _stackoverflow.com_

### Q20: What are async functions in Node? Provide some examples. ☆☆☆☆

**Respuesta:**
Las funciones `async/await` (introducidas en ES2017) son una forma más limpia de trabajar con código asíncrono.
- Una función declarada con `async` siempre devuelve una Promesa.
- Dentro de la función, la palabra clave `await` pausa la ejecución hasta que la Promesa se resuelva (o rechace), permitiendo escribir código asíncrono como si fuera secuencial y no bloqueante.

---
## Preguntas Nivel SENIOR (Avanzado)


### Q1: Consider following code snippet ☆☆☆☆☆

**Detalles:**
Considera el siguiente fragmento de código:
```js
{
  console.time("loop");
  for (var i = 0; i < 1000000; i += 1) {
    // No hacer nada
  }
  console.timeEnd("loop");
}
```
El tiempo necesario para ejecutar este código en Google Chrome es considerablemente mayor que el tiempo necesario para ejecutarlo en Node.js. Explica por qué ocurre esto, aunque ambos usen el motor JavaScript V8.

**Respuesta:**
Dentro de un navegador web como Chrome, declarar la variable `i` fuera del alcance de cualquier función la hace global y, por lo tanto, la vincula como una propiedad del objeto `window`. Como resultado, ejecutar este código en un navegador requiere resolver repetidamente la propiedad `i` dentro del espacio de nombres `window` (muy poblado) en cada iteración del bucle `for`.
En Node.js, sin embargo, declarar cualquier variable fuera del alcance de una función la vincula solo al alcance propio del módulo (no al objeto `window`), lo que por lo tanto hace que sea mucho más fácil y rápido de resolver.

**Fuente:** _toptal.com_

### Q2: Can Node.js use other engines than V8? ☆☆☆☆☆

**Respuesta:**
Sí. Microsoft Chakra es otro motor de JavaScript que puede usarse con Node.js (mediante el proyecto node-chakracore). También se está trabajando en independizar Node de V8 para poder intercambiar el motor.

**Fuente:** _codeforgeek.com_

### Q3: How would you scale Node application? ☆☆☆☆☆

**Respuesta:**
Podemos escalar una aplicación Node de las siguientes maneras:
- **Escalado horizontal (Clonación):** usando el módulo _Cluster_ o gestores como PM2.
- **Descomposición:** separando la aplicación en servicios más pequeños, es decir, arquitectura de microservicios.

**Fuente:** _codeforgeek.com_

### Q4: What is the difference between process.nextTick() and setImmediate() ? ☆☆☆☆☆

**Respuesta:**
La diferencia es que `process.nextTick()` aplaza la ejecución de una acción hasta justo después de que termine la operación actual, *antes* de continuar con el bucle de eventos. Por otro lado, `setImmediate()` ejecuta un callback en el *siguiente ciclo* (fase de check) del bucle de eventos, cediendo el paso al bucle para ejecutar cualquier operación de I/O pendiente.

**Fuente:** _codingdefined.com_

### Q5: How to solve "Process out of Memory Exception" in Node.js ? ☆☆☆☆☆

**Respuesta:**
Para solucionar la excepción de proceso sin memoria en Node.js necesitamos aumentar el límite `max-old-space-size`. Por defecto, el tamaño máximo es de 512 MB (o 1.4GB en sistemas de 64 bits modernos), el cual puedes incrementar con el comando:
```sh
node --max-old-space-size=1024 file.js
```
*Nota Senior:* Esto es solo un parche; la solución real es diagnosticar y arreglar la fuga de memoria (memory leak) o manejar los flujos de datos grandes mediante Streams para no saturar la RAM.

**Fuente:** _codingdefined.com_

### Q6: Explain what is Reactor Pattern in Node.js? ☆☆☆☆☆

**Respuesta:**
El **Patrón Reactor** es la idea central detrás de las operaciones I/O no bloqueantes en Node.js. Este patrón proporciona un manejador (handler, en Node.js una _función callback_) que está asociado con cada operación de I/O. Cuando se genera una petición de I/O, se envía a un _demultiplexor_.
El _demultiplexor_ es una interfaz de notificación que se usa para manejar la concurrencia en modo no bloqueante; recolecta cada petición en forma de un evento y encola cada evento en una cola de eventos (_Event Queue_).
Al mismo tiempo, hay un Bucle de Eventos (_Event Loop_) que itera sobre los elementos en la Cola de Eventos. Cada evento tiene una función callback asociada a él, y esa función se invoca cuando el Event Loop itera.

**Fuente:** _hackernoon.com_

### Q7: Explain some Error Handling approaches in Node.js you know about. Which one will you use? ☆☆☆☆☆

**Respuesta:**
Hay cuatro patrones principales de manejo de errores en Node:
1. **Retornar el error (Error return value):** No funciona asíncronamente.
2. **Lanzar el error (Error throwing):** `throw new Error()`. Un `try/catch` síncrono no atrapará errores asíncronos.
3. **Error Callback:** Retornar un error vía un callback (error-first callback) es el patrón más común en Node.js antiguo.
4. **Emisión de Errores (Error emitting):** Usando `EventEmitter` para emitir el evento 'error'.
5. **Promesas y Async/Await (Recomendado):** Permite usar `try...catch` de manera limpia en funciones asíncronas.

**Fuente:** _gist.github.com_

### Q8: Why should you separate Express 'app' and 'server'? ☆☆☆☆☆

**Respuesta:**
Mantener la declaración de la API (app) separada de la configuración relacionada con la red (puerto, protocolo, etc. - servidor) permite probar la API en proceso, sin realizar llamadas de red reales. Esto trae beneficios como una ejecución rápida de pruebas y fácil obtención de métricas de cobertura. También permite desplegar la misma API bajo diferentes condiciones de red. Extra: mejor separación de responsabilidades (separation of concerns) y código más limpio.

**Fuente:** _github.com/i0natan/nodebestpractices_

### Q10: How many threads does Node actually create? ☆☆☆☆☆

**Respuesta:**
Además del hilo principal (main thread), se crean **4 hilos extra** por defecto para el uso de `libuv` (el thread pool para operaciones pesadas de I/O de disco y criptografía) y otros hilos internos que V8 usa para recolección de basura (GC) y tareas de compilación optimizadora.

**Fuente:** _stackoverflow.com_

### Q12: How the V8 engine works? ☆☆☆☆

**Respuesta:**
**V8** es un motor de JavaScript construido por Google. Es de código abierto y está escrito en C++. 
Para obtener velocidad, V8 traduce el código JavaScript a un código máquina más eficiente en lugar de usar un intérprete tradicional. Compila el código en tiempo de ejecución implementando un compilador **JIT (Just-In-Time)**. La diferencia principal de V8 es que no produce bytecode ni ningún código intermedio, compila directamente a código máquina.

**Fuente:** _nodejs.org_

### Q13: What is the purpose of using hidden classes in V8? ☆☆☆☆☆

**Respuesta:**
JavaScript es de tipado dinámico y los objetos pueden cambiar su forma en tiempo de ejecución añadiendo o eliminando propiedades. En lugar de usar una estructura de datos tipo diccionario (que es lenta), V8 crea **clases ocultas (hidden classes)** en tiempo de ejecución para tener una representación interna del sistema de tipos y mejorar enormemente el tiempo de acceso a las propiedades.

**Fuente:** _thibaultlaurens.github.io_

### Q15: How does libuv work under the hood? ☆☆☆☆☆

**Respuesta:**
La ejecución de callbacks es realizada por el bucle de eventos provisto por **libuv**. `libuv` también crea por defecto un grupo de hilos (thread pool) con 4 hilos para delegar el trabajo asíncrono pesado. Siempre que sea posible, usará interfaces asíncronas del SO en lugar del thread pool.
El bucle de eventos es un conjunto de fases con tareas específicas:
- **timers (temporizadores):** ejecuta callbacks programados por `setTimeout` y `setInterval`.
- **pending callbacks:** callbacks de I/O diferidos.
- **idle, prepare:** uso interno.
- **poll (sondeo):** recupera nuevos eventos de I/O; ejecuta casi todos los callbacks relacionados con I/O (excepto los de temporizadores o cierres).
- **check (comprobación):** los callbacks de `setImmediate` se invocan aquí.
- **close callbacks:** callbacks de cierre, ej. `socket.on('close')`.

**Fuente:** _nodejs.org_

### Q16: How does the cluster module work? What’s the difference between it and a load balancer? ☆☆☆☆

**Respuesta:**
El módulo cluster hace un clon (fork) de tu servidor, creando múltiples procesos hijos (workers). Soporta dos métodos para distribuir conexiones entrantes:
- Round-robin (por defecto): el proceso maestro (master) escucha en un puerto y distribuye las conexiones a los workers.
- El maestro crea el socket y se lo envía a los workers interesados, quienes aceptan las conexiones directamente.
La diferencia con un balanceador de carga externo es que el módulo cluster distribuye la carga entre procesos dentro de la misma máquina, mientras que un balanceador distribuye las peticiones entre diferentes servidores o máquinas.

**Fuente:** _imasters.com_

### Q18: Why do we need C++ Addons in Node.js? ☆☆☆☆☆

**Respuesta:**
Los **Addons (Complementos)** de Node.js son objetos compartidos enlazados dinámicamente, escritos en C++, que pueden cargarse en Node.js usando `require()`. Las razones para escribirlos son:
1. Acceder a APIs nativas que son difíciles de acceder solo con JS.
2. Integrar una biblioteca de terceros escrita en C/C++ y usarla directamente en Node.js.
3. Reescribir módulos en C++ por razones puramente de rendimiento.

**Fuente:** _nodejs.org_

### Q1: Explain the result of this code execution (Event Emit Infinity) ☆☆☆☆☆

**Detalles:**
```js
crazy.on("event1", function () { crazy.emit("event2"); });
crazy.on("event2", function () { crazy.emit("event3"); });
crazy.on("event3", function () { crazy.emit("event1"); });
crazy.emit("event1");
```
**Respuesta:**
Obtendrás una excepción de `Maximum call stack size exceeded` (Desbordamiento de pila). ¿Por qué? Cada `emit` invoca código síncrono. Debido a que todos los callbacks se ejecutan de manera síncrona, simplemente se llamará a sí mismo recursivamente hasta el infinito.

**Fuente:** _codementor.io_

### Q2: Explain the result of this code execution (setImmediate) ☆☆☆☆☆

**Detalles:**
```js
crazy.on("event1", function () { setImmediate(function () { crazy.emit("event2"); }); });
// igual para event2 y event3...
```
**Respuesta:**
La aplicación se ejecutará infinitamente. Al usar `setImmediate()`, cada callback se ejecuta como parte de la *siguiente iteración* del bucle de eventos, por lo que no ocurre una recursión profunda y no hay desbordamiento de pila (stack overflow).

**Fuente:** _codementor.io_

### Q3: What will happen when that code will be executed? (nextTick) ☆☆☆☆☆

**Detalles:**
```js
crazy.on("event1", function () { process.nextTick(function () { crazy.emit("event2"); }); });
// igual para event2 y event3...
```
**Respuesta:**
¡Se quedará atascado! Y eventualmente dará una excepción de "process out of memory" (falta de memoria). El problema no es el desbordamiento de pila, sino que `process.nextTick` bloquea completamente el bucle de eventos, porque siempre hay otro callback en la cola de `nextTick` que debe procesarse antes de avanzar a la siguiente fase. Eventualmente la recolección de basura (GC) no podrá recuperar la memoria.

**Fuente:** _codementor.io_

### Q4: Consider the code (Async/Await continuation) ☆☆☆☆☆

**Respuesta:**
El código demuestra cómo convertir múltiples promesas encadenadas con `.then()` en código secuencial mucho más limpio utilizando la sintaxis moderna `async` / `await`.

```js
async function addAsync(x) {
  const a = await doubleAfter2Seconds(10);
  const b = await doubleAfter2Seconds(20);
  const c = await doubleAfter2Seconds(30);
  return x + a + b + c;
}
```

**Fuente:** _medium.com_

### Q5: How do you handle backpressure in Node.js Streams? ☆☆☆☆☆

**Respuesta:**
La contrapresión (backpressure) ocurre cuando un stream de escritura procesa datos más lento de lo que el stream de lectura los emite. Esto hace que los datos se almacenen en la memoria (buffer), lo que puede provocar una caída por `heap out of memory`.
En Node.js, el método `.pipe()` maneja automáticamente la contrapresión deteniendo el stream de lectura cuando el buffer del stream de escritura alcanza su límite (`highWaterMark`), y reanudándolo una vez que se emite el evento `drain`.

### Q6: Explain the difference between Worker Threads, Cluster, and Child Processes in Node.js. ☆☆☆☆☆

**Respuesta:**
- **Cluster:** Usado para escalado horizontal. Clona (fork) el proceso principal de Node para crear múltiples instancias (workers) que comparten el mismo puerto de red. Ideal para manejar más tráfico web concurrente usando varios núcleos de CPU.
- **Child Processes:** Inicia un proceso del sistema operativo completamente independiente. No comparten memoria. La comunicación se realiza mediante IPC. Ideal para ejecutar scripts externos (ej. Python) o binarios del sistema.
- **Worker Threads:** Inicia un nuevo hilo dentro del *mismo* proceso de Node.js. Pueden compartir memoria usando `SharedArrayBuffer`. Ideal para tareas pesadas vinculadas a la CPU (procesamiento de imágenes, criptografía pesada) sin bloquear el bucle de eventos principal.

### Q7: How do you identify and fix memory leaks in a Node.js application? ☆☆☆☆☆

**Respuesta:**
Las fugas de memoria en Node.js suelen ocurrir debido a la acumulación de objetos en variables globales, closures no cerrados o event listeners abandonados.
Para identificarlos:
1. **Monitoreo:** Usar herramientas como `process.memoryUsage()`, o APMs (DataDog, New Relic) para observar el crecimiento de memoria.
2. **Heap Snapshots (Capturas de memoria):** Iniciar la app con `--inspect` y usar Chrome DevTools para tomar instantáneas. Compararlas para encontrar objetos que no están siendo recolectados por el Garbage Collector.
Para solucionarlos, asegúrate de remover los listeners de eventos (`.removeListener()`), evitar cachés globales masivos y cerrar limpiamente las conexiones.

### Q8: What is the difference between Macrotasks and Microtasks in the Event Loop? ☆☆☆☆☆

**Respuesta:**
En el bucle de eventos, las operaciones se encolan de manera diferente:
- **Microtareas (Microtasks):** Encoladas por `Promises` y `process.nextTick()`. Tienen prioridad absoluta. La cola de microtareas siempre se drena *completamente* antes de que el bucle de eventos pase a la siguiente fase de macrotareas.
- **Macrotareas (Macrotasks):** Incluyen `setTimeout`, `setInterval`, `setImmediate`, y callbacks de I/O. Se ejecutan en fases específicas del bucle de eventos.

### Q9: How do you implement Graceful Shutdown in a Node.js server? ☆☆☆☆☆

**Respuesta:**
Un apagado elegante (graceful shutdown) garantiza que cuando el servidor recibe una señal de apagado (como `SIGTERM` de Kubernetes/Docker), deje de aceptar nuevas conexiones pero termine de procesar las peticiones en curso antes de salir.
Pasos:
1. Escuchar la señal: `process.on('SIGTERM', gracefulShutdown);`
2. Dejar de aceptar peticiones: `server.close(() => { console.log('HTTP cerrado'); });`
3. Cerrar conexiones activas de base de datos.
4. Establecer un tiempo de espera (timeout) para forzar la salida `process.exit(1)` si el apagado tarda demasiado.
5. Salir limpiamente: `process.exit(0)`.
