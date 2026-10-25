FROM maven:3.9.15-eclipse-temurin-17-alpine AS maven

WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package


FROM alpine/java:17

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN chown appuser:appgroup /app

COPY --from=maven /app/target/demo-0.0.1-SNAPSHOT.jar app.jar

USER appuser

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
