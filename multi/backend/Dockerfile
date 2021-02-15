FROM node

WORKDIR /app

COPY package.json .

RUN npm install

EXPOSE 80

COPY . .

CMD ["node", "app.js"]