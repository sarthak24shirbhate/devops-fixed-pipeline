FROM node:20
WORKDIR /usr/src/application
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node","server.js"]
