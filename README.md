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

## Make it interactive with terminal
podman exec -it my-container /bin/bash

### Copy files in and out of conatiners
podman cp /host/path/file.txt my-container:/container/path/

### Pause/unpause container
podman pause my-container, podman unpause my-container

### Inspect container
podman inspect my-container

### Logging (--tail 20 for num of lines, -f for live log, --since 10m for specific time)
podman logs --tail 20 -f --since 10m my-container


