# Preguntas de Entrevista de JavaScript

## Que es hoisting?

El hoisting es un comportamiento de JavaScript en el que las declaraciones de variables y funciones son movidas a la parte superior de su contexto de ejecución antes de que el código sea ejecutado. Esto significa que puedes usar variables y funciones antes de declararlas en tu código, aunque solo las declaraciones son hoisted, no las asignaciones.

Ejemplo de hoisting con variables:

```javascript
console.log(x); // undefined
var x = 5;
```

## Que es un closure?

Un closure es una función que tiene acceso a las variables de su ámbito externo, incluso después de que la función externa haya terminado de ejecutarse. Esto permite que la función interna "recuerde" el entorno en el que fue creada. Los closures son útiles para crear funciones privadas y para mantener el estado entre llamadas a funciones.

```javascript
function outerFunction() {
  var outerVariable = "I am from the outer function";

  function innerFunction() {
    console.log(outerVariable); // Accede a la variable del ámbito externo
  }
  return innerFunction;
}
var closureExample = outerFunction();
closureExample(); // Output: I am from the outer function
```

### P1: Explica la igualdad en JavaScript ☆

**Respuesta:**
JavaScript tiene comparaciones estrictas y comparaciones con conversión de tipo:

- **Comparación estricta (por ejemplo, ===)** verifica la igualdad de valor sin permitir _coerción_
- **Comparación abstracta (por ejemplo, ==)** verifica la igualdad de valor permitiendo la _coerción_

```js
var a = "42";
var b = 42;

a == b; // true
a === b; // false
```

Algunas reglas simples de igualdad:

- Si cualquiera de los valores en una comparación podría ser `true` o `false`, evita `==` y usa `===`.
- Si cualquiera de los valores podría ser `0`, `""` o `[]` (arreglo vacío), evita `==` y usa `===`.
- En los demás casos, puedes usar `==` de forma segura. No solo es seguro, sino que en muchos casos simplifica tu código y mejora la legibilidad.

### P2: Explica Null y Undefined en JavaScript ☆☆

**Respuesta:**
JavaScript (y por extensión TypeScript) tiene dos tipos base: `null` y `undefined`. Están _diseñados_ para significar cosas diferentes:

- Algo no ha sido inicializado: `undefined`.
- Algo está actualmente no disponible: `null`.

### P3: ¿Qué es el Scope en JavaScript? ☆

**Respuesta:**
En JavaScript, cada función tiene su propio _scope_ (ámbito). Scope es básicamente una colección de variables y las reglas de cómo se accede a esas variables por nombre. Solo el código dentro de esa función puede acceder a las variables de ese scope.

Un nombre de variable debe ser único dentro del mismo scope. Un scope puede estar anidado dentro de otro. Si un scope está anidado, el código dentro del scope más interno puede acceder a variables de ambos scopes.

### P4: Explica Valores y Tipos en JavaScript ☆☆

**Respuesta:**
JavaScript tiene valores tipados, no variables tipadas. Los siguientes tipos integrados están disponibles:

- `string`
- `number`
- `boolean`
- `null` y `undefined`
- `object`
- `symbol` (nuevo en ES6)

### P5: ¿Qué es el operador typeof? ☆

**Respuesta:**
JavaScript provee el operador `typeof` que examina un valor y te dice qué tipo es:

### P6: ¿Qué es el tipo objeto? ☆

**Respuesta:**
El tipo objeto se refiere a un valor compuesto donde puedes establecer propiedades (ubicaciones nombradas) que pueden contener valores de cualquier tipo.

```js
var obj = {
  a: "hola mundo", // propiedad
  b: 42,
  c: true,
};

obj.a; // "hola mundo", notación punto
obj.b; // 42
obj.c; // true

obj["a"]; // "hola mundo", notación corchetes
obj["b"]; // 42
obj["c"]; // true
```

La notación de corchetes también es útil si quieres acceder a una propiedad pero el nombre está almacenado en otra variable:

```js
var obj = {
  a: "hola mundo",
  b: 42,
};

var b = "a";

obj[b]; // "hola mundo"
obj["b"]; // 42
```

### P7: Explica los arreglos en JavaScript ☆

**Respuesta:**
Un `array` es un objeto que contiene valores (de cualquier tipo) no en propiedades nombradas, sino en posiciones indexadas numéricamente:

```js
var arr = ["hola mundo", 42, true];

arr[0]; // "hola mundo"
arr[1]; // 42
arr[2]; // true
arr.length; // 3

typeof arr; // "object"
```

### P8: ¿Qué es la coerción en JavaScript?

**Respuesta:**
En JavaScript, la conversión entre diferentes tipos integrados se llama `coerción`. Hay dos formas: _explícita_ e _implícita_.

Ejemplo de coerción explícita:

```js
var a = "42";

var b = Number(a);

a; // "42"
b; // 42 -- ¡el número!
```

Ejemplo de coerción implícita:

```js
var a = "42";

var b = a * 1; // "42" se convierte implícitamente a 42

a; // "42"
b; // 42 -- ¡el número!
```

