FROM maven:3.9.6-eclipse-temurin-17-alpine AS build
WORKDIR /app
# Apuntamos a la carpeta del proyecto desde la raíz
COPY ./back-Despachos_SpringBoot/pom.xml .
RUN mvn dependency:go-offline -B
COPY ./back-Despachos_SpringBoot/src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8085
ENTRYPOINT ["java", "-jar", "-Dserver.port=8085", "app.jar"]