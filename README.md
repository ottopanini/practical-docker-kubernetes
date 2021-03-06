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

And use environment variables (see also the Dockerfile):
```
docker run --name goals-backend -v logs:/app/logs -v ~/practical-docker-kubernetes/multi/backend:/app -v /app/node_modules -e MONGODB_USERNAME=max -d --rm --network goals-net -p 80:80 goals-node
```
### Live Source Code Updates for the React Container (with Bind Mounts)
Setup the frontend for reflecting code changes:
```
docker run --name goals-frontend -v ~/practical-docker-kubernetes/multi/frontend/src:/app/src -p 3000:3000 --rm -it goals-react
```
## Docker Compose: Elegant Multi-Container Orchestration
Docker compose is Docker build and run commands combined in a configuration file to start/stop a whole setup with just ***ONE*** command.

Things needed to configure:
- Services (containers)
    - published Ports
    - volumes
    - networks
    - Environment Variables

### Creating a Compose File

`services` needs at least one child element. `network` isn't needed here because docker-compose creates a network out of the box for all services within one docker-compose file.

and run it...
```
dockedr-compose up -d 
```
`-d` is for detachted mode.  
`docker-compose down` stops all containers and deletes them including images. Volumes are ignored. To delete also volumes option `-v` can be used.  
```yaml
...
    build: ./backend
...
```
Optionally you can use `dockerfile` to specify the Dockerfile file name if it's not the 'Dockerfile' default. Additionally args can be set if your Dockerfile processes ARGS.
```yaml
...
    build: ./backend
    dockerfile: Dockerfile
    args:
        some-arg: 1
...
```
Environment variables can be set via 3 different options. You can simply use key value pairs:
```yaml
...
  environment: 
    MONGO_INITDB_ROOT_USERNAME: max 
    MONGO_INITDB_ROOT_PASSWORD: secret 
...
```
or as list of variables:
```yaml
...
  environment: 
    - MONGO_INITDB_ROOT_USERNAME=max 
    - MONGO_INITDB_ROOT_PASSWORD=secret 
...
```
or as own files:
```yaml
...
  env_file: 
    - ./env/mongo.env
...
```
with the env file content:
```
MONGO_INITDB_ROOT_USERNAME=max 
MONGO_INITDB_ROOT_PASSWORD=secret 
```
Bind mounts can be created with relative paths for docker-compose.
```yaml
...
    volumes: 
      - ./backend:/app
...
```
`-it` transformation for docker-compose:
```yaml
...
    stdin_open: true
    tty: true
...
``` 
So the whole configuration for the compose Project should look like:
```yaml
version: "3.8"
services: 
  mongodb:
    image: 'mongo'
    name: 'mongodb'
    env_file: 
      - ./env/mongo.env
    volumes: 
      - data:/data/db
  backend:
    build: ./backend
    ports: 
      - '80:80'
    volumes: 
      - logs:/app/logs
      - ./backend:/app
      - /app/node_modules
    environment: 
      MONGODB_USERNAME: max
      MONGODB_PASSWORD: secret
    depends_on: 
      - mongodb
  frontend:
    build: ./frontend
    ports: 
      - '3000:3000'
    volumes: 
      - ./frontend/src:/app/src
    stdin_open: true
    tty: true
    depends_on: 
      - backend

volumes:
  data:
  logs:
```
`docker-compose up` can be executed with `--build` to enforce docker to build the images.  
`docker-compose build` can be used to just only build the images.

FYI: Container names can be set in the docker-compose file with: 
```yaml
...
  container_name: mongodb
...
```
## Working with "Utility Containers" & Executing Commands In Containers
![](utility-containers-1.png)


```
docker run -it node
```
can be used to start a node container.
```
docker run -it -d node
```
starts it in detached mode but waiting for commands. Now we can pass commands via:
```
docker exec -it <container id> npm init
```
this creates a node project inside the container.   

To execute commands directly you can run it with: 
```
docker run -it <image-id> [<commands to run after container start>]*

docker run -it node npm init
```
The commands replace the commands in the Dockerfile in the `CMD` section.

### Building a First Utility Container [utility]
The Dockerfile used here doesn't provide any CMD so that a user can run any command against the container.
```docker
FROM node:14-alpine 

WORKDIR /app
```
Notice we are using an extra lightweight image here. The we build the image with:
```
docker build -t node-util .
```
Now with:
```
docker run -it -v ~/practical-docker-kubernetes/utility:/app node-util npm init
```
a node project in the specified folder of the volume can be created.

