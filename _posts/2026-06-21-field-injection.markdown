---
layout: post
title:  "¿Por qué la inyección por campos (@Autowired) se desaconseja en Spring?"
date:   2026-06-21 18:37:58 +0200
categories: jekyll update
---

El uso de la anotación ```@Autowired``` es una práctica muy extendida en los repositorios de Spring Boot. Como ingeniero de Software que acaba de comenzar en este framework pensaba que era una buena práctica, pero hace un par de semanas en una conversación con un ingeniero con mucha más experiencia que yo me dijo que no solo no era una buena práctica sino que era un antipatrón. 

El objetivo de esta entrada es analizar cómo funciona la inyección de campos en Spring (actualmente la anotación ```@Autowired```) y por qué Spring no recomienda utilizarla. Esta entrada está completamente centrada en Spring, pero en esta primera sección trataremos lo que es la inyección de dependencias, usaremos Java como lenguaje vehicular pero es un concepto que no está ligado a ningún lenguaje o framework.

## El patrón de inyección de dependencias

Esta sección tampoco pretende ser una guía completa de la inyección de dependencias, puesto que hay múltiples artículos donde se trata con mucho más detalle, pero sí que pretende dar una idea conceptual.

La idea detrás de este patrón de diseño es muy sencilla: si una clase necesita utilizar otro objeto para realizar su trabajo, no tiene por qué ser ella misma quien lo construya. Basta con que lo reciba ya creado y se limite a utilizarlo.

Al evitar que una clase instancie sus propios atributos/dependencias (usando el operador ```new``` o cualquier sintaxis de construcción según el lenguaje), eliminamos el acoplamiento fuerte. Esto hace que el código sea modular, fácil de mantener y, sobre todo, testable, ya que nos permite sustituir las dependencias reales por componentes de prueba (mocks) fácilmente.

Este cambio de paradigma se conoce como Inversión de Control (IoC): el componente ya no controla la creación de sus herramientas, sino que delega esa responsabilidad en un agente externo. Veámoslo con un ejemplo muy sencillo.

Si tenemos un coche que utiliza un motor de gasolina, si no utilizamos inyección de dependencias el código en Java podría tener esta forma:

```java
public class Car {

  private DieselEngine engine; 

    public Car() {
        this.engine = new DieselEngine(); 
    }

    public void drive() {
        this.engine.start();
        System.out.println("Let's go...");
    }
}
```

Este código tiene numerosos problemas de diseño. Por ejemplo, ¿qué pasaría si queremos generar objetos de la clase ```Car``` con otro tipo de motor que no sea diésel? No podríamos hacerlo directamente. Por otro lado, ¿qué más le da al coche qué motor se use? Mientras el motor que tenga arranque, le da igual el tipo de motor que utilice. Si aplicamos el IoC nuestro código tendría esta forma:

```java
public interface Engine {
    void start();
}

public class Car {

    private final Engine engine; 

    public Car(Engine engine) { 
        this.engine = engine; 
    }

    public void drive() {
        this.engine.start();
        System.out.println("Let's go...");
    }
}
```

Como se puede observar ```Car``` ya no se encarga de crear el motor, simplemente lo recibe y lo utiliza. Además, ya no tenemos el acoplamiento fuerte con respecto al motor, la clase va a funcionar igual sin importar si el ```Engine``` concreto es de diésel, eléctrico o funciona con gasolina.

Con esta idea en mente, vamos a ver cómo Spring utiliza esta idea a través de la inyección de campos.


##  La anotación @Autowired en Spring

La anotación ```@Autowired``` sirve para conseguir hacer un tipo de inyección de dependencias que se denomina inyección de campos. Se introdujo en la versión de Spring Framework 2.5 en noviembre de 2007. Con esta anotación podemos conseguir que una clase reciba las instancias que necesite utilizar sin siquiera declararlo en el constructor. Siguiendo el ejemplo anterior si quisiéramos hacer inyección de campos con el ```Engine``` tendríamos lo siguiente:

