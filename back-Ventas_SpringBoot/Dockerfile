FROM maven:3.9.6-eclipse-temurin-17-alpine AS build
WORKDIR /app

# Buscamos el código bajando un nivel hacia la subcarpeta interna de Ventas
COPY ./back-Ventas_SpringBoot/Springboot-API-REST/pom.xml .
RUN mvn dependency:go-offline -B
COPY ./back-Ventas_SpringBoot/Springboot-API-REST/src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8086
ENTRYPOINT ["java", "-jar", "-Dserver.port=8086", "app.jar"]