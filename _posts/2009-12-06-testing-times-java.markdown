---
layout: post
title:  "Testing Times in Java"
date:   2009-12-06 19:33:27
categories: misc
---

This post first appeared on the [LShift web site](http://www.lshift.net/blog/2009/12/06/testing-times-in-java/), which also has an example project.

How do you test a Java application that uses the current date or time as the basis for a calculation? I’ve seen a couple of ways:

* Make sure all date/time operations use a service class to get the current date, which is then mocked
* Adjust the system clock

Option 1) is a PITA for an existing codebase, or if any 3rd party libraries do date processing. Option 2) rules out any sensible automation, especially on a machine used for anything else.

Another option is to mock system date related classes. [JMockit](https://code.google.com/p/jmockit/) is a mock testing framework that uses Java 5 instrumentation to allow bytecode to be modified at runtime. Using this, known date values can be inserted into tests with no interference with production code or rewriting of legacy code. If Java 6 is available, it is possible to go further and mock the `System` class itself.

We have an example application that performs the onerous task of checking whether a supplied date is the future or not:

{% highlight java %}
public class App {
  public static boolean isInFuture(Date date) {
    return new Date().after(date);
  }
}
{% endhighlight %}

To test this under Java 5, we need to mock out the `java.util.Date` class. First of all, we need to configure JMockit. The project (linked to from the original LShift article) uses [Maven](http://maven.apache.org/) for building and testing. The POM is pretty much the standard archetype with the following modifications:

1\. JMockit lives at java.net, so their repo needs to be added:

{% highlight xml %}
<repositories>
  <repository>
    <id>download.java.net</id>
    <url>http://download.java.net/maven/2</url>
  </repository>
</repositories>
{% endhighlight %}

2\. The library needs to be added as a dependency for testing:

{% highlight xml %}
<dependency>
  <groupId>mockit</groupId>
  <artifactId>jmockit</artifactId>
  <version>0.993</version>
  <scope>test</scope>
</dependency>
{% endhighlight %}

3\. Finally, the JMockit JAR needs to be set as the Java agent in order to hook into the instrumentation API. This is done in the Surefire plugin configuration by passing the correct argument to the JVM.

{% highlight xml %}
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <argLine>-javaagent:"${settings.localRepository}/mockit/jmockit/0.993/jmockit-0.993.jar"</argLine>
  </configuration>
</plugin>
{% endhighlight %}

With JMockit configured, a mock for the `java.util.Date` class can be created:

{% highlight java %}
1.   @MockClass(realClass = Date.class)
     public class MockDate {
         private Calendar now = null;
         public Date it;

         public MockDate(Calendar now) {
             this.now = now;
         }

2.       @Mock
3.       public void $init() throws Exception {
4.           Field field = Date.class.getDeclaredField("fastTime");
             field.setAccessible(true);
             field.set(it, now.getTimeInMillis());
         }
     }
{% endhighlight %}
     
1. The MockClass annotation declares the class as containing mock methods or constructors
2. The Mock annotation declares the following method / constructor as a replacement for that in the real class
3. The special `$init()` method overrides the default constructor in the real class
4. The private field `fastTime` holds the underlying date represented by a `Date` object, and so this is set to be the date passed to the mock

The mock object can now be initialised and used in a JUnit test class:

{% highlight java %}
   public class AppTest extends TestCase {
        private static Calendar TODAY = new GregorianCalendar(2009, JANUARY, 5);
        private static Calendar YESTERDAY = new GregorianCalendar(2009, JANUARY, 4);
        private static Calendar TOMORROW = new GregorianCalendar(2009, JANUARY, 6);

        ...

        @Override
        protected void setUp() {
1.          mockit.Mockit.setUpMocks(new MockDate(TODAY));
        }

        @Override
        protected void tearDown() {
2.          mockit.Mockit.restoreOriginalDefinition(Date.class);
        }

        public void testDateInFuture() {
           assertTrue(App.isInFuture(TOMORROW.getTime()));
        }

        public void testDateInPast() {
          assertFalse(App.isInFuture(YESTERDAY.getTime()));
        }
   }
{% endhighlight %}

1. Tells JMockit which classes are to be mocked
2. Restores mocked classes to the original versions. Failure to do this can lead to interesting and often subtle errors in subsequent tests

The same approach can be taken for the` java.util.Calendar` class.

Java 6 allows the instrumentation of native methods which means that the `System.currentTimeMillis()` method can be mocked. This is used in both the `java.util.Date` or `java.util.Calendar` classes and so is the simplest way of covering all bases if using Java 6:

{% highlight java %}
@MockClass(realClass = System.class)
public class MockSystem {
    private Calendar now = null;

    public MockSystem(Calendar now) {
        this.now = now;
    }

    @Mock
    public long currentTimeMillis() {
        return now.getTimeInMillis();
    }
}

{% endhighlight %}
