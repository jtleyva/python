# Guía de Conceptos de Frontend, JavaScript y Vue.js para Entrevista Técnica

## 1. Fundamentos de Frontend

- **HTML**: Estructura básica de las páginas web. Semántica, etiquetas principales, formularios, accesibilidad.
- **CSS**: Estilos, selectores, especificidad, box model, flexbox, grid, responsive design, preprocesadores (Sass, Less).
- **JavaScript**: Lenguaje de programación principal del navegador. Manipulación del DOM, eventos, asincronía, promesas, fetch API.
- **Web APIs**: LocalStorage, SessionStorage, Service Workers, WebSockets, APIs de geolocalización, etc.
- **Performance**: Lazy loading, optimización de imágenes, minimización de recursos, critical rendering path.
- **Accesibilidad (a11y)**: Buenas prácticas, atributos ARIA, navegación por teclado.
- **SEO básico**: Meta tags, sitemap, robots.txt, SSR vs CSR.

## 2. JavaScript Profundo

- **Tipos de datos**: Primitivos, objetos, arrays, funciones.
- **Scope y Hoisting**: Var, let, const, closures, contexto de ejecución.
- **This**: Cómo funciona en distintos contextos, bind, call, apply.
- **Funciones**: Declarativas, expresiones, arrow functions, callbacks, IIFE.
- **Asincronía**: Callbacks, Promesas, async/await, manejo de errores.
- **Event Loop**: Stack, heap, queue, microtasks vs macrotasks.
- **Módulos**: CommonJS, ES Modules (import/export).
- **Prototipos y Herencia**: Prototype chain, clases ES6, super, extends.
- **Manipulación del DOM**: querySelector, addEventListener, creación y eliminación de nodos.
- **JSON**: Serialización y deserialización, diferencias con objetos JS.
- **Testing**: Vitest, Jest, Mocha, testing de componentes y funciones.

## 3. Vue.js (v2 y v3)

- **Instancia de Vue**: Ciclo de vida, hooks principales (created, mounted, updated, destroyed).
- **Componentes**: Props, eventos personalizados, slots, scoped slots, composición de componentes.
- **Reactividad**: Data, computed, watch, diferencias entre v2 y v3 (Composition API: ref, reactive, setup).
- **Directivas**: v-if, v-for, v-bind, v-model, v-on, directivas personalizadas.
- **Gestión de Estado**: Vuex (v2), Pinia (v3), patrones de lifting state up.
- **Ruteo**: Vue Router, rutas anidadas, navegación programática, guards.
- **Comunicación entre componentes**: Props, eventos, provide/inject, event bus (anti-patrón).
- **Plugins y Ecosistema**: Uso de plugins, librerías de UI (Vuetify, Element), internacionalización.
- **Testing en Vue**: Vue Test Utils, testing de componentes, mocks y spies.
- **SSR y SSG**: Nuxt.js, diferencias entre renderizado del lado del cliente y del servidor.

## 4. Herramientas y Buenas Prácticas

- **Control de versiones**: Git, flujos de trabajo (feature branch, pull requests).
- **Bundlers**: Webpack, Vite, diferencias y casos de uso.
- **Linting y Formateo**: ESLint, Prettier.
- **CI/CD**: Automatización de pruebas y despliegues.
- **Documentación**: Storybook, documentación de componentes.

---

## 1.1 Preguntas Comunes de HTML

- ¿Qué es el DOM y cómo se relaciona con HTML?

  Respuesta: El DOM (Document Object Model) es una representación en árbol del documento HTML que el navegador construye al parsear el HTML. Es una API viva: cualquier cambio en el DOM se refleja inmediatamente en la pantalla. Es importante distinguir el **DOM real** del **Virtual DOM** que usan frameworks como Vue o React — este último es una abstracción en memoria que minimiza los reflows y repaints costosos al comparar diffs antes de actualizar el DOM real. Para un senior, también es relevante entender el **Shadow DOM**, usado en Web Components para encapsular estilos y estructura.

