# Build stage
FROM maven:3.9.6-eclipse-temurin-17 AS builder
WORKDIR /app

COPY pom.xml .
COPY common-library/pom.xml common-library/
COPY auth-service/pom.xml auth-service/
COPY user-service/pom.xml user-service/
COPY admin-service/pom.xml admin-service/
COPY employee-service/pom.xml employee-service/
COPY customer-service/pom.xml customer-service/
COPY hr-service/pom.xml hr-service/
COPY task-service/pom.xml task-service/
COPY gateway-service/pom.xml gateway-service/

COPY common-library/src common-library/src
COPY employee-service/src employee-service/src

RUN mvn clean package -pl employee-service -am -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

RUN addgroup -S crm && adduser -S crm -G crm
USER crm

COPY --from=builder /app/employee-service/target/employee-service-*.jar app.jar

EXPOSE 8083
ENTRYPOINT ["java", "-jar", "app.jar"]
