FROM node:20-alpine AS base

WORKDIR /app/chatService

COPY package*.json ./
RUN npm ci --omit=dev

COPY . ./
ENV NODE_ENV=production
EXPOSE 3004

CMD ["npm", "run", "start"]

