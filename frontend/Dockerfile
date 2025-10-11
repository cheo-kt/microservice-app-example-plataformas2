FROM node:18 AS build

WORKDIR /app

COPY package*.json ./

# Instalar Python y herramientas de compilación requeridas
RUN apt-get update && apt-get install -y python3 make g++ && ln -s /usr/bin/python3 /usr/bin/python

RUN npm install
COPY . .

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html

