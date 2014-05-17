OkHttp requires Java 7 to build and run tests. Runtime compatibility with Java 6 is enforced as part of the build to ensure compliance with Android and older versions of the JVM.

### Desktop Testing

Run OkHttp tests on the desktop with Maven. Running HTTP/2 and SPDY tests on the desktop uses [Jetty-NPN][1] when running OpenJDK 7 or [Jetty-ALPN][2] when OpenJDK 8.

```
mvn clean test
```

### Device Testing

OkHttp's test suite creates an in-process HTTPS server. Prior to Android 2.3, SSL server sockets were broken, and so HTTPS tests will time out when run on such devices.

Test on a USB-attached Android using [Vogar][3]. Unfortunately `dx` requires that you build with Java 6, otherwise the test class will be silently omitted from the `.dex` file.

```
mvn clean
mvn package -DskipTests
vogar \
    --classpath ~/.m2/repository/org/bouncycastle/bcprov-jdk15on/1.48/bcprov-jdk15on-1.48.jar \
    --classpath mockwebserver/target/mockwebserver-2.0.0-SNAPSHOT.jar \
    --classpath okhttp-protocols/target/okhttp-protocols-2.0.0-SNAPSHOT.jar \
    --classpath okhttp/target/okhttp-2.0.0-SNAPSHOT.jar \
    okhttp/src/test
```

 [1]: https://github.com/jetty-project/jetty-npn
 [2]: https://github.com/jetty-project/jetty-alpn
 [3]: https://code.google.com/p/vogar/