We can define an entry point in the Dockerfile to shrink or specify kind of a base command the commands for running the container are appended to.
In the Dockerfile:
```docker
ENTRYPOINT [ "npm" ]
``` 
Just build the container again with `docker build -t mynpm .` and then any npm command can be executed:
```
docker run -it -v ~/practical-docker-kubernetes/utility:/app mynpm init
```
### Using Docker Compose
Now we add a docker-compose.yaml
```yaml
version: "3.8"
services:
  npm:
    build: ./
    stdin_open: true
    tty: true
    volumes:
      - ./:/app
```
and start it via 
```
docker-compose run npm init 
```
here the `--rm` option can be used to have the containers removed after the run command is executed.

## A More Complex Setup: A Laravel & PHP Dockerized Project [laravel]
The target setup:
![](laravel-1.png)

###  Adding a Nginx (Web Server) Container
```yaml
...
  server:
    image: 'nginx:stable-alpine'
    ports:
      - '8000:80'
    volumes:
      - ./nginx/nginx.comf:/etc/nginx/nginx.conf
...
```
### Adding a PHP Container
Here we need a Dockerfile for the specific setup. We use a folder 'dockerfiles' to store needed Dockerfile image definitions. To seperate the Dcokerfiles of the different services we prefix their names with their service name. To use these files in the docker-compose file it can be specified with:
```yaml
    build: 
      context: ./dockerfiles
      dockerfile: <service-name>.dockerfile

    build: 
      context: ./dockerfiles
      dockerfile: php.dockerfile
    
```
For the bind mount volume we use a sepcial mechanism `delegated` which is a performance optimization for the reflection process of changes.
```yaml
...
  php:
    build: 
      context: ./dockerfiles
      dockerfile: php.dockerfile
    volumes:
      - ./src:/var/www/html:delegated
...
```
### Adding a MySQL Container
```yaml
...
  mysql:
    image: mysql:5.7
    env_file: 
      - ./env/mysql.env
...
```
### Adding a Composer Utility Container
A Laravel Project can be created ith the composer. We can find a composer docker image at dock hub but needs a little customization:
```docker
FROM composer:latest

WORKDIR /var/www/html

ENTRYPOINT [ "composer", "--ignore-platform-reqs" ]
```
and the changes in the docker-compose-file are:
```yaml
...
  composer:
    build:
      context: ./dockerfiles
      dockerfile: composer.dockerfile
    volumes:
      - ./src:/var/www/html
...
``` 
The command to start a creation of the laravel project is then:
```
docker-compose run --rm composer create-project --prefer-dist laravel/laravel .
``` 
*FYI*: This runs only the single composer container.

Some changes needs to be done though. This changes the db configuration in laravel to use the containerized MySQL database. In `src/.env` change:
```
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret
```
Now after last changes to the nginx configuration
```yaml
...
    volumes:
      - ./src:/var/www/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - php
      - mysql
...
```
the whole suite can be started.
```
docker-compose up -d --build server
```
### Adding More Utility Containers
#### Artisan
Initializes database with data. The Dockerfile of the php container is reused but with a different entry point here (artisan is a php tool):
```yaml
...
  artisan:
    build: 
      context: ./dockerfiles
      dockerfile: php.dockerfile
    volumes:
      - ./src:/var/www/html
    entrypoint: ["php", "/var/www/html/artisan"]
...    
```
To start data migration we execute: 
```
docker-compose run --rm artisan migrate
```
#### NPM
```yaml
...
  npm:
    image: node:14
    working_dir: /var/www/html
    entrypoint: ["npm"]
    volumes:
      - ./src:/var/www/html
...
```
### The complete docker-compose config
```yaml
version: "3.8"
services:
  server:
    image: 'nginx:stable-alpine'
    ports:
      - '8000:80'
    volumes:
      - ./src:/var/www/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - php
      - mysql
  php:
    build: 
      context: ./dockerfiles
      dockerfile: php.dockerfile
    volumes:
      - ./src:/var/www/html:delegated
  mysql:
    image: mysql:5.7
    env_file: 
      - ./env/mysql.env
  composer:
    build:
      context: ./dockerfiles
      dockerfile: composer.dockerfile
    volumes:
      - ./src:/var/www/html
  artisan:
    build: 
      context: ./dockerfiles
      dockerfile: php.dockerfile
    volumes:
      - ./src:/var/www/html
    entrypoint: ["php", "/var/www/html/artisan"]
  npm:
    image: node:14
    working_dir: /var/www/html
    entrypoint: ["npm"]
    volumes:
      - ./src:/var/www/html
```
## Deploying Docker Containers
- install docker on a remote host machine (SSH)
- push our docker image to docker registry
- pull from there to our remote host
- run the image there
- expose all needed ports to the WWW
- testing in browser

