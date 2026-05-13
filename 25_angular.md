# Angular questions

**Questions Source:** _[angularspace.com](https://www.angularspace.com/senior-angular-interview-questions/)_

## General Questions:

1. What does it mean to be a senior developer ?
**Respuesta:** Ser un desarrollador senior va más allá de solo escribir código funcional. Implica:
- **Arquitectura y Diseño:** Tomar decisiones de alto nivel, comprender la arquitectura del sistema y anticipar la escalabilidad futura.
- **Mentoría:** Guiar a desarrolladores junior y semi-senior, realizar revisiones de código exhaustivas y fomentar una buena cultura de ingeniería.
- **Comprensión del Negocio:** Alinear las decisiones técnicas con los objetivos comerciales y comprender los pros y contras de diferentes enfoques.
- **Calidad y Rendimiento:** Promover las mejores prácticas, estrategias de prueba, CI/CD y optimizaciones de rendimiento.
- **Resolución de Problemas:** Ser capaz de abordar de forma independiente problemas complejos no definidos y asumir la propiedad de las funcionalidades desde la concepción hasta el despliegue.

2. Do you prefer declarative or imperative programming?
**Respuesta:** En el contexto de Angular moderno, se prefiere en gran medida la **programación declarativa**. Se enfoca en describir *cómo* debería verse la interfaz de usuario basada en el estado, en lugar de *cómo* mutarla paso a paso (imperativo).
El código declarativo usando RxJS o Signals es más predecible, más fácil de probar y reduce los efectos secundarios.
```typescript
// Imperativo (Evitar)
ngOnInit() {
  this.userService.getUsers().subscribe(users => {
    this.users = users;
  });
}

// Declarativo (Preferido)
users$ = this.userService.getUsers();
// En el template: @for(user of users$ | async; track user.id) { ... }
```

3. Would you use a state management library or a custom implementation?
**Respuesta:** Depende de la escala y complejidad del proyecto:
- **Implementación Personalizada:** Para aplicaciones pequeñas a medianas, el uso de Angular Signals o `BehaviorSubject` de RxJS dentro de servicios suele ser suficiente. Evita el código repetitivo innecesario (boilerplate) y mantiene la arquitectura simple.
- **Librería de Gestión de Estado (NgRx, Akita, NGXS):** Para aplicaciones empresariales a gran escala con un estado complejo compartido en muchos módulos diferentes, actualizaciones en tiempo real o lógicas complejas de deshacer/rehacer.

4. What are the advantages and disadvantages of using a state management library?
**Respuesta:**
**Ventajas:**
- **Única Fuente de Verdad:** El estado centralizado facilita la comprensión del flujo de datos.
- **Predictibilidad:** Flujo de datos unidireccional estricto (Acciones -> Reducers -> Estado).
- **Herramientas:** Excelentes herramientas de depuración como Redux DevTools (depuración de viaje en el tiempo).
- **Separación de Responsabilidades:** La lógica de negocio está desacoplada de los componentes de la interfaz de usuario.

**Desventajas:**
- **Boilerplate (Código repetitivo):** Requiere escribir acciones, reducers, selectores y efectos, lo que puede ralentizar el desarrollo para funcionalidades simples.
- **Curva de Aprendizaje:** Alta barrera de entrada para desarrolladores junior.
- **Excesivo (Overkill):** Puede complicar en exceso aplicaciones simples donde un servicio básico sería suficiente.

## Angular Questions - General:

5. How would you achieve a parent - child component communication ?
**Respuesta:**
- **Padre a Hijo:** Usando decoradores `@Input()` (o el nuevo `input()` de Signal).
- **Hijo a Padre:** Usando decoradores `@Output()` con `EventEmitter` (o `output()` de Signal).
- **Padre accediendo a Hijo:** Usando `@ViewChild()` o `@ViewChildren()` para acceder a la API pública del hijo.
- **Servicio Compartido:** Para componentes profundamente anidados o que no tienen una relación directa, utilizando un servicio compartido con RxJS o Signals.
```typescript
// Componente Hijo
export class ChildComponent {
  // Input con Signal (Angular 17.1+)
  data = input<string>();
  // Output con Signal
  dataChanged = output<string>();

  notify() {
    this.dataChanged.emit('Nuevos Datos');
  }
}
```

6. What is the role of **NgZone** in Angular, and when would you opt out of Angular's change detection?
**Respuesta:** `NgZone` es el envoltorio (wrapper) de Angular sobre `zone.js`. Rastrea las operaciones asíncronas (como `setTimeout`, solicitudes HTTP y eventos del DOM) para notificar a Angular cuándo ejecutar la Detección de Cambios (Change Detection).
Deberías optar por salir usando `ngZone.runOutsideAngular()` para eventos de alta frecuencia (como `mousemove`, `scroll` o `requestAnimationFrame`) o cálculos pesados que no requieren una actualización inmediata de la interfaz de usuario. Esto evita cuellos de botella en el rendimiento.
```typescript
constructor(private ngZone: NgZone) {}

listenToScroll() {
  this.ngZone.runOutsideAngular(() => {
    window.addEventListener('scroll', () => {
      // Lógica que no necesita actualizar la vista inmediatamente
    });
  });
}
```

7. What is and when to use an Injection Token ?
**Respuesta:** Un `InjectionToken` se usa para crear un token único para la Inyección de Dependencias (DI) cuando el elemento inyectado no es un tipo de clase (por ejemplo, interfaces, objetos de configuración o primitivos). Evita las colisiones de nombres que podrían ocurrir si se usaran tokens de cadena de texto (strings).
```typescript
export interface AppConfig { apiUrl: string; }
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// En los providers:
{ provide: APP_CONFIG, useValue: { apiUrl: 'https://api.com' } }

// En el constructor:
constructor(@Inject(APP_CONFIG) private config: AppConfig) {}
```

8. What are resolution modifiers and how to use them ?
**Respuesta:** Los modificadores de resolución cambian cómo el framework de inyección de dependencias de Angular resuelve una dependencia:
- `@Optional()`: Retorna `null` en lugar de lanzar un error si no se encuentra la dependencia.
- `@Self()`: Restringe la búsqueda al inyector del propio componente.
- `@SkipSelf()`: Inicia la búsqueda en el inyector del componente padre, omitiendo el inyector del componente actual.
- `@Host()`: Detiene la búsqueda en el inyector del componente anfitrión (útil para directivas).
```typescript
constructor(@Optional() @SkipSelf() private parentService: ParentService) {}
```

9. Why would you use a track function in a for-loop and how it works ?
**Respuesta:** La función `track` (o `trackBy` en versiones anteriores) ayuda a Angular a identificar elementos en un iterable. Por defecto, Angular rastrea los objetos por su identidad (referencia). Si se reemplaza un array (ej., desde una llamada a una API), Angular elimina y recrea todos los elementos del DOM, lo cual es costoso.
Al usar `track` con un identificador único, Angular solo actualiza, agrega o elimina los nodos exactos del DOM que cambiaron.
```html
<!-- Control Flow de Angular 17+ -->
@for (user of users; track user.id) {
  <li>{{ user.name }}</li>
}
```

10. What is the the difference between providers and viewProviders ? Can you provide an example when to use either of them ?
**Respuesta:** 
- `providers`: El servicio provisto está disponible para el propio componente, sus componentes hijos de la vista (view children), Y el contenido proyectado en él a través de `<ng-content>`.
- `viewProviders`: El servicio provisto está disponible para el componente y sus componentes hijos de la vista, pero **NO** para el contenido proyectado.
**Ejemplo:** Si creas un componente genérico de pestañas altamente encapsulado y provees un `TabStateService` interno, deberías usar `viewProviders` para que los componentes proyectados dentro de las pestañas no inyecten ni manipulen accidentalmente el estado interno de la pestaña.

11. Why pipes are considered safe in a template, but regular function calls (not signals) are not ?
**Respuesta:** Las llamadas a funciones regulares en un template se ejecutan durante **cada ciclo de detección de cambios**, incluso si sus entradas (inputs) no han cambiado. Si la función contiene una lógica pesada, degradará el rendimiento.
Los pipes, por otro lado, son **puros por defecto**. Angular almacena en caché su salida y solo vuelve a ejecutar el método `transform` del pipe cuando cambian los argumentos de entrada. Esto los hace altamente optimizados y seguros para su uso en templates.

## Angular Questions - Signals:

12. How would you convince your team to migrate a project from Observables to signals ?
**Respuesta:** Me centraría en los beneficios tangibles:
- **Simplicidad:** Los Signals eliminan la necesidad de gestionar suscripciones (`subscribe`/`unsubscribe`) y evitan fugas de memoria. El modelo mental es síncrono y más fácil de comprender para los nuevos desarrolladores.
- **Reactividad Granular:** Angular sabe exactamente qué signals cambiaron y puede actualizar solo las partes de la UI dependientes sin verificar todo el árbol de componentes (futuro sin `zone.js`).
- **Sin Errores Temporales (Glitches):** Los Signals garantizan una ejecución sin errores temporales (no hay estados inválidos intermedios).
- **Interoperabilidad con RxJS:** No tenemos que reescribir todo a la vez. Podemos usar `toSignal` y `toObservable` para migrar de forma incremental.

13. Can you explain the diamond problem in Observables and why it doesn't occur with signals?
**Respuesta:** El problema del diamante ocurre cuando dos valores calculados dependen de una única fuente, y un tercer valor depende de ambos valores calculados. En RxJS, si la fuente cambia, el tercer valor podría evaluarse dos veces (un "glitch" o estado inválido intermedio) porque recibe la actualización a través de dos caminos diferentes en momentos ligeramente distintos.
**Los Signals resuelven esto** utilizando un mecanismo híbrido de pull/push. Cuando un signal cambia, marca a sus dependientes como "sucios" (push) pero no los recalcula inmediatamente. El recálculo real solo ocurre cuando se lee el valor (pull), asegurando que para el momento en que se lee, todas las dependencias estén completamente resueltas.

14. When to use effect and untracked in a signal based application ?
**Respuesta:**
- **`effect()`:** Úsalo para efectos secundarios que deberían ejecutarse cuando un signal cambia, como registrar logs, sincronizar datos en `localStorage` o dibujar en un canvas. **Nunca** uses `effect()` para propagar cambios de estado (usa `computed()` en su lugar).
- **`untracked()`:** Úsalo dentro de un `effect` o `computed` cuando quieras leer el valor de un signal *sin* crear una dependencia reactiva. Esto evita que el efecto se vuelva a ejecutar cuando ese signal en específico cambia.
```typescript
effect(() => {
  const user = this.currentUser();
  // Queremos registrar el cambio de usuario, pero no queremos que este efecto 
  // se vuelva a ejecutar si solo cambia el ID de rastreo de analíticas.
  const trackingId = untracked(this.trackingId);
  this.analytics.log(`Usuario ${user.name} cambió`, trackingId);
});
```

15. Are life-cycle hooks still needed in a fully signal based application ?
**Respuesta:** Aunque los Signals manejan perfectamente la reactividad y la derivación del estado (`computed`), los hooks del ciclo de vida siguen siendo necesarios para eventos específicos del framework:
- **Inicialización del DOM:** `ngAfterViewInit` sigue siendo necesario si debes interactuar con elementos del DOM mediante `@ViewChild` (aunque las queries basadas en Signals ayudan a mitigar esto).
- **Destrucción del Componente:** `ngOnDestroy` podría seguir siendo necesario para limpiar recursos que no son de Angular (como WebSockets o event listeners de JS puro), aunque ahora a menudo se prefiere el uso de `DestroyRef`.
- Sin embargo, hooks como `ngOnChanges` o `ngDoCheck` se vuelven en gran medida obsoletos gracias a `computed()` y `effect()`.

## Angular Questions - RxJS:

16. What are the differences between Observables and Promises, and when would you choose one over the other?
**Respuesta:**
- **Promesas:** Manejan un único valor futuro. Son impacientes (eager, comienzan a ejecutarse inmediatamente) y no se pueden cancelar.
- **Observables:** Manejan flujos de múltiples valores a lo largo del tiempo. Son perezosos (lazy, no se ejecutan hasta que se llama a `subscribe`), se pueden cancelar (`unsubscribe`) y ofrecen un rico conjunto de operadores (`map`, `filter`, `switchMap`).
**Cuándo elegir:** Usa Promesas para operaciones asíncronas simples y de una sola vez (como `fetch`). Usa Observables para eventos recurrentes (websockets, eventos del DOM), flujos de trabajo asíncronos complejos que requieran cancelación (autocompletado de búsqueda) o para la gestión reactiva del estado.

17. Can you explain the concept of "cold" and "hot" Observables in RxJS, and provide examples of each?
**Respuesta:**
- **Observables Fríos (Cold):** El productor de datos se crea *dentro* del observable. Cada suscriptor obtiene su propia ejecución independiente y recibe los valores desde el principio.
  *Ejemplo:* `http.get('/api/data')`. Si 3 componentes se suscriben, se realizan 3 solicitudes HTTP.
- **Observables Calientes (Hot):** El productor de datos se crea *fuera* del observable. Todos los suscriptores comparten la misma ejecución y reciben los valores desde el momento en que se suscriben (podrían perderse valores anteriores).
  *Ejemplo:* `fromEvent(document, 'click')` o un `Subject`.

18. How do you handle error handling in RxJS, and what are some common strategies for managing errors in a reactive programming context?
**Respuesta:**
- **`catchError`:** Intercepta un error y devuelve un nuevo observable (datos de respaldo) o vuelve a lanzar el error.
- **`retry` / `retryWhen`:** Se suscribe automáticamente de nuevo al observable de origen al ocurrir un error.
- **`EMPTY`:** Absorbe silenciosamente el error y completa el flujo.
```typescript
this.http.get('/api/data').pipe(
  retry(2), // Intenta 2 veces más antes de fallar
  catchError(error => {
    this.logger.log(error);
    return of([]); // Devuelve datos de respaldo
  })
).subscribe();
```

19. What are some common operators in RxJS, and how do they work? Can you provide examples of how to use them in a real-world application?
**Respuesta:**
- **`map`:** Transforma los valores emitidos (ej., extrayendo una propiedad anidada de la respuesta de una API).
- **`filter`:** Solo permite valores que pasen una condición (ej., procesar solo consultas de búsqueda > 3 caracteres).
- **`tap`:** Realiza efectos secundarios sin alterar el flujo (ej., `console.log` para depuración o establecer un spinner de carga en falso).
- **`switchMap`:** Cancela el observable interno anterior y se suscribe a uno nuevo. (ej., Búsqueda con autocompletado: si el usuario escribe rápidamente, cancela la solicitud HTTP anterior y solo resuelve la más reciente).
- **`combineLatest`:** Emite cuando cualquiera de los observables de origen emite, proporcionando los valores más recientes de todos ellos (ej., combinar criterios de filtrado de múltiples menús desplegables de la UI para desencadenar la obtención de datos).

20. How do you manage subscriptions in RxJS to prevent memory leaks, and what are some best practices for cleaning up subscriptions in Angular components?
**Respuesta:**
- **El pipe `async`:** La mejor práctica. Angular se suscribe automáticamente en el template y se desuscribe cuando el componente es destruido.
- **`takeUntilDestroyed` (Angular 16+):** Un operador que se desuscribe automáticamente basándose en la `DestroyRef` del componente.
- **`takeUntil(subject)`:** Usar un `Subject` que emite en `ngOnDestroy`.
- **`unsubscribe()` manual:** Mantener una referencia a una `Subscription` y llamar a `.unsubscribe()` en `ngOnDestroy`.

21. What is a higher-order observable and how they differ ?
**Respuesta:** Un observable de orden superior es un observable que emite *otros observables* (en lugar de valores simples como strings o números). Para aplanarlos de nuevo en un único flujo de valores, usamos operadores de aplanamiento (flattening):
- **`switchMap`:** Cancela el observable interno anterior y cambia al nuevo (Bueno para solicitudes HTTP GET).
- **`concatMap`:** Pone en cola los observables internos, ejecutándolos en orden uno por uno (Bueno para HTTP POST/PUT donde el orden importa).
- **`mergeMap`:** Ejecuta todos los observables internos de manera concurrente (Bueno para solicitudes paralelas independientes).
- **`exhaustMap`:** Ignora nuevos observables internos mientras el actual todavía se está ejecutando (Bueno para botones de login para evitar dobles clics).

22. What is the difference between share() and shareReplay() ?
**Respuesta:** Ambos se usan para convertir un observable frío en uno caliente (multidifusión), evitando ejecuciones múltiples (ej., evitando solicitudes HTTP duplicadas).
- **`share()`:** Los suscriptores solo obtienen los valores emitidos *después* de suscribirse. Los suscriptores tardíos no obtienen nada hasta la siguiente emisión.
- **`shareReplay(N)`:** Almacena en caché los últimos `N` valores emitidos. Los suscriptores tardíos reciben inmediatamente los valores en caché tras la suscripción. Esto se usa mucho para almacenar en caché respuestas HTTP en servicios.

23. What does this code do ? - scan() + expand()
**Respuesta:**
- **`scan(accumulator, seed)`:** Funciona exactamente como `Array.prototype.reduce`, pero emite el estado acumulado después de *cada* emisión de la fuente (útil para la gestión de estado tipo Redux en RxJS).
- **`expand(project)`:** Proyecta recursivamente cada valor de origen a un Observable que se fusiona en el Observable de salida. A menudo se usa para **paginación**. Toma un valor, lo emite y lo retroalimenta en la función para obtener el siguiente observable hasta que devuelve `EMPTY`.


26. What are some common performance pitfalls in Angular applications, and how do you avoid them?
**Respuesta:**
- **Llamadas a funciones en templates:** Evita `<div *ngIf="isUserValid()">`. Usa pipes puros o precalcula el valor en la clase del componente (o usa un `computed` de Signal).
- **Fugas de Memoria:** Olvidar desuscribirse de Observables de larga duración. Usa el pipe `async` o `takeUntilDestroyed()`.
- **Bundles grandes:** No usar carga perezosa, importar librerías completas (como `lodash` en lugar de `lodash-es`), o no usar la configuración de compilación `production`.
- **Falta de `track` en bucles:** Re-renderizar listas enteras cuando solo cambia un elemento.

27. How do you use lazy loading in Angular, and what are the benefits of using it to improve performance?
**Respuesta:** La carga perezosa (lazy loading) pospone la inicialización del código hasta que realmente se necesita, reduciendo significativamente el tamaño del bundle inicial y el tiempo de carga.
Con los componentes standalone, usamos `loadComponent` o `loadChildren` en el router:
```typescript
const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent)
    // O cargar un conjunto de rutas hijas:
    // loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
  }
];
```
Nuevo en Angular 17 son las **Vistas Diferibles** (`@defer`), que permiten la carga perezosa de componentes *dentro de un template* basándose en desencadenadores como la visibilidad en la pantalla o la interacción.

28. Can you explain the concept of Ahead-of-Time (AOT) compilation in Angular, and how it differs from Just-in-Time (JIT) compilation? What are the advantages of using AOT compilation for performance optimization in Angular applications?
**Respuesta:**
- **JIT (Just-in-Time):** El compilador de Angular se envía al navegador junto con el código de la aplicación. Los templates se compilan en el navegador en tiempo de ejecución. Arranque más lento, tamaño de bundle más grande.
- **AOT (Ahead-of-Time):** El compilador se ejecuta una vez durante el proceso de construcción en el servidor. El navegador descarga código ejecutable precompilado y optimizado.
**Ventajas de AOT:** Renderizado más rápido (el navegador no compila), tamaño de bundle más pequeño (no se envía el compilador de Angular) y detección temprana de errores en templates durante el paso de construcción. Es el predeterminado para construcciones de producción.

29. How do you use the Angular CLI to optimize the build process and improve performance in an Angular application?
**Respuesta:** Usar el comando `ng build --configuration production` habilita:
- **Compilación AOT**
- **Minificación y Ofuscación** (Terser)
- **Tree-shaking:** Eliminación de código no utilizado (siempre que se use `providedIn: 'root'`).
- **Eliminación de código muerto.**
- **Presupuestos (Budgeting):** Falla la construcción si los tamaños de los bundles superan los límites definidos en `angular.json`.

30. What are some best practices for optimizing the performance of an Angular application, and how do you implement them in your projects?
**Respuesta:** Además de OnPush, Lazy Loading y Signals (mencionados anteriormente):
- **Web Workers:** Mueve cálculos pesados vinculados a la CPU (como parsear archivos CSV enormes) fuera del hilo principal para evitar que la interfaz de usuario se congele.
- **Scroll Virtual:** Usa `@angular/cdk/scrolling` para renderizar listas masivas. Solo renderiza los nodos del DOM que actualmente son visibles en la pantalla.
- **Estrategias de Precarga:** Usa `PreloadAllModules` o una estrategia de precarga personalizada para cargar módulos perezosos en segundo plano después de que se haya cargado el cascarón (app shell) inicial de la aplicación.

## Angular Questions - Testing:

31. How do you approach testing in Angular, and what are some common strategies for writing effective tests?
**Respuesta:** 
- **Pruebas Unitarias:** Céntrate en la lógica de negocio aislada en servicios y pipes.
- **Pruebas de Componente (Superficiales/Integración):** Prueba las interacciones del DOM del componente, sus entradas (inputs) y salidas (outputs), simulando (mocking) las dependencias pesadas.
- **Pruebas E2E:** Prueba flujos críticos de usuario en un entorno de navegador real.
- **Estrategia:** Sigue el "Trofeo de Pruebas" (muchas pruebas de integración, una cantidad decente de pruebas unitarias, menos pruebas E2E). Usa el patrón AAA (Arrange, Act, Assert - Preparar, Actuar, Afirmar).

32. Can you explain the difference between unit testing, integration testing, and end-to-end testing in the context of Angular applications, and when would you use each type of testing?
**Respuesta:**
- **Pruebas Unitarias:** Prueba una clase aislada (Servicio, Pipe, función pura) sin sus dependencias o el DOM. *Cuándo:* Algoritmos complejos, transformaciones de datos.
- **Pruebas de Integración:** Prueba cómo los componentes interactúan con sus templates y servicios inyectados (generalmente usando `TestBed`). *Cuándo:* Vinculaciones de componentes, eventos del DOM, conexiones de entrada/salida.
- **Pruebas End-to-End (E2E):** Prueba la aplicación completa ejecutándose en un navegador contra un backend real o simulado (Cypress, Playwright). *Cuándo:* Flujos de trabajo comerciales críticos como el pago o el inicio de sesión.

33. How do you use the Angular TestBed to set up and configure tests for Angular components and services, and what are some best practices for using TestBed effectively?
**Respuesta:** `TestBed` crea un módulo dinámico de Angular específicamente para pruebas.
```typescript
beforeEach(async () => {
  await TestBed.configureTestingModule({
    imports: [MyComponent], // Componente Standalone
    providers: [
      { provide: DataService, useValue: mockDataService }
    ]
  }).compileComponents();
  
  fixture = TestBed.createComponent(MyComponent);
  component = fixture.componentInstance;
  fixture.detectChanges();
});
```
**Mejores Prácticas:** Siempre simula (mock) las dependencias externas (HTTP, Router) para asegurar aislamiento y velocidad. Usa `NO_ERRORS_SCHEMA` con cuidado, ya que podría ocultar errores legítimos en el template.

34. What are some common testing frameworks and libraries used in Angular testing, and how do you choose the right ones for your projects?
**Respuesta:**
- **Karma/Jasmine:** El valor predeterminado tradicional. Bueno, pero Karma está siendo deprecado.
- **Jest:** Reemplazo muy popular para pruebas unitarias/integración. Más rápido, sin interfaz gráfica (headless), excelentes capacidades de simulación.
- **Cypress / Playwright:** Los estándares modernos para pruebas E2E y de componentes. Playwright es cada vez más favorecido por su soporte multi-navegador y velocidad.
- **Angular Testing Library:** Un excelente contenedor (wrapper) alrededor de TestBed que fomenta las pruebas desde la perspectiva del usuario (encontrando elementos por texto/rol) en lugar de probar detalles de implementación.

35. How do you handle asynchronous testing in Angular, and what are some strategies for testing asynchronous code effectively?
**Respuesta:**
- **`fakeAsync` & `tick()`:** El enfoque más común. Ejecuta código en una zona de tiempo falsa, permitiéndote avanzar el tiempo manualmente (`tick(1000)`) sin tener que esperar.
- **`waitForAsync`:** Se usa cuando se prueba código que involucra promesas reales o llamadas XHR que no pueden ser simuladas por `fakeAsync`.
- **callback `done`:** La forma estándar en Jasmine/Jest de esperar a que se complete un observable o una promesa.
```typescript
it('debería cargar los datos', fakeAsync(() => {
  component.load();
  tick(); // avanza rápidamente las microtareas (Promesas/Observables)
  expect(component.data).toBeDefined();
}));
```

## Angular Questions - Architecture and Design:

36. How do you approach designing the architecture of an Angular application, and what are some common architectural patterns used in Angular development?
**Respuesta:** Sigo una arquitectura basada en características (features) y orientada al dominio (a menudo basada en espacios de trabajo Nx o características standalone de Angular).
- **Core (Núcleo):** Servicios singleton, guards, interceptores (cargados una vez en `app.config.ts`).
- **Shared (Compartido):** Componentes de interfaz de usuario, pipes y directivas reutilizables (ej., botones, modales).
- **Features (Características):** Áreas cargadas de forma perezosa específicas del dominio (ej., `perfil-usuario`, `pago`).
- **Patrón de Componentes Smart/Dumb (Inteligente/Tonto):** Los componentes "Inteligentes" se comunican con los servicios y gestionan el estado; los componentes "Tontos" (presentacionales) solo reciben `@Input` y emiten `@Output`.

37. Can you explain the concept of modularity in Angular, and how do you structure your Angular applications to promote modularity and maintainability?
**Respuesta:** La modularidad significa dividir la aplicación en bloques de funcionalidad cohesivos. Históricamente, esto se hacía usando `NgModules`. En el Angular moderno (15+), la modularidad se logra a través de **Componentes Standalone** y módulos ES.
Estructura: Mantén la lógica de la característica aislada en su propia carpeta. Una característica no debería importar directamente desde los archivos internos de otra característica; en su lugar, usa un `index.ts` (archivo barril de API pública) para exponer solo lo que sea necesario, imponiendo límites estrictos.

38. How do you manage state in an Angular application, and what are some common state management patterns and libraries used in Angular development?
**Respuesta:**
- **Estado Local:** Gestionado dentro del componente usando Signals o propiedades estándar.
- **Servicio-con-un-Subject:** Un simple `BehaviorSubject` de RxJS o un Signal de Angular en un servicio para compartir el estado entre unos pocos componentes.
- **Estado Global/Complejo:** Usando librerías como **NgRx** (patrón Redux), **NGXS** (patrón CQRS) o **Akita** cuando el estado es muy complejo, requiere depuración de viaje en el tiempo (time-travel), o se comparte ampliamente a través de módulos distantes.

39. What are some best practices for organizing and structuring Angular components, services, and other application code, and how do you implement these practices in your projects?
**Respuesta:**
- **Principio LIFT:** Localizar (Locate) código rápidamente, Identificar (Identify) el código de un vistazo, Mantener la estructura lo más Plana (Flattest) que puedas, Intentar ser DRY (Don't Repeat Yourself - No te repitas).
- **Nomenclatura de Archivos:** `caracteristica.tipo.ts` (ej., `usuario.service.ts`, `login.component.ts`).
- **Responsabilidad Única:** Una clase/componente por archivo.
- **Delegar Lógica:** Mantén los componentes ligeros; mueve la lógica de negocio y la obtención de datos a los Servicios.

40. How do you approach code reuse in Angular, and what are some strategies for creating reusable components, directives, and services in Angular applications?
**Respuesta:**
- **Proyección de Contenido:** Usa `<ng-content>` para crear componentes contenedores flexibles (como Tarjetas o Modales) donde el consumidor proporciona el HTML interno.
- **Directivas:** Extrae la manipulación del DOM compartida o la lógica de manejo de eventos en Directivas de Atributo (ej., un `ClickOutsideDirective`).
- **Composición:** Construye componentes "tontos" pequeños y enfocados y compónlos para construir una interfaz de usuario compleja.
- **Pipes:** Reutiliza la lógica de transformación de datos.

41. Can you explain the concept of dependency injection in Angular, and how does it work? How do you use dependency injection to manage dependencies and promote loose coupling in Angular applications?
**Respuesta:** La Inyección de Dependencias (DI) es un patrón de diseño donde una clase recibe sus dependencias de una fuente externa en lugar de crearlas ella misma.
Angular tiene un sistema jerárquico de DI. Cuando un componente solicita una dependencia en su constructor (o vía `inject()`), Angular busca en el inyector del componente. Si no la encuentra, sube por el árbol de componentes, y finalmente verifica el inyector del entorno raíz.
Promueve el bajo acoplamiento porque un componente depende de una abstracción (o token), lo que hace extremadamente fácil intercambiar implementaciones (ej., intercambiar `RealHttpService` con `MockHttpService` en las pruebas).

42. How do you handle cross-cutting concerns in Angular applications, such as logging, error handling, and authentication, and what are some strategies for implementing these concerns in a modular and maintainable way?
**Respuesta:**
- **Interceptores HTTP:** Para adjuntar tokens de autenticación, atrapar globalmente errores HTTP y registrar solicitudes/respuestas.
- **Manejador de Errores Global:** Implementar la interfaz `ErrorHandler` de Angular para atrapar excepciones del frontend no manejadas y registrarlas en un servicio externo (como Sentry).
- **Guards y Resolvers de Ruta:** Para manejar la autenticación y las comprobaciones de autorización antes de que un usuario ingrese a un área específica.

43. What are some best practices for designing and implementing APIs in Angular applications, and how do you ensure that your APIs are well-designed, maintainable, and scalable?
**Respuesta:**
- **Variables de Entorno:** Almacena las URLs base de la API en `environment.ts` (o cárgalas dinámicamente en tiempo de ejecución a través de un servicio de configuración).
- **Tipado Fuerte:** Define interfaces/tipos estrictos en TypeScript para todas las solicitudes y respuestas de la API.
- **Patrón de Repositorio:** Crea servicios de API dedicados (`UserService`, `ProductService`) que solo manejen llamadas HTTP, y sepáralos de los servicios de Estado.
- **Usar Interceptores:** Evita codificar de forma rígida los encabezados o el manejo de errores en cada llamada HTTP.

44. How do you approach performance optimization in Angular applications, and what are some strategies for improving the performance of your Angular applications through architectural and design choices?
**Respuesta:** (Consultar P24-30). A nivel de arquitectura, esto significa usar por defecto componentes Standalone, carga perezosa estricta por ruta, utilizar Vistas Diferibles (`@defer`), y diseñar la gestión del estado de tal manera que podamos confiar en gran medida en `ChangeDetectionStrategy.OnPush` y Signals.

45. Can you explain the concept of micro frontends, and how do you implement micro frontend architecture in Angular applications? What are the benefits and challenges of using micro frontends in Angular development?
**Respuesta:** Los micro frontends implican dividir un frontend monolítico grande en aplicaciones más pequeñas e independientes que pueden ser desarrolladas, probadas y desplegadas por equipos separados, para luego unirse en tiempo de ejecución.
**Implementación:** Usualmente se logra usando **Webpack Module Federation** (ej., `@angular-architects/module-federation`).
**Beneficios:** Despliegues independientes, autonomía de los equipos, actualizaciones incrementales (podrías tener un shell en Angular 15 y un micro frontend en Angular 17).
**Desafíos:** Pipelines de construcción/despliegue complejos, la gestión del estado compartido entre aplicaciones es difícil, posibles conflictos de estilos, y tiempos de carga iniciales más largos si las dependencias no se comparten adecuadamente.

## Angular Questions - Miscellaneous:

46. How do you stay up-to-date with the latest developments and best practices in Angular development, and what are some resources you use to continue learning and improving your skills as an Angular developer?
**Respuesta:** Sigo el blog oficial de Angular, los lanzamientos en GitHub y las notas de la versión. Participo en la comunidad a través de Twitter/X (siguiendo a miembros del equipo como Minko Gechev), leo artículos en Angular in Depth, veo charlas de ng-conf en YouTube y experimento con versiones RC (Release Candidate) en proyectos personales.

47. Can you share an example of a challenging problem you faced in an Angular project, and how you approached solving it? What was the outcome, and what did you learn from the experience?
**Respuesta:** *(Esta es una pregunta de comportamiento personal. Ejemplo de respuesta:)* "Teníamos fugas de memoria masivas en un dashboard en tiempo real. Usé el perfilador de memoria de Chrome DevTools para identificar nodos DOM desconectados. Descubrí que múltiples suscripciones RxJS a un WebSocket no se estaban limpiando. Refactoricé el componente para usar el pipe `async` y `takeUntilDestroyed()`. El resultado fue una huella de memoria estable. Aprendí la importancia crítica de la gestión declarativa de las suscripciones."

48. How do you approach debugging and troubleshooting in Angular applications, and what are some common tools and techniques you use to identify and resolve issues in Angular applications?
**Respuesta:**
- **Angular DevTools (Extensión de Chrome):** Para inspeccionar el árbol de componentes, el estado y perfilar los ciclos de detección de cambios.
- **Redux DevTools:** Si se usa NgRx/NGXS.
- **Pestaña Network (Red) del Navegador:** Para fallos de la API.
- **Operador `tap()`:** Para inspeccionar flujos de RxJS sin mutarlos.
- **Debugger:** Colocar `debugger;` en el código para recorrer la lógica paso a paso en el navegador.

49. Can you explain the concept of Angular Universal, and how do you implement server-side rendering in Angular applications? What are the benefits and challenges of using Angular Universal for server-side rendering in Angular development?
**Respuesta:** Angular Universal (ahora integrado como `@angular/ssr`) renderiza aplicaciones Angular en el servidor (Node.js) en HTML estático antes de enviarlas al navegador.
**Beneficios:** Mejora enormemente el SEO (los motores de búsqueda pueden rastrear el HTML inmediatamente), mejora el First Contentful Paint (FCP) en dispositivos lentos y proporciona vistas previas de enlaces ricas para las redes sociales.
**Desafíos:** No puedes usar objetos específicos del navegador (`window`, `document`) directamente en los componentes sin verificar `isPlatformBrowser`. Aumenta los costos del servidor y la complejidad de despliegue en comparación con el alojamiento estático.

50. How do you approach accessibility in Angular applications, and what are some best practices for designing and implementing accessible Angular applications? What are some common accessibility issues in Angular applications, and how do you address them to ensure that your applications are usable by all users, including those with disabilities?
**Respuesta:**
- **HTML Semántico:** Usar elementos nativos (`<button>`, `<nav>`, `<main>`) en lugar de `<div>` con manejadores de clic.
- **Atributos ARIA:** Usar `aria-label`, `aria-hidden`, etc., para proporcionar contexto a los lectores de pantalla.
- **Navegación por Teclado:** Asegurar que todos los elementos interactivos puedan ser enfocados y activados mediante el teclado (`tabindex`).
- **Angular CDK:** Usar `@angular/cdk/a11y` (FocusMonitor, LiveAnnouncer).
- **Problema común:** Modales que no atrapan el foco. Soluciono esto usando la directiva `cdkTrapFocus` del CDK para asegurar que los usuarios de teclado no puedan tabular fuera de un modal abierto.
