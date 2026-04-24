FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY server.js ./

FROM node:20-alpine

LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.authors="Gabriel et Mathieu"
LABEL org.opencontainers.image.source="github.com/Gaper8/TaskFlow"

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=builder /app/server.js ./

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --retries=3 CMD wget -qO- http://localhost:3001/health || exit 1

CMD ["npm", "start"]