# docker-compose notes

## Running separated

### Create network
```bash
docker network create mongo-network
```

### List Docker networks
```bash
docker network ls
```

### Run Docker project (Mongo)
```bash
docker run -d \
-p 27017:27017 \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD=supersecret \
--network mongo-network \
--name mongodb
mongo
```

### Run Docker project (Mongo Express)
```bash
docker run -d \
-p 8081:8081 \
-e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
-e ME_CONFIG_MONGODB_ADMINPASSWORD=supersecret \
-e ME_CONFIG_MONGODB_SERVER=mongodb \
--network mongo-network \
--name mongo-express
mongo-express
```

### List Docker processes
```bash
docker ps
```

### Stop Docker processes
```bash
docker stop <process_id_1> <process_id_2>
```

### Remove Docker processes
```bash
docker rm <process_id_1> <process_id_2>
```

### Remove Docker network
```bash
docker network rm mongo-network
```

## Compose File

Above containers will be defined in a single YAML file - `compose.yaml`

### Header
Version of docker-compose
```bash
version: '3.1'
```

### Services
container name
ports - host:container
```bash
services:
   mongodb:
      image: mongo
      ports:
        - 27017:27017
      environment:
        - MONGO_INITDB_ROOT_USERNAME=admin
        - MONGO_INITDB_ROOT_PASSWORD=supersecret
```

### Network
* By default: Single network for your app
* Communication via container name
* Use `networks` key to specify network options

## Start docker-compose
```bash
docker-compose -f mongo-services.yaml up
```
* Network name(s) is printed.
* Printed container names are prefixed with project and suffixed with index.
* Logs output is aggregated

### Set project name
```bash
docker-compose --project-name projects -f mongo-services.yaml up
```

### Set starting dependency
```bash
      depends_on:
        - "mongodb"
```

### Use Dockerfile
Path is `.`
```bash
services:
   myapp:
      build: .
```

## Commands
### Stop
* Ctrl + C
* `docker stop ...`
* Graceful stop
```bash
docker-compose -f mongo-services.yaml stop
```

### Restart stopped
```bash
docker-compose -f mongo-services.yaml start
```

### Start detached
```bash
docker-compose -f mongo-services.yaml up -d
```

### Complete stop 
Removes containers, data and networks
```bash
docker-compose -f mongo-services.yaml down
```

## Variables
* DO NOT hardcode sensitive data
* Define variables
```bash
      environment:
        - MONGO_INITDB_ROOT_USERNAME:${MONGO_ADMIN_USER}
        - MONGO_INITDB_ROOT_PASSWoRD:${MONGO_ADMIN_PASS}
```
* Use these variables in file
* Can be set as env variables - still risk to security
```bash
export MONGO_ADMIN_USER=admin
```
* Use secrets - set as separate section
```bash
services:
   my_app:
      secrets:
         - my_secret
secrets:
   my_secret:
      file: ./my_secret.txt
```


## Reference
* [Ultimate Docker Compose Tutorial](https://www.youtube.com/watch?v=SXwC9fSwct8)
* [Git repo](https://gitlab.com/twn-youtube/docker-compose-crash-course)
* [Docker Installation](https://docs.docker.com/get-docker/)
* [Using Secrets in Compose](https://docs.docker.com/compose/use-secrets/)
* <https://hub.docker.com/_/mongo>
* <https://hub.docker.com/_/mongo-express>