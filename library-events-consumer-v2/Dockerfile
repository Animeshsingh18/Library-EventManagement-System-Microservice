# Base image with Java runtime
FROM eclipse-temurin:25-jre

# Set working directory inside the container
WORKDIR /app

# Build-time argument for selecting the JAR artifact
ARG JAR_FILE=build/libs/*-SNAPSHOT.jar

# Copy built Spring Boot JAR into the container
COPY ${JAR_FILE} /app/app.jar

# Application port
EXPOSE 8080

# Start the application
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