```java
public class Car {

  @Autowired
  private Engine engine; 

  public void drive() {
    this.engine.start();
    System.out.println("Let's go...");
  }
}
```
¿Y cómo sabe Spring qué motor concreto inyectar?
Para eso se usan las anotaciones ```@Primary``` y ```@Qualifier(name)```. En caso de que solo haya una implementación no es necesario usar ninguna, si hay más de una utilizará la implementación marcada con ```@Primary``` y si queremos usar una en concreto tendremos que indicar el @Qualifier que Spring debe inyectar.

Antes de esta anotación esto se hacía a través de XML. Como curiosidad histórica, un ejemplo de repositorio donde se puede observar la inyección de dependencias por XML (tanto por campos como por constructor) es el siguiente : [https://github.com/lucabixio/dependency-injection-xml](https://github.com/lucabixio/dependency-injection-xml).


## ¿Cómo funciona ```@Autowired``` por debajo?

Cuando ponemos que un campo es ```@Autowired```, parece casi mágico cómo Spring añade la dependencia. Pero es conveniente que veamos exactamente cómo funciona para poder entender bien por qué Spring Framework desaconseja su uso. Supongamos que tenemos la siguiente clase:

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

}
```
Lo que hace Spring es primero generar el UserService y después añadir ```UserRepository```. Se podría decir con muchas comillas que hace algo parecido a esto (no hace esto por debajo, este código es solo para facilitar entender los pasos):

```java
UserService service = new UserService();

Field field = UserService.class.getDeclaredField("userRepository");
field.setAccessible(true);

UserRepository repo =
    applicationContext.getBean(UserRepository.class);

field.set(service, repo);
```
En primer lugar lo que hace Spring es instanciar el ```UserService```, dejando el atributo userRepository vacío (null). Después de pasar por este estado donde la clase tiene sus atributos en null, comienza la inyección.

Para la anotación de ```@Autowired```, la clase que nos interesa se puede encontrar en : [AutowiredAnnotationBeanPostProcessor](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java). En esta clase es donde verdaderamente ocurre la magia (o, mejor dicho, la reflexión) en la fase posterior a la instanciación. Se encarga de escanear la clase buscando metadatos, específicamente anotaciones como ```@Autowired```, ```@Value``` o ```@Inject```. Todo ello en el método [PostProcessProperties()](https://github.com/spring-projects/spring-framework/blob/8a2e4a9e0aff3636a9bf8e55a7c15c4e72a1c155/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java#L490):

Al identificar el campo, busca en el ```ApplicationContext``` qué bean encaja con el tipo requerido (en nuestro caso, ```UserRepository```).

Después, utiliza la API de Reflection de Java, la cual permite a un programa inspeccionar, modificar y consultar la estructura interna de sus propias clases, campos y métodos en tiempo de ejecución. En este caso se utiliza para romper el encapsulamiento e inyectar la instancia que encontró. Esto lo podemos ver en el método  ```inject(bean, beanName, pvs)``` de la clase ```AutowiredAnnotationBeanPostProcessor```, que hace el proceso que acabamos de discutir:

```java
@Override
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
  Field field = (Field) this.member;
  Object value;
  if (this.cached) {
    try {
      value = resolveCachedArgument(beanName, this.cachedFieldValue);
    }
    catch (BeansException ex) {
      // Unexpected target bean mismatch for cached argument -> re-resolve
      this.cached = false;
      logger.debug("Failed to resolve cached argument", ex);
      value = resolveFieldValue(field, bean, beanName);
    }
  }
  else {
    value = resolveFieldValue(field, bean, beanName);
  }
  if (value != null) {
    ReflectionUtils.makeAccessible(field);
    field.set(bean, value);
  }
}
```


## Desventajas de la inyección por campos

Una vez explicado cómo funciona el ```@Autowired```, veamos por qué Spring desaconseja su uso. Para ello he decidido crear un repositorio muy sencillo para ejemplificarlo.

En este repositorio he definido una interfaz de Servicio (```ServiceI```) y dos servicios que lo implementan (```ServiceA``` y ```ServiceB```). El único método que tienen definido es ``` public String showSomething();```, que simplemente devuelve una frase que identifica cada servicio. Y he hecho dos controladores, uno con inyección por campos y otro por constructor para poder hacer una comparación. La estructura es simplemente:

![Diagrama UML arquitectura repositorio de Ejemplo](/assets/images/spring-boot/field-injection/RepositoryUML.png)


Y el Controlador en el que nos centraremos es:
```java
@RestController
@RequestMapping("/api/autowired")
public class AutowiredController {

