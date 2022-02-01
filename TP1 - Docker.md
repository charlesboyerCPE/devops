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
sudo docker run -d -p 8080:8080 --name db_view --network net adminer

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