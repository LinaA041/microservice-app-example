# Etapa 1: Compilación
FROM maven:3.9.6-eclipse-temurin-8 AS builder

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copiar el código fuente
COPY src ./src

# Compilar y empaquetar la app (sin ejecutar tests)
RUN mvn clean package -DskipTests

# Etapa 2: Imagen final liviana
FROM eclipse-temurin:8-jre-alpine

WORKDIR /app

# Copiar el .jar generado
COPY --from=builder /app/target/users-api-0.0.1-SNAPSHOT.jar /app/users-api.jar

# Variables de entorno
ENV SERVER_PORT=8083 \
    JWT_SECRET=PRFT

EXPOSE 8083

CMD ["java", "-jar", "/app/users-api.jar"]
