# conf, env and logging

Jooby delegates configuration management to [config](https://github.com/typesafehub/config). By defaults Jooby will attempt to load an ```application.conf``` file from root of classpath. Inside the file you can add/override any property you want.

## property access

via script:

```java
{
  get("/", req -> {
    Config conf = req.require(Config.class);
    String myprop = conf.getString("myprop");
    ...
  });
}
```

or via ```@Named``` annotation:

```java
public class Controller {

  @Inject
  public Controller(@Named("myprop") String myprop) {
    ...
  }
}
```

### type conversion

Automatic type conversion is provided when a type:

1) Is a primitive, primitive wrapper or String

2) Is an enum

3) Has a public **constructor** that accepts a single **String** argument

4) Has a static method **valueOf** that accepts a single **String** argument

5) Has a static method **fromString** that accepts a single **String** argument. Like ```java.util.UUID```

6) Has a static method **forName** that accepts a single **String** argument. Like ```java.nio.charset.Charset```

7) There is custom [Guice](https://github.com/google/guice) type converter for the type

It is also possible to inject the root ```com.typesafe.config.Config``` object or a child of it.

## application.env

Jooby internals and also the module system rely on the ```application.env``` property. By defaults, this property is set to: ```dev```.

This special property is represented at runtime with the [Env]({{apidocs}}/org/jooby/Env.html) class.

For example, the [development stage](https://github.com/google/guice/wiki/Bootstrap) is set in [Guice](https://github.com/google/guice) when ```application.env == dev```.

A module provider, might decided to create a connection pool, cache, etc when ```application.env != dev ```.

## turn on/off features

As described before the ```application.env``` property defines the environment where the application is being executed. It is possible to turn on/off specific features base on the application environment:

```java
{
  on("dev", () -> {
    use(new DevModule());
  });

  on("prod", () -> {
    use(new ProdModule());
  });
}
```

There is a complement operator: ```.orElse``` too:

```java
{
  on("dev", () -> {
    use(new DevModule());
  }).orElse(() -> {
    use(new ProdModule());
  });
}
```

The ```env callback``` can access to ```config``` object, like:

```java
{
  on("dev", conf -> {
    use(new DevModule(conf.getString("myprop")));
  });
}
```


## special properties

### application.secret

If present, the session cookie will be signed with the ```application.secret```.

### default properties

Here is the list of default properties provided by  Jooby:

* **application.name**: describes the name of your application. Default is: *app.getClass().getSimpleName()*
* **application.tmpdir**: location of the temporary directory. Default is: *${java.io.tmpdir}/${application.name}*
* **application.charset**: charset to use. Default is: *UTF-8*
* **application.lang**: locale to use. Default is: *Locale.getDefault()*. A ```java.util.Locale``` can be injected.
* **application.dateFormat**: date format to use. Default is: *dd-MM-yyyy*. A ```java.time.format.DateTimeFormatter``` can be injected.
* **application.numberFormat**: number format to use. Default is: *DecimalFormat.getInstance("application.lang")*
* **application.tz**: time zone to use. Default is: *ZoneId.systemDefault().getId()*. A ```java.time.ZoneId``` can be injected.

## precedence

Config files are loaded in the following order (first-listed are higher priority)

* system properties
* arguments properties
* (file://[application].[mode].[conf])?
* (cp://[application].[mode].[conf])?
* ([application].[conf])?
* [modules in reverse].[conf]*


It looks kind of complex, right?
It does, but at the same time it is very intuitive and makes a lot of sense. Let's review why.

### system properties

System properties can override any other property. A sys property is set at startup time, like: 

    java -Dapplication.env=prod -jar myapp.jar

### arguments properties

Arguments properties can override any other property. A argument property is set at startup time, like: 

    java -jar myapp.jar application.env=prod

### file://[application].[mode].[conf] 

The use of this conf file is optional, because Jooby recommend to deploy your application as a **fat jar** and all the properties files should be bundled inside the jar.

If you find this impractical, then this option is for you.

Let's say your app includes a default property file: ```application.conf``` bundled with your **fat jar**. Now if you want/need to override two or more properties, just do this:

* find a directory to deploy your app
* inside that directory create a file: ```application.conf```
* start the app from same directory

That's all. The file system conf file will take precedence over the classpath config file, overriding any property.

A good practice is to start up your app with a **env**, like:

    java -jar myapp.jar prod

The process is the same, except this time you can name your file as:

    application.prod.conf

### cp://[application].[mode].[conf]

Again, the use of this conf file is optional and works like previous config option, except here the **fat jar** was bundled with all your config files (dev, stage, prod, etc.)

Example: you have two config files: ```application.conf``` and ```application.prod.conf````. Both files were bundled inside the **fat jar**, starting the app in **prod** env:

    java -jar myapp.jar application.env=prod

So here the ```application.prod.conf``` will takes precedence over the ```application.conf``` conf file.

This is the recommended option from Jooby, because your app doesn't have an external dependency. If you need to deploy the app in a new server all you need is your **fat jar**

### [application].[conf]

This is the default config file and it should be bundle inside the **fat jar**. As mentioned early, the default name is: **application.conf**

### [modules in reverse].[conf]

As mentioned in the [modules](#modules) section a module might define his own set of properties.

```
{
   use(new M1());
   use(new M2());
}
```

In the previous example the M2 modules properties will take precedence over M1 properties.

The config system is very powerful and allow you to create a single distribution with different configuration per environment.
 