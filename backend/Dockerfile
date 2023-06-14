FROM tomcat:latest

COPY ./target/backend-0.0.1-SNAPSHOT.war /usr/local/tomcat/webapps/ROOT.war

RUN rm -rf /usr/local/tomcat/webapps/ROOT

EXPOSE 8080

CMD ["catalina.sh", "run"]