- ¿Cuál es la diferencia entre etiquetas semánticas y no semánticas?

  Respuesta: Las etiquetas semánticas en HTML tienen un significado claro y describen el propósito del contenido que contienen, como `<header>`, `<nav>`, `<article>`, `<section>`, etc. Estas etiquetas mejoran la accesibilidad y el SEO. Por otro lado, las etiquetas no semánticas, como `<div>` y `<span>`, no tienen un significado específico y se utilizan principalmente para agrupar elementos o aplicar estilos.

- ¿Cómo se implementa la accesibilidad en HTML?

  Respuesta: La accesibilidad se implementa en múltiples capas. A nivel HTML: usar etiquetas semánticas (`<nav>`, `<main>`, `<article>`, `<button>` en vez de `<div>` clickeable), `alt` en imágenes, `label` asociado a inputs, y orden lógico del DOM. A nivel ARIA: añadir `role`, `aria-label`, `aria-describedby`, `aria-expanded`, `aria-live` (para regiones dinámicas) solo cuando el HTML semántico no alcanza — **ARIA no reemplaza HTML semántico, lo complementa**. Para senior: asegurarse de que los componentes custom (dropdowns, modals, tooltips) sigan el patrón WAI-ARIA correcto, que el foco sea manejado explícitamente (focus trap en modals), y que el contraste de color cumpla WCAG 2.1 nivel AA (ratio mínimo 4.5:1).

- ¿Qué son los formularios y cómo se manejan en HTML?

  Respuesta: Los formularios en HTML se utilizan para recopilar datos del usuario. Se crean utilizando la etiqueta `<form>` y pueden contener diversos elementos de entrada como `<input>`, `<textarea>`, `<select>`, etc. El manejo de formularios implica la validación de los datos, el envío de la información al servidor (usando el atributo `action` y el método `method`), y la gestión de eventos para proporcionar retroalimentación al usuario.

## 1.2 Preguntas Comunes de CSS

- ¿Qué es el box model y cómo afecta al diseño de una página?

  Respuesta: Describe cómo se calculan las dimensiones y el espacio de los elementos en una página web. Cada elemento se representa como una caja rectangular que consta de cuatro partes: el contenido, el padding (relleno), el border (borde) y el margin (margen). El tamaño total de un elemento se calcula sumando estas partes, lo que afecta al diseño y la disposición de los elementos en la página.

- ¿Cómo funciona la especificidad en CSS?

  Respuesta: La especificidad se calcula como un vector de tres componentes `(A, B, C)` donde A = número de IDs, B = número de clases/atributos/pseudoclases, C = número de tipos/pseudoelementos. Se comparan de izquierda a derecha: `(1,0,0)` siempre gana a `(0,99,99)`. El `!important` rompe este sistema y crea su propio orden de cascada — su uso indiscriminado es una **deuda técnica seria** en proyectos grandes. Para senior: entender la **cascade** completa (origen, importancia, especificidad, orden) y saber que el `inline style` tiene especificidad `(1,0,0,0)` (cuatro componentes cuando se incluye `!important`). Estrategias como BEM, CSS Modules o utility-first (Tailwind) existen precisamente para evitar guerras de especificidad.

- ¿Qué es el responsive design y cómo se implementa?

  Respuesta: El responsive design es una técnica de diseño web que permite que una página se adapte a diferentes tamaños de pantalla y dispositivos. Se implementa utilizando media queries en CSS para aplicar estilos específicos según el ancho del dispositivo, así como utilizando unidades relativas (como %, em, rem) en lugar de unidades fijas (px) para garantizar que los elementos se escalen adecuadamente.

- ¿Cuáles son las diferencias entre flexbox y grid?

  Respuesta: Flexbox es **unidimensional**: organiza elementos en un eje (fila o columna) y deja que el otro eje fluya naturalmente. Grid es **bidimensional**: controla filas y columnas simultáneamente desde el contenedor. La regla práctica: usa Flexbox para componentes (nav, botones, alineación de ítems dentro de una card) y Grid para layouts de página. No son excluyentes — un layout Grid puede contener Flex containers. Para senior: entender `auto-fit` vs `auto-fill` en Grid, el modelo de alineación `justify-content` vs `align-items` vs `place-items`, y cuándo `fr` (fracción) es más correcto que `%`. También conocer las implicaciones de performance: Grid genera menos DOM manipulation en resizes comparado con soluciones JavaScript.

