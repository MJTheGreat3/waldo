# Stage 1: stamp the build version into the page
FROM alpine:3.20 AS build
ARG APP_VERSION=dev
WORKDIR /src
COPY public/ ./public/
RUN sed -i "s/__VERSION__/${APP_VERSION}/g" public/index.html

# Stage 2: serve as a non-root nginx
FROM nginxinc/nginx-unprivileged:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /src/public/ /usr/share/nginx/html/
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:8080/healthz || exit 1
