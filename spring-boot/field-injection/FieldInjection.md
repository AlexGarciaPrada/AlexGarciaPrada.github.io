---
layout: post
title: ¿Por qué la inyección por campos (@Autowired) está dejando de utilizarse en Spring?
date: 2026-06-21
last_modified_at: 2026-06-21
---

# ¿Por qué la inyección por campos (@Autowired) está dejando de utilizarse en Spring?

El uso de la anotación @Autowired es una práctica muy extendida en los repositorios de Spring Boot. Como ingeniero de Software que acaba de comenzar en este framework pensaba que era una buena práctica, pero hace un par de semanas en una conversación con un ingeniero con mucha más experiencia que yo me dijo que no solo no era una buena práctica. El objetivo de esta entrada es analizar cómo funciona la inyección de campos en Spring (actualmente la notación @Autowired) y por qué Spring no recomienda utilizarlo.

## Historia de la anotación @Autowired

La anotación @Autowired se introdujo en la versión de Spring Framework 2.5 en noviembre de 2007. Concretamente se añadió el soporte de annotation-based dependency injection, entre las que se encuentran algunas anotaciones como @Autowired, @Component, @Service y @Repository.

¿Y cómo se hacía la inyección por campos antes de estas anotaciones?

Pues se hacía a través de XML. Por ejemplo, si tenemos el siguiente Repository:

```java
public class UserRepository {
    public String findUser() {
        return "Alex";
    }
}
```
Y queríamos inyectarlo en el siguiente Service:

```java
public class UserService {

    private UserRepository userRepository;

    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public void printUser() {
        System.out.println(userRepository.findUser());
    }
}
```
Y para que Spring supiera que userRepository era necesario inyectarlo había que declararlo en el archivo xml de configuración, de una forma similar a esta:

```xml
<beans>
    <bean id="userRepository" class="com.example.UserRepository"/>

    <bean id="userService" class="com.example.UserService">
        <property name="userRepository" ref="userRepository"/>
    </bean>
</beans>
```
Un ejemplo de repositorio donde se puede observar la inyección de dependencias por xml (tanto por campos como por constructor) es el siguiente : [https://github.com/lucabixio/dependency-injection-xml](https://github.com/lucabixio/dependency-injection-xml).

Es decir, que con la versión 2.5 de Spring en vez de basarse en la configuración de un xml para inyectar dependencias empezaron a utilizarse las anotaciones previamente descritas. Pero, ¿Cómo funciona realmente @Autowired por debajo?

## ¿Cómo funciona @Autowired por debajo?

Cuando ponemos que un campo es @Autowired, parece casi mágico cómo Spring añade la dependencia. Pero es conveniente que veamos exactamente cómo funciona para poder entender bien por qué la fundación de Spring desaconseja su uso. Supongamos que tenemos la siguiente clase:

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

}
```
Lo que hace Spring es primero generar el UserService y después añadir UserRepository. Se podría decir con muchas comillas que hace algo parecido a esto (no hace esto por debajo, este código es solo para facilitar entender los pasos):

```java
UserService service = new UserService();

Field field = UserService.class.getDeclaredField("userRepository");
field.setAccessible(true);

UserRepository repo =
    applicationContext.getBean(UserRepository.class);

field.set(service, repo);
```
En primer lugar lo que hace Spring es instanciar el UserService, dejando el atributo userRepository vacío (null). Cuando esta se pone en marcha el BeanPostProcessor.

¿Qué es el BeanPostProcessor?

Pues simplemente es esta interfaz de la configuración de Spring:


```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;
import org.springframework.lang.Nullable;

public interface BeanPostProcessor {
   @Nullable
   default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }

   @Nullable
   default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }
}
```

Como se puede observar tenemos dos "hooks", uno para antes de inicializar el bean y otro para después de la inicialización.

¿En cuál de los dos se inyectan las dependencias? 

En ninguna. Para llegar a la inyección por campos tenemos que bucear un poco más en distintas intefaces:

![Diagrama Interfaces Autowired](resources\AutowiredIntefacesUML.png)

Para la anotación de @Autowired, la clase que nos interesa se puede encontrar en : [AutowiredAnnotationBeanPostProcessor](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java).

En esta clase es donde verdaderamente ocurre la magia (o, mejor dicho, la reflexión) en la fase posterior a la instanciación. Se encarga de escanear la clase buscando metadatos, específicamente anotaciones como @Autowired, @Value o @Inject. Todo ello en el método PostProcessProperties():

```java
	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}