- ¿Qué son los preprocesadores CSS y cuáles son sus ventajas?

  Respuesta: Los preprocesadores como Sass y Less compilan a CSS estándar y aportan variables, mixins, funciones, anidamiento e importaciones modulares. Hoy en día, **CSS nativo ha cerrado gran parte de la brecha**: `custom properties` (`--color: red`) son variables nativas con soporte en cascada (algo que Sass no tiene), y `@layer` permite controlar la cascada explícitamente. Para senior: saber cuándo un preprocesador agrega valor real vs cuándo es overhead innecesario. En proyectos modernos con Vue/React, **CSS Modules** o **CSS-in-JS** (styled-components, UnoCSS, Tailwind) suelen resolver mejor el problema de encapsulación y colisión de estilos que Sass solo.

## 1.3 Preguntas Comunes de JavaScript

- ¿Qué es el event loop y cómo funciona?

  Respuesta: El event loop es el mecanismo que permite a JavaScript (single-threaded) manejar concurrencia sin bloquear el hilo principal. Tiene tres componentes clave: **Call Stack** (ejecuta el código síncrono), **Microtask Queue** (Promises, `queueMicrotask`, `MutationObserver`) y **Macrotask Queue** (setTimeout, setInterval, I/O, eventos DOM). El orden de ejecución en cada "tick": 1) ejecuta el stack hasta vaciarlo, 2) vacía completamente la microtask queue (incluso si se agregan nuevas durante el proceso), 3) toma **una** macrotask y repite. Este orden es crítico: las Promises resuelven ANTES que un `setTimeout(..., 0)`.

  ```javascript
  console.log("1"); // sync
  setTimeout(() => console.log("2"), 0); // macrotask
  Promise.resolve().then(() => console.log("3")); // microtask
  console.log("4"); // sync
  // Output: 1, 4, 3, 2
  ```

  Para senior: entender por qué una microtask que genera microtasks infinitas puede **bloquear el render** del navegador, y cómo `requestAnimationFrame` se ubica entre macrotasks para sincronizar con el ciclo de pintura.

- ¿Cuál es la diferencia entre `var`, `let` y `const`?
  Respuesta: `var` es una declaración de variable que tiene un alcance de función y se puede redeclarar. `let` tiene un alcance de bloque y no se puede redeclarar dentro del mismo bloque. `const` también tiene un alcance de bloque, pero además no permite reasignar el valor después de su inicialización.

- ¿Qué son las promesas y cómo se usan?

  Respuesta: Las Promises representan un valor que puede estar disponible ahora, en el futuro, o nunca. Tienen tres estados: `pending`, `fulfilled`, `rejected` — y una vez que transicionan a fulfilled/rejected son **inmutables**. Los métodos esenciales para un senior:
  - `Promise.all([p1, p2])` — espera todas; falla rápido si cualquiera rechaza. Ideal para operaciones paralelas independientes.
  - `Promise.allSettled([p1, p2])` — espera todas sin importar si fallan. útil cuando necesitas el resultado de cada una individualmente.
  - `Promise.race([p1, p2])` — resuelve/rechaza con la primera que complete. Útil para timeouts.
  - `Promise.any([p1, p2])` — resuelve con la primera que tenga éxito; rechaza solo si todas fallan.

  ```javascript
  // Patrón común: paralelismo con manejo de errores individual
  const results = await Promise.allSettled([fetchUser(), fetchOrders()]);
  const [user, orders] = results.map((r) =>
    r.status === "fulfilled" ? r.value : null,
  );
  ```

  Anti-patrón a evitar: **Promise Hell** (anidar `.then()` como si fueran callbacks) y **await en un loop** sin necesidad (serializa operaciones que podrían ir en paralelo).

