# Copyright The OpenTelemetry Authors
# SPDX-License-Identifier: Apache-2.0

# -----------------------------------
# Stage 1: Install dependencies
# -----------------------------------
FROM --platform=linux/amd64 node:20-bullseye-slim AS deps

WORKDIR /app

# Prevent Cypress from downloading binary if not needed during build
ENV CYPRESS_INSTALL_BINARY=0

COPY ./src/frontend/package*.json ./
RUN npm ci

# -----------------------------------
# Stage 2: Build application
# -----------------------------------
FROM --platform=linux/amd64 node:20-bullseye-slim AS builder

WORKDIR /app

# Install protoc for grpc:generate
RUN apt-get update && apt-get install -y \
    protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*

COPY --from=deps /app/node_modules ./node_modules
COPY ./pb ./pb
COPY ./src/frontend .

RUN npm run grpc:generate
RUN npm run build

# -----------------------------------
# Stage 3: Runtime image
# -----------------------------------
FROM --platform=linux/amd64 node:20-bullseye-slim AS runner

WORKDIR /app

# Optional if your app needs protobuf at runtime
RUN apt-get update && apt-get install -y \
    protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*

ENV NODE_ENV=production

# Add non-root user
RUN addgroup --system nodejs && adduser --system --ingroup nodejs nextjs

COPY --from=builder /app/next.config.js ./
COPY --from=builder /app/utils/telemetry/Instrumentation.js ./
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=deps /app/node_modules ./node_modules

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

ENV PORT=8080
EXPOSE ${PORT}

ENTRYPOINT ["npm", "start"]
