#-- create image ------
FROM node:12

WORKDIR /app

COPY package.json /app

RUN npm install

COPY . /app

EXPOSE 80
#-- image done --------

#-- run command -------
CMD [ "node", "server.js" ]