    @Autowired
    private ServiceI service;

    @Autowired
    private ApplicationContext context;

    @GetMapping("/activate")
    public ResponseEntity<String> activate() {
        return new ResponseEntity<>(service.showSomething(), HttpStatus.OK);
    }
}
```
### Dificultad detectando violación del Single Responsibility Principle (SRP)

Uno de los argumentos que más he encontrado documentándome para la redacción de la presente entrada es que es sencillo incumplir la responsabilidad única. Es decir, que como es muy "barato" escribir la anotación ```@Autowired``` es fácil olvidarse de que la clase está realizando demasiadas funciones (por utilizar demasiados servicios). No estoy del todo de acuerdo con este argumento porque si sabes lo que está haciendo por debajo un atributo con ```@Autowired``` no debería existir este problema, incluso si no hay un constructor kilométrico.

### Riesgo de NullPointerException

Hemos visto en la explicación de cómo funciona ```@Autowired```, que la clase como tal se instancia sin las dependencias; se añaden a posteriori. La pregunta es: ¿Qué pasaría si la clase se instanciara en alguna parte del código? La respuesta es que esas dependencias serían null. ¿Y cómo podríamos añadir esas dependencias? Pues la realidad es que no podríamos, nos quedaríamos con una clase a la que le falta rellenar atributos de forma irreversible. Es decir, que como tal podríamos instanciar la clase sin que el compilador se queje pero no sería totalmente funcional. 

Como tal este punto podría parecer que no es un problema; en nuestro ejemplo del controlador no esperamos que alguien lo instancie en ninguna otra clase. Pero hay un caso donde sí que es necesario y va a ser el tema del siguiente punto: el testing

### Tests unitarios poco mantenibles

En el punto anterior tratamos el problema al que nos enfrentamos si intentamos instanciar manualmente una clase con atributos ```@Autowired```. Supongamos que queremos aislar nuestra clase para hacer un test unitario rápido de ```AutowiredController```. Como los campos son privados, nuestra única salida es recurrir a librerías de testing como Mockito y su anotación ```@InjectMocks```. Un test básico de nuestra clase se podría tener esta forma:

```java
@ExtendWith(MockitoExtension.class)
class AutowiredControllerTest {

    @Mock
    private ServiceI service;

    @InjectMocks
    private AutowiredController controller;

    @Test
    void activate_ShouldReturnOkResponseWithServiceMessage() {
        String expectedMessage = "Mensaje de prueba";
        when(service.showSomething()).thenReturn(expectedMessage);
        ResponseEntity<String> response = controller.activate();
        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertEquals(expectedMessage, response.getBody());
        verify(service).showSomething();
    }
}
```

En un primer vistazo podría parecer que no hay ningún problema con esto. Pero la realidad es otra. Por ejemplo, si cambiamos la clase ```AutowiredController``` añadiendo un nuevo atributo con la anotación ```@Autowired``` el test seguiría funcionando. Si la dependencia no está declarada con ```@Mock```, Mockito dejará este nuevo atributo como null. Un cambio que es relativamente profundo a nivel funcional haría que los tests de la funcionalidad antigua pasaran de forma automática. Los tests no están probando la composición real del objeto.

Otro problema que tiene este test es la alta dependencia que tiene con Mockito. Dependemos totalmente de que Mockito use reflexión por debajo para romper el encapsulamiento de nuestra clase y forzar la inyección. Si el día de mañana queremos probar nuestro controlador instanciándolo como un objeto Java normal, no podremos hacerlo de forma directa.

### Mutabilidad de los campos inyectados

Por la naturaleza de las aplicaciones en Spring, es habitual que múltiples peticiones se ejecuten en paralelo, lo que implica concurrencia.

En este contexto, los beans gestionados por Spring son singletons por defecto, es decir, una única instancia compartida entre todas las peticiones.

Esto plantea una pregunta interesante:

¿qué ocurre si modificamos el estado interno de un bean durante la ejecución de una petición?

Veamos un ejemplo en el que el controlador cambia dinámicamente la implementación de un servicio durante la ejecución:
```java
@RestController
@RequestMapping("/api/autowired")
public class AutowiredController {
    
