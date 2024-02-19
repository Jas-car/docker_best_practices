FROM node:16.16-slim AS build-env
#FROM node:18.19-alpine3.18

WORKDIR /app
RUN groupadd -g 1002  nodegroup && useradd -u 1002 -g 1002 usernode && chown -R usernode /app && chmod 550 /app

COPY ./package.json /app/package.json
COPY ./package-lock.json /app/package-lock.json

RUN npm ci
RUN chmod 770 /app/node_modules
USER usernode
COPY . .

CMD ["node", "server.js"]