### P9: ¿Qué es el modo estricto? ☆☆

**Respuesta:**
_El modo estricto_ es una característica de ECMAScript 5 que permite poner un programa o función en un contexto "estricto". Este contexto previene ciertas acciones y lanza más excepciones.

```js
// Código no estricto...

(function () {
  "use strict";

  // Define tu librería estrictamente...
})();

// Código no estricto...
```

### P10: ¿Qué es la palabra clave let en JavaScript? ☆☆

**Respuesta:**
Además de crear declaraciones de variables a nivel de función, ES6 permite declarar variables que pertenecen a bloques individuales (pares de { .. }) usando la palabra clave `let`.

**Fuente:** _github.com/getify_

### P11: ¿Qué es un Polyfill? ☆☆

**Respuesta:**
Un polyfill es código (o plugin) que permite que una funcionalidad esperada en navegadores modernos también funcione en otros navegadores que no la soportan de forma nativa.

- Los polyfills no son parte del estándar HTML5
- Polyfilling no está limitado a Javascript

**Fuente:** _programmerinterview.com_

### P12: Dado un arreglo desordenado que contiene (n - 1) de n números consecutivos (donde los límites están definidos), encuentra el número faltante en O(n) tiempo ☆☆

**Respuesta:**

```js
// El resultado de la función debe ser 8
var arrayOfIntegers = [2, 5, 1, 4, 9, 6, 3, 7];
var upperBound = 9;
var lowerBound = 1;

findMissingNumber(arrayOfIntegers, upperBound, lowerBound); // 8

function findMissingNumber(arrayOfIntegers, upperBound, lowerBound) {
  // Itera para encontrar la suma de los números
  var sumOfIntegers = 0;
  for (var i = 0; i < arrayOfIntegers.length; i++) {
    sumOfIntegers += arrayOfIntegers[i];
  }

  // Suma teórica usando una variación de la suma de Gauss.
  // Fórmula: [(N * (N + 1)) / 2] - [(M * (M - 1)) / 2];
  // N es el límite superior y M el inferior

  upperLimitSum = (upperBound * (upperBound + 1)) / 2;
  lowerLimitSum = (lowerBound * (lowerBound - 1)) / 2;

  theoreticalSum = upperLimitSum - lowerLimitSum;

  return theoreticalSum - sumOfIntegers;
}
```

**Fuente:** _https://github.com/kennymkchan_

### P13: Elimina duplicados de un arreglo y retorna solo los elementos únicos ☆☆

**Respuesta:**

```js
// Implementación ES6
var array = [1, 2, 3, 5, 1, 5, 9, 1, 2, 8];

Array.from(new Set(array)); // [1, 2, 3, 5, 9, 8]

// Implementación ES5
var array = [1, 2, 3, 5, 1, 5, 9, 1, 2, 8];

uniqueArray(array); // [1, 2, 3, 5, 9, 8]

function uniqueArray(array) {
  var hashmap = {};
  var unique = [];

  for (var i = 0; i < array.length; i++) {
    // Si la clave es undefined (único), se evalúa como falso.
    if (!hashmap.hasOwnProperty(array[i])) {
      hashmap[array[i]] = 1;
      unique.push(array[i]);
    }
  }

  return unique;
}
```

**Fuente:** _https://github.com/kennymkchan_

### P14: Dada una cadena, invierte cada palabra en la oración ☆☆

**Detalles:**
Por ejemplo, `Welcome to this Javascript Guide!` debería convertirse en `emocleW ot siht tpircsavaJ !ediuG`

**Respuesta:**

```js
var string = "Welcome to this Javascript Guide!";

// Salida: !ediuG tpircsavaJ siht ot emocleW
var reverseEntireSentence = reverseBySeparator(string, "");

// Salida: emocleW ot siht tpircsavaJ !ediuG
var reverseEachWord = reverseBySeparator(reverseEntireSentence, " ");

function reverseBySeparator(string, separator) {
  return string.split(separator).reverse().join(separator);
}
```

**Fuente:** _https://github.com/kennymkchan_

### P15: Implementa enqueue y dequeue usando solo dos pilas ☆☆

**Respuesta:**
_Encolar_ significa agregar un elemento, _desencolar_ eliminarlo.

```js
var inputStack = []; // Primera pila
var outputStack = []; // Segunda pila

// Para encolar, solo haz push en la primera pila
function enqueue(stackInput, item) {
  return stackInput.push(item);
}

function dequeue(stackInput, stackOutput) {
  // Invierte la pila para que el primer elemento de la salida
  // sea el último de la entrada. Luego haz pop para obtener el primero que se insertó.
  if (stackOutput.length <= 0) {
    while (stackInput.length > 0) {
      var elementToOutput = stackInput.pop();
      stackOutput.push(elementToOutput);
    }
  }

  return stackOutput.pop();
}
```

**Fuente:** _https://github.com/kennymkchan_

### P16: Explica el event bubbling y cómo prevenirlo ☆☆

