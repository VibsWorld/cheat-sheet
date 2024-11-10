
## DOCKER RUN
```
# Running with a docker image in a deattached mode
docker run --name <CustomContainerName> -p 8080:8080 -d <ImageName>


```

### DOCKER IMAGES
```
# List images
docker images

# Remove all unused images
docker images prune

# Show history associated with a image
# docker image history <docker_registry>/<image>:<version>
docker image history vibs2006/test01:v1 
```

### DOCKER START
```
docker start <container_name>

# Starting docker in attached mode
docker start -a <container_name>

# Starting docker in attached and interactive mode both
docker start -a -i <container_name>
```

### DOCKER PS
`docker ps -a`

### DOCKER ATTACH and DOCKER LOGS
```
docker attach <container_name>

# Only prints logs
docker logs <container_name>

# Prints past logs and also attaches itself to container and block ui again to show live logs 
docker logs -f <container_name> 
```

DOCKER RM and DOCKER RMI (Delete)
```
# Docker delete
docker rm <container_name>

#Delete images and its all related layers (rmi , i means Images)
docker rmi <image_id> 

#Delete multiple images
docker rimi <image1> <image2> 
```

DOCKER TAG (Renaming Images)
```
# Image Rename
docker tag 0e5574283393 fedora/httpd:version1.0do

# Container Rename Image Example (vibs2006/padauk-api:latest is created on hub)
docker tag padaukapi:latest vibs2006/padauk-api:latest

# Change any given version to latest
docker tag vibs2006/image:v2 vibs2006/image:latest
```

DOCKER CONTAINER RENAME
```
docker rename my_container my_new_container
```

DOCKER EXEC (Opening Interactive Terminal in an existing container)
```
Opening Interactive Terminal in an existing container
#Usage
docker exec -it <container_name> bash

#Exanoke
docker exec -it rabbitmq bash
```

Update a container's restart policy (--restart)
You can change a container's restart policy on a running container. The new restart policy takes effect instantly after you run docker update on a container.
To update restart policy for one or more containers:
 ```
docker update --restart=on-failure:3 abebf7571666 hopeful_morse
```

