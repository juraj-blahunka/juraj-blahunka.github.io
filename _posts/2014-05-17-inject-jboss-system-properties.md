---
layout: post
title: Inject JBoss 7 system properties into your Java EE application
---

How do you make your Java EE application configurable? Do you use Maven profiles? Properties file? Environment properties?
If you have developed with JBoss 7 before, you have probably already heard about [JBoss 7 System Properties](https://community.jboss.org/wiki/JBossAS7SystemProperties).

They are usually specified in configuration file of mode specific directory, for example ```standalone```, they are
persistent (live through server restarts) and can be configured, most usually via ```jboss-cli```.

One of the great things about Java EE application development is its flexibility. Why not use this flexibility to
inject these properties into runtime?

In following lines you will see:

* how to inject JBoss system properties into runtime
* REST resource demonstration
* that you don't need to read it, just [fork the project on GitHub](https://github.com/juraj-blahunka/site-examples/tree/master/inject-jboss-system-properties)


## CDI annotations to the rescue


First, we need to define, how would our default access pattern look. Let's leverage the ```@Inject``` annotation with a
custom ```@Qualifier```:

~~~ java
@Inject
@SystemProperty("example.foo")
String foo;
~~~

Now, let's define the ```@SystemProperty``` annotation:

~~~ java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.PARAMETER, ElementType.FIELD, ElementType.METHOD, ElementType.TYPE})
public @interface SystemProperty {

	/**
	 * Full name of the system property, for example "user.home"
	 */
	@Nonbinding String value();

}
~~~

Note, that we do not define ```default ""``` next to our ```value()```, since the property name should be *always* defined.
Therefore we automatically disallow usage of empty system property ```@SystemProperty()```.

And how will the System property get from JBoss to our little ```@SystemProperty``` annotated variable?
Let's define a provider, who uses the ```@Produces``` annotation and provides our application with
concrete system properties:

~~~ java
public class SystemPropertyProvider {
^
	@Produces
	@SystemProperty("")
	String findProperty(InjectionPoint ip) {
		SystemProperty annotation = ip.getAnnotated()
			.getAnnotation(SystemProperty.class);

		String name = annotation.value();
		String found = System.getProperty(name);
		if (found == null) {
			throw new IllegalStateException("System property '" + name + "' is not defined!");
		}
		return found;
	}

}
~~~


## Demonstration

After we have defined our deliver mechanism, we probably want to use our new ```@SystemProperty``` annotation.
Let's define an example REST resource:

~~~ java
@Path("example")
public class ExampleResource {

	@Inject
	@SystemProperty("example.foo")
	String foo;

	@Inject
	@SystemProperty("example.bar")
	String bar;

	@GET
	@Produces(MediaType.TEXT_PLAIN)
	public String printSomeSystemProperties() {
		return "foo=" + foo + ", bar=" + bar;
	}

}
~~~

Deploy to JBoss 7 and hit ```http://localhost:8080/inject-jboss-system-properties/example```.
But what happened? We are getting an exception:

~~~
java.lang.IllegalStateException: System property 'example.foo' is not defined!
	sk.blahunka.jbossinject.properties.SystemPropertyProvider.findProperty(SystemPropertyProvider.java:15)
~~~

We forgot about our properties, to define them, we will use the good old jboss-cli.
On windows, fire up the ```jboss-cli.bat``` executable and type in:

~~~
connect

/system-property=example.foo:add(value="Special Foo Value")
/system-property=example.bar:add(value="Why is Foo so special?")
~~~

Then **hit refresh** in your browser and voila, our properties were injected:

~~~
foo=Special Foo Value, bar=Why is Foo so special?
~~~

You can fork the [inject-jboss-system-properties](https://github.com/juraj-blahunka/site-examples/tree/master/inject-jboss-system-properties) project on GitHub.