- ¿Cómo funciona el `this` en JavaScript?
  Respuesta: El valor de `this` se determina en **tiempo de ejecución**, no de definición (excepto en arrow functions). Las reglas en orden de prioridad:
  1. **`new` binding**: cuando se usa `new`, `this` es el objeto recien creado.
  2. **Explicit binding**: `call(ctx)`, `apply(ctx)`, `bind(ctx)` fijan `this` explícitamente.
  3. **Implicit binding**: `obj.method()` — `this` es `obj`.
  4. **Default binding**: función llamada sola — `this` es `globalThis` (o `undefined` en strict mode).
  5. **Arrow functions**: NO tienen su propio `this`; capturan el `this` del scope léxico donde fueron definidas. No pueden ser usadas como constructores ni con `bind`.

  ```javascript
  const obj = {
    name: "Alice",
    greet() {
      return `Hi ${this.name}`;
    }, // implicit
    greetArrow: () => `Hi ${this.name}`, // this = outer scope (NO obj)
  };
  const fn = obj.greet;
  fn(); // undefined (default binding, strict) — bug clásico en callbacks
  ```

  Para senior: el bug más común es perder `this` al pasar métodos como callbacks (ej. `setTimeout(obj.greet, 0)`). La solución idiomática hoy es usar arrow functions en class fields o `bind` en el constructor.

- ¿Qué es el hoisting y cómo afecta a las variables y funciones?

  Respuesta: Hoisting es el proceso por el cual el motor de JavaScript mueve las **declaraciones** (no las inicializaciones) al inicio de su scope durante la fase de compilación, antes de ejecutar el código. Los comportamientos difieren:
  - **`var`**: la declaración se eleva e inicializa como `undefined`. El valor se asigna en tiempo de ejecución.
  - **`let` / `const`**: la declaración se eleva pero NO se inicializa. Existen en la **Temporal Dead Zone (TDZ)** desde el inicio del bloque hasta la línea de declaración — acceder en ese intervalo lanza `ReferenceError`.
  - **Function declarations**: se elevan completamente (nombre + cuerpo), por eso pueden llamarse antes de declararse.
  - **Function expressions / arrow functions**: se comportan como `var` o `let`/`const` según cómo se declaren.

  ```javascript
  console.log(x); // undefined (var hoisted)
  var x = 5;

  console.log(y); // ReferenceError: Cannot access 'y' before initialization (TDZ)
  let y = 10;

  greet(); // ✅ funciona — function declaration es fully hoisted
  function greet() {
    return "hello";
  }

  sayBye(); // ❌ TypeError: sayBye is not a function
  var sayBye = () => "bye";
  ```

  Para senior: la TDZ es intencional — evita bugs de acceso antes de inicialización. Entender esto diferencia a alguien que usa `let`/`const` por moda de alguien que entiende el motor.

- ¿Qué es una closure y cómo se utiliza en JavaScript?

  Respuesta: Una closure es una función que "cierra sobre" las variables de su scope léxico, manteniendo una referencia viva a ese entorno incluso después de que la función externa haya retornado. No es una característica opcional — **en JavaScript todas las funciones son closures**.

  ```javascript
  function outer() {
    let count = 0;
    return function inner() {
      count++;
      console.log(count);
    };
  }

  const counter = outer();
  counter(); // 1
  counter(); // 2
  ```

  Usos reales y patrones senior:
  - **Módulo pattern**: encapsular estado privado antes de que existieran clases.
  - **Partial application / currying**: `const add5 = add(5)` — fija argumentos parcialmente.
  - **Memoization**: cachear resultados en una variable del scope externo.
  - **Callbacks y event handlers** que necesitan contexto.

  **Gotcha crítico (memory leaks)**: si una closure referencia un objeto DOM grande o datos pesados, ese objeto no puede ser garbage-collected mientras la closure exista. En SPAs con muchos listeners no removidos, esto genera leaks reales. La regla: si registras un listener con una closure, desregistralo en cleanup (`onUnmounted`, `removeEventListener`).

