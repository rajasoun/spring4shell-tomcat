FROM rajasoun/tomcat-manager:1.0
COPY target/spring4shell.war /usr/local/tomcat/webapps/spring4shell.war
RUN apt-get update -y 
RUN apt-get install vim -y 
CMD ["catalina.sh", "run"]