Main hosting providers:
- AWS
- Azure 
- Google Cloud

### Getting Started With An Example
- ***create and launch EC2 instance***, VPC and security group
- ***Configure security group*** to expose all required ports to WWWW
- ***Connect*** to instance (SSH), ***install Docker*** and ***run container***

### Bind Mounts In Production
![](deployment-1.png)

### Introducing AWS & EC2
![](deployment-2.png)

![](deployment-3.png)

![](deployment-4.png)

![](deployment-5.png)

![](deployment-6.png)

![](deployment-7.png)

![](deployment-8.png)

![](deployment-9.png)

Download and save the keyfile to connect to the remote machine  
&rarr; Then click launch instances  
&rarr; View Instances   

### Pushing our local Image to the Cloud
1. Deploy source
    - build image on remote machine
    - push source code to remote machine, `run docker build` and then `docker run`
    - unnecessary complexity
2. Deploy built image
    - Build image before deployment (local machine)
    - just execute `docker run`
    - avoid unnecessary remote server work

2nd option can be utilized using docker hub.

First build the image as usual. Then give the image the right name:
```
docker tag node-example-1 <accountname>/node-example-1
```
and push it
```
docker push <accountname>/node-example-1
```

### Managing & Updating the Container / Image
Implement changes and then:
```
docker build -t node-example-1 .
docker tag node-example-1 <accountname>/node-example-1
docker push <accountname>/node-example-1
```
On the remote machine stop the running container:
```
docker ps
docker stop <container name>
```
and rerun it:
```
sudo docker pull <accountname>/node-example-1
sudo docker run -d -rm -p 80:80 <accountname>/node-example-1
```
`docker pull` is needed here to pull the latest version (if the image was downloaded by docker before).

Do-it-yourself Approach - disadvantages
- responsibility 
  - security
  - updates
  - manage networks, security groups, firewall  
- SSHing can get annoying

Recommandation: **Managed Services**

### From Manual Deployment to Managed Services
![](deployment-10.png)

#### ECS on AWS
![](deployment-11.png)

![](deployment-12.png)

Set Port 80

![](deployment-13.png)

![](deployment-14.png)

![](deployment-15.png)

... and ready to go.

![](deployment-16.png)

### Creating a "build-only" Container
In some development environments like Angular, there is a difference between development builds and production builds. Development builds often contain serving the results as production only generate the expected Javascript but don't provide any server.
![](deployment-17.png)

Create a prod Dockerfile.
```docker
FROM node:14-alpine

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

CMD ["npm", "run", "build"]
```
This was only the first 'stage'...

### Introducing Multi-Stage Builds
```docker
FROM node:14-alpine as build

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

RUN npm run build

FROM nginx:stable-alpine

WORKDIR /app

COPY --from=build /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
`--from=<image>` allows us to copy content from a previous `FROM` stage into the current stage.

Build the image with:
```
docker build -f Dockerfile.prod .
```
`-f` let us specify a specific Dockerfile file name (with relative path). The context `.` is needed also this way.

It is also possible to just only build a container up until a specific stage
```
docker build [--target <stage name>]

