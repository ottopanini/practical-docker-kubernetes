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
`docker run -it <container>`to get a console inside the running container.

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
docker run <id>
```
One step missing. Expose 80 in Dockerfile alone isn't sufficient. A port must be provided at startup:
```
docker run -p 3000:80 <id>
```
Actually `EXPOSE 80`in the Dockerfile is optional. It just documents that a process in the container will expose this port.


