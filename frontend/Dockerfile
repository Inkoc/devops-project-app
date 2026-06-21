FROM node:24-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:24-alpine AS dev
WORKDIR /app
COPY package*.json ./
RUN npm ci
USER node
EXPOSE 3000
CMD ["npm", "run", "dev"]


FROM node:24-alpine AS prod
WORKDIR /app
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node src/ ./src/
USER node
EXPOSE 3000
CMD ["node", "src/server.js"]