docker build --target build -f Dockerfile.prod .
```
by this processing of the dockerfile will stop in the above build script after the `RUN npm run build`

# Kubernetes
## Getting Started with Kubernetes
### Core Concepts
![](k8s-core-concepts-1.png)

***Cluster***: A set of *Node* machines which are running *containerized* Application (*Worker Nodes*) or control other nodes (*Master Node*).  
***Nodes***: *Physical or virtual machine* with a certain hardware capacity with hosts *one or multiple Pods* and *communicates* with the Cluster.  
  ***Master Node***: Cluster *Control Plane*, *managing the Pods* across Worker Nodes
  ***Worker Node***: Hosts Pods, *running App containers* (+ *resources*)   
***Pods***: ... *hold the actual running App Containers* + their *required resources* (e.g. volumes).   
***Containers***: Normal (Docker) Containers  
***Services***: A *logical set (group) of Pods* with a unique, Pod- and Container-*independent IP address*

### A Closer Look at the Worker Nodes
![](worker-node-1.png)

### A Closer Look at the Master Node
![](master-node-1.png)

## Kubernetes in Action - Diving into the Core Concepts
### Understanding Kubernetes Objects (Resources)
Kubernetes API Objects:
- Pods
- Deployments
- Services
- Volumes
- ...

... can be defined declarativly or imperativly.

#### The Pod
- contains and runs one or multiple containers
- contain shared resources (volumes) for *all* pod containers
- has a cluster-internal IP (`ClusterIP`) by default 
- containers inside the pod can talk to eah other via `localhost`
***
***Important:*** Pods are designed to be ***ephemeral***: Kubernetes will *start*, *stop* and *replace* them as needed.
***
For *Pods* to be managed for you, you need a "Controller" (e.g. a "Deployment")

#### The Deployment
- a **Deployment** *controls* (multiple) *Pods*
  - the desired state is defined and Kubernetes changes the actual state to follow
  - define which *Pods* and *Containers* to run and *number of instances*
- **Deployments** can be *paused*, *deleted* and *rolled back*
- **Deployments** can be *scaled dynamically* (and *automatically*)
  - change number of desired Pods as needed
- *multiple* **Deployments** can be managed
- typically Pods aren't controlled directly, instead Deployments are used to set up the desired state.

### A First Deployment - Using the Imperative Approach [kube-action]
We still need to use the images:
```docker
FROM node:14-alpine

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

EXPOSE 8080

CMD [ "node", "app.js" ]
```  
and build it:
```
docker build -t kube-first-app .
```
Push it to your docker registry (minikube doesn't know a thing about the images in our local docker desktop).
```
docker tag kube-first-app registry/kube-first-app
docker push registry/kube-first-app
```
and use it to create a deployment
```
kubectl create deployment first-app --image=<registry/image>

kubectl create deployment first-app --image=registry/kube-first-app
```
`kubectl get deployments` lists all *deployments*  
`kubectl get pods` lists all *pods*

### The "Service" Object (Resource)
- Pods have an internal IP by default - but it changes when a Pod is replaced   
&rarr; **Services** group Pods with a shared IP
- Sercvies can allow external access to Pods
  - the default (internal only) can be overwritten with a service   

Without **Services**, Pods are very hard to reach and communication is difficult.
Reaching it from outside the Cluster would be even impossible at all.
With 
```
kubectl expose deployment first-app --type=LoadBalancer --port=8080
```
Types:
- ClusterIP - only reachable inside the cluster
- NodePort - reachable also outside th cluster
- LoadBalancer - Layer 4 load balancing 

`kubectl get services` lists all services.   
With
```
minikube service first-app
```
minikube can give us an external IP to reach the service.  
With `minikube dashboard` the kubernetes dashboard can be opened.
![](dashboard-1.png)

### Scaling in Action
```
kubectl scale deployment <deployment name> --replicas=<replica count>

kubectl scale deployment first-app --replicas=3
```
With `watch -n 1 kubectl get pods` the 3 pods can now be observed. Start the minkube service with `minikube service first-app` and just use `curl <minikube service url>` to get the page and `curl <minikube service url>/error` to provoke pod error and restart.

### Updating Deployments

Change some code in our app.js
```html
...
    <h1>Hello from this NodeJS app!!!!!!</h1>
    <p>This is new!</p>
...
```
Then the image needs to be rebuild with `docker build -t <repo name>/kube-first-app .`, push it to the hub `docker push <repo name>/kube-first-app` and it's ready to update:
```
kubectl set image deployment first-app kube-first-app=<repo name>/kube-first-app 
```
But wait checking the result with `curl <minikube service url>` there is no change... and this is why:
***
The change detection mechanism uses tags to decide the necessity to replace the running pods by newer versions. 
***
So lets build the image with a version tag now:
```
docker build -t <repo name>/kube-first-app:2 .
docker push <repo name>/kube-first-app:2
kubectl set image deployment first-app kube-first-app=<repo name>/kube-first-app:2
```
The Status of the updating can then be get by:
```
kubectl rollout status deployment first-app
``` 

### Deployment Rollbacks & History
Deploying a non existing verion with `kubectl set image deployment first-app kube-first-app=<repo name>/kube-first-app:3` causes an error:
![](err-img-pull-1.png)

To Undo the last change just:
```
kubectl rollout undo deployment <deployment name> [--to-revision=<revNr>]

