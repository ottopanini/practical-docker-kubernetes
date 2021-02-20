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

### My NodeJS App [nodejs-app]
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
***
Always pay attention to the JSON array format for the CMD command parameter! If missing the comma it will throw an error at container runtime startup with
message: `/bin/sh: 1: [: node: unexpected operator`
***
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

### Entering Interactive Mode [python-app]
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
## 3: Managing Data & Working with Volumes
- application data
    - stored in image
- temporary app data
    - stored in container (extra container layer)
- permanent app data
    - stored in volumes

### Analyzing a Real App [data-volumes]
After added a Dockerfile... build the image:
```
docker build -t feedback-node .      
```
and run it:
```
docker run -p 3000:80 -d --name feedback-app --rm feedback-node
```
Feedback will be stored as files in the feedback folder (inside the container). But will be lost when container is removed...
### Introducing volumes
Volumes are *folders on your host machine* hard drive which are *mounted* ("made available", mapped) *into containers*.

Volumes can be defined in the Dockerfile like:
```docker
...
VOLUME [ "<path inside container>" ]
...
#example
VOLUME [ "/app/feedback" ]
```
Another problem is, that used directories in the example mount as different devices. A little change to the code must have been made and keep in mind:

In server.js:
```js
...
await fs.rename(tempFilePath, finalFilePath);
...
```
to move the files from temp folder to feedback filder must be replaced with proper handling:
```js
...
await fs.copyFile(tempFilePath, finalFilePath);
await fs.unlink(tempFilePath);
...
```
Data is still not going into the right folder...
Let's check it out.
```
docker volume ls
```
The first thing we see is, our volume is an **anonymous volume**. By removing our container (when using the --rm tag) the anonymous volume gets lost. **Named volumes** instead survive container removals (of --rm runs).

**Named volumes** can't be created by the Dockerfile. It must be defined with the container startup command:
```
docker run ... [-v <volume name>:<volume path in container>]

docker run --name feedback-app -p 3000:80 --rm -d -v feedback:/app/feedback feedback-node:volumes
```
**Named volumes** are great to store persistent data which must not edited directly (from outside the container). **Named Volumes** can also be shared between containers.

### Getting Started With Bind Mounts (Code Sharing)

Here the path on the host can be specified.
```
docker run ... [-v <absolute path to folder>:<path inside container>]

docker run --name feedback-app -p 3000:80 --rm -d -v feedback:/app/feedback -v "/home/cpress/practical-docker-kubernetes/data-volumes:/app" feedback-node:volumes
```
By this make shure the specified folder is accessible by docker. In Docker Desktop this can be achieved via Settings &rarr; Resources &rarr; File Sharing (if this setting isn't available you are on a windows machine using the wsl integration of docker desktop and therefore this isn't needed anyway).

![](bind-mounts-1.png)

