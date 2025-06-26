FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/notificacao-api-0.0.1-SNAPSHOT.jar /app/notificacao-api.jar

EXPOSE 8082

CMD ["java", "-jar", "/app/notificacao-api.jar"]


