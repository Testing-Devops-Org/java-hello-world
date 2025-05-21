# Use a slim OpenJDK base image for the runtime environment.
# This provides Java 17 for your application.
FROM openjdk:17-jdk-slim

# Set the working directory inside the Docker container.
# This is where your application's files will reside.
WORKDIR /app

# Copy the compiled JAR file from the Maven build output.
# The 'target/' directory is where Maven puts the JAR.
# 'java-hello-world-1.0-SNAPSHOT.jar' is the name Maven gives it based on your pom.xml.
# We rename it to 'app.jar' for simplicity inside the container.
COPY target/java-hello-world-1.0-SNAPSHOT.jar app.jar

# Expose the port your Java application will listen on (if applicable).
# Even for a simple console app, if you ever add web capabilities (like Spring Boot),
# this is where you'd specify the port. Jenkins will use this for validation.
EXPOSE 8080

# Define the command to run your Java application when the container starts.
# This tells Docker to execute the JAR file.
CMD ["java", "-jar", "app.jar"]