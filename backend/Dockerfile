# syntax=docker/dockerfile:1

# Compile the TypeScript server and install production dependencies separately.
FROM node:22-alpine AS build
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY tsconfig.json ./
COPY src ./src
RUN npm run build

FROM node:22-alpine AS production
WORKDIR /app
ENV NODE_ENV=production \
    PORT=5000

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=build /app/dist ./dist
# server.ts reads swagger.json from src at runtime.
COPY --from=build /app/src/swagger.json ./src/swagger.json

# All required runtime configuration is supplied with --env-file or --env.
EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:' + (process.env.PORT || 5000) + '/api/health', r => process.exit(r.statusCode < 400 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["npm", "start"]