    @Autowired
    private ServiceI service;


    @Autowired
    private ApplicationContext context;
    
    @GetMapping("/activate")
    public ResponseEntity<String> activate(){
        return new ResponseEntity<>(service.showSomething(), HttpStatus.OK);
    }

    @GetMapping("/switch")
    public ResponseEntity<String> triggerUndefinedBehavior() throws InterruptedException {
        ServiceI originalService = this.service;

        ServiceI serviceB = (ServiceI) context.getBean("serviceB");

        this.service = serviceB;

        long randomTime = ThreadLocalRandom.current().nextLong(1000, 10001);
        Thread.sleep(randomTime);

        this.service = originalService;

        return new ResponseEntity<>(
            "Cambio realizado. Tiempo: " + randomTime + " ms",
            HttpStatus.OK);
    }
}
```
Durante la ejecución del método ```triggerUndefinedBehavior()```, el estado interno del controlador cambia temporalmente. Esto significa que otras peticiones concurrentes a ```activate()``` podrían observar un estado intermedio del objeto, dependiendo del momento exacto de ejecución.

Esto introduce una condición de carrera, donde el comportamiento depende del timing de los hilos, lo que puede provocar respuestas inconsistentes.

¿Debería permitirse este tipo de diseño?

En general, no. Los beans en Spring deberían ser inmutables o, como mínimo, no modificar su estado interno en función de la lógica de negocio.

Si es necesario utilizar distintas implementaciones de un mismo servicio, lo correcto es inyectarlas de forma explícita como dependencias separadas, en lugar de mutar el estado del bean en tiempo de ejecución.

## Mejoras utilizando inyección por constructor

Ahora vamos a ver cómo todas estas desventajas que hemos mencionado se subsanan utilizando la recomendación oficial de Spring; la inyección por constructor. Nuestro controlador sería el siguiente:

```java

@RestController
@RequestMapping("/api/constructor")
public class ConstructorController {
    

    private final ServiceI service;

    ConstructorController(ServiceI service) {
        this.service = service;
    }

    @GetMapping("/activate")
    public ResponseEntity<String> activate(){
        return new ResponseEntity<>(service.showSomething(), HttpStatus.OK);
    }
}
```

### Más difícil ignorar el SRP

En primer lugar vemos cómo ahora cada servicio que queremos utilizar se pasa por el constructor y como hace más explícitas las dependencias se facilita detectar clases con demasiadas responsabilidades. Aunque por otro lado, alguien podría argumentar que es más verboso. Para esto podemos utilizar la solución que ya proponía en 2013 Oliver Drotbohm en "Why field injection is evil" que es utilizar Lombok. Concretamente utilizando la anotación ```@RequiredArgsConstructor``` se genera automáticamente el constructor de todos los atributos que son final.
```java
@RestController
@RequestMapping("/api/constructor")
@RequiredArgsConstructor
public class ConstructorController {

    private final ServiceI service;

    @GetMapping("/activate")
    public ResponseEntity<String> activate() {
        return new ResponseEntity<>(service.showSomething(), HttpStatus.OK);
    }
}
```

### Evitamos NullPointerException

Cuando utilizamos la inyección por constructor no pasamos por el estado intermedio donde tenemos la clase instanciada y las dependencias son nulas. Es decir, que si queremos instanciar la clase debemos hacerlo correctamente; con todos los atributos finales.

### Testing sin depender de Mockito

Para subsanar las dificultades que teníamos creando mocks de la clase cuando usábamos inyección por campos utilizábamos la anotación de Mockito ```@InjectMocks```. Gracias a que ahora los atributos se pasan en el constructor no tenemos ese problema:

```java
class ConstructorControllerTest {