- ¿Qué es el scope y cómo funciona en JavaScript?

  Respuesta: El scope (ámbito) en JavaScript se refiere al contexto en el que las variables y funciones son accesibles. Hay tres tipos principales de scope: global, de función y de bloque. El scope global es accesible desde cualquier parte del código, el scope de función es accesible solo dentro de la función donde se declara, y el scope de bloque (introducido con `let` y `const`) es accesible solo dentro del bloque donde se declara (como un `if` o un `for`). El scope también determina la visibilidad y el ciclo de vida de las variables.

## 1.4 Preguntas Comunes de Vue.js

- ¿Qué es el ciclo de vida de un componente en Vue.js?

  Respuesta: El ciclo de vida de un componente en Vue.js se refiere a las etapas por las que pasa un componente desde su creación hasta su destrucción. Las etapas principales incluyen: `beforeCreate`, `created`, `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeDestroy`, y `destroyed`. Cada etapa tiene hooks específicos que permiten ejecutar código en momentos clave del ciclo de vida del componente.

- ¿Cómo se comunican los componentes en Vue.js?

  Respuesta: Para la comunicación de padre a hijo, se utilizan props para pasar datos. Para la comunicación de hijo a padre, se emiten eventos personalizados que el componente padre puede escuchar. Para la comunicación entre componentes hermanos o no relacionados, se pueden usar un bus de eventos (aunque es un anti-patrón) o una solución de gestión de estado como Vuex o Pinia.

- ¿Qué es la reactividad en Vue.js y cómo funciona?

  Respuesta: La reactividad es el sistema que conecta el estado (datos) con la vista (DOM) de forma automática. La implementación difiere radicalmente entre versiones:
  - **Vue 2**: usa `Object.defineProperty` para interceptar getters/setters. Limitación crítica: **no puede detectar propiedades agregadas o eliminadas dinámicamente** ni cambios en índices de arrays. Por eso existía `Vue.set()` / `this.$set()`.
  - **Vue 3**: usa `Proxy` de ES6 que envuelve el objeto completo. Detecta **cualquier mutación** (agregar/eliminar propiedades, cambiar índices). Más eficiente y sin las limitaciones de Vue 2.

  ```javascript
  // Vue 3 — sistema reactivo
  import { reactive, ref, computed, watchEffect } from "vue";

  const state = reactive({ count: 0 }); // Proxy-based
  const double = computed(() => state.count * 2); // lazy, cached
  watchEffect(() => console.log(state.count)); // se re-ejecuta automáticamente

  state.newProp = "x"; // ✅ Vue 3 lo detecta; Vue 2 no lo hubiera detectado
  ```

  Para senior: entender la diferencia entre `ref` (envuelve primitivos en `{ value }`) y `reactive` (envuelve objetos con Proxy), y por qué no deberías desestructurar un `reactive` — pierdes la reactividad.

- ¿Cuáles son las diferencias entre Vue 2 y Vue 3?

  Respuesta: Las diferencias arquitecturales clave que debe conocer un senior:

  | Área               | Vue 2                              | Vue 3                                      |
  | ------------------ | ---------------------------------- | ------------------------------------------ |
  | Reactividad        | `Object.defineProperty` (limitada) | `Proxy` (completa)                         |
  | API de componentes | Options API                        | Options API + **Composition API**          |
  | Bundle size        | Mayor                              | Tree-shakeable — solo importas lo que usas |
  | TypeScript         | Soporte parcial                    | First-class support                        |
  | Fragments          | No (un solo root)                  | Sí (múltiples roots)                       |
  | Teleport           | No                                 | Sí (`<Teleport>` para modals/portals)      |
  | Suspense           | No                                 | Experimental (async components)            |
  | Performance        | Baseline                           | Hasta 2x más rápido en benchmarks          |

  La diferencia **conceptual más importante**: el Composition API no es solo sintaxis nueva — permite extraer lógica con estado en **composables** reutilizables, algo que con Options API requeria mixins (que tienen colisión de nombres, origen opaco y no son type-safe). Los composables resuelven estos problemas.

