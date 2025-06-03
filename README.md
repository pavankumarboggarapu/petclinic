# Build Pipeline for Petclinic
The goal of this exercise is to utilize GitHub Actions and JFrog  As part of the exercise, we will:
- Use Spring pet-clinic (https://github.com/spring-projects/spring-petclinic) as your project source code
- Build a Github Actions pipeline with the following steps:
- Compile the code
- Run the tests
- Get a free trial instance of the JFrog SaaS environment for 14 days(https://jfrog.com/start/), using this JFrog instance to finish the below task.
- Package the project as a runnable Docker image
- Publish the image to JFrog Artifactory in your pipeline
- Make sure all dependencies are resolved from Maven Central

This repository contains everything you need to complete the lab except for the three prerequisites listed below.

# Prerequisites

1. [signup](https://github.com/signup) for a free GitHub Account
2. [create](https://pavankumarb.jfrog.io/ui/admin/configuration/security/access_tokens) a JFrog Access Token
3. Create Docker file 

# Clone repository

1. Clone this repository into your GitHub account. _GitHub → New → Import a Repository
   - enter https://github.com/spring-projects/spring-petclinic
   - enter repository name, e.g. petclinic
   - leave as public 

# Setup workflow

1. Confirm all GitHub Actions are allowed. _GitHub → Project → Settings → Actions → General → Actions Permissions_
2. Add the following variables, adding JF_ACCESS_TOKEN as a **secret**. _GitHub → Project → Settings → Secrets and Variables → Actions_
   - JF_URL
   - JF_ACCESS_TOKEN
4. Add a Dockerfile to the project repository. _GitHub → Project → Add file → Create new file_

```
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

5. From the JFrog UI, [create a reposiory for docker]
6. Create a new workflow. _GitHub → Project → Actions → New Workflow → Setup a workflow yourself_ 

```
name: JF

on:
  push:
    branches: [main, master, develop, stage]
  pull_request:
    branches: [main, master, develop, stage]
  workflow_dispatch:
jobs:
  jf:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Setup Java JDK
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: temurin
          cache: maven

      - name: Build with Maven
        run: mvn package -DskipTests

      - name: Test with Maven
        run: mvn test

      - name: Setup JFrog CLI
        uses: jfrog/setup-jfrog-cli@v4
        env:
          JF_URL: ${{ vars.JF_URL }}
          JF_ACCESS_TOKEN: ${{ secrets.JF_ACCESS_TOKEN }}

      - name: Build Tag and Push Docker Image
        env:
          IMAGE_NAME: pavankumarb.jfrog.io/petclinic-docker/petclinic:${{ github.run_number }}
        run: |
          jf docker build -t $IMAGE_NAME .
          jf docker push $IMAGE_NAME
          jf docker scan
      - name: Publish Build info With JFrog CLI
        env:
          # Generated and maintained by GitHub
          JFROG_CLI_BUILD_NAME: jfrog-docker-build-example
          # JFrog organization secret
          JFROG_CLI_BUILD_NUMBER : ${{ github.run_number }}
        run: |
          # Export the build name and build nuber
          # Collect environment variables for the build
          jf rt build-collect-env
          # Collect VCS details from git and add them to the build
          jf rt build-add-git
          # Publish build info
          jf rt build-publish
```

