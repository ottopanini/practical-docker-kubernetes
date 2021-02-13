# practical-docker-kubernetes
## Getting Started
### first-demo-starting-setup
Runs simple node.js Server. Go into folder and type:
```
docker build .
```
builds image. Grab the id of the image for the next step:
```
docker run -p 3000:3000 <ID>
```
Use a browser and visit: [http://localhost:3000](http://localhost:3000)

## Docker Images & Containers: The Core Building Blocks

![](image&container1.png) 

![](image&container2.png)  
Base images can be found at [https://hub.docker.com/](https://hub.docker.com/).

You can find the node image there.
To run it simply put:
```
docker run node
```
to create a container by the image.

**Useful commands**:  
`docker ps` to get all running containers. with `-a` to get all containers including stopped ones.  
`docker run -it <container>` to get a console inside the running container.

### My NodeJS App (nodejs-app)
Build my own custom image.
![](a-nodejs-app1.png)  
Create a 'Dockerfile' in project directory:
```docker
#-- create image ------
FROM node

WORKDIR /app

# copy everything in current directory to 'app' in container
COPY . /app

RUN npm install

EXPOSE 80
#-- image done --------

#-- run command -------
CMD [ "node", "server.js" ]
```
**To start the container:**  
build the image:
```cmd
docker build
```
grab the Id at the end und use it with:
```cmd
docker run <image id>
```
One step missing. Expose 80 in Dockerfile alone isn't sufficient. A port must be provided at startup:
```
docker run -p 3000:80 <image id>
```
[http://localhost:3000](http://localhost:3000/)
nothing there...

Actually `EXPOSE 80` in the Dockerfile is optional. It just documents that a process in the container will expose this port.

Now we change something in the served html and call the run command again. But... the html output in the browser didn't change...

The image must be build again first. `docker build .`
Start container again... and there it is.

![](a-nodejs-app-2.png)  

### Understanding Image Layers 

![](image-layers-1.png)  

When an image is rebuild then all steps are reexecuted which are underneath the first step that has changed.

Lets have an improvement of the Dockerfile though:
```docker
#-- create image ------
FROM node

WORKDIR /app

COPY package.json /app

RUN npm install

COPY . /app

EXPOSE 80
#-- image done --------

#-- run command -------
CMD [ "node", "server.js" ]
```
Now install is only executed when some node modules really changed.

![](summary-1.png)

### Managing Images & Containers

`--help` can be added to every docker command.

`docker run ..` starts a NEW container. By default in foreground (STDOUT can be seen).
With `docker ps -a` we can check for stopped containers and restart it with:
```
docker start <id/name>
```
By this the container is started in the background (in opposite to `run`s default mode; with `-a` start can be attached).  
`docker run -d ..` with runs the container in the background.  
`docker attach <id>` can be used to have a running container in the foreground again.  
`docker logs <id>` can be used to get STDOUT log output in the container. With option `-f`the follow mode can be activated.

### Entering Interactive Mode (python-app)
'Atached' doesn't mean we're inside the container and can work init. Let's have it more interactive:
```
docker run -it <image id>
```
or restart:
```
docker start -a -i <container id>
```
### Deleting Images & Containers
Delete container:
```
docker rm <container id> [<container nextId>]
```
list images:
```
docker images 

docker image ls
```
remove image:
```
docker rmi <imageid>
```
remove all unused images:
```
docker image prune
```
You can also start/run a container with `--rm` to have it remove itself completely.

### A Look Behind the Scenes: Inspecting Images
```
docker image inspect <id>
```
Output: `ContainerConfig` with some `Env` Variables. `Cmd` with the running command. `Os`with the operting system the image is based on. `Layers` with the checksums of all the layers of the container.
### Copying Files Into & From A Container
Can be done using:
```
docker cp [<container name>:]<srcpath> [<container name>:]<destpath>

docker cp copy-into-container/. boring_vaughan:/test
```
### Naming & Tagging Containers and Images
Naming Containers:
```
docker run [--name <name>] <id>

docker run -p 3000:80 -d --rm --name goalsapp ec276
```
Tagging images:
Images are tagged using a *name* and a *tag* specifying a version.  
![](image-tagging-1.png)  
In Dockerfile (@see nodejs-app):
```docker
FROM <name>[:<tag>]
...

# example
FROM node:12
...
```
build tagged image:
```
docker build [-t <image name>[:<tag>]]

docker build -t goals:latest .
```
If you want to remove all unused images including tagged images you need to run:
```
docker image prune -a
```
### Sharing Images
How to provide a reliable & reproducible environment:
1. We want to have the *exact same environment for development and production* &rarr; This ensures that it works exactly as tested
2. It should be easy to *share a common development environmment*/setup with new employees and collegues
3. We *don't want to uninstall and re-install* local dependencies and runtimes all the times
![](sharing-images-1.png)  

Sharing via:
![](sharing-images-2.png)  

Sharing via docker hub [https://hub.docker.com](https://hub.docker.com):
![](share-hub-1.jpg)  

![](share-hub-2.jpg)  

![](share-hub-3.png)  

take the first part: `docker push <dockerid>/<image>`  
but first name/tag the image accordingly:
```
docker tag node-demo:latest mydockerid/node-hello-world:latest
```
next:
```
docker push mydockerid/node-hello-world
``` 
(you must be logged in via `docker login`)

### Pulling & Using Shared Images
Download/import image:
```
docker pull <dockerid>/<image>[:<tag>]

docker pull mydockerid/node-hello-world:latest
```

