# TP1 : Docker

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

Lancer le docker-compose
sudo docker-compose up --build

Eteindre les containers
sudo docker-compose down

Nous pushons sur le service en ligne pour avoir une sauvegarde. De plus nous pouvons maintenant le modifier sereinement pendant qu'une version fonctionnelle est en production