- ¿Cómo se implementa el manejo de estado en Vue.js?

  Respuesta: Vuex en Vue 2 o Pinia en Vue 3. Estas son bibliotecas de gestión de estado que permiten centralizar el estado de la aplicación y facilitar la comunicación entre componentes. También es posible manejar el estado localmente dentro de los componentes utilizando `data` y `computed`, o compartir estado entre componentes utilizando `provide` e `inject`.

- ¿Qué son las directivas en Vue.js y cómo se usan?

  Respuesta: Las directivas en Vue.js son atributos especiales que se utilizan para aplicar comportamientos reactivos a los elementos del DOM. Algunas directivas comunes incluyen `v-if` para renderizado condicional, `v-for` para renderizado de listas, `v-bind` para enlazar atributos, `v-model` para enlace bidireccional de datos, y `v-on` para manejar eventos. Las directivas pueden ser personalizadas para agregar funcionalidades específicas a los componentes.

- ¿Qué es el Composition API en Vue 3 y cómo se diferencia del Options API?

  Respuesta: El Options API organiza el código por **tipo de opción** (`data`, `methods`, `computed`, `watch`) — en componentes grandes, la lógica relacionada queda fragmentada. El Composition API organiza el código por **preocupación lógica** dentro de `setup()`, permitiendo agrupar todo lo de una feature junta.

  La herramienta más poderosa que habilita el Composition API son los **composables**: funciones que encapsulan lógica reactiva y son reutilizables entre componentes.

  ```javascript
  // composables/useCounter.js — lógica extraída y reutilizable
  import { ref, computed } from "vue";

  export function useCounter(initial = 0) {
    const count = ref(initial);
    const double = computed(() => count.value * 2);
    const increment = () => count.value++;
    return { count, double, increment };
  }

  // En cualquier componente:
  import { useCounter } from "@/composables/useCounter";
  const { count, increment } = useCounter(10);
  ```

  Para senior: los composables reemplazan a los **mixins** con ventajas claras — origen explícito de cada valor, sin colisión de nombres, completamente type-safe. Con `<script setup>` (azucar sintáctico sobre `setup()`), el código es aún más limpio y tiene mejor rendimiento en compilación.

- ¿Cómo se maneja el ruteo en Vue.js?
  Respuesta: Vue Router es la solución oficial. Para un senior, lo importante va más allá de definir rutas básicas:
  - **Lazy loading de rutas** con `() => import('./views/Home.vue')` — imprescindible para performance, reduce el bundle inicial.
  - **Navigation Guards**: `beforeEach` (global), `beforeEnter` (por ruta), `beforeRouteEnter/Leave/Update` (en componente). Se usan para autenticación, autorización, o guardar estado antes de salir.
  - **Rutas dinámicas**: `/user/:id` — con `useRoute().params.id` en Composition API.
  - **Rutas anidadas**: layouts compartidos sin duplicar markup.
  - **Scroll behavior**: controlar la posición del scroll al navegar.

  ```javascript
  // Guard de autenticación con Composition API
  router.beforeEach((to, from) => {
    const auth = useAuthStore();
    if (to.meta.requiresAuth && !auth.isLoggedIn) {
      return { name: "Login", query: { redirect: to.fullPath } };
    }
  });
  ```

  Para senior: el anti-patrón más común es hacer lógica de negocio dentro de los guards en vez de delegar a stores (Pinia). Los guards solo deben decidir si navegar o redirigir, no hacer fetches.

- ¿Qué es Nuxt.js y cómo se relaciona con Vue.js?

  Respuesta: Nuxt.js es un framework de alto nivel basado en Vue.js que facilita el desarrollo de aplicaciones universales (isomórficas) y de una sola página (SPA). Proporciona características como renderizado del lado del servidor (SSR), generación de sitios estáticos (SSG), manejo automático de rutas, y una estructura de proyecto predefinida. Nuxt.js es ideal para mejorar el SEO y el rendimiento de las aplicaciones Vue.

