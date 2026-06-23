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


## PRACTICE EXAM ANSWERS
### 1. Run an httpd container detached, publish container port 80 to host port 8081, give it a custom --name and a custom --hostname, and confirm both with podman inspect
podman run -d --name web-server --hostname my-httpd-host -p 8081:80 docker.io/library/httpd:latest
podman inspect web-server

### 2. Run a container with --restart=always, kill its main process, and prove podman restarts it (check podman ps and the restart behavior)
podman run -d --name my-container --restart=always docker.io/library/alpine:latest
podman kill my-container
podman ps --all 

### 3. Run a container with --memory=256m --cpus=0.5 and verify the limits are applied using podman stats and podman inspect
podman run -d --name my-container --memory=256m --cpus=0.5 docker.io/library/alpine:latest
podman stats
podman inspect my-container

### 4. Start one container with --env KEY=VALUE and another with --env-file, then show the environment differences with podman exec <c> env
podman run -d --name my-container-1 --env KEY=VALUE my-image
podman run -d --name my-container-2 --env-file ./file.env my-image
podman exec my-container-1 env
podman exec my-container-2 env

### 5. Create a user-defined bridge network with podman network create, run a container attached to it, and inspect the network to find the container's IP
podman network create my-network
podman run -d --name my-container --net my-network my-image
podman network inspect my-network

### 6. Pause and unpause a running container and explain what state its processes are in while paused
podman pause my-container
podman unpause my-container

### 7. Rename an existing container with podman rename and confirm the change.
podman rename old-container-name new-container-name
podman ps --all

### 8. Use podman logs with --tail, --since, and -f, and explain what each flag does.
podman logs --tail 20 my-container
podman logs --since 10m my-container
podman logs -f my-container

### 9. Extract a single field (e.g. the IP address or restart count) using podman inspect --format '{{ ... }}' Gotemplate syntax
podman inspect --format '{{.NetworkSettings.IPAddress}}' my-container

### 10. Copy a file from the host into a running container and back out again with podman cp.
podman cp ./local-file.txt my-container:/tmp/local-file.txt
podman cp my-container:/tmp/local-file.txt ./restored-file.txt

### 11. Exec an interactive shell into a running container, install a package, and explain whether that change survives the container being recreated.
podman exec -it my-container /bin/bash
apk add curl

### 12. Use podman top and podman stats to show the processes and live resource usage of a container.
podmam top my-container
podman stats my-container

### 13. Run a container with --rm, and explain when auto-removal is useful and what you lose by using it.
podman run --rm my-container

### 14. Bind-mount a host directory into a container read-only (-v hostdir:/path:ro) and prove that writes from inside are rejected.
podman run -it --rm -v hostdir:/mnt/data:ro my-image /bin/bash
touch /mnt/data/test.txt

### 14. Bind-mount a host directory into a container read-only (-v hostdir:/path:ro) and prove that writes from inside are rejected.
podman run -it --rm -v hostdir:/mnt/data:ro my-image /bin/bash
touch /mnt/data/test.txt

### 15. Add custom --labels to two containers, then list only the ones matching a label using podman ps --filter.
podman run -d --name app-frontend --label tier=frontend my-image
podman run -d --name app-backend --label tier=backend my-image
podman ps --filter "label=tier=frontend"

### 16. Commit a modified running container into a new image with podman commit, then run a container from that image
podman commit my-container my-new-image:v1
podman run -d --name new-container my-new-image:v1

### 17. Show the filesystem changes made inside a running container with podman diff and interpret the A/C/D markers
podman diff new-container

### 18. Run a container with --network none, demonstrate it has no external connectivity, and explain a use case.
podman run -d --name my-container --network none my-image
podman exec my-container ping -c 1 8.8.8.8

### 19. Publish a port bound to a specific host IP only (-p 127.0.0.1:8082:80) and verify it is not reachable on the VM's other IPs.
podman run -d --name my-container -p 127.0.0.1:8082:8000 my-image

### 20. Use podman port to list a container's published ports and cross-check with podman inspect.
podman port my-container
podman inspect my-container

### 21. Run a container as a non-root user with --user, and verify the effective UID inside with id / whoami.
podman run --user 1001 my-image id
podman run --user 1001 my-image whoami

### 22. Generate a systemd unit (or a Quadlet .container file) for a container and explain how it makes the container start on boot.
mkdir -p ~/.config/containers/systemd/

cat << 'EOF' > ~/.config/containers/systemd/sample-app.container
Description=My Sample Podman Quadlet Application
After=network-online.target

Image=registry.access.redhat.com/ubi9/ubi-minimal:9.5
Exec=sleep infinity
ContainerName=quadlet-sample-app

WantedBy=default.target
EOF

### 23. Run podman events in one terminal while you start/stop a container in another, and describe the lifecycle events you observe.
podman events
podman run -d --name my-container my-image
podman stop my-container
podman rm my-container


