FROM node:24-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:24-alpine AS dev
WORKDIR /app
COPY package*.json ./
RUN npm ci
USER node
EXPOSE 8080
CMD ["npm", "run", "dev"]


FROM gcr.io/distroless/nodejs24-debian13 AS prod
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY package*.json ./
COPY src/ ./src/
USER nonroot
EXPOSE 8080
CMD ["src/server.js"]