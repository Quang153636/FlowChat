# syntax=docker/dockerfile:1

# Build the Vite static site.
FROM node:22-alpine AS build
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . ./

# Vite substitutes VITE_* variables at build time. These must be public URLs
# reachable by the browser, not Docker-internal service names.
ARG VITE_API_URL=https://192.168.100.168.nip.io:5000/api
ARG VITE_SOCKET_URL=https://192.168.100.168.nip.io:5000
ARG VITE_GOOGLE_WEB_CLIENT_ID
ARG VITE_WEBRTC_STUN_URL=stun:stun.l.google.com:19302
ARG VITE_WEBRTC_TURN_URL
ARG VITE_WEBRTC_TURN_USERNAME
ARG VITE_WEBRTC_TURN_CREDENTIAL

ENV VITE_API_URL=${VITE_API_URL} \
    VITE_SOCKET_URL=${VITE_SOCKET_URL} \
    VITE_GOOGLE_WEB_CLIENT_ID=${VITE_GOOGLE_WEB_CLIENT_ID} \
    VITE_WEBRTC_STUN_URL=${VITE_WEBRTC_STUN_URL} \
    VITE_WEBRTC_TURN_URL=${VITE_WEBRTC_TURN_URL} \
    VITE_WEBRTC_TURN_USERNAME=${VITE_WEBRTC_TURN_USERNAME} \
    VITE_WEBRTC_TURN_CREDENTIAL=${VITE_WEBRTC_TURN_CREDENTIAL}

RUN npm run build

# Serve the generated SPA through Nginx.
FROM nginx:1.27-alpine AS production
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --spider -q http://127.0.0.1/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