kubectl rollout undo deployment first-app
```
It is possible to get even to a state longer back in time. To get the history of a deployment you can usse:
```
kubectl rollout history deployment <deployment name> [--revision=<revNr>]

kubectl rollout history deployment first-app
kubectl rollout history deployment first-app --revision=3
```

### Clean up 
```
kubectl delete service first-app
kubectl delete deployment first-app
```

### Creating a Deployment Configuration File (Declarative Approach)
Deployment configs can be specified in a file:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: second-app-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: second-app
    spec:
      containers:
      - name: second-node
        image: christianpress/kube-first-app:2

```
and applied by:
```
kubectl apply -f <config yaml path>

k apply -f deployment.yaml
```
(this is comparable to docker-compose)

### Creating a Service Declaratively
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: second-app
    tier: backend
  ports:
  - protocol: 'TCP'
    port: 80
    targetPort: 8080
  type: LoadBalancer
```
and deploy the service `kubectl apply -f service.yaml`   
It is possible to simply change a yaml and apply it again using `kubectl apply -f <config yaml>`  
To remove the resources related to a config yaml use:
```
kubectl delete -f <config yaml>
```
It is also possible to put several configs in one file. Therefore they must be delimited by `---`. 
***
It is considered best practise if a service and a deployment is merged to start with the service first.
*** 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: second-app
    tier: backend
  ports:
  - protocol: 'TCP'
    port: 80
    targetPort: 8080
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: second-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: second-app
      tier: backend
  template:
    metadata:
      labels:
        app: second-app
        tier: backend
    spec:
      containers:
      - name: second-node
        image: christianpress/kube-first-app:2
```

### More on Labels & Selectors
Are used to connect Api Objects such as services and deployments. For deployments you can also use `matchExpressions` instead `matchLabels`.
```yaml
...
    matchExpressions: 
      - {key: app, operator: In, values: [second-app, first-app]}
...
```
Operators can be: `In`, `NotIn`    
Selectors can also be used for the imperative approach via the CLI.
```
kubectl delete deployments,services -l group=example
```

### Liveness Probes
In deployment configs add for the containers node:
```yaml
...
      containers:
        ...
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          periodSeconds: 10
          initialDelaySeconds: 5  
...
```
`path: /` is the default though.

## Managing Data & Volumes with Kubernetes [kube-data]
You can use `docker-compose up -d --build` to build and run the projects image. The project works using persistence data on docker volumes. Let's try to translate this into the kubernetes world...

First we need a deployment:
```yaml
kind:apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
      - name: story
        image: christianpress/kube-data-demo
```
and a service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: story-service
spec:
  selector:
    app: story
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```
Know we create a repo at [docker hub](https://hub.docker.com) with the name `kube-data-demo` and build and push the image:
```
docker build -t christianpress/kube-data-demo .
docker push christianpress/kube-data-demo
```
and deploy it to our kubernetes cluster:
```
kubectl apply -f deployment.yaml -f service.yaml
```
For the later steps we add an additional endpoint to make the application exit (app.js):
```javascript
...
app.get('/error', () => {
  process.exit(1);
});
...
```
We build the image again and tag it with version 1:
```
docker build -t christianpress/kube-data-demo:1 .
docker push christianpress/kube-data-demo:1
```
We can check the app is working with curl commands:
```
curl -X POST -H "Content-Type: application/json" -d '{"text": "foo"}' <minikube url>/story
curl <minikube url>/story
curl <minikube url>/error
```

### A First Volume: The "emptyDir" Type
EmptyDir creates a volume when the pod starts but survives container restarts.  
Downsides: Replicas > 1 - emptyDirs are bound to the pod.

Add a volumes section under `spec`:
```yaml
      volumes:
        - name: story-volume
          emptyDir: {}
```
and refernce this volume in the container spec:
```yaml
...
      containers:
        ...
        volumeMounts:
          - mountPath: /app/story
            name: story-volume
...
```

### A Second Volume: The "hostPath" Type
Creates a shared volume on the host file system.
```yaml
...
      volumes:
        - name: story-volume
          hostPath:
            path: /data
            type: DirectoryOrCreate
