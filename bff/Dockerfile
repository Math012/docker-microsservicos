FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY target/bff-0.0.1-SNAPSHOT.jar /app/bff.jar

EXPOSE 8083

CMD ["java", "-jar", "/app/bff.jar"]