**Respuesta:**
**Event bubbling** es el concepto en el que un evento se dispara en el elemento más profundo posible y luego en los elementos padre en orden de anidamiento. Como resultado, al hacer clic en un hijo puede activarse el manejador del padre.

Una forma de prevenir el event bubbling es usando `event.stopPropagation()` o `event.cancelBubble` en IE < 9.

**Fuente:** _https://github.com/kennymkchan_

### P17: Escribe una función "mul" que funcione correctamente con la siguiente sintaxis. ☆☆

**Detalles:**

```javascript
console.log(mul(2)(3)(4)); // salida: 24
console.log(mul(4)(3)(4)); // salida: 48
```

**Respuesta:**

```javascript
function mul(x) {
  return function (y) {
    // función anónima
    return function (z) {
      // función anónima
      return x * y * z;
    };
  };
}
```

Aquí, la función `mul` acepta el primer argumento y retorna una función anónima que toma el segundo parámetro y retorna otra función anónima que toma el tercer parámetro y retorna la multiplicación de los argumentos pasados sucesivamente.

En JavaScript, una función definida dentro de otra tiene acceso a las variables externas y las funciones son objetos de primera clase, por lo que pueden ser retornadas o pasadas como argumento.

- Una función es una instancia del tipo Object
- Una función puede tener propiedades y referencia a su constructor
- Una función puede almacenarse en una variable
- Puede pasarse como parámetro a otra función
- Puede ser retornada desde otra función

**Fuente:** _github.com/ganqqwerty_

### P18: ¿Cómo vaciar un arreglo en JavaScript? ☆☆

**Detalles:**

```js
var arrayList = ["a", "b", "c", "d", "e", "f"];
```

¿Cómo podríamos vaciar el arreglo anterior?

**Respuesta:**
**Método 1**

```javascript
arrayList = [];
```

Este código asigna una nueva lista vacía. Es recomendable si **no tienes referencias al arreglo original** en otro lugar. Si tienes referencias, solo el nombre cambia, pero el arreglo original sigue igual.

Por ejemplo:

```javascript
var arrayList = ["a", "b", "c", "d", "e", "f"];
var anotherArrayList = arrayList;
arrayList = [];
console.log(anotherArrayList); // ['a', 'b', 'c', 'd', 'e', 'f']
```

**Método 2**

```javascript
arrayList.length = 0;
```

Esto vacía el arreglo y actualiza todas las referencias.

Por ejemplo:

```javascript
var arrayList = ["a", "b", "c", "d", "e", "f"];
var anotherArrayList = arrayList;
arrayList.length = 0;
console.log(anotherArrayList); // []
```

**Método 3**

```javascript
arrayList.splice(0, arrayList.length);
```

Esto también vacía el arreglo y actualiza todas las referencias.

```javascript
var arrayList = ["a", "b", "c", "d", "e", "f"];
var anotherArrayList = arrayList;
arrayList.splice(0, arrayList.length);
console.log(anotherArrayList); // []
```

**Método 4**

```javascript
while (arrayList.length) {
  arrayList.pop();
}
```

No se recomienda usar este método frecuentemente.

**Fuente:** _github.com/ganqqwerty_

### P19: ¿Cómo saber si un objeto es un arreglo o no? Da código. ☆☆

**Respuesta:**

> La mejor forma es usando el método `toString` de `Object.prototype`.

```javascript
var arrayList = [1, 2, 3];
```

Un caso común es cuando sobrecargas métodos y necesitas saber si el parámetro es un arreglo:

```javascript
function greet(param) {
  if (typeof param === "string") {
    // lógica para string
  } else {
    // lógica para arreglo
  }
}
```

Pero si el parámetro puede ser string, arreglo u objeto, necesitas una comprobación más robusta:

```javascript
if (Object.prototype.toString.call(arrayList) === "[object Array]") {
  console.log("¡Es un arreglo!");
}
```

Si usas jQuery, puedes usar `$.isArray`:

```javascript
if ($.isArray(arrayList)) {
  console.log("Arreglo");
} else {
  console.log("No es un arreglo");
}
```

En navegadores modernos:

```javascript
Array.isArray(arrayList);
```

`Array.isArray` es soportado por Chrome 5, Firefox 4.0, IE 9, Opera 10.5 y Safari 5

**Fuente:** _github.com/ganqqwerty_

### P20: ¿Cómo usar un closure para crear un contador privado? ☆☆

**Respuesta:**
Puedes crear una función dentro de otra (closure) que permita actualizar una variable privada, pero esa variable no será accesible desde fuera sin una función auxiliar.

```js
function counter() {
  var _counter = 0;
  // retorna un objeto con funciones para modificar la variable privada
  return {
    add: function (increment) {
      _counter += increment;
    },
    retrieve: function () {
      return "El contador está en: " + _counter;
    },
  };
}

// error si intentamos acceder a _counter directamente
// _counter;

// uso del contador
var c = counter();
c.add(5);
c.add(9);

// ahora accedemos a la variable privada así
c.retrieve(); // => El contador está en: 14
```

**Fuente:** _coderbyte.com_
