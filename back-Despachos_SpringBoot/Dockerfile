FROM maven:3.9.6-eclipse-temurin-17-alpine AS build
WORKDIR /app

# ➡️ CORREGIDO: Le decimos a Docker que busque dentro de la subcarpeta
COPY ./Springboot-API-REST-DESPACHO/pom.xml .
RUN mvn dependency:go-offline -B
COPY ./Springboot-API-REST-DESPACHO/src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8085
ENTRYPOINT ["java", "-jar", "-Dserver.port=8085", "app.jar"]