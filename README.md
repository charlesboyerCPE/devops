# devops
Module "DevOps"

---

Ce dépot contient tous les TP du module DevOps

---

TP1 : Docker

## BOYER Charles

---

## Database

---

Fichier Dockerfile :

```docker

    FROM postgres:11.6-alpine
    ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

```

### Lancement des containers

sudo docker build -t charlesboyercpe/postgredb .

sudo docker network create net
sudo docker run -d --name postgredb --network net charlesboyercpe/postgredb
sudo docker run -d -p 8080:8080 --name db_view --network net adminer

### Lancement des scripts à l'init

- Ajout dans le Dockerfile :
COPY sql/01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY sql/02-InsertData.sql /docker-entrypoint-initdb.d/

- Arrêt des containers et rebuild :
sudo docker rm -f adminer
sudo docker rm -f postgredb

sudo docker build -t charlesboyercpe/postgredb .
sudo docker run -d --name postgredb --network net charlesboyercpe/postgredb
sudo docker run -d -p 8081:8081 --name db_view --network net adminer

### Persistence des données

sudo docker run -d --name postgredb -v /data:/var/lib/postgresql/data --network net charlesboyercpe/postgredb

Les commandes les plus importantes sont :

- docker build : Pour construire notre image docker
  
- docker run : Pour lancer notre image dans un container

Nous avons aussi besoin d'un fichier Dockerfile, essentiel pour construire notre image

---

## Backend API

---

Sur ma machine, lancement de la commande : javac Main.java

Contenu Dockerfile :

```docker
    FROM openjdk:11

    COPY java/Main.class /

    CMD cd / ; java Main
```

### Build et lancement

sudo docker build -t charlesboyercpe/backend .
sudo docker run --name backend charlesboyercpe/backend

### Simple API

Contenu Dockerfile :

```docker
    # Build
    FROM maven:3.6.3-jdk-11 AS myapp-build
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY simple-api/pom.xml .
    COPY simple-api/src ./src
    RUN mvn package -DskipTests

    # Run
    FROM openjdk:11-jre
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
    ENTRYPOINT java -jar myapp.jar
```

sudo docker build -t charlesboyercpe/simple-api .
sudo docker run -d --name simple-api -p 8080:8080 charlesboyercpe/simple-api

### Simple API-2

Contenu Dockerfile :

```docker
    # Build
    FROM maven:3.6.3-jdk-11 AS myapp-build
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY simple-api-2/pom.xml .
    COPY simple-api-2/src ./src
    RUN mvn package -DskipTests

    # Run
    FROM openjdk:11-jre
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
    ENTRYPOINT java -jar myapp.jar
```

Contenu application.yml

```yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://postgredb:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops
```

sudo docker rm -f simple-api
sudo docker build -t charlesboyercpe/simple-api-2 .
sudo docker run --name simple-api-2 -p 8080:8080 --network net charlesboyercpe/simple-api-2

---

## HTTP Server

---

Récupération du container
docker pull httpd

Configuration :
sudo docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > my-httpd.conf

Contenu Dockerfile :

```docker
    FROM httpd:2.4
    COPY ./my-httpd.conf /usr/local/apache2/conf/httpd.conf
```

Build et lancement :
sudo docker build -t charlesboyercpe/httpd .
sudo docker run -d --name httpd -p 8082:80 charlesboyercpe/httpd


### Reverse Proxy

Ajout dans le my-httpd.conf :

```conf
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

ServerName localhost
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://simple-api-2:8080/
    ProxyPassReverse / http://simple-api-2:8080/
</VirtualHost>
```

Build et lancement :
sudo docker build -t charlesboyercpe/httpd .
sudo docker run -d --name httpd -p 80:80 --network net charlesboyercpe/httpd

---

## Link Application

---

### Docker Compose

Contenu docker-compose.yml :

```docker
version: '3.7'
services:
  backend: # Service Backend
    build:
      ./Backend_API/simple-api-2
    networks:
      - my-network
    depends_on:
      - database # Définition de la priorite de lancement

  database:
    build:
      ./Database # Definition du repertoire vers le dockerfile
    networks:
      - my-network

  httpd:
    build:
      ./Http_Server
    ports:
      - 80:80 # Definition des ports
    networks:
      - my-network
    depends_on:
      - backend

networks:
  my-network: # Creation de notre network
```

Commandes réalisées :

Lancer le docker-compose :
sudo docker-compose up --build

Eteindre les containers :
sudo docker-compose down

### Publication

Nous pushons sur le repository en ligne pour avoir une sauvegarde. De plus nous pouvons maintenant le modifier sereinement pendant qu'une version fonctionnelle est en production

TP2 : GIT CI_CD

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

TP3 : Ansible

---

## Intro

---
Fichiers de base :

- /etc/ansible/hosts : Fichier permettant de spécifier les hosts
- inventories/setup.yml : Fichier permettant de créer un inventaire

Commande de bases :

- "--become" permet de devenir super-user
- ansible all -m -ping : Permet de pinger toutes les machines
- ansible all -m yum -a “name=<SERVICE>” --private-key=<path_to_your_ssh_key> -u centos : Permet d'installer un services sur une machine
- ansible all --list-hosts : Permet de lister tous les hosts présents
- ansible all -i inventories/setup.yml -m ping : Permet lancer le test de l'inventaire en pingant la machine esclave

---

## Playbooks

---

### First playbook

Contenue playbook.yml :

```yaml
- hosts: all
  gather_facts: false
  become: yes

  tasks:
    - name: Test connection
      ping:
```

Commandes réalisés :

- --syntax-check : "Permet de checker la syntaxe"
- ansible-playbook -i inventories/setup.yml playbook.yml : Permet de lancer un playbook

### Advanced playbook

```yaml
- hosts: all
  gather_facts: false
  become: yes

# Install Docker
  tasks:
  - name: Clean packages
    command:
      cmd: dnf clean -y packages

  - name: Install device-mapper-persistent-data
    dnf:
      name: device-mapper-persistent-data
      state: latest
  
  - name: Install lvm2
    dnf:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    dnf:
      name: docker-ce
      state: present

  - name: Install Python3
    dnf:
      name: python3

  - name: Pip Install
    pip:
      name: docker

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```

Commandes réalisés :

- ansible-playbook -i inventories/setup.yml playbook.yml : Permet de lancer un playbook
- ansible-galaxy init roles/docker : Permet de récupérer le module : rôle Docker

---

## Deploy your app

---

Création des différents rôles :

- ansible-galaxy init roles/proxy
- ansible-galaxy init roles/app
- ansible-galaxy init roles/network
- ansible-galaxy init roles/database

Playbook.yml final :

```yaml
- hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```