    @Test
    void activate_ShouldReturnOkResponseWithServiceMessage() {

        ServiceI service = () -> "Mensaje de prueba";

        ConstructorController controller =
                new ConstructorController(service);

        ResponseEntity<String> response = controller.activate();

        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertEquals("Mensaje de prueba", response.getBody());
    }
}
```
Por supuesto podemos utilizar Mockito igualmente, pero ya no es necesario. Y si el día de mañana decidimos migrar a otro framework basado en Java los tests van a seguir funcionando. Además, ahora sí que se prueba la composición del objeto en los tests y si añadimos una nueva dependencia (cambio funcional profundo), se obliga al desarrollador a cambiar los tests.

## Conclusión

La inyección por campos con ```@Autowired``` ha sido durante años una forma cómoda y muy extendida de trabajar en Spring, especialmente por su simplicidad y rapidez de uso. Sin embargo, como hemos visto a lo largo del análisis, esa “comodidad” oculta una serie de problemas estructurales que afectan directamente al diseño, la testabilidad y la robustez del código.

El principal problema de la inyección por campos no es su funcionamiento en sí, sino el hecho de que introduce dependencias ocultas en la clase. Esto debilita la claridad del diseño, dificulta la creación de objetos en estado válido fuera del contenedor de Spring y obliga a depender de mecanismos de reflexión o frameworks adicionales en los tests. Además, abre la puerta a estados inconsistentes y a errores que no son evidentes en tiempo de compilación.

Por el contrario, la inyección por constructor refuerza principios fundamentales del diseño orientado a objetos como la inmutabilidad y la claridad de dependencias. Obliga a que una clase se construya siempre en un estado válido, facilita el testing sin herramientas externas y hace explícito qué necesita cada componente para funcionar. Esto no solo mejora la mantenibilidad, sino que también reduce la probabilidad de errores en sistemas complejos.

Por estos motivos, Spring actualmente recomienda priorizar la inyección por constructor frente a la inyección por campos. Herramientas como Lombok han eliminado gran parte de la “verbosidad” que históricamente se le criticaba, lo que hace que hoy en día esta aproximación sea tan sencilla como la alternativa menos recomendable.

En definitiva, aunque ```@Autowired``` en campos sigue siendo funcional, su uso tiende a considerarse un antipatrón en diseños modernos con Spring. Adoptar la inyección por constructor no es solo una cuestión de estilo, sino una decisión que impacta positivamente en la calidad, la escalabilidad y la testabilidad del software.

## Referencias

- Baeldung – Field Injection is not recommended  
  [https://www.baeldung.com/java-spring-field-injection-cons](https://www.baeldung.com/java-spring-field-injection-cons)

- Baeldung – XML-based Dependency Injection in Spring  
  [https://www.baeldung.com/spring-xml-injection](https://www.baeldung.com/spring-xml-injection)  

- Marc Nuri – Inyección de campos desaconsejada  
  [https://blog.marcnuri.com/inyeccion-de-campos-desaconsejada-field-injection-not-recommended-spring-ioc](https://blog.marcnuri.com/inyeccion-de-campos-desaconsejada-field-injection-not-recommended-spring-ioc)

- GeeksforGeeks – Why field injection is not recommended in Spring  
  [https://www.geeksforgeeks.org/advance-java/why-is-field-injection-not-recommended-in-spring/](https://www.geeksforgeeks.org/advance-java/why-is-field-injection-not-recommended-in-spring/)  

- Oliver Drotbohm – Why field injection is evil  
  [https://odrotbohm.de/2013/11/why-field-injection-is-evil/](https://odrotbohm.de/2013/11/why-field-injection-is-evil/)

- Spring Framework (código fuente oficial)  
  [https://github.com/spring-projects/spring-framework/tree/main](https://github.com/spring-projects/spring-framework/tree/main)

- Ejemplo histórico de inyección por XML  
  [https://github.com/lucabixio/dependency-injection-xml](https://github.com/lucabixio/dependency-injection-xml)