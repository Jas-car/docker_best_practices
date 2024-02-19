#FROM node:16.16-slim AS build-env
FROM node@sha256:e9dbce470d22c34e5cc91e305f5ad3fd14b3f02e36fd8a7746e3e5a9e4de4655 AS build-env
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


#FROM gcr.io/distroless/nodejs20-debian11
FROM gcr.io/distroless/nodejs@sha256:0f7fbe2f3853fd719204ff417dda421eea2f8db8e17875820fddac5d3a8f572c
COPY --from=build-env --chown=1002:1002 /app /app
COPY --from=build-env /etc/passwd /etc/passwd
COPY --from=build-env /etc/group /etc/group

USER 1002

WORKDIR /app
CMD [ "server.js"]