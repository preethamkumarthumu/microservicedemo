# Build stage
FROM maven:3.9.6-eclipse-temurin-17 AS builder

WORKDIR /app

# Copy root POM
COPY pom.xml .

# Copy POMs for current Maven modules
COPY common-library/pom.xml common-library/
COPY auth-service/pom.xml auth-service/
COPY user-service/pom.xml user-service/
COPY admin-service/pom.xml admin-service/
COPY employee-service/pom.xml employee-service/
COPY customer-service/pom.xml customer-service/
COPY hr-service/pom.xml hr-service/
COPY task-service/pom.xml task-service/
COPY gateway-service/pom.xml gateway-service/

# Copy common library source
COPY common-library/src common-library/src

# Copy service source
COPY customer-service/src customer-service/src

# Build this service and required dependencies
RUN mvn clean package -pl customer-service -am -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -S crm && adduser -S crm -G crm
USER crm

COPY --from=builder /app/customer-service/target/customer-service-*.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
