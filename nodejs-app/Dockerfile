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