- Como funciona el SSR, SSG y CSR en Vue.js?

  Respuesta:
  - **CSR (Client-Side Rendering)**: El navegador descarga un HTML mínimo y luego ejecuta JavaScript para renderizar la aplicación. Es rápido para interacciones después de la carga inicial, pero el tiempo hasta el primer render puede ser lento, especialmente en dispositivos móviles o conexiones lentas.
  - **SSR (Server-Side Rendering)**: El servidor genera el HTML completo para cada solicitud, lo que mejora el tiempo hasta el primer render y es beneficioso para SEO. Sin embargo, requiere más recursos del servidor y puede ser más complejo de implementar.
  - **SSG (Static Site Generation)**: Similar a SSR, pero el HTML se genera en tiempo de construcción (build time) en lugar de en cada solicitud. Ideal para sitios con contenido estático o que no cambia frecuentemente.
    Nuxt.js soporta los tres modos, permitiendo elegir el más adecuado según las necesidades del proyecto.

- ¿Cómo se realiza el testing de componentes en Vue.js?

  Respuesta: El stack recomendado en proyectos Vue modernos (con Vite) es **Vitest + Vue Test Utils + @testing-library/vue**.

  **¿Por qué Vitest y no Jest?**
  - Vitest usa la misma pipeline de transformación que Vite: entiende nativo los `.vue`, TypeScript, JSX y aliases de paths **sin configuración extra**.
  - Jest requiere `babel-jest` o `ts-jest` + transformers para `.vue` — más configuración, más puntos de falla.
  - La API de Vitest es compatible con Jest (`describe`, `it`, `expect`, `vi.fn()`), por lo que la curva de migración es mínima.
  - Vitest tiene modo watch ultra-rápido gracias al HMR de Vite y soporte nativo de ESM.

  Los dos enfoques de testing de componentes:
  - **Vue Test Utils (VTU)**: la librería oficial. Permite montar componentes (`mount`, `shallowMount`), simular eventos, inspeccionar el DOM renderizado y hacer assertions sobre el estado del componente.
  - **Testing Library para Vue** (`@testing-library/vue`): construida sobre VTU pero con una filosofía diferente — testea **comportamiento del usuario**, no detalles de implementación. Es la opción preferida para tests robustos.

  ```javascript
  // vitest.config.ts
  import { defineConfig } from "vitest/config";
  import vue from "@vitejs/plugin-vue";

  export default defineConfig({
    plugins: [vue()],
    test: {
      environment: "jsdom", // o 'happy-dom' (más rápido)
      globals: true, // describe/it/expect sin imports
    },
  });
  ```

  ```javascript
  // Con @testing-library/vue — filosofía: "test lo que el usuario ve/hace"
  import { render, screen, fireEvent } from "@testing-library/vue";
  import Counter from "./Counter.vue";

  test("incrementa el contador al hacer click", async () => {
    render(Counter);
    const btn = screen.getByRole("button", { name: /incrementar/i });
    await fireEvent.click(btn);
    expect(screen.getByText("1")).toBeInTheDocument();
  });
  ```

  Para senior: preferir queries semánticas (`getByRole`, `getByLabelText`) sobre `querySelector` o `wrapper.find('.btn')` — las primeras fallan si el componente deja de ser accesible, convirtiendo el test en un detector de regresión de a11y también. Mockear dependencias externas (APIs, stores) pero **no** mockear sub-componentes propios sin razón — los tests de integración dan más confianza que los unitarios aislados en exceso.

---

# Preguntas Senior

## HTML

- CSS se renderiza antes o despues del DOM? ¿Cómo afecta esto al rendimiento y a la experiencia del usuario?

Respuesta: El CSS se renderiza después de que el navegador ha construido el DOM, pero antes de que se pinte la página en la pantalla. Esto significa que el navegador necesita descargar y procesar el CSS antes de mostrar cualquier contenido al usuario. Si el CSS es bloqueante (por ejemplo, si se incluye en el `<head>` sin `media` o `async`), puede retrasar significativamente el tiempo hasta el primer render (First Contentful Paint), lo que afecta negativamente la experiencia del usuario. Para mejorar el rendimiento, es recomendable optimizar la carga de CSS, utilizando técnicas como la división de CSS, la carga asíncrona o el uso de `media` para estilos específicos.

