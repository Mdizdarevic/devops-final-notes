# intro-to-devops-final-notes

### Log in to a secure container image registry:
podman login registry.redhat.io

### Log out 
podman logout --all 

### Download container image to my host 
podman pull registry.access.redhat.com/ubi9/ubi-minimal:9.5 

### List images
podman images 

### Run container image (to check if it's working)
podman run registry.access.redhat.com/ubi9/ubi-minimal:9.5 echo 'hello red hat'

### Run container image and automatically remove after running 
podman run --rm registry.access.redhat.com/ubi9/ubi-minimal:9.5 echo 'Red Hat'

### Run container image in the background (detached mode)
podman run -d registry.access.redhat.com/ubi9/ubi-minimal:9.5

### Assign name to container
podman run --name my-container-name my-image

### Map public port on my host machine (host first : public port)
podman run -p 8081:8080 registry.access.redhat.com/ubi9/ubi-minimal:9.5

### Map public port on my host machine to a specifi IP
podman run -p 127.0.0.1:8082:8080 registry.access.redhat.com/ubi9/ubi-minimal:9.5

### List port mapping for a container
podman port instance-name

### List ports for all containers
podman port --all

### Add environmental variable
podman run --env KEY='Value' IMAGE

### List all currently active and running containers on the host
podman ps

### List all containers on the system (even if they're not running)
podman ps --all

### Stop a running container 
podman stop my-container

### Stop all running containers
podman stop --all

### Force kill active container
podman kill my-container

### Remove a container
podman rm my-container

### Forcefully remove container
podman rm my-container --force

### Remoe all non-running containers
podman rm --all

### Start process in container
podman exec my-container

### Make it interactive with terminal
podman exec -it my-container /bin/bash

### Copy files in and out of conatiners
podman cp /host/path/file.txt my-container:/container/path/

### Pause/unpause container
podman pause my-container, podman unpause my-container

### Inspect container
podman inspect my-container

### Logging (--tail 20 for num of lines, -f for live log, --since 10m for specific time)
podman logs --tail 20 -f --since 10m my-container

### Create podman network
podman network create my-network

### Block traffic from other networks
podman network create -o isolate=TRUE my-network

### List existing networks
podman network ls 

### Check config data for the network
podman network inspect my-network

### Remove netowrk
podman network rm my-network

### Remove all networks not being used by containers
podman network prune 

### Connect running container to existing network
podman network connect 

### Disconnect a container from a network
podman network disconnect 

### Get IP addrees of container within network
podman inspect app-name --format ‘{{.NetworkSettings.IPAddress}}’ 

### Build an image
podman build --file CONTAINERFILE --tag IMAGETAG

### Push an image
podman push IMAGETAG quay.io/QUAYUSERNAME/IMAGENAME:TAG 

### Inspect image
podman image push IMAGE_REGISTRY_URL

### Remove image
podman image rm REGISTRY/NAMESPACE/IMAGE_NAME:TAG

### Creat volume 
podman volume create VOLUME_NAME

### Import volume
podman volume import VOLUME_NAME ARCHIVE_NAME

### To export a volume
podman volume export VOLUME_NAME --output ARCHIVE_NAME

### Compose file

name: my-app-stack

services:
  web-service:
    container_name: frontend-web
    image: docker.io/library/httpd:latest
    ports:
      - "8080:80"
    networks:
      - app-net
    environment:
      ENV_MODE: production

  db-service:
    container_name: backend-db
    image: registry.access.redhat.com/rhel9/postgresql-13:1
    environment:
      POSTGRESQL_USER: admin
      POSTGRESQL_PASSWORD: redhat
    volumes:
      - db-data:/var/lib/pgsql/data:Z
    networks:
      - app-net

volumes:
  db-data: {}

networks:
  app-net: {}

### Execute compose file
podman-compose up

### Stop and remove containers defined as services 
podman-compose down

### Stop the containers that are defined in the file 
podman-compose stop

### Create a podman secret
echo “secret texrt > SECRET_FILENAME
podman secret create SECRET_NAME SECRET_FILENAME 

### Specify driver
podman secret create SECRET_NAME --driver pass 

### Delete secret
podman secret rm SECRET_NAME

### Pass secret as environmental variable
podman run -it --secret dbsecret,type=env,target=DB_SECRET \
--name myapp registry.access.redhat.com/ubi8/ubi /bin/bash



