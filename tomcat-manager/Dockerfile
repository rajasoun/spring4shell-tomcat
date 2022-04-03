FROM tomcat:9.0.30-jdk11-openjdk
COPY config/tomcat-users.xml /usr/local/tomcat/conf/tomcat-users.xml
COPY config/webapps /usr/local/tomcat/webapps
CMD ["catalina.sh", "run"]
