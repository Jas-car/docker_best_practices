FROM node@sha256:e9dbce470d22c34e5cc91e305f5ad3fd14b3f02e36fd8a7746e3e5a9e4de4655 AS build
#FROM node:16.16-slim as build
#FROM node:18.19-alpine3.18 as build
ARG REACT_APP_SERVICES_HOST=/services/m

WORKDIR /app

COPY ./package.json /app/package.json
COPY ./package-lock.json /app/package-lock.json
#RUN useradd  usernode && chown -R usernode /app
RUN yarn install

COPY . .
#USER usernode
RUN yarn build



#FROM debian:11 as builder
FROM debian@sha256:2c22645bfe97aa1ed1c930adf5970fee3454f9a42a19214051ec677cba805712 as builder

ENV USER=nonroot
ENV UID=10001 

RUN adduser \    
    --disabled-password \    
    --gecos "" \    
    --home "/nonexistent" \    
    --shell "/sbin/nologin" \    
    --no-create-home \    
    --uid "${UID}" \    
    "${USER}"

RUN apt update && apt -y install wget gnupg binutils && \
    wget "https://nginx.org/keys/nginx_signing.key" && \
    apt-key add nginx_signing.key && \
    echo "deb https://nginx.org/packages/mainline/debian/ bullseye nginx" >/etc/apt/sources.list.d/nginx.list && \
    echo "deb-src https://nginx.org/packages/mainline/debian/ bullseye nginx" >>/etc/apt/sources.list.d/nginx.list && \
    apt update && \
    apt install -y nginx && \
    apt install --download-only --reinstall nginx lsb-base  libgcc-s1 libc6 libcrypt1 libpcre2-8-0 libssl1.1 zlib1g && \
    apt install --download-only --reinstall libidn2-0 libnss-nis libnss-nisplus  debconf gcc-10-base && \
    for f in /var/cache/apt/archives/*.deb; do dpkg-deb -xv $f /packages; done && \
    rm -rf /packages/usr/share/bash-completion  \
        /packages/usr/share/debconf \
        /packages/usr/share/doc \
        /packages/usr/share/lintian	\
        /packages/usr/share/locale	\
        /packages/usr/share/man \
        /packages/usr/share/perl5  \
        /packages/usr/share/pixmaps \
        /packages/usr/bin \
        /packages/usr/bin/dpkg* \
        /packages/usr/bin/nginx-debug && \
        mkdir -p /packages/var/run

RUN chown -R nonroot:nonroot /var/cache /var/run

RUN touch /var/run/nginx.pid && \
    chown -R nonroot:nonroot /var/run/nginx.pid

FROM scratch
COPY --from=builder /packages /
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group
COPY --from=builder /var/cache /var/cache
COPY --from=builder /var/run /var/run

COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
#COPY nginx_configs/default.conf /etc/nginx/conf.d/default.conf

COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/build /usr/share/nginx/html

RUN --mount=type=bind,from=builder,target=/mount ["/mount/bin/ln", "-sf", "/dev/stdout", "/var/log/nginx/access.log"]
RUN --mount=type=bind,from=builder,target=/mount ["/mount/bin/ln", "-sf", "/dev/stderr", "/var/log/nginx/error.log"]

EXPOSE 8000

USER nonroot

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]