```
`DirectoryOrCreate` creates the directory if not exists.   
*hostPath* works only in a 1 node environment like minikube.

### From Volumes to Persistent Volumes
**Persistent volumes** are *pod and node independent*. They are kind of an external environment of the nodes. **Peristent Volume Claims** are then used to connect Pods/Containers to the *Peristent Volumes*.
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
    type: DirectoryOrCreate
```
**volumeMode**: `Block` or `FileSystem`   
**accessModes**:   
`ReadWriteOnce` - Read and write by a single node.  
`ReadOnlyMany` - Readonly by multiple nodes.  
`ReadWriteMany` - Read and write by many nodes.  
With 
```
kubectl get sc
```
we can get the storage class. Storage class allows administrators to have a more fine grained control over storage use. The result for minikube is `standard`.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-pvc
spec:
  volumeName: host-pv
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```
and some changes to be done in the deployment:
```yaml
...
      volumes:
      - name: story-volume
        persistentVolumeClaim:
          claimName: host-pvc
 ```
Now let's apply the new volume settings:
```
kubectl apply -f host-pv.yaml
kubectl apply -f host-pvc.yaml
kubectl apply -f deployment.yaml
```
with
```
kubectl get pv
```
we can have a look at the newly generated persistent volume.
![](pv-1.png)
and with
```
kubectl get pvc
```
the created persistent volume claims can be listed.
![](pvc-1.png)

### Using Environment Variables
We have defined a 'story' folder to use in the app.js and we defined a path in the deployment to provide this folder. Wouldn't it be nice we could have it defined once with a environment variable and use that in both places? Here is how this works...  
First we change the source code in app.js:
```javascript
...
const filePath = path.join(__dirname, 'story', 'text.txt');
...
```
to:
```javascript
...
const filePath = path.join(__dirname, process.env.STORY_FOLDER, 'text.txt');
...
```
and add the environment variable to the deployment:
```yaml
...
      containers:
        ...
        image: christianpress/kube-data-demo:2
        env:
          - name: STORY_FOLDER
            value: 'story'
        ...
```
now we need to build the new version of the image
```
docker build -t <repo name>/kube-data-demo:2 .
docker push <repo name>/kube-data-demo:2
```
and apply the changed deployment with `kubectl apply -f deployment.yaml`

### Environment Variables & ConfigMaps
Environment variables can also be defined to be usable across different deployments using config maps. To define such a config map we create a file environment.yaml (name is free):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: data-store-env
data:
  folder: 'story'
  ...
  key: value
```
and apply this too
```
kubectl apply -f environment.yaml
```
Now we need to use this config map in our deployment:
```yaml
...
        env:
          - name: STORY_FOLDER
            valueFrom: 
              configMapKeyRef: 
                name: data-store-env
                key: folder
...
```
and apply it again.

Cleanup:
```
kubectl delete -f deployment.yaml
kubectl delete -f host-pvc.yaml
kubectl delete -f host-pv.yaml
kubectl delete -f environment.yaml
kubectl delete -f service.yaml
```

## Kubernetes Networking [kube-network]
General architecture:
![](kube-network-1.png)

The application can be started locally by `docker-compose up -d --build`.

With
```
curl -X POST -H "Content-Type: application/json" -d '{"email": "test@test.com", "password": "testers"}' localhost:8080/login
```
we are able to get a login token
```json
{
  "token": "abc"
}
```
With
```
curl -H "Authorization: Bearer abc" localhost:8000/tasks 
```
Tasks can be retrieved and new tasks can be stored with:
```
curl -X POST -H "Authorization: Bearer abc" -H "Content-Type: application/json" -d '{"title": "a task", "text": "do this, do that"}' localhost:8000/tasks
```

### Creating a First Deployment
First we need to comment out the axios calls in users-app.js and replace its result with a dummy value in line 26:
```javascript
    //const hashedPW = await axios.get('http://auth/hashed-password/' + password);
    const hashedPW = 'dummy text';
```
and in line 57:
```javascript
  // const response = await axios.get(
  //   'http://auth/token/' + hashedPassword + '/' + password
  // );
  const response = {status: 200, data: {token: 'abc'}};
```
just for now.  

Now we create and push the docker image to the hub
```
docker build -t <hub account>/kube-demo-users .
docker push <hub account>/kube-demo-users

```
The users deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: users
  template:
    metadata:
      labels:
        app: users
    spec:
      containers:
        - name: users
          image: <hub account>/kube-demo-users
