# TP2 : GIT CI_CD

## BOYER Charles

---

## First steps into the CI world

---

L'objectif de la commande "mvn clean verify" est de supprimer les jar présents afin de les recréer. La fonction "verify" va procéder aux tests d'intégrations pour vérifier la qualité du code et que ces derniers respèctent certains critères.

Les tests unitaires permettent de vérifier et tester une portion d'une application. Quant au tests d'intégrations, ils permettent de vérifier que les composants s'intègrent bien ensemble.

2.1- testContainers est une dépendance que l'on va utiliser dans le projet.

Contenu final main.yml :

```yaml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

env:
  GITHUB_TOKEN: $(secrets.GITHUB_TOKEN)

jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'

      #finally build your app with the latest command
      - name: Build and test with Maven
        working-directory: ./Backend_API/simple-api-2
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=charlesboyerCPE_devops -Dsonar.organization=charlesboyercpe -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{secrets.SONAR_TOKEN }} --file ./pom.xml

        # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
    
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Login to DockerHub
        run: docker login -u ${{secrets.DOCKERHUB_USERNAME}} -p ${{secrets.DOCKERHUB_TOKEN}}

      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./Backend_API/simple-api-2
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/backend
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./Database
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/database
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./Http_Server
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/httpd
          push: ${{ github.ref == 'refs/heads/main' }}
```
