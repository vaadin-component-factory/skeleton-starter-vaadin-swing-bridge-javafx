# Vaadin SwingBridge: JavaFX starter

## Overview
This minimal skeleton shows how to use the **JavaFX Extension for Vaadin SwingBridge** to run an existing JavaFX application inside a **Vaadin** application, so that end users can reach it through the web in no time.

It runs **WarehousePro**, a sample JavaFX inventory application, whose jar is in `applibs/`. Every browser session gets its own instance of the application. Sign in with `admin / admin` or `manager / manager`.

This project uses Maven for managing the dependencies and configurations.
Similar to any other Vaadin project, you can run it from the command line with `mvn clean spring-boot:run`.
For production builds, see the [Production Build](#production-build) section.

To keep the initial setup simple, the bridge assumes a few things are in place, mostly where the JavaFX application's jar file(s) are and where JavaFX itself comes from.
Please read [Important Directories](#important-directories) for a smooth experience. :+1:

## Important Directories

### applibs
This is the directory that is automatically scanned for the jar files of the JavaFX application.
JavaFX applications are packaged in different ways, from one fat jar to a light main jar with its dependencies next to it.
In any case, all the jar files should be in this directory so that they are available to the bridge.
**Do not put JavaFX itself (the `javafx-*` jars) here**: the bridge provides JavaFX, see below.

> **_NOTE:_** The `applibs` name of this directory is important and case-sensitive.
If you wish to change it, you can pass the path to a directory that contains all the jar files with `-Dapplibs.dir` on the command line.
It is also possible to configure this in the `pom.xml`. There is a commented placeholder for this in the `<systemPropertyVariables>` section of the `pom.xml`.

### SwingBridge
The JavaFX extension builds on Vaadin SwingBridge, so `pom.xml` declares both: `com.vaadin:swing-bridge-flow` (SwingBridge, version `swing-bridge.version`) and `org.vaadin.addons.componentfactory:swing-bridge-javafx-flow` (the extension, version `swing-bridge-javafx.version`). SwingBridge is a commercial product and needs a SwingBridge license.

Keep `swing-bridge.version` at the version the extension release was built against (1.4.0-alpha1 for this release): the extension uses SwingBridge internals that can change between versions.

### Repositories
The [JavaFX extension](https://github.com/vaadin-component-factory/swing-bridge-javafx-addon) is published through the Vaadin Directory, to Vaadin's add-on repository. SwingBridge 1.4.0-alpha1 is a pre-release, from Vaadin's pre-release repository. `pom.xml` declares both under `<repositories>`; everything else comes from Maven Central.

Unlike the Swing starter, no `java.desktop` patch and no SwingBridge jars on the boot class path are needed; JavaFX does not render through AWT.

### target/javafx-modules and target/javafx-bridge
The build (`process-resources` phase, so any `mvn spring-boot:run` or `mvn package`) copies the JavaFX jars for your operating system to `target/javafx-modules`, and unpacks the bridge's two JVM-level jars from the extension's jar to `target/javafx-bridge`. The JVM flags point at these directories.

JavaFX is not bundled into the application: it has to be on the JVM's **module path**, because the bridge patches the `javafx.graphics` module. That is also why it is in `provided` scope and excluded from the Spring Boot jar.

## Prerequisites
- **JDK 21 or later.**
- **OpenJFX** comes from Maven Central; nothing to install. The version is the `openjfx.version` property in `pom.xml`: 21.x works on JDK 21 and newer, while OpenJFX 23 and newer need JDK 23 or newer.
- **Build on the platform you run on.** The OpenJFX jars are platform-specific (they contain native libraries), and a Maven profile picks them for the operating system the build runs on. To build for another platform, pass `-Djavafx.platform=linux` (or `linux-aarch64`, `mac`, `mac-aarch64`, `win`).
- **No native windows.** The bridge renders JavaFX into the browser with JavaFX's software renderer (`-Dprism.order=sw`) and its own windowing platform (`-Dglass.platform=Vaadin`); the application never opens a window on the server.

Maven itself does not need to be installed: the project ships the Maven wrapper, so `./mvnw` (or `mvnw.cmd` on Windows) works out of the box. The commands below use `mvn`; substitute `./mvnw` if you prefer the wrapper.

## How to Run
The `pom.xml` is configured with the necessary command-line arguments and configurations, so you can run it in development mode simply by running:
```
mvn clean spring-boot:run
```

Then open http://localhost:8888. To attach a debugger, connect your IDE to `localhost:5005`.

## Embedding a JavaFX application in Vaadin
The whole integration is one component, `JavaFXBridge`, which you add to any Vaadin view like any other component. This starter's only view, `src/main/java/com/example/javafxbridge/ui/WarehouseView.java`, does exactly that:

```java
@Route(value = "")
public class WarehouseView extends VerticalLayout {
    private static final String MAIN_CLASS = "com.example.warehouse.fx.WarehouseApp";

    public WarehouseView() {
        setSizeFull();
        JavaFXBridge app = new JavaFXBridge(MAIN_CLASS);
        add(app);
        expand(app);
    }
}
```

The bridge loads the application's classes from `applibs/`, launches the `javafx.application.Application` subclass named by `MAIN_CLASS` for the user's session, and renders its primary stage into the component. Because it is an ordinary component, you can put it in any layout: next to a Vaadin menu, inside an `AppLayout`, on one route or several.

To run your own JavaFX application instead of WarehousePro:
1. Replace the jar in `applibs/` with your application's jar(s), without JavaFX.
2. Set `MAIN_CLASS` in `WarehouseView.java` to your `javafx.application.Application` subclass.
3. If your application uses JavaFX modules beyond `javafx.controls` (for example `javafx.fxml` or `javafx.web`), add them as `provided` dependencies in `pom.xml` and to `--add-modules` in the JVM flags.

Keep in mind that every user's application runs in the same JVM:
- Never call `Platform.exit()`: it stops JavaFX for every user. Close the window instead.
- The primary stage fills the page in the browser; any other stage opens as a dialog.

## Interoperability between Vaadin and the JavaFX application
Embedding the whole application is the first step. Interoperability lets you migrate gradually after that: native Vaadin views and the JavaFX application call each other, so you can rebuild one screen at a time in Vaadin while the rest stays in JavaFX. It uses the same annotations and code generator as SwingBridge:
- Vaadin calls methods that the application marks with `@ExposedMethod`. These calls run on the JavaFX Application Thread, in the user's session.
- The application's listeners call view methods marked with `@VaadinCallback`.

**1. In the JavaFX application**, mark the methods to expose, and give the bridge a way to reach the running instance. You can use a static factory marked `@InstanceProvider` (or a static `getInstance()`), or expose only static methods. Each user gets their own copy of the application's classes, so a static field set in `start()` is per user:

```java
public class CustomersApp extends Application {
    private static CustomersApp current;

    @InstanceProvider
    public static CustomersApp current() { return current; }

    @Override
    public void start(Stage stage) { current = this; /* ... */ }

    @ExposedMethod
    public void showCustomer(String id) { /* ... */ }

    @ExposedMethod(invocation = Invocation.ASYNC)   // CompletableFuture<Integer> in Vaadin
    public int customerCount() { /* ... */ }

    @ExposedMethod   // returns a remover, run when the view unregisters
    public Runnable addSelectionListener(CustomerSelectionListener listener) { /* ... */ }
}
```

The annotations come from `com.vaadin:swing-bridge-annotations`. Declare it with `provided` scope in the JavaFX application: the Vaadin application already has it.

**2. In this project's `pom.xml`**, add SwingBridge's code generator. It reads the jars in `applibs/` and generates a typed interface for each annotated class (for example `CustomersAppBridge`) into `target/generated-sources`:

```xml
<plugin>
  <groupId>com.vaadin</groupId>
  <artifactId>swing-bridge-codegen-maven-plugin</artifactId>
  <version>${swing-bridge.version}</version>
  <configuration>
    <applibsDir>${project.basedir}/applibs</applibsDir>
  </configuration>
  <executions>
    <execution>
      <goals><goal>generate-bridge</goal></goals>
    </execution>
  </executions>
</plugin>
```

**3. In a Vaadin view**, get a handle and call the application through it. Calls made before the application has started wait until it has:

```java
JavaFXBridgeHandle<CustomersAppBridge> app = JavaFXBridge
        .interop("com.example.CustomersApp").of(CustomersAppBridge.class);

app.onReady(bridge -> bridge.showCustomer(id));
app.requestAsync(CustomersAppBridge::customerCount)
        .thenAccept(n -> ui.access(() -> count.setText("" + n)));

@VaadinCallback(observerFor = CustomerSelectionListener.class)
public void customerSelected(CustomerSelectedEvent event) { /* ... */ }

// onAttach: registration = app.registerCallback(this);  onDetach: registration.remove();
```

Things to know:
- **Shared types.** Listener interfaces and event classes are used on both sides, so both sides must load the same class. Put them in a small jar on this project's class path, not only in `applibs/`. JDK types, such as `java.util.function.Consumer<String>`, need nothing extra.
- **One JavaFX Application Thread for all users.** Use `Invocation.ASYNC` for anything slow. Keep the callback's default `Dispatch.ACCESS`: `ACCESS_SYNCHRONOUSLY` makes the JavaFX thread wait for the user's Vaadin session.
- **Several views on one application.** Within one user's session, every `JavaFXBridge` for the same application class shares one running instance. So you can have one route per module of the application, switch modules through interop, and the user keeps their state when navigating. To show a stage other than the main window, override `stageResolver()` in a `JavaFXBridge` subclass.

## Configuring the bridge
The bridge reads its optional settings from JVM system properties. The `<systemPropertyVariables>` section of the `pom.xml` lists them as commented-out entries with a one-line explanation each: screen size, HiDPI rendering, keyboard focus, frame delivery, and the launch-failure error view. Uncomment what you need.

`swingbridge.screenSize`, `swingbridge.hidpi` and `swingbridge.errorReporting.enabled` work as in SwingBridge; see also the [SwingBridge documentation](https://vaadin.com/docs/latest/tools/modernization-toolkit/swing-bridge/configuration).

## Production Build
Similar to any Spring Boot Vaadin application, you can build it with:
```
mvn clean package
```

The result is a Spring Boot fat jar in `target/`, next to the `target/javafx-modules` and `target/javafx-bridge` directories the build also creates. Run it from the project directory with the commands below; to deploy elsewhere, copy the jar, both directories and `applibs` together and adjust the paths.

> **_NOTE:_** These flags are the same set the `pom.xml` passes to `spring-boot:run`, minus the debug agent. Keep the two in sync if you change one.

**Linux/macOS:**
```bash
java \
  --module-path target/javafx-modules \
  --add-modules javafx.controls \
  --patch-module javafx.graphics=target/javafx-bridge/swing-bridge-javafx-patch.jar \
  -Xbootclasspath/a:target/javafx-bridge/swing-bridge-javafx-graphics.jar \
  --add-reads javafx.graphics=ALL-UNNAMED,java.logging \
  --add-exports=javafx.graphics/com.sun.glass.ui=ALL-UNNAMED \
  --add-exports=javafx.graphics/com.sun.glass.ui.vaadin=ALL-UNNAMED \
  --add-exports=javafx.graphics/com.sun.glass.events=ALL-UNNAMED \
  --add-exports=javafx.graphics/com.sun.javafx.tk=ALL-UNNAMED \
  --add-exports=javafx.graphics/com.sun.prism=ALL-UNNAMED \
  --enable-native-access=javafx.graphics \
  --add-opens=java.base/java.lang=ALL-UNNAMED \
  -Dglass.platform=Vaadin \
  -Dprism.order=sw \
  -XX:+UseStringDeduplication \
  -Dapplibs.dir=./applibs \
  -jar target/skeleton-starter-vaadin-swing-bridge-javafx-1.0-SNAPSHOT.jar
```

**Windows (Command Prompt):**
```cmd
java ^
  --module-path target\javafx-modules ^
  --add-modules javafx.controls ^
  --patch-module javafx.graphics=target\javafx-bridge\swing-bridge-javafx-patch.jar ^
  -Xbootclasspath/a:target\javafx-bridge\swing-bridge-javafx-graphics.jar ^
  --add-reads javafx.graphics=ALL-UNNAMED,java.logging ^
  --add-exports=javafx.graphics/com.sun.glass.ui=ALL-UNNAMED ^
  --add-exports=javafx.graphics/com.sun.glass.ui.vaadin=ALL-UNNAMED ^
  --add-exports=javafx.graphics/com.sun.glass.events=ALL-UNNAMED ^
  --add-exports=javafx.graphics/com.sun.javafx.tk=ALL-UNNAMED ^
  --add-exports=javafx.graphics/com.sun.prism=ALL-UNNAMED ^
  --enable-native-access=javafx.graphics ^
  --add-opens=java.base/java.lang=ALL-UNNAMED ^
  -Dglass.platform=Vaadin ^
  -Dprism.order=sw ^
  -XX:+UseStringDeduplication ^
  -Dapplibs.dir=.\applibs ^
  -jar target\skeleton-starter-vaadin-swing-bridge-javafx-1.0-SNAPSHOT.jar
```

`-XX:+UseStringDeduplication` is optional but recommended: every user's copy of the application holds many equal strings, which the garbage collector then merges.

## Learn more
- [Vaadin SwingBridge documentation](https://vaadin.com/docs/latest/tools/modernization-toolkit/swing-bridge).
- [Vaadin documentation](https://vaadin.com/docs/latest).