```
and apply it to kubernetes.  
Create a users service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: users-service
spec:
  selector:
    app: users
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
```
and apply it too.
```
kubectl apply -f kubernetes/users-service.yaml
```
Now we can run `minikube service users-service` and try the login:
```
curl -X POST -H "Content-Type: application/json" -d '{"email": "test@test.com", "password": "testers"}' http://127.0.0.1:35785/login
```
### Multiple Containers in One Pod
Now we can reactivate the previously out commented lines in users-app.js to establish real communications between pods. But this time we will use Environment variables to define the address of the auth endpoint.
```javascript
...
const hashedPW = await axios.get(`http://${process.env.AUTH_ADDRESS}/hashed-password/` + password);
...
const response = await axios.get(
    `http://${process.env.AUTH_ADDRESS}/token/` + hashedPassword + '/' + password
  );
```
Build and push this new image (`docker build -t <hub account>/kube-demo-users .` and `docker push <hub account>/kube-demo-users`). The auth container should be deployed as second container inside the same pod as the users api. The containers section in the users-deployment must be changed:
```yaml
...
      containers:
        - name: users
          image: <hub account>/kube-demo-users:latest
          env: 
          - name: AUTH_ADDRESS
            value: localhost
        - name: auth
          image: <hub account>/kube-demo-auth:latest
```
Deploy the new deployment with `kubectl apply -f users-deplyment.yaml`and check the result:
```
curl -X POST -H "Content-Type: application/json" -d '{"email": "test@test.com", "password": "testers"}' http://127.0.0.1:38163/login
```
### Creating Multiple Deployments
First we change our general setup so that the auth container gets its own pod. 
![](kube-network-2.png)
create an auth-deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
        - name: auth
          image: <hub account>/kube-demo-auth:latest
```
and a service so that the new pod is reachable inside the cluster:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
```
apply it `kubectl apply -f kubernetes/auth-service.yaml -f kubernetes/auth-deployment.yaml` and change the AUTH_ADDRESS in the users deployment environment... but first let's get the correct address by `kubectl get services` and use it:
```yaml
...
          env: 
            - name: AUTH_ADDRESS
              value: "10.97.231.106"
```
finally apply the changes (`kubectl apply -f kubernetes/users-deployment.yaml`).

Next improvement by using automatic environment variables kubernetes creates in js code.
```javascript
...
const response = await axios.get(
    `http://${process.env.AUTH_SERVICE_SERVICE_HOST}/token/` + hashedPassword + '/' + password
  );
...
```
These variables are automatically provided by kubernetes and follow always this pattern
```
<Snake Case Service Name>_SERVICE_HOST
```
Now again the changes must be built into an image and pushed to the hub.

### Using DNS for Pod-to-Pod Communication
Kubernetes clusters by default come with a default service named CoreDNS. These are cluster internal domain name providers. We can use it in our users-deployment:
```yaml
         env: 
          - name: AUTH_ADDRESS
            value: "auth-service"
```
When communicating over namespace the namespace can be added
```yaml
            value: "<service-name>.<namespace>"
```

### Add Tasks deployment and service 
Code changes:
```javascript
...
  const response = await axios.get(`http://${process.env.AUTH_ADDRESS}/verify-token/` + token);
...
```
build the docker image and push it to the hub. Then add the deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tasks-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tasks
  template:
    metadata:
      labels:
        app: tasks
    spec:
      containers:
      - name: tasks
        image: <hub account>/kube-demo-tasks
        env:
          - name: TASKS_FOLDER
            value: "tasks"
          - name: AUTH_ADDRESS
            value: auth-service
```
and the corresponding service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tasks-service
spec:
  selector:
    app: tasks
  type: LoadBalancer
  ports:
  - port: 8000
    targetPort: 8000
    protocol: TCP
```
### Adding a Containerized Frontend
Some lines need to be changed to work with our deployed minkube services. Just copy the ip listed by `minikube service tasks-service` into frontend/App.js
```javascript
    fetch('http://192.168.49.2:31264/tasks', {
```
then build the image
```
docker build -t <hub account>/kube-demo-frontend .
```
and run it
```
docker run -d --rm --name frontend -p 80:80 <hub account>/kube-demo-frontend
```
***
To deploy images without having to use a remote registry you can use
```
eval $(minikube -p minikube docker-env)
```
After that just push the images as usual tagged and kubernetes will fetch it from the local docker env.
***