- HTML es renderizado incrementalmente. Podemos decir lo mismo de CSS? ¿Cómo afecta esto a la experiencia del usuario?

Respuesta: El HTML se renderiza de manera incremental a medida que el navegador lo procesa, lo que permite mostrar contenido al usuario incluso antes de que toda la página haya sido descargada. Sin embargo, el CSS no se renderiza de manera incremental; el navegador necesita descargar y procesar todo el CSS antes de aplicar los estilos a la página. Esto significa que si el CSS es bloqueante, el usuario puede experimentar un retraso significativo antes de ver cualquier contenido estilizado, lo que puede resultar en una experiencia de usuario pobre. Para mejorar esto, es importante optimizar la carga del CSS y evitar incluir estilos innecesarios en el bloque inicial.

- Cual es interpretado luego que el HTML esta complementamente cargado? async o defer script?

Respuesta: El atributo `defer` indica que el script debe ser ejecutado después de que el documento HTML haya sido completamente analizado, pero antes de que se dispare el evento `DOMContentLoaded`. Esto significa que los scripts con `defer` se ejecutan en orden, pero solo después de que el HTML esté completamente cargado. Por otro lado, el atributo `async` indica que el script debe ser ejecutado tan pronto como se haya descargado, sin esperar a que el HTML esté completamente cargado. Esto puede resultar en que los scripts con `async` se ejecuten antes de que el HTML esté completamente cargado, lo que puede causar problemas si el script depende de elementos del DOM que aún no han sido procesados.

- Dime 3 formas de optimizar la carga de CSS en una página web.

Respuesta: Para optimizar la carga de CSS en una página web, se pueden utilizar las siguientes técnicas:

1. **División de CSS**: Dividir el CSS en archivos más pequeños y específicos para cada página o sección de la aplicación, lo que permite cargar solo el CSS necesario para cada vista.
2. **Carga asíncrona**: Utilizar el atributo `media` para cargar estilos específicos solo cuando se necesiten, o cargar el CSS de manera asíncrona utilizando JavaScript para evitar que el CSS bloquee el renderizado inicial.
3. **Minificación y compresión**: Minificar el CSS para reducir su tamaño y utilizar técnicas de compresión como Gzip o Brotli para disminuir el tiempo de descarga.

- Ventajas y desventajas de usar clouseres en JavaScript.

1. Permiten la encapsulación de datos y la creación de variables privadas, lo que mejora la seguridad y la modularidad del código.
   Las desventajas de usar closures incluyen:
2. Pueden llevar a problemas de memoria si no se gestionan adecuadamente, ya que las variables referenciadas por el closure no pueden ser garbage-collected mientras el closure exista.

- ¿Cómo manejarías la gestión de estado en una aplicación Vue grande sin usar Vuex o Pinia?

Respuesta: Para una aplicación Vue grande, es recomendable usar una solución de gestión de estado como Vuex o Pinia debido a su estructura y características diseñadas para manejar estados complejos. Sin embargo, si por alguna razón no se desea usar estas bibliotecas, se pueden considerar las siguientes alternativas:

1. **Provide/Inject**: Esta es una API nativa de Vue que permite compartir datos entre componentes sin necesidad de pasar props a través de múltiples niveles. Es útil para compartir estado entre componentes relacionados, pero no es ideal para manejar estados globales complejos.
2. **Event Bus**: Aunque es un anti-patrón, se puede usar un bus de eventos para comunicar componentes entre sí. Sin embargo, esto puede llevar a un código difícil de mantener y depurar, especialmente en aplicaciones grandes.
3. **Composables**: Con el Composition API de Vue 3, se pueden crear composables para encapsular lógica de estado y compartirla entre componentes. Esto permite una gestión de estado más modular y reutilizable, aunque no proporciona las características avanzadas de Vuex o Pinia, como el soporte para plugins, devtools, o time-travel debugging.