```

Al identificar el campo, busca en el ApplicationContext qué bean encaja con el tipo requerido (en nuestro caso, UserRepository).

Utiliza la API de Reflection de Java para romper el encapsulamiento llamando a field.setAccessible(true) e inyectar la instancia que encontró. Esto lo podemos ver en el código anterior en el metadata.inject(bean, beanName, pvs), que por debajo hace:

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

Una vez explicado como funciona el @Autowired, veamos por qué Spring desaconseja su uso. Para ello he decidido crear un repositorio muy sencillo para ejemplificarlo.

En este repositorio he definido una interfaz de Servicio "ServiceI" y dos servicios que lo implementan (ServiceA y ServiceB). El único método que tienen definido es ```java public String showSomething();```. Que simplemente devuelve una frase que identifica cada servicio. Y he hecho dos controladores, uno con inyección por campos y otro por constructor para poder hacer una comparación. La estructura es simplemente:

![Diagrama UML arquitectura repositorio de Ejemplo](resources\RepositoryUML.png){: style="max-height: 600px;"}


Y el controlador en el que nos centraremos es:
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
### Violación del Single Responsibility Principle

Uno de los argumentos que más he encontrado documentándome para la redacción de la presente entrada es que es sencillo incumplir la responsabilidad única. Es decir, que como es muy "barato" escribir la anotación @Autowired es fácil olvidarse de que la clase está realizando demasiadas funciones (por utilizar demasiados servicios). No estoy del todo de acuerdo con este argumento porque si sabes lo que está haciendo por debajo un atributo con @Autowired no debería existir este problema, incluso si no hay un constructor kilométrico.

### Riesgo de NullPointerException

Hemos visto en la explicación de cómo funciona @Autowired, que la clase como tal se instancia sin las dependencias; se añaden a posteriori. La pregunta es: ¿Qué pasaría si la clase se instanciara en alguna parte del código? La respuesta es que esas dependencias serían null. ¿Y cómo podríamos añadir esas dependencias? Pues la realidad es que no podríamos, nos quedaríamos con una clase a la que le falta rellenar atributos de forma irreversible. Es decir, que como tal podríamos instanciar la clase sin que el compilador se queje pero no sería totalmente funcional. 

Como tal este punto podría parecer que no es un problema; en nuestro ejemplo del controlador no esperamos que alguien lo instancie en ninguna otra clase. Pero hay un caso donde sí que es necesario y va a ser el tema del siguiente punto: el testing

### Test unitarios poco mantenibles

En el punto anterior tratamos el problema al que nos enfrentamos si intentamos instanciar manualmente una clase con atributos @Autowired. Supongamos que queremos aislar nuestra clase para hacer un test unitario rápido de AutowiredController. Como los campos son privados, nuestra única salida es recurrir a librerías de testing como Mockito y su anotación @InjectMocks. Un test básico de nuestra clase se podría tener esta forma:

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

En un primer vistazo podría parecer que no hay ningún problema con esto. Pero la realidad es otra. Por ejemplo, si cambiamos la clase AutowiredController añadiendo un nuevo atributo con la anotación @Autowired el test seguiría funcionando. Si la dependencia no está declarada con @Mock, Mockito dejará este nuevo atributo como null. Un cambio que es relativamente profundo a nivel funcional haría que los tests de la funcionalidad antigua pasaran de forma automática. Los tests no están probando la composición real del objeto.

Otro problema que tiene este test es la alta dependencia que tiene con Mockito. Dependemos totalmente de que Mockito use reflexión por debajo para romper el encapsulamiento de nuestra clase y forzar la inyección. Si el día de mañana queremos probar nuestro controlador instanciándolo como un objeto Java normal, no podremos hacerlo de forma directa.

### Mutabilidad de los campos inyectados

Por la naturaleza de las aplicaciones por Spring es muy común esperar que varios usuarios puedan lanzar operaciones al mismo tiempo y con esto tendríamos concurrencia.

¿Podría llegar a ocurrir que un usuario a través de una operación cambiara el servicio y afectara a otro usuario?

Pues la respuesta es que sí, y obtendríamos un "undefined behaviour" como le gusta decir a un amigo mío. Vamos a generar un método para cambiar el servicio de A a B y luego vuelva a ponerlo de A a B. Para ello hemos añadido el atributo ApplicationContext para poder obtener fácilmente el Service B y hemos implementado el nuevo método:



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
Cuando se hace un switch de servicio, durante un tiempo aleatorio todos los activates van a devolver una respuesta que no somos capaces de predecir.

¿Deberíamos permitir esto?

Claramente no. Si es necesario realizar una operación con otro servicio aunque dependa de la misma interfaz deberíamos utilizar dos atributos distintos para que no afecte al resto de operaciones que realizamos.

## Mejoras utilizando inyección por constructor

Ahora vamos a ver cómo todas estas desventajas que hemos utilizado se subsanan utilizando la recomendación oficial de Spring; la inyección por constructor. Nuestro controlador sería el siguiente:

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

### Single Responsibility Principle

En primer lugar vemos como ahora cada servicio que queremos utilizar se pasa por el constructor, lo que hace más difícil ignorar una violación del Single Responsibility Principle. Aunque por otro lado, alguien podría argumentar que es más verboso. Para esto podemos utilizar la solución que ya proponía en 2013 Oliver Drotbohm en "Why field injection is evil" que es utilizar Lombok. Concretamente utilizando la anotación @RequiredArgsConstructor se genera automáticamente el constructor de todos los atributos que son final.
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

Cuando utilizamos la inyección por campos no pasamos por el estado intermedio donde tenemos la clase instanciada y las dependencias son nulas. Es decir, que si queremos instanciar la clase debemos hacerlo correctamente; con todos los atributos finales.

### Testing sin depender de Mockito

Para subsanar las dificultades que teníamos creando mocks de la clase cuando usábamos inyección por campos utilizábamos la anotación de Mockito @InjectMocks. Gracias a que ahora los atributos se pasan en el constructor no tenemos ese problema:

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

La inyección por campos con @Autowired ha sido durante años una forma cómoda y muy extendida de trabajar en Spring, especialmente por su simplicidad y rapidez de uso. Sin embargo, como hemos visto a lo largo del análisis, esa “comodidad” oculta una serie de problemas estructurales que afectan directamente al diseño, la testabilidad y la robustez del código.

El principal problema de la inyección por campos no es su funcionamiento en sí, sino el hecho de que introduce dependencias ocultas en la clase. Esto debilita la claridad del diseño, dificulta la creación de objetos en estado válido fuera del contenedor de Spring y obliga a depender de mecanismos de reflexión o frameworks adicionales en los tests. Además, abre la puerta a estados inconsistentes y a errores que no son evidentes en tiempo de compilación.

Por el contrario, la inyección por constructor refuerza principios fundamentales del diseño orientado a objetos como la inmutabilidad y la claridad de dependencias. Obliga a que una clase se construya siempre en un estado válido, facilita el testing sin herramientas externas y hace explícito qué necesita cada componente para funcionar. Esto no solo mejora la mantenibilidad, sino que también reduce la probabilidad de errores en sistemas complejos.

Por estos motivos, Spring actualmente recomienda priorizar la inyección por constructor frente a la inyección por campos. Herramientas como Lombok han eliminado gran parte de la “verbosidad” que históricamente se le criticaba, lo que hace que hoy en día esta aproximación sea tan sencilla como la alternativa menos recomendable.

En definitiva, aunque @Autowired en campos sigue siendo funcional, su uso tiende a considerarse un antipatrón en diseños modernos con Spring. Adoptar la inyección por constructor no es solo una cuestión de estilo, sino una decisión que impacta positivamente en la calidad, la escalabilidad y la testabilidad del software.

## Referencias

- Baeldung – Field Injection is not recommended  
  https://www.baeldung.com/java-spring-field-injection-cons  

- Baeldung – XML-based Dependency Injection in Spring  
  https://www.baeldung.com/spring-xml-injection  

- Marc Nuri – Inyección de campos desaconsejada  
  https://blog.marcnuri.com/inyeccion-de-campos-desaconsejada-field-injection-not-recommended-spring-ioc  

- GeeksforGeeks – Why field injection is not recommended in Spring  
  https://www.geeksforgeeks.org/advance-java/why-is-field-injection-not-recommended-in-spring/  

- Oliver Drotbohm – Why field injection is evil  
  https://odrotbohm.de/2013/11/why-field-injection-is-evil/  

- Spring Framework (código fuente oficial)  
  https://github.com/spring-projects/spring-framework/tree/main  

- Ejemplo histórico de inyección por XML  
  https://github.com/lucabixio/dependency-injection-xml