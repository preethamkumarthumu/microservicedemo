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

# Copy common-library source because gateway may depend on it
COPY common-library/src common-library/src

# Copy gateway source
COPY gateway-service/src gateway-service/src

# Build gateway and required dependencies
RUN mvn clean package -pl gateway-service -am -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Run as non-root user
RUN addgroup -S crm && adduser -S crm -G crm
USER crm

COPY --from=builder /app/gateway-service/target/gateway-service-*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