Using Docker Toolbox: [https://headsigned.com/posts/mounting-docker-volumes-with-docker-toolbox-for-windows/](https://headsigned.com/posts/mounting-docker-volumes-with-docker-toolbox-for-windows/)

If you don't always want to copy and use the full path, you can use these OS dependant shortcuts:
macOS / Linux: `-v $(pwd):/app`    
Windows: `-v "%cd%":/app`

Attention: make shure that for code sharing the WORKDIR isn't overridden by container creation (the npm install in the data-volumes is for now overridden by the later mount bind). This can be prevented by using an anonymous volume:
```
docker run --name feedback-app -p 3000:80 --rm -v feedback:/app/feedback -v "/home/cpress/practical-docker-kubernetes/data-volumes:/app" -v /app/node_modules feedback-node:volumes
```
(node_modules dosn't get overridden here bacause docker has a rule that for volume mounts the more specific path wins)

On Windows with docker on WSL be shure, the folders trying to mount are inside the WSL file system **hints:**[WSL2 PDF](https://att-b.udemycdn.com/2020-11-06_09-26-40-a5b3131a849aa7ea0ad3dc1a26de5da3/original.pdf?secure=5KKxaoyUuq-rVSkRe0vYbg==,1613322384&filename=windows-wsl2-file-events.pdf)

**Bind Mounts** can be shared between containers. 

### A Look at Read-Only Volumes

Files should only be changed in one direction by the code sharing **bind mount** - from outside the container into the container.
```
docker run --name feedback-app -p 3000:80 --rm -d -v feedback:/app/feedback -v "/home/cpress/practical-docker-kubernetes/data-volumes:/app:ro" -v /app/temp -v /app/node_modules feedback-node:volumes
```
We just added the `:ro` to make clear the volume isn't changed from inside out. Therefore the `-v /app/temp` must be specified for the *data-volumes* project though, so that it can write into that folder *inside* the container. 

### Managing Docker Volumes
**Bind mounts** are not managed by docker. `docker volume ls` will not show it.
With 
```
docker volume create <volume name>

docker volume create feedback-files

```
you can create a named volume by hand.

`docker volume inspect <volume>` shows volume specific info.   
`docker volume rm <volume-id/-name>` can be used to remove a volume.  
`docker volume prune` removes unused volumes.

FYI: When using **bind mounts** the COPY inside the Dockerfile can be omitted but be aware the image definition might be used in production and then the container won't get started with the code sharing **bind mounts**.

A `.dockerignore` file can be used simillar to .gitignore to specify files or folders to be ommitted by the COPY command.

### Working with Environment Variables & ".env" Files
![](env-vars-1.png)

In server.js of the "data-volumes" Project. we like to have the port more flexible. Instead:
```javascript
...
app.listen(80);

// we use now
app.listen(process.env.PORT);
```
With a little change in the dockerfile, the env var can be provided:
```docker
...
ENV PORT 80

# we can also use this var in the Dockerfile 
EXPOSE $PORT
...
```

Now it's also possible to override the env vars on container run:
```
docker run --name feedback-app --env PORT=8000 -p 3000:8000 --rm -d -v feedback:/app/feedback -v "/home/cpress/practical-docker-kubernetes/data-volumes:/app:ro" -v /app/temp -v /app/node_modules feedback-node:volumes 
```
`--env` can be replaced shorter with just `-e`. Another possible solution is to place a .env file in the project directory and run the container with the `--env-file` option.
```
docker run ... --env-file ...

docker run --name feedback-app --env-file -p 3000:8000 --rm -d -v feedback:/app/feedback -v "/home/cpress/practical-docker-kubernetes/data-volumes:/app:ro" -v /app/temp -v /app/node_modules feedback-node:volumes
```
### Using Build Arguments (ARG)
With the `ARG` keyword, we can set args available in the dockerfile for image build time. 
```docker
...
ARG <property>=<value>

ARG DEFAULT_PORT=80

# and use it like
ENV PORT $DEFAULT_PORT
...
``` 
(not usable in `CMD`)

To use a different port we can also override this value with build option:
```
docker build ... --build-arg <key>=<value> ...

docker build -t feedback-node:volumes --build-arg DEFAULT_PORT=8000 .
```
***Advice***: place vars to the latest possible position in the dockerfile.

## Networking: (Cross-)Container Communication [networks]
![](networks-1.png)
### Case 1: Container to WWW Communication
The example project uses AXIOS to make GET requests against a Star Wars Dummy API of the Web. This works out of the box.
### Case 2: Container to Local Host Machine Communication
The example project uses mongoDB which runs on the host machine to store persistent data.
`localhost` usage in project code should be replaced with `host.docker.internal` to maked this work.
### Case 3: Container to Container Communication
The example project also talks to a SQL Database in another container.
***
Advice: A container should just do ***ONE*** main thing.
***
Running the example:
build the image:
```
docker build -t favorite-node .
```
And start mongodb:
```
docker run -d --name mongodb mongo
```
Then you can inspect the running container with 
```
docker container inspect mongodb | grep IPAddress
```
to get the ip-address for use in the mongoose.connect statement
and after all start the container:
```
docker run --name favorites -d --rm -p 3000:3000 favorite-node
```

The better way then inspecting for the ip of another container would be to create a container network.

### Introducing Docker Networks: Elegant Container to Container Communication
First create the network:
```
docker network create favorites-net
```
and with the existing network:
```
docker run -d --name mongodb --network favorites-net mongo
```
we can mongoDB let can use it.

`docker network ls` lists all defined networks.  
In the implementation of the app you can now use just the name of the container as host address. The last thing needed is to put the container into the same network:
```
docker run --name favorites --network favorites-net -d --rm -p 3000:3000 favorite-node
```
*Info*: in container networks it isn't needed to expose ports via Dockerfile.

## Building Multi-Container Applications with Docker [multi]
### The Multi Demo App
Backend is talking to Mongo DB and provides an API to the Frontend React SPA. The purpose of the App is to manage goals.

- Database
    - Mongo DB
    - Data must persist
    - Access should be limited
- Backend
    - NodeJS REST API
    - Data must persist
    - Live source code update
- Frontend
    - REACT single page application (SPA)
    - Live source code update

start the db:
```
docker run --name mongodb -p 27017:27017 --rm -d mongo
```
For now the BE could be build and run with `node app.js` but e want it dockerized. To build the image create the Dockerfile:
```docker
FROM node

WORKDIR /app

COPY package.json .

RUN npm install

EXPOSE 80

COPY . .

CMD ["node", "app.js"]
```
... and build the image with:
```
docker build -t goals-node backend 
```
and run it
```
docker run --name goals-backend --rm -d -p 80:80 goals-node
```
The Frontend has a similar Dockerfile:
```
FROM node

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```
build it:
```
docker build -t goals-react frontend
```
and run it
```
docker run --name goals-frontend  -p 3000:3000 --rm -it goals-react
```
`-it` is needed because the npm start only works this way (for now).

### Adding Docker Networks for Efficient Cross-Container Communication
Create a network:
```
docker network create goals-net
``` 
Run mongo db in the created network:
```
docker run --name mongodb --network goals-net --rm -d mongo
```
Build and run the Backend (@see also the code changes in app.js):
```
docker build -t goals-node backend
docker run --name goals-backend -d --rm --network goals-net -p 80:80 goals-node
```
Hint: The Appp.js of the frontend doesn't need any change, because it's a web application and is executed inside the browser - not the container. The container is only used in this case to serve the web page.

###  Adding Data Persistence to MongoDB with Volumes
Add a volume to mongo db:
```
docker run --name mongodb --network goals-net --rm -d -v data:/data/db mongo
```
and add security to it:
```
docker run --name mongodb --network goals-net -e MONGO_INITDB_ROOT_USERNAME=max -e MONGO_INITDB_ROOT_PASSWORD=secret --rm -d -v data:/data/db mongo
```
![](multi-1.png)

Now the Backend must talk using credentials to the mongodb database. In app.js add the credentials to the connect statement:
```javascript
...
mongoose.connect(
  'mongodb://max:secret@mongodb:27017/course-goals?authSource=admin',
  {
...
```
Rebuild the backend and start it again and everything should work fine again.

### Volumes, Bind Mounts & Polishing for the NodeJS Container
bind volume for logs:
```
docker run --name goals-backend -v logs:/app/logs -d --rm --network goals-net -p 80:80 goals-node
```
and a bind mount to have any changes be reflected into the container:
```
docker run --name goals-backend -v logs:/app/logs -v ~/practical-docker-kubernetes/multi/backend:/app -d --rm --network goals-net -p 80:80 goals-node
```
The node_modules folder inside the container created by image build should not be overriden by the non-existing folder of the mounted (bind mount) source directory:
```
docker run --name goals-backend -v logs:/app/logs -v ~/practical-docker-kubernetes/multi/backend:/app -v /app/node_modules -d --rm --network goals-net -p 80:80 goals-node
```
And finally add nodemon change detection to have the running container application detect and reflect changes in the source code.









