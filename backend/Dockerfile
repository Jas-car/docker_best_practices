FROM node:16.16-slim
#FROM node:18.19-alpine3.18

WORKDIR /app

COPY ./package.json /app/package.json
COPY ./package-lock.json /app/package-lock.json
RUN useradd  usernode && chown -R usernode /app
RUN npm ci

COPY . .
USER usernode
CMD ["node", "server.js"]