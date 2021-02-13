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