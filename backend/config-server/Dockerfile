# syntax=docker/dockerfile:1

FROM maven:3.9.16-amazoncorretto-25 AS build

WORKDIR /workspace

COPY pom.xml ./

RUN mvn --batch-mode --no-transfer-progress dependency:go-offline

COPY src/ src/

RUN mvn --batch-mode --no-transfer-progress verify


FROM amazoncorretto:25 AS runtime

WORKDIR /app

ENV HOME=/home/config-server

RUN mkdir -p "$HOME" \
    && chown 10001:10001 "$HOME"

COPY --from=build /workspace/target/*.jar config-server.jar

EXPOSE 8080

USER 10001:10001

ENTRYPOINT ["java", "-jar", "config-server.jar"]