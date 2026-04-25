FROM node:20-alpine

WORKDIR /app

# Create non-root user early so we can assign file ownership on copy
RUN addgroup -S nodeuser && adduser -S nodeuser -G nodeuser

# Copy manifests first — this layer is cached unless dependencies change
COPY --chown=nodeuser:nodeuser app/package*.json ./
RUN npm ci

# Copy the rest of the application source
COPY --chown=nodeuser:nodeuser app/ ./

ENV PORT=3000

EXPOSE $PORT

# Probe /health every 30 s; fail fast after 3 consecutive misses
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:$PORT/health || exit 1

USER nodeuser

CMD ["node", "index.js"]
