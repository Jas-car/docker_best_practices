FROM node:16.16-slim as build
#FROM node:18.19-alpine3.18 as build
ARG REACT_APP_SERVICES_HOST=/services/m

WORKDIR /app

COPY ./package.json /app/package.json
COPY ./package-lock.json /app/package-lock.json
RUN useradd  usernode && chown -R usernode /app
RUN yarn install

COPY . .
USER usernode
RUN yarn build



FROM  nginxinc/nginx-unprivileged:latest
#FROM  nginx:alpine3.18-slim
COPY ./nginx/nginx.conf /etc/nginx/conf.d/default.conf

COPY --from=build /app/build /usr/share/nginx/html
#RUN useradd usernginx 
#USER usernginx
