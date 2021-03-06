## What is ADP4J ?

Annotation Driven Properties For Java is a library that allows you to inject configuration properties in your objects in a declarative way using annotations.

Configuration properties can be loaded from a variety of sources:

 - System properties passed to JVM
 - Properties files
 - Resource bundles
 - Database, etc

Usually, we write a lot of boilerplate code to load these properties into objects, convert them to typed values, etc.

The idea behind ADP4J is to implement the "Inversion Of Control" principle : Instead of having objects looking actively for configuration properties, these objects simply declare configuration properties they need and these properties will be provided to them by a tool, ADP4J for instance!

Let's see an example. Suppose you have a java object of type `Bean` which should be configured with:

 - An Integer property "threshold" from a system property passed to the JVM with -Dthreshold=100

 - A String property "bean.name" from a properties file named "myProperties.properties"

To load these properties in your `Bean` object using ADP4J, you annotate fields to declare needed configuration properties as follows:

```java
public class Bean {

    @SystemProperty(value = "threshold", defaultValue = "50")
    private int threshold;

    @Property(source = "myProperties.properties", key = "bean.name")
    private String beanName;

    //getters and setters omitted
}
```

and instructs ADP4J to inject these configuration properties in the annotated fields :

```java
//Instantiate your object
Bean bean = new Bean();

//Instantiate ADP4J properties injector
PropertiesInjector propertiesInjector = new PropertiesInjectorBuilder().build();

//Inject properties in your object
propertiesInjector.injectProperties(bean);
```

That it! ADP4J will introspect the `Bean` type instance looking for fields annotated with `@Property` and `@SystemProperty`,
 convert each property value to the field type and inject that value into the annotated field.

Without ADP4J, you would write something like this :

```java
public class Bean {

    private int threshold;

    private String beanName;

    public Bean() {

        //Load 'threshold' property from system properties
        String thresholdProperty = System.getProperty("threshold");
        try {
            threshold = Integer.parseInt(thresholdProperty);
        } catch (NumberFormatException e) {
            // log exception
            threshold = 50; //default threshold value;
        }

        //Load 'bean.name' property from properties file
        Properties properties = new Properties();
        try {
            InputStream inputStream = this.getClass().getClassLoader().getResourceAsStream("myProperties.properties");
            if (inputStream != null) {
                properties.load(inputStream);
                beanName = properties.getProperty("bean.name");
            }
        } catch (IOException ex) {
            // log exception
            beanName = "FOO"; // default bean name value
        }

    }

    //getters and setters omitted

}
```

As you can see, a lot of plumbing code is written to load two properties, convert them to the right type, etc.

ADP4J handles all this plumbing for you, which makes your code cleaner, more readable and maintainable.

## Built-in Annotations

By default, ADP4J provides 4 annotations to load configuration properties from a variety of sources.

### @SystemProperty

This annotation allows you to inject a property from the system properties passed to your java program at JVM startup using the -D prefix.

The @SystemProperty annotation can be declared on a field and have the following attributes :

| Attribute    | Type    | Required | Description                                                         |
|:-------------|:-------:|:--------:|---------------------------------------------------------------------|
| value        | String  | yes      | The system property to inject in the annotated field.               |
| defaultValue | String  | no       | The default value to set in case the system property does not exist |

Example:

```java
@SystemProperty("user.home")
private String userHome;
```

In this example, ADP4J will look for the system property `user.home` and set its value to the `userHome` field.

### @Properties

This annotation can be declared on a field of type `java.util.Properties` and allows you to inject all properties of a properties file in that field.

The annotation have a single attribute which value corresponds to the properties file name. Example:

```java
@Properties("myProperties.properties")
private java.util.Properties myProperties;
```

In this example, ADP4J will populate the `myProperties` field with properties from the file `myProperties.properties`.

### @Property

This annotation can be used to load a property from a java properties file.

The annotation can be declared on a field to tell ADP4J to inject the specified key value from the properties file into the annotated field.

Attributes of this annotation are described in the following table:

| Attribute  | Type    | Required | Description                                       |
|:-----------|:-------:|:--------:|---------------------------------------------------|
| source     | String  | yes      | The properties file name                          |
| key        | String  | yes      | The key to load from the source properties file   |

Example :

```java
@Property(source = "myProperties.properties", key = "bean.name")
private String beanName;
```

In this example, ADP4J will look for the property `bean.name` in the `myProperties.properties` properties file in the classpath and set its value to the `userHome` field.

### @I18NProperty

This annotation allows you to inject a property from a java resource bundle for a specified locale.

The annotation can be declared on a field to tell ADP4J to inject the specified key value from the resource bundle into the annotated field.

Attributes of this annotation are described in the following table:

| Attribute  | Type     | Required | Description                                                      |
|:-----------|:--------:|:--------:|------------------------------------------------------------------|
| bundle     | String   | yes      | The resource bundle containing the property to load              |
| key        | String   | yes      | The key to load from the resource bundle                         |
| language   | String   | no       | The locale language to use (default to default locale language)  |
| country    | String   | no       | The locale country to use (default to default locale country)    |
| variant    | String   | no       | The locale variant to use (default to default locale variant)    |

Example :

```java
@I18NProperty(bundle = "i18n/messages", key = "my.message")
private String message;
```

In this example, ADP4J will look for the property `my.message` in the resource bundle `i18n/messages.properties` in the classpath and set its value to the `message` field.

Note that this annotation is not suited to applications where the locale may change during the application lifetime. This is because java annotation attributes can only have constant values.

## Custom Annotations

With ADP4J, you can write your own annotations to load configuration properties from a custom location.
The following steps describe how to create your own annotation and how to use it with ADP4J.

### 1. Create custom annotation

First, you need to create your annotation with all the necessary information about where to load properties, which property to load, etc.
You can specify these parameters using public methods in your annotation interface.
You can see how the built-in annotations are defined in the `net.benas.adp4j.annotations` [package][annotationsPackage].

### 2. Implement the `AnnotationProcessor` interface

The `AnnotationProcessor` interface allows you to specify how to process your annotation to load all the necessary parameters specified by the user.

The following is the definition of this interface:

 ```java
public interface AnnotationProcessor<T extends Annotation> {

    /**
     * Process an annotation of type T to be introspected by ADP4J.
     *
     * @param annotation the annotation to process.
     * @param field the field annotated with the annotation.
     * @param object the object being configured.
     * @throws Exception thrown if an exception occurs during annotation processing
     */
    void processAnnotation(T annotation, Field field, Object object) throws Exception;

}
 ```

You can see some examples of how the built-in annotations are processed in the `net.benas.adp4j.processors` [package][processorsPackage].

### 3. Register the annotation processor within ADP4J

To register your custom annotation processor within ADP4J, you can use the `PropertiesInjectorBuilder` API as follows:

 ```java
 //Instantiate ADP4J properties injector and register custom annotation processor
 PropertiesInjector propertiesInjector = new PropertiesInjectorBuilder()
                             .registerAnnotationProcessor(MyAnnotation.class, new myAnnotationProcessor())
                             .build();
 ```

Now you can annotate any field with your annotation and ADP4J will automatically use the registered annotation processor to
load the value, convert it to the field type and set it to the field value.

## Getting started

ADP4J is a single jar file with no dependencies. To build it from sources, you need to have maven installed and set up.

To use ADP4J, please follow these instructions :

 * $>`git clone https://github.com/benas/adp4j.git`

 * $>`mvn package`

 * Add the generated jar `target/adp4j-${version}.jar` to your application's classpath

If you use maven, you should build/install the jar to your local maven repository and add the following dependency to your pom.xml :
```xml
<dependency>
    <groupId>net.benas</groupId>
    <artifactId>adp4j</artifactId>
    <version>${version}</version>
</dependency>
```

## License
ADP4J is released under the [MIT License][].

[annotationsPackage]: https://github.com/benas/adp4j/tree/master/src/main/java/net/benas/adp4j/annotations
[processorsPackage]: https://github.com/benas/adp4j/tree/master/src/main/java/net/benas/adp4j/processors
[MIT License]: http://opensource.org/licenses/mit-license.php/