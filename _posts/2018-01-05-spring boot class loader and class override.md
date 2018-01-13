---
layout: post
title: Spring Boot ClassLoader and Class Override
comments: true
---
Article explains the Spring Boot classloader (_LaunchedURLClassLoader_) and a way to temporarily override library classes with your custom ones.

&nbsp;

#### Just a Little Fix
Let's say you found a bug in some third party jar your app uses. As a good scout you fixed it and created a pull request with a solution.
The pull request was merged, but the fix is critical for you and you can't wait till next library release.
Is using library snapshot the only way? Wouldn't it be great if there existed a solution overriding temporarily only few particular classes?

&nbsp;

As an imaginary example ([follow the code](https://github.com/dkublik/spring-boot-loader-play)) let's say you found a bug in [SpringBootBanner](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/SpringBootBanner.java) class
and already have a solution fixing the banner's colors - [SpringBootBanner fixed](https://github.com/dkublik/spring-boot-loader-play/blob/master/src/main/java/org/springframework/boot/SpringBootBanner.java)

(_I know we can easily define custom banners in Spring Boot, it's just an useful example - it will be super easy to spot if the 'fix' is working or not_)

So what can we do to have the solution working immediately?  
Les's just take the class (with the package) and paste is into our project (_src/main/java_).

Now let's run the App from IDE and everything seems to work

![Working Fix in an IDE]({{ site.baseurl }}/images/2018-01-06-spring-boot-class-loader/banner-ide-working.png "Working Fix in an IDE")

&nbsp;

Great! But the joy is premature...  
If you build the app and run it

```
./gradlew build
cd build/libs
java -jar spring-boot-loader-play-0.0.1-SNAPSHOT.jar
```

Original banner is still being displayed, and this is not about a terminal not suporting ANSI colors.  
The banner class (_SpringBootBanner_) was simply not overriden.

![Not Working Fix When Running Jar]({{ site.baseurl }}/images/2018-01-06-spring-boot-class-loader/banner-jar-not-working.png "Not Working Fix When Running Jar")

&nbsp;

The difference is that when you launch the app from an IDE you have two kinds of artifacts: classes and jars. Classes are loaded before jars so even though you have two versions of a class (your fix in _/src/main/java_ and original in _spring-boot-2.0.0.M7.jar_ lib) only the fix will be loaded. (ClassLoaders don't care about duplicates - the class that is found first is loaded).

&nbsp;

#### Spring Boot ClassLoader

With jar the situation is harder.
It's a Spring Boot fat jar, with structure as below 

```
+--- spring-boot-loader-play-0.0.1-SNAPSHOT.jar
     +--- META-INF
     +--- BOOT-INF
     |    +--- classes                            # 1 - project classes
     |    |     +--- org.springframework.boot
     |    |     |	 \--- SpringBootBanner.class    # this is our fix
     |    |     |	 
     |    |     +--- pl.dk.loaderplay
     |    |          \--- SpringBootLoaderApplication.class
     |    |
     |    +--- lib                                # 2 - nested jar libraries
     |          +--- javax.annotation-api-1.3.1
     |          +--- spring-boot-2.0.0.M7.jar     # original banner class inside
     |          \--- (...)
     |
     +--- org.springframework.boot.loader         # Spring Boot loader classes
          +--- JarLauncher.class
          +--- LaunchedURLClassLoader.class
          \--- (...)
```
so actually it contains three type of entries:

1. Project classes
2. Nested jar libraries
3. Spring Boot loader classes

Both Project classes (_BOOT-INF/classes_) and nested jars (_BOOT-INF/lib_) are handled by the same classLoader which in turn resides in the root of the jar (_org.springframework.boot.loader.LaunchedURLClassLoader_).

On might expect that _LaunchedURLClassLoader_ will load the _classes_ content before the _lib_ content but the loader seems not the have that preference.  
[LaunchedURLClassLoader](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-loader/src/main/java/org/springframework/boot/loader/LaunchedURLClassLoader.java) extends [java.net.URLClassLoader](https://github.com/frohoff/jdk8u-jdk/blob/master/src/share/classes/java/net/URLClassLoader.java) which is created with a set of Urls that will be used for classloading.  
Url might point to a location like jar archive or classes folder. When classloading all of the resources specified by urls will be traversed in the order the urls were provided and the first resource containing the searched class will be used.  
So how the urls are provided to LaunchedURLClassLoader? Simply - the jar archive is parsed from up to bottom and when an archive is found it's added to url's list.

In our example:
```
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-boot-starter-2.0.0.M7.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-boot-autoconfigure-2.0.0.M7.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-boot-2.0.0.M7.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-boot-starter-logging-2.0.0.M7.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/javax.annotation-api-1.3.1.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-context-5.0.2.RELEASE.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-aop-5.0.2.RELEASE.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-beans-5.0.2.RELEASE.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-expression-5.0.2.RELEASE.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-core-5.0.2.RELEASE.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/snakeyaml-1.19.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/logback-classic-1.2.3.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/log4j-to-slf4j-2.10.0.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/jul-to-slf4j-1.7.25.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-jcl-5.0.2.RELEASE.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/logback-core-1.2.3.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/slf4j-api-1.7.25.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/log4j-api-2.10.0.jar!/"
"jar:file:/Users/dk/spring-boot-loader-play/build/libs/spring-boot-loader-play-0.0.1-SNAPSHOT.jar!/BOOT-INF/classes!/"
```

As we can see _/BOOT-INF/classes_ is the last entry on the list - far after _/BOOT-INF/lib/spring-boot-2.0.0.M7.jar_ so in the search for _SpringBootBanner.class_ the version from the latter will be used - not an outcome we hoped for.

&nbsp;

On our quest to "what can we do about it" it's worth zooming into how classloaders work in hierachies.  
Basically classLoaders form hierarchies, with every child loader having a reference to it's parent.  
With _LaunchedURLClassLoader_ being the youngest descendant in Spring Boot case we end up with simple hierarchy like this:

```
+--- sun.misc.Launcher$ExtClassLoader     # loading classes /jre/lib/ext/
     +--- sun.misc.Launcher.Launcher$AppClassLoader    # loading classes from the root of the jar - spring-boot-loader-play-0.0.1-SNAPSHOT.jar
          +--- org.springframework.boot.loader.LaunchedUrlClassLoader    # loading classes from /BOOT-INF/lib/ & /BOOT-INF/classes/

```
 
In Spring Boot when the class is about to be load we always start with _LaunchedUrlClassLoader_ but the "parent first" rule applies. This means that child loader will try to load a given class only if the parent doesn't find it.

&nbsp;

#### First Idea - AppClassLoader

If _LaunchedUrlClassLoader_ firstly delegates classloading to _AppClassLoader_ then why not to use this one to load our class before it's loaded by _LaunchedUrlClassLoader_?

You might be tempted to simply do 

```java
Thread.currentThread().getContextClassLoader()
        .getParent()
        .loadClass("org.springframework.boot.SpringBootBanner");
```

but this won't work. Yes - _Thread.currentThread().getContextClassLoader().getParent()_ gets us the correct _AppClassLoader_ but this one is designed to work with standard jars - where classes (with packages) are placed in the root of the jar. So where _AppClassLoader_ have no problems handling the _org.springframework.boot.loader_ classes (see the jar directory tree above) it will not find classes in _BOOT-INF/classes_.

```java
Thread.currentThread().getContextClassLoader()
        .getParent()
        .loadClass("BOOT-INF/classes/org/springframework/boot/SpringBootBanner");
```

won't work neither. Yes - the class will be found, but the package will not match the path.

&nbsp;

It appears there is no other way than copying _SpringBootBanner_ from  _BOOT-INF/classes/org/_ into the root of the jar. If we do it there is no need to call _AppClassLoader_ directly to load the class as it will always have precedence before _LaunchedUrlClassLoader_.

This is easily done with gradle build:
```groovy
bootJar {
    with copySpec {
        from "$buildDir/classes/java/main/org"
        into 'org'
    }
}
```

We can launch the app and find that trully _SpringBootBanner_ was copied to the jar root and _AppClassLoader_ was used to load it but it won't work.
The problem is that _SpringBootBanner_ depends on other classes - loaded by child _LaunchedUrlClassLoader_.
One thing we forgot about classLoaders hierarchy is that classes loaded by parents don't see classes loaded by children.  
The "load by AppClassLoader" idea seems to be a dead end - but worry not. We will use the knowledge with the second one!

&nbsp;

#### LaunchedUrlClassLoader - resources order

It appears that parent loaders is not an option and we are stuck with the last loader in the hierarchy - _LaunchedUrlClassLoader_.
You might remember that _LaunchedUrlClassLoader_ loads classes traversing nested resources in the order they were provided to it. So let's try to manupulate the order, so that _/BOOT-INF/classes/_ resource is first - not last on the list.

With [org.springframework.boot.loader.JarLauncher](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-loader/src/main/java/org/springframework/boot/loader/JarLauncher.java) this seems to be an easy task as it provides
```
protected void postProcessClassPathArchives(List<Archive> archives)
```
method to manipulate archives just before they are given to _LaunchedUrlClassLoader_.

So let's write a custom launcher using this functionality

```java
public class ClassesFirstJarLauncher extends JarLauncher {

    @Override
    protected void postProcessClassPathArchives(List<Archive> archives)
            throws MalformedURLException {
        for (int i = archives.size() - 1; i >= 0; i--) {
            Archive archive = archives.get(i);
            if (archive.getUrl().getPath().endsWith("/classes!/")) {
                archives.remove(archive);
                archives.add(0, archive);
                break;
            }
        }
    }

    public static void main(String[] args) throws Exception {
        new ClassesFirstJarLauncher().launch(args);
    }
}
```

&nbsp;

Quick reminder is that _JarLauncher_ is the class launching your Spring Boot app. Check any Spring Boot _MANIFEST.MF_ and you will find sth like:

```
Manifest-Version: 1.0
Start-Class: pl.dk.loaderplay.SpringBootLoaderApplication
Main-Class: org.springframework.boot.loader.JarLauncher
```

_Main-Class_ being the class with main method launched when 

```
java -jar spring-boot-loader-play-0.0.1-SNAPSHOT.jar
```

&nbsp;

JarLauncher must be loaded by _AppClassLoader_ (_LaunchedUrlClassLoader_ is not even loaded itself yet) and to do that it's must be placed in the root of the jar.
Let's use the trick we learned before

```
bootJar {
    with copySpec {
        from "$buildDir/classes/java/main/pl/dk/loaderplay/ClassesFirstJarLauncher.class"
        into 'pl/dk/loaderplay'
    }
}
```

&nbsp;

What remained is to replace _Main-Class_ in _MANIFEST.MF_.
_Spring Boot Gradle Plugin_ provides a way to do it

```
bootJar {
    manifest {
        attributes 'Main-Class': 'pl.dk.loaderplay.ClassesFirstJarLauncher'
    }
}
```

Unfortunately, when replacing _Main-Class_ , the original Spring Boot loader / launcher classes are not copied to the root of the jar - and we still need them. This is how _Spring Boot Gradle Plugin_ works and I have not found a way around it.  
(It happens because in plugin's [BootZipCopyAction](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-gradle-plugin/src/main/java/org/springframework/boot/gradle/tasks/bundling/BootZipCopyAction.java) decision on either or not to copy the loader files is based upon the fact if original _JarLauncher_ was used or not)

So changing _Main-Class_ by _bootJar_ configuration is no use to us. One can try to change it in some other way. For me - it was enough to leave the original _Main-Class_ in the  manifesto and simply specify start class when launching the app.

```
java -cp spring-boot-loader-play-0.0.1-SNAPSHOT.jar \
pl.dk.loaderplay.ClassesFirstJarLauncher
```

&nbsp;

When doing so the class was finally overriden:

![Working Fix When Running Jar]({{ site.baseurl }}/images/2018-01-06-spring-boot-class-loader/banner-jar-working.png "Working Fix When Running Jar")

&nbsp;

#### Quick summary

To summarize shortly

##### Goal

override
```
spring-boot-loader-play-0.0.1-SNAPSHOT.jar
/BOOT-INF/lib/spring-boot-2.0.0.M7.jar/org/springframework/boot/SpringBootBanner.class
```
with
```
spring-boot-loader-play-0.0.1-SNAPSHOT.jar
/BOOT-INF/classes/org/springframework/boot/bootSpringBootBanner.class
```

##### Steps
1. Place the overriding [SpringBootBanner](https://github.com/dkublik/spring-boot-loader-play/blob/master/src/main/java/org/springframework/boot/SpringBootBanner.java) in _src/main/java_
2. Create custom launcher ordering resources from which classes are load - [ClassesFirstJarLauncher](https://github.com/dkublik/spring-boot-loader-play/blob/master/src/main/java/pl/dk/loaderplay/ClassesFirstJarLauncher.java) 
3. Copy the launcher to root of the jar by [bootJar gradle task](https://github.com/dkublik/spring-boot-loader-play/blob/master/build.gradle)
4. Launch the archive specifying the launcher class
```
java -cp spring-boot-loader-play-0.0.1-SNAPSHOT.jar \
pl.dk.loaderplay.ClassesFirstJarLauncher
```

&nbsp;

Again you may check the code [here](https://github.com/dkublik/spring-boot-loader-play)

&nbsp;
