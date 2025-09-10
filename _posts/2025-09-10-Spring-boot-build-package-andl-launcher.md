<h1>Spring-boot builds an executable package and launcher</h1>

let’s dig into how Spring Boot turns your project into an executable JAR and how that JAR actually runs, step-by-step and from an implementation perspective. I’ll cover the repackaging done at build time, the on-disk layout produced, and the runtime launcher + class-loader magic that makes nested JARs work.

1) High level: two phases

Build time (repackaging) — spring-boot-maven-plugin / spring-boot-gradle-plugin rearranges your artifact and your dependencies into a single JAR with a specific layout and special launcher classes and manifest entries.

Runtime (launching) — JVM starts the process (java -jar app.jar) and runs the special Spring Boot launcher (a class included in the JAR). The launcher sets up a custom classloader that can load classes from the nested JARs and then delegates to your application main().

2) What the plugin does at build time

When you run mvn package with the Spring Boot repackage goal (or the Gradle plugin), the plugin:

Gathers your compiled classes and resources (your app).

Collects all dependency JARs.

Creates a new JAR that contains:

Your app classes at BOOT-INF/classes/

All dependency JARs at BOOT-INF/lib/*.jar

Spring Boot loader classes (the launcher implementation) placed at the top level of the final JAR (so they are directly loadable)

A META-INF/MANIFEST.MF with special attributes:

Main-Class: org.springframework.boot.loader.JarLauncher (or another launcher)

Start-Class: com.example.YourApplication (your app’s main)

Spring-Boot-Version: x.y.z and other metadata

Handles merging/processing of META-INF/* (e.g. merging service files or spring.factories if needed).

Optionally supports layering, relocations, and other plugin options.

Important: Spring Boot does not (by default) flatten all dependency classes into the top-level JAR; it puts dependencies as nested JAR files under BOOT-INF/lib. That is a deliberate design to:

Keep dependencies intact (avoid class name collisions resolution problems that flattening can cause).

Simplify repackaging and maintain original JAR metadata.

3) The on-disk JAR layout (typical)
myapp.jar
├─ META-INF/
│   └─ MANIFEST.MF  (Main-Class=org.springframework.boot.loader.JarLauncher, Start-Class=...)
├─ org/springframework/boot/loader/...   (the launcher classes)
├─ BOOT-INF/
│   ├─ classes/      (your app .class files & resources, preserving package directories)
│   └─ lib/
│       ├─ dependency-a.jar
│       ├─ dependency-b.jar
│       └─ ...


Notes:

Launcher classes (org.springframework.boot.loader.*) are placed so the JVM can load them as the jar’s entry point.

BOOT-INF/classes and BOOT-INF/lib are the actual runtime classpath roots that the launcher will expose to the application classloader it creates.

4) Runtime: what happens when you run java -jar myapp.jar

JVM startup
The JVM loads the outer jar as the application JAR (because of -jar) and looks at the manifest Main-Class. That points to one of Spring Boot’s launcher classes — for a simple app that’s usually org.springframework.boot.loader.JarLauncher.

Launcher class is loaded
Because the launcher classes are at the top level of the JAR (not inside BOOT-INF), the system/application classloader can load JarLauncher directly.

JarLauncher reads the archive
JarLauncher (or its base Launcher) opens the currently executing JAR as an archive and inspects it. It looks for:

BOOT-INF/classes/ (application classes)

BOOT-INF/lib/*.jar (dependencies)

Create a custom classloader
The launcher builds a LaunchedURLClassLoader (a Spring Boot custom subclass of URLClassLoader), supplying it with a list of URLs / classpath entries that represent:

BOOT-INF/classes (logical URL pointing to classes inside the outer JAR)

each nested JAR under BOOT-INF/lib (logical URLs / handlers that let the classloader read those nested JARs)

This custom classloader knows how to load classes/resources from nested JAR entries (more on how below).

Set thread context classloader
The launcher sets the new classloader as the thread context classloader so libraries using the TCCL (ServiceLoader, resource lookups, etc.) will see the application classloader.

Find Start-Class and invoke main
The launcher uses the Start-Class value from the manifest (or discovers the main class) and uses the custom classloader to load that class and invoke its main(String[] args).

Application runs inside the custom classloader
All your application classes and dependencies are loaded by the LaunchedURLClassLoader from entries inside the outer JAR.

5) How does Spring Boot actually read nested JARs? (implementation notes & tradeoffs)

URLClassLoader can easily load classes from directories or normal JARs on the file system. It cannot, out of the box, treat a JAR inside another JAR as a simple classpath entry. Spring Boot’s loader solves this:

Custom archive/jar abstraction — Spring Boot provides classes in org.springframework.boot.loader.archive.* and org.springframework.boot.loader.jar.* that represent the outer jar and the nested entries as “archives”.

Virtual URLs / handlers — For each nested JAR (an entry inside the outer JAR), Spring Boot creates a URL that the LaunchedURLClassLoader can use. Internally a special URLStreamHandler / resource accessor is used to map that URL to the nested jar content.

Random access to nested JAR entries — Spring Boot implements logic to open the outer JAR (a normal JarFile) and then read nested JARs as streams or virtual archives. In older versions or edge cases (e.g., Windows/locking), Spring Boot might extract a nested JAR to a temporary file to provide a true JarFile for random access; in newer versions it does more in-memory / streaming handling where possible.

No recursive default support in JVM — the core reason for all this is that the JVM’s standard classloaders and JarURLConnection do not provide a stable, portable way to treat a JAR inside a JAR as a classpath element — so Spring Boot implements the required plumbing itself.

Because of this, Spring Boot does not rely on standard URLClassLoader semantics alone; it uses its LaunchedURLClassLoader + archive abstraction to expose nested jars as classpath entries.

6) Launcher types

Spring Boot provides multiple launchers depending on packaging needs:

JarLauncher — default for executable JARs.

WarLauncher — for executable WARs (it reads WEB-INF/classes and WEB-INF/lib).

PropertiesLauncher — a flexible launcher that can read loader.path or a custom loader.properties to load external directories/jars at runtime (useful for more dynamic classpaths).

Others exist or can be extended by advanced users.

The Main-Class in the manifest is set to the chosen launcher. Start-Class points to your application entry.

7) What the plugin/loader also takes care of

Manifest attributes — it sets Main-Class and Start-Class (so the launcher can find your app main).

Resource merging — many META-INF/* resources could be present in multiple deps; the repackager can merge service files sensibly (e.g. spring.factories, META-INF/spring.handlers).

Layered jars — newer Spring Boot supports layered JAR layout to enable efficient Docker layering (separate dependencies, resources, classes into layers).

Class-path isolation — the app gets its own classloader that isolates app classes from loader classes.

8) Pseudo-code / control flow (simplified)
java -jar myapp.jar
  -> JVM loads Main-Class org.springframework.boot.loader.JarLauncher (from top-level of myapp.jar)
  -> JarLauncher:
       archive = new JarFileArchive(currentJarFile)
       ClassLoader appClassLoader = createClassLoader(archive)
           // createClassLoader:
           //   find entries: BOOT-INF/classes + BOOT-INF/lib/*.jar
           //   for each nested jar create a virtual URL / archive entry
           //   return new LaunchedURLClassLoader(urls, parent)
       Thread.currentThread().setContextClassLoader(appClassLoader)
       mainClass = manifest.get("Start-Class")
       Class.forName(mainClass, true, appClassLoader).getMethod("main", String[].class).invoke(...)

9) Why Spring Boot chose nested JARs (pros & cons)

Pros

Keeps dependency JARs intact (no class file merging).

Easier to repackage / maintain original JAR metadata.

Smaller changes to source artifacts, fewer conflicts.

Cons

JVM can’t natively treat nested JARs as classpath elements → needs a custom loader.

Slight startup overhead: scanning outer JAR, enumerating nested jars.

Some platform-specific workarounds (temporary extraction) may be needed in older platforms.

Alternative approach (Maven Shade) flattens classes into a single JAR so URLClassLoader can handle it, but that can cause resource collisions and needs special merging of service files and META-INF.

10) Extra features you might see/use

loader.path with PropertiesLauncher — allows the runtime to add folders/jars outside the executable to the classpath. Useful for “deployables” where you drop extra jars next to the executable jar.

Layered jars — for optimized Docker builds (dependencies go into separate layers).

Fat JAR vs nested JAR — Spring Boot is a nested-JAR approach by default (with loader support), not a flattened fat JAR.

TL;DR (short summary)

The Spring Boot Maven/Gradle plugin repackages your app into a JAR with BOOT-INF/classes (your classes) and BOOT-INF/lib (dependency JARs), and it includes Spring Boot launcher classes at the root.

The manifest points Main-Class at the launcher (org.springframework.boot.loader.JarLauncher) and sets Start-Class to your application.

At runtime the launcher is loaded by the JVM, inspects the outer JAR, builds a custom classloader (LaunchedURLClassLoader) that knows how to read nested JARs, sets it as the app classloader, then invokes